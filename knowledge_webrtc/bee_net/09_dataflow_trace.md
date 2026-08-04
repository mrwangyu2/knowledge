# BeeNet 数据流追踪：HTTPS 下载完整生命周期

> 追踪日期：2026-06-04  
> 追踪链路：`bee_open("https://example.com/data")` → `bee_read` → `bee_close`

---

## 线程模型

```
┌──────────────────┐     ┌──────────────────────────────────┐
│   Caller Thread   │     │      Async Worker Thread          │
│  (用户调用线程)     │     │  (BeeManager 单线程事件循环)        │
├──────────────────┤     ├──────────────────────────────────┤
│ bee_open()       │     │  BeeOpenCmd::Process()            │
│   ↓              │     │    ↓                              │
│ PostCmd ─────────┼────→│  CmdQueue::GetCmd()               │
│   ↓              │     │    ↓                              │
│ future.get()     │     │  CreateCache → StartLua           │
│   ║ BLOCK        │     │    ↓                              │
│   ║              │     │  Lua bee_open → http.get          │
│   ║              │     │    ↓                              │
│   ║              │     │  lua_yield ═══ SUSPEND ═══       │
│   ║              │     │    ↓                              │
│   ║              │     │  curl_multi_perform               │
│   ║              │     │    ├─ GetHeader (callback)        │
│   ║              │     │    ├─ GetBody (callback) × N      │
│   ║              │     │    └─ OnCompleteEvent              │
│   ║              │     │    ↓                              │
│   ║              │     │  lua_resume → CacheFinished       │
│   ║              │     │    ↓                              │
│   ║              │     │  promise.set_value(fd) ──────────→│
│   ╚══════════════╪════╝                                   │
│   ↓              │                                         │
│  return fd       │                                         │
└──────────────────┘                                         │
                                                             │
┌──────────────────┐     ┌──────────────────────────────────┐
│ bee_read(fd,...) │     │  BeeReadCmd::Process()            │
│ PostCmd ─────────┼────→│    ↓                              │
│ future.get()     │     │  GetCache(fd)                     │
│   ║ BLOCK        │     │    ↓                              │
│   ║              │     │  GetDataAsync(onrecv, len, res)   │
│   ║              │     │    ├─ 数据足够 → 立即回调           │
│   ║              │     │    └─ 数据不足 → 暂存回调等待       │
│   ║              │     │  memcpy → promise.set_value(n) ──→│
│   ╚══════════════╪════╝                                   │
│   ↓              │                                         │
│  return n        │                                         │
└──────────────────┘                                         │
```

---

## 阶段 1：bee_open — 连接建立

### Step 1.1：用户线程 → 创建命令

**文件**：`interface.cpp:bee_open()`

```cpp
int bee_open(const char *url, const char *opaque_json) {
    // ① 从对象池分配 BeeOpenCmd（底层是 CmdMaker<BeeOpenCmd>::Create()）
    CmdHandler *cmd = BeeManager::CreateCmd<BeeOpenCmd>();

    // ② 栈上分配 Result（含 std::promise<int>）
    BeeOpenCmd::Result result;

    // ③ 填充命令参数
    BP<BeeOpenCmd>(cmd)->url_ = url;            // "https://example.com/data"
    BP<BeeOpenCmd>(cmd)->opaque_ = opaque_json;  // "{}"
    BP<BeeOpenCmd>(cmd)->result_ = &result;

    // ④ 投递到命令队列（Prod-Cons，跨线程）
    BeeManager::PostCmd(cmd);

    // ⑤ 阻塞等待结果（std::future::get()）
    return result.ret.get_future().get();
}
```

**关键数据流**：
- `url` / `opaque_json` ：被拷贝到命令的 `std::string` 成员 → 跨线程安全
- `result` ：栈上对象，promise/future 配对提供跨线程同步

### Step 1.2：PostCmd — 命令投递

**文件**：`manager.cpp:BeeManager::PostCmd()`

```cpp
void BeeManager::PostCmd(CmdHandler* cmd) {
    // 检查调用者是否就是工作线程（重入检测）
    if (myself_.async_worker_reentrant_ > 0) {
        // 工作线程内部 PostCmd → 直接执行（避免死锁）
        BP<BaseCmd>(cmd)->Process();
    } else {
        // 外部线程 → 入队 + 唤醒工作线程
        myself_.cmd_queue_.PutCmd(cmd);
        CURLMcode err = curl_multi_wakeup(myself_.curl_multi_handle_.get());
    }
}
```

**重入保护的设计意图**：工作线程消费命令 A 时，若命令 A 内部又 `PostCmd(B)`，B 直接执行而非入队（因为工作线程同时在 `GetCmd` 循环中，入队会导致 B 等下一轮循环才被处理，且可能 deadlock）。

### Step 1.3：BeeOpenCmd::Process() — 在异步工作线程中

**文件**：`opencmd.cpp:BeeOpenCmd::Process()`

```cpp
void BeeOpenCmd::Process() {
    // ① 分配 fd 和 DataCache
    int fd = BeeManager::CreateCache();

    // ② 获取刚创建的 DataCache
    auto *cache = BeeManager::GetCache(fd);

    // ③ 绑定完成回调 → 触发 promise.set_value()
    static auto on_open = +[](int err, Result *result) {
        result->ret.set_value(err);  // 将 fd 或错误码传回调用线程
    };
    cache->on_open_or_close = std::bind(on_open, std::placeholders::_1, result_);

    // ④ 启动 Lua 处理
    cache->Startup(wrap_lua_bee_open, url_, opaque_);
}
```

### Step 1.4：CreateCache — 分配 fd + DataCache

**文件**：`manager.cpp:BeeManager::CreateCache()`

```cpp
int BeeManager::CreateCache() {
    // ① 从空闲池取 fd（环形缓冲区 pop_front）
    int cache_fd = myself_.fd2cache_free_fds_.front();

    // ② 在 Lua 堆中分配 DataCache（lua_newuserdata 保证 Lua GC 管理）
    auto cache_in_lua = (DataCache *)lua_newuserdata(L.get(), sizeof(DataCache));
    new(cache_in_lua) DataCache(L, cache_fd);  // placement new

    // ③ 注册到 fd 映射表
    myself_.fd2cache_vec_[cache_fd & 0x3ff] = cache_in_lua;
    myself_.fd2cache_free_fds_.pop_front();

    return cache_fd;
}
```

**DataCache 构造函数**（`datacache.cpp`）：
- `recv_buffer_(2*1024*1024)` — 2MB 接收缓冲区
- `luaL_ref(L, REGISTRYINDEX)` — 防止被 Lua GC 回收
- 所有回调引用 = `LUA_NOREF`

### Step 1.5：Startup — 启动 Lua 协程

**文件**：`datacache.cpp:DataCache::Startup()`

```cpp
void DataCache::Startup(int (*lua_function)(lua_State*), const string &url, const string &opaque) {
    // ① 在主 Lua VM 中创建协程
    lua_State *co = lua_newthread(lua_main_.get());
    ref_co_main_ = luaL_ref(lua_main_.get(), LUA_REGISTRYINDEX);

    // ② 压栈：C 闭包 + self(userdata) + metatable + url + opaque
    lua_pushcfunction(co, lua_function);         // wrap_lua_bee_open
    lua_pushnil(co);
    lua_rawgeti(co, REGISTRYINDEX, ref_cache_);  // self (DataCache userdata)
    luaL_getmetatable(co, "datacache");          // 元表
    lua_setmetatable(co, -2);
    lua_pushlstring(co, url.data(), url.length());
    lua_pushlstring(co, opaque.data(), opaque.length());

    // ③ 启动协程（4 个参数）
    // Lua 5.3: lua_resume(co, nullptr, 4) → nresult 为返回值
    // Lua 5.4: lua_resume(co, nullptr, 4, &nresult)
    int err = lua_resume(co, nullptr, 4);
}
```

**此时 Lua 协程栈的状态**：
```
栈底 → [1] C closure (wrap_lua_bee_open)
       [2] self (DataCache userdata + metatable)
       [3] "https://example.com/data"  (url)
       [4] "{}"                         (opaque)
栈顶
```

### Step 1.6：wrap_lua_bee_open — Lua 入口包装

**文件**：`opencmd.cpp:wrap_lua_bee_open()`

```cpp
static int wrap_lua_bee_open(lua_State *L) {
    // ① 获取 Lua 全局函数 bee_open
    lua_getglobal(L, "bee_open");

    // ② 将 self 移到位置 1（替换 bee_open）
    lua_replace(L, 1);

    // ③ pcallk 调用 bee_open(self, url, opaque)
    // continuation K 在函数返回（或 yield 后 resume）时执行
    return K(L, lua_pcallk(L, 3, 1, 0, (intptr_t)cache, K), (intptr_t)cache);
}
```

**continuation K**（Lua 函数完成后执行）：
```cpp
static auto K = [](lua_State *L, int status, lua_KContext ctx) {
    auto *cache = (DataCache *)ctx;
    if (status != LUA_OK && status != LUA_YIELD)
        cache->CacheFinished(MKERR('E','L','U','A'));  // Lua 运行错误
    else if (lua_toboolean(L, -1))
        cache->CacheFinished(MKERR('E','F','I','N'));  // 返回 true → SUCCESS
    else
        cache->CacheFinished(MKERR('E','F','A','L'));  // 返回 false → FAILED
};
```

**CacheFinished 触发**：`stage_` 设为 SUCCESS/FAILED → 调用 `on_open_or_close` → `result_->ret.set_value(fd)` → **bee_open 用户线程返回 fd**。

### Step 1.7：Lua bee_open → http.get

Lua 脚本中的 `bee_open(self, url, opaque)` 判定 URL scheme 为 `https`，调用 `http.get(url, options)`。

**文件**：`http_exec.cpp:LuaHttpGet()`

```cpp
int HTTPExecuter::LuaHttpGet(lua_State *L) {
    // ① 在 Lua 堆分配 HTTPExecuter
    auto http = (HTTPExecuter *)lua_newuserdata(L, sizeof(HTTPExecuter));
    new (http) HTTPExecuter;

    // 设置元表 "http"
    luaL_getmetatable(L, "http");
    lua_setmetatable(L, -2);

    // ② 创建回调协程
    lua_State *co = lua_newthread(L);
    http->ref_cb_co_ = luaL_ref(L, REGISTRYINDEX);

    // ③ 初始化 curl easy handle
    http->Initialize(co);
    //    └─ curl_easy_init()
    //    └─ CURLOPT_PRIVATE = http (用于回调中找回 HTTPExecuter)
    //    └─ CURLOPT_SSL_CTX_FUNCTION (内置 CA) 或 CURLSSLOPT_NATIVE_CA
    //    └─ TCP keepalive 参数

    // ④ 设置 curl 选项
    EasySetOption(handle, CURLOPT_ACCEPT_ENCODING, "");        // 接受压缩
    EasySetOption(handle, CURLOPT_HEADERFUNCTION, GetHeader);   // 响应头回调
    EasySetOption(handle, CURLOPT_HEADERDATA, http);            // 传递 this
    EasySetOption(handle, CURLOPT_WRITEFUNCTION, GetBody);      // Body 回调
    EasySetOption(handle, CURLOPT_WRITEDATA, http);             // 传递 this
    EasySetOption(handle, CURLOPT_LOW_SPEED_LIMIT, 1L);         // 最低 1B/s
    EasySetOption(handle, CURLOPT_LOW_SPEED_TIME, 5L);          // 超时 5s

    // ⑤ 发起请求 → 注册到 BeeManager
    http->SendRequest(url);
    //    └─ EasySetOption(CURLOPT_URL, url)
    //    └─ BeeManager::AddExecuter(this) → 插入 executer_list_head_ 链表

    // ⑥ 挂起协程 → 等待 HTTP 响应完成
    return lua_yieldk(L, 0, ref_lua_co, K);
}
```

**此时的数据状态**：
- HTTPExecuter 注册在 BeeManager 的 `executer_list_head_` 双向链表中
- curl easy handle 已关联 URL，由 curl multi handle 驱动
- Lua 协程已 yield，等待 OnCompleteEvent 时 resume

---

## 阶段 2：curl 事件循环 — 数据接收

### Step 2.1：async_worker_main 主循环

**文件**：`manager.cpp:async_worker_main()`

```cpp
// 简化后的主循环
while (async_worker_state_ == RUNNING) {
    // ① 消费命令队列
    while (auto *cmd = urge_cmd_queue_.GetCmd()) { BP<BaseCmd>(cmd)->Process(); }
    while (auto *cmd = cmd_queue_.GetCmd())       { BP<BaseCmd>(cmd)->Process(); }

    // ② 检查超时定时器
    TimeScheduler::CheckTimer();

    // ③ 驱动 curl 传输
    int running_handles = 0;
    curl_multi_perform(curl_multi_handle_.get(), &running_handles);

    // ④ 等待 socket 事件（内部调用 select/poll/epoll）
    curl_multi_poll(curl_multi_handle_.get(), NULL, 0, timeout_ms, &nfds);

    // ⑤ 处理完成的传输
    while (auto *msg = curl_multi_info_read(curl_multi_handle_.get(), &msgs_left)) {
        if (msg->msg == CURLMSG_DONE) {
            // 从 PRIVATE 指针找回 Executer
            Executer *executer = nullptr;
            curl_easy_getinfo(msg->easy_handle, CURLINFO_PRIVATE, &executer);
            executer->OnCompleteEvent();  // 通知协议处理器
        }
    }
}
```

### Step 2.2：curl 回调链路

**GetHeader 回调**（收到 HTTP 响应头时触发）：

```cpp
static size_t GetHeader(char *buffer, size_t size, size_t nitems, void *userp) {
    auto *http = (HTTPExecuter *)userp;  // 从 CURLOPT_HEADERDATA 恢复

    // 解析 HTTP 状态行："HTTP/1.1 200 OK"
    Tokenizer tokenizer(buffer, nitems);
    if (http->stage_ == INIT) {
        // 提取状态码 → 记录到 stage_ = HEAD
        // 缓存回调协程引用
    }

    // 解析 Content-Type / Content-Length
    if (tokenizer == "Content-Type:")
        http->response_content_type_ = ...;
    else if (tokenizer == "Content-Length:")
        http->response_content_length_ = ...;

    // 调用 Lua onhead 回调（如果设置了）
    if (http->custom_get_head_ != LUA_NOREF)
        lua_rawgeti(L, REGISTRYINDEX, http->custom_get_head_);
}
```

**GetBody 回调**（收到 HTTP body 数据时触发）：

```cpp
static size_t GetBody(char *buffer, size_t size, size_t nitems, void *userp) {
    auto *http = (HTTPExecuter *)userp;
    size_t real_size = size * nitems;

    if (http->stage_ == HEAD) {
        http->stage_ = BODY;  // 首次 body → 触发一次 onhead
    }

    // 追加到 HTTPExecuter 内部的 response_ 缓冲区
    http->response_.concat(buffer, real_size);

    // 调用 Lua onbody 回调（如果设置了）
    // 注意：onbody 回调中通常会调用 DataCache:append(data)
    if (http->custom_get_body_ != LUA_NOREF)
        lua_rawgeti(L, REGISTRYINDEX, http->custom_get_body_);

    return real_size;
}
```

### Step 2.3：OnCompleteEvent — 传输完成

**文件**：`http_exec.cpp:OnCompleteEvent()`

```cpp
bool HTTPExecuter::OnCompleteEvent() {
    // 从 curl multi handle 移除
    BeeManager::RemoveExecuter(this);

    // 获取响应信息
    EasyGetInfo(handler(), CURLINFO_RESPONSE_CODE, &http_code);
    EasyGetInfo(handler(), CURLINFO_HTTP_VERSION, &http_version);
    EasyGetInfo(handler(), CURLINFO_SIZE_DOWNLOAD_T, &dl_size);
    // ... 其他指标

    // 写入 Lua 对象元表
    lua_rawgeti(lua_co_, REGISTRYINDEX, ref_obj_);  // self
    lua_getmetatable(lua_co_, -1);
    lua_pushinteger(lua_co_, http_code);
    lua_setfield(lua_co_, -2, "http_code");  // self.http_code = 200
    // ... 其他字段

    // Resume Lua 协程
    // 传入 nil (无错误) + self 对象
    lua_pushnil(lua_co_);
    lua_rawgeti(lua_co_, REGISTRYINDEX, ref_obj_);
    // 协程从 lua_yieldk 处恢复，lua_pcallk 的 continuation K 被调用
}
```

### Step 2.4：continuation K → CacheFinished → promise.set_value

```
lua_resume → continuation K
  └─ luaL_unref(L, REGISTRYINDEX, ref_lua_co)  // 释放协程引用
  └─ 返回 2 个值：(nil, self)

→ wrap_lua_bee_open 的 continuation K
  └─ cache->CacheFinished(MKERR('E','F','I','N'))
      └─ stage_ = SUCCESS
      └─ on_open_or_close(MKERR('E','F','I','N'))
          └─ on_open(err, result_)  // 来自 std::bind
              └─ result_->ret.set_value(fd)  // ← promise 被 fulfill！
```

**bee_open 用户线程返回 fd**。

---

## 阶段 3：bee_read — 数据读取

### Step 3.1：命令投递

用户线程调用 `bee_read(fd, buffer, len)`：

```cpp
// interface.cpp
int bee_read(int fd, void *buffer, size_t buflen) {
    CmdHandler *cmd = BeeManager::CreateCmd<BeeReadCmd>();
    BeeReadCmd::Result result = { .buffer = buffer, .buflen = buflen };
    BP<BeeReadCmd>(cmd)->fd_ = fd;
    BP<BeeReadCmd>(cmd)->result_ = &result;
    BeeManager::PostCmd(cmd);
    return result.ret.get_future().get();  // BLOCK
}
```

### Step 3.2：BeeReadCmd::Process() — 数据拷贝

```cpp
void BeeReadCmd::Process() {
    DataCache *cache = BeeManager::GetCache(fd_);

    static auto on_recvmsg = [](const void *buffer, size_t length, void *userp) {
        auto *result = (BeeReadCmd::Result *)userp;
        if (nullptr == buffer) {
            // buffer 为空 → 数据结束或错误
            result->ret.set_value(static_cast<int>(length));  // 0 = EOF, <0 = 错误
        } else {
            // 拷贝数据到用户 buffer
            size_t copy_len = std::min(result->buflen, length);
            memcpy(result->buffer, buffer, copy_len);
            result->ret.set_value(static_cast<int>(copy_len));
        }
    };

    // 调用 GetDataAsync → 可能立即回调，也可能暂存等待更多数据
    cache->GetDataAsync(on_recvmsg, result_->buflen, result_);
}
```

### Step 3.3：GetDataAsync — 数据调度

```cpp
void DataCache::GetDataAsync(onrecv, expected_bytes, userptr) {
    // 情况 A：缓冲区已有足够数据 → 立即回调
    if (recv_buffer_.size() >= expected_bytes) {
        onrecv(recv_buffer_.ptr(), expected_bytes, userptr);
        return;
    }

    // 情况 B：数据不足但流已结束 → 返回所有剩余数据
    if (stage_ & (FINISHED | SUCCESS | FAILED)) {
        if (recv_buffer_.size() > 0)
            onrecv(recv_buffer_.ptr(), recv_buffer_.size(), userptr);
        else
            onrecv(nullptr, 0, userptr);  // EOF
        return;
    }

    // 情况 C：数据不足，流未结束 → 暂存回调，等待更多数据
    onread_buffer_ = userptr;
    onread_callback_ = onrecv;
    onread_expected_ = expected_bytes;
}
```

### Step 3.4：Lua 追加数据 → 触发等待中的 read

当 HTTP body 数据到达时，Lua `DataCache:append(data)` 被调用：

```cpp
int DataCache::LuaAppendData(lua_State *L) {
    // ① 追加到 recv_buffer_
    cache->recv_buffer_.concat(data, len);

    // ② 检查是否有等待的 read 回调
    if (cache->onread_callback_ && cache->recv_buffer_.size() >= cache->onread_expected_) {
        cache->onread_callback_(cache->recv_buffer_.ptr(),
                                cache->onread_expected_,
                                cache->onread_buffer_);
    }

    // ③ 若有 onmessage 回调 → 调用（WebSocket/WebRTC 场景）
    if (LUA_NOREF != cache->ref_on_message_)
        lua_rawgeti(L, REGISTRYINDEX, cache->ref_on_message_);

    // ④ 若有等待的 Lua 协程 → resume
    if (YES == cache->waiting_on_receive_) {
        lua_rawgeti(L, REGISTRYINDEX, cache->ref_co_read_);
        lua_resume(...);
    }
}
```

**数据流终点**：用户 buffer 被 memcpy 填充 → promise.set_value → bee_read 返回字节数。

---

## 阶段 4：bee_close — 连接关闭

### Step 4.1：命令投递 → CacheFinished

```cpp
// BeeCloseCmd::Process()
void BeeCloseCmd::Process() {
    DataCache *cache = BeeManager::GetCache(fd_);
    cache->on_open_or_close = std::bind(on_close, _1, result_);
    cache->CacheFinished(MKERR('C','L','O','S'));
}
```

### Step 4.2：CacheFinished → 析构链

```
DataCache::CacheFinished(MKERR('C','L','O','S'))
  │
  ├─ stage_ = CLOSEWAIT | 之前的 SUCCESS/FAILED
  ├─ on_open_or_close(MKERR('C','L','O','S'))
  │    └─ promise.set_value(0)  → bee_close 返回
  │
  ├─ ref_co_main_ unref（释放 Lua 主协程引用）
  │    └─ Lua GC 可回收协程和关联的 HTTPExecuter
  │
  └─ stage_ = CLOSEWAIT
       └─ 下次工作线程检测到 CLOSEWAIT:
            BeeManager::RemoveCache(fd)
              ├─ fd2cache_vec_[fd & 0x3ff] = nullptr
              ├─ cache->Destroy() → ~DataCache()
              │    ├─ 清理 wakeup_q_ 中的所有协程引用
              │    ├─ 清理所有 Lua 回调引用 (ref_on_*)
              │    ├─ 清理所有协程引用 (ref_co_*)
              │    └─ luaL_unref(REGISTRYINDEX, ref_cache_)
              │
              └─ 生成新 fd (原 fd + 1024) → 放回空闲池
```

### Step 4.3：HTTPExecuter 析构

DataCache 析构后，Lua GC 回收 HTTPExecuter userdata：

```
~HTTPExecuter()
  ├─ stage_ = DESTROY
  ├─ 清理 Lua 回调引用 (custom_get_head_, custom_get_body_, custom_progress_)
  ├─ 清理协程引用 (ref_cb_co_, ref_obj_)
  ├─ curl_slist_free_all(custom_headers_)
  ├─ curl_mime_free(post_form_)
  └─ 基类 ~Executer() 确保已从链表移除
```

---

## 完整时间线总结

```
T0: bee_open("https://example.com/data")       [Caller Thread]
T1:   ├─ BeeOpenCmd created (object pool)
T2:   ├─ PostCmd → cmd_queue_.PutCmd()         [Caller → Queue]
T3:   ├─ curl_multi_wakeup()                    [Wake Worker]
T4:   └─ future.get() → BLOCK                  [Caller BLOCKS]

T5: async_worker_main loop iteration            [Worker Thread]
T6:   ├─ cmd_queue_.GetCmd() → BeeOpenCmd
T7:   ├─ CreateCache() → fd=1, DataCache(2MB)
T8:   ├─ Startup(wrap_lua_bee_open)
T9:   │   ├─ lua_newthread → coroutine
T10:  │   ├─ lua_resume → Lua bee_open(self, url, opaque)
T11:  │   │   └─ http.get(url, ...)
T12:  │   │       ├─ HTTPExecuter created (curl handle)
T13:  │   │       ├─ SendRequest → AddExecuter
T14:  │   │       └─ lua_yieldk → SUSPEND       [Lua SUSPENDS]

T15:  ├─ curl_multi_perform()                    [Curl drives HTTP]
T16:  │   ├─ DNS resolve (c-ares async)
T17:  │   ├─ TCP connect + TLS handshake
T18:  │   └─ HTTP request sent

T19:  ├─ curl_multi_poll() → wait for socket    [Worker waits]

T20:  ├─ GetHeader callback                      [Response header arrives]
T21:  │   └─ Parse: HTTP/1.1 200 OK, Content-Length: 1048576

T22:  ├─ GetBody callback × N                    [Body data streaming]
T23:  │   ├─ response_.concat(data)
T24:  │   └─ Lua onbody → DataCache:append(data)
T25:  │       └─ recv_buffer_.concat(data)

T26:  ├─ curl_multi_perform → running=0          [Transfer complete]
T27:  ├─ curl_multi_info_read → CURLMSG_DONE
T28:  └─ OnCompleteEvent()
T29:      ├─ GetInfo: http_code=200, size=1MB...
T30:      ├─ lua_resume → yield 处恢复            [Lua RESUMES]
T31:      │   └─ continuation K → unref coroutine
T32:      └─ → wrap_lua_bee_open continuation K
T33:          └─ CacheFinished(MKERR('E','F','I','N'))
T34:              └─ on_open_or_close(1) → promise.set_value(1)
T35:                  └─ bee_open returns 1       [Caller UNBLOCKS]

T36: bee_read(1, buf, 4096)                      [Caller Thread]
T37:   ├─ BeeReadCmd → PostCmd                   [Caller → Queue]
T38:   └─ future.get() → BLOCK

T39: BeeReadCmd::Process()                        [Worker Thread]
T40:   ├─ GetCache(1) → DataCache
T41:   ├─ GetDataAsync(onrecv, 4096, result)
T42:   │   └─ recv_buffer_.size() >= 4096 → 立即回调
T43:   ├─ memcpy(result->buffer, recv_buffer_.ptr(), 4096)
T44:   └─ promise.set_value(4096)
T45:       └─ bee_read returns 4096              [Caller UNBLOCKS]

T46: bee_close(1)                                 [Caller Thread]
T47:   ├─ BeeCloseCmd → PostCmd
T48:   └─ future.get() → BLOCK

T49: BeeCloseCmd::Process()                       [Worker Thread]
T50:   ├─ CacheFinished(MKERR('C','L','O','S'))
T51:   ├─ promise.set_value(0) → bee_close returns 0
T52:   ├─ RemoveCache(1) → fd recycled
T53:   └─ ~DataCache() → Lua unref all
```

---

## 设计的核心特征

| 特征 | 实现方式 |
|---|---|
| **跨线程同步** | `std::promise/future` 桥接调用者线程和工作线程 |
| **零拷贝?** | 否。body 数据有两次拷贝：curl→response_ (64KB buf) → recv_buffer_ (2MB) → 用户 buffer |
| **协程挂起/恢复** | Lua `yield/resume` 实现阻塞式语法的异步 I/O |
| **curl 集成** | CURLOPT_PRIVATE 传 this 指针，静态回调中恢复对象 |
| **事件驱动** | curl_multi_perform + curl_multi_poll 替代手动 select/epoll |
| **内存布局** | DataCache 分配在 Lua 堆中，生命周期受 Lua GC 和 C 双重管理 |
| **fd 回收安全** | 释放后生成新 fd（+1024），防止 ABA 问题 |
