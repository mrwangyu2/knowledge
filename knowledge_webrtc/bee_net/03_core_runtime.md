# BeeNet 模块3：核心运行时

> 精读日期：2026-06-04  
> 覆盖文件：`interface.h/cpp`, `manager.h/cpp`, `datacache.h/cpp`

---

## 文件依赖关系

```
interface.cpp ──► manager.h, opencmd.h, initcmd.h, logger.h
manager.cpp   ──► peerconnection.h, media_frame.h, executer.h, http_exec.h, ws_exec.h, ca_exec.h
                   initcmd.h, deflua.h, timer.h
datacache.cpp ──► manager.h, opencmd.h, media_frame.h, tokenizer.h
```

interface 和 datacache 依赖 manager；manager 依赖所有协议处理器和 WebRTC 组件。

---

## 1. interface.cpp — C API 桥接层

所有公共 C API 函数的实现。**设计模式统一**——每个函数遵循相同模板：

### 同步 API 模板（以 `bee_open` 为例）

```cpp
int bee_open(const char *url, const char *opaque_json) {
  CmdHandler *cmd = BeeManager::CreateCmd<BeeOpenCmd>();   // 1. 从对象池创建命令
  if (nullptr == cmd) return -ENOMEM;                       // 2. 检查分配失败
  BeeOpenCmd::Result result;                                // 3. 栈上创建 promise result
  BP<BeeOpenCmd>(cmd)->url_ = url;                          // 4. 填充命令参数
  BP<BeeOpenCmd>(cmd)->result_ = &result;
  BeeManager::PostCmd(cmd);                                 // 5. 投递到工作线程
  return result.ret.get_future().get();                     // 6. 阻塞等待结果
}
```

### 异步 API 模板（以 `bee_open_async` 为例）

```cpp
void bee_open_async(const char *url, const char *opaque_json,
                    void (*onopen)(int, void*), void *userp) {
  CmdHandler *cmd = BeeManager::CreateCmd<BeeAsyncOpenCmd>();
  if (nullptr == cmd) return onopen(-ENOMEM, userp);        // 失败直接回调
  BP<BeeAsyncOpenCmd>(cmd)->url_ = url;
  BP<BeeAsyncOpenCmd>(cmd)->on_open_ = onopen;
  BP<BeeAsyncOpenCmd>(cmd)->user_ptr_ = userp;
  BeeManager::PostCmd(cmd);                                 // 投递后立即返回
}
```

### 完整 API 对应关系

| C 函数 | 命令类 | Result 类型 | 特殊处理 |
|---|---|---|---|
| `bee_env_init` | `EnvInitCmd` | `bool` | 合并配置 JSON |
| `bee_set_*` (5个) | 直接调用 | — | 无异步，直接设值 |
| `bee_env_cleanup` | 直接调用 | — | `manager->Cleanup()` |
| `bee_open` | `BeeOpenCmd` | `int` (fd/err) | — |
| `bee_read` | `BeeReadCmd` | `int` (bytes) | `memcpy` 到用户 buffer |
| `bee_send` | `BeeSendCmd` | `int` (err) | — |
| `bee_stat` | `BeeStatCmd` | `int` (length) | `memcpy` 到用户 buffer |
| `bee_seek` | `BeeSeekCmd` | `off_t` (offset) | — |
| `bee_close` | `BeeCloseCmd` | `int` (err) | — |
| `bee_*_async` (6个) | 对应 `BeeAsync*Cmd` | 回调 | swap 提取回调 |

**C++20 指定初始化器**: `BeeReadCmd::Result` 等使用 C 风格的指定初始化 `.buffer = buffer, .buflen = buflen`。

---

## 2. manager.cpp — BeeManager 运行时核心

### `BeeManager` 构造函数

- **初始化顺序**: `async_worker_state_(INIT)` → OpenSSL 线程安全设置 → 内置 CA 证书解密 → fd 池初始化（1~1024）→ Executer 双向链表哨兵初始化
- **OpenSSL 初始化**: `OPENSSL_init_crypto(LOAD_CRYPTO_STRINGS | ADD_ALL_CIPHERS | ADD_ALL_DIGESTS | NO_LOAD_CONFIG | ASYNC | NO_ATEXIT)`
- **fd 池**: `fd2cache_free_fds_` (环形缓冲) 预填充 1~1024 的 fd 值
- **哨兵节点**: `executer_list_head_` 的 prev/next 均指向自身 (`dummy_executer_` 数组的首地址)，形成空循环链表

### `Create()` — 首次初始化

- **调用流程**:
  1. `curl_global_init(CURL_GLOBAL_ALL)` — 初始化 libcurl
  2. `curl_multi_handle_` 创建 — 设置 `CURLMOPT_MAX_TOTAL_CONNECTIONS` 为 64
  3. `curl_share_object_` 创建 — DNS 和 SSL 会话共享
  4. 加载 Lua：优先从 `lua_file_` 路径读取 → 解密（xxtea rc4）→ 解压（gz）→ 加载到 VM
  5. 若无外部文件则从 `deflua.c` 的 `def_lua[]` 数组解密加载（RC4 解密，密钥来自 `DEF_GLSB_KKK`）
  6. 创建 `SignalingServer`（继承 `WebRTCSocketServer`）并启动异步工作线程
  7. `async_worker_state_` 设为 `RUNNING`

### `Cleanup(terminate)` — 停机和清理

- **调用流程**:
  1. 若工作线程存在：设 `terminate_async_worker_` → `curl_multi_wakeup` 唤醒等待 → 等待 `async_worker_state_ == CLEAR`
  2. 断言所有 Executer 链表已清空
  3. 断言 `cleanup_queue_` 为空
  4. 释放 curl multi handle → `curl_global_cleanup()`
  5. `async_worker_state_` 设为 `DESTROY` 或 `INIT`

### `CreateLuaContext(script, length, keyid, key, init_args)` — Lua 脚本加载

- **调用流程**:
  1. `luaL_newstate()` → 创建新 VM
  2. `luaL_openlibs(L)` → 加载所有标准库
  3. 注册 C 扩展：`lua_cjson` → `lua_zlib` → `PeerConnection` → `DataChannel` → `VideoFrame` → `AudioFrame` → `HTTPExecuter` → `WSExecuter` → `DataCache`
  4. 注册日志桥接：`LuaBeeLoggerLog` 绑定到 `bee_log` 全局
  5. xxtea 解密脚本 → gz 解压 → `luaL_loadbuffer` 加载
  6. 创建 `RuntimeInitCmd` 投递到 urgent 队列
- **密码方案**: 外层 RC4（key=DEF_GLSB_KKK），内层 gzip 压缩 + xxtea 加密（keyid+key）

### `ChangeLuaContext(...)` — 运行时热替换

- **调用流程**:
  1. 检查 `CanUpdateLuaContext()` — 确保 Executer 列表为空且当前非 QUITTING
  2. `CreateLuaContext` 创建新的 Lua VM
  3. 将旧 VM 移入 `lua_release_q_`（延迟释放队列）
  4. 原子交换 `lua_` 指向新 VM
- **延迟释放**: 旧 VM 被放入 `lua_release_q_`，在工作线程主循环中检测 `use_count() == 1` 时释放（确保没有协程还在使用）

### `async_worker_main()` — 工作线程主循环

这是整个 SDK 的核心事件循环：

```
while (state == RUNNING) {
  1. lua_gc(L, LUA_GCCOLLECT)                        // 触发 Lua GC
  2. 消费 urge_cmd_queue_ 和 cmd_queue_ 中的命令     // 处理 pending 操作
  3. CheckTimer()                                     // 检查超时定时器
  4. 清理 lua_release_q_ 中的旧 VM                    // 延迟释放
  5. curl_multi_perform(handle, &running)              // 驱动 curl 传输
  6. curl_multi_poll(handle, NULL, 0, timeout, &nfds)  // 等待 socket 事件
  7. 处理 timeout=0 时的 already_read/written 标志     // 防止忙等
  8. 检查 CURLMsg (CURLMSG_DONE)                       // 处理完成的事件
  9. executer->OnCompleteEvent()                       // 通知 Executer
}
```

- **双模式等待**: `curl_multi_poll` 既做 IO 多路复用（类似 `select`/`epoll`），也做定时等待。`timeout=0` 时有特殊的忙等保护逻辑。
- **Wait 唤醒**: `SignalingServer::Wait` (WebRTC 需要) 通过 `curl_multi_wakeup` 实现中断。

### Executer 链表管理

#### `AddExecuter(pexecuter)`
- **用途**: 将 Executer 注册到活跃列表
- **核心逻辑**: 头插法插入 `executer_list_head_` 循环双向链表

#### `RemoveExecuter(pexecuter)`
- **用途**: 从链表中移除 Executer
- **核心逻辑**: 操作 prev/next 指针，如果链表只剩哨兵则断言 `prev==next==dummy`

### Cache 管理

#### `CreateCache()` → fd
- **用途**: 分配 fd 并创建 DataCache
- **调用流程**:
  1. 从 `fd2cache_free_fds_` 取空闲 fd
  2. `lua_newuserdata(L, sizeof(DataCache))` — 在 Lua 堆中分配 DataCache 内存
  3. placement new 构造 DataCache
  4. `fd2cache_vec_[fd & 0x3ff] = cache` — 注册到映射表
- **fd 编码**: fd 低 10 位做索引（`& 0x3ff`），高位用于区分同一槽位的不同生命周期

#### `RemoveCache(cache_fd)`
- **用途**: 释放 fd 和 DataCache
- **核心逻辑**:
  1. 从映射表移除
  2. `cache->Destroy()` — 显式调用析构
  3. 生成新 fd（原 fd + 1024）放回空闲池
- **fd 重用保护**: 每次释放生成新 fd（加 1024 且 `& 0x7fffffff`），防止 ABA 问题

#### `GetCache(cache_fd)` → DataCache* / nullptr
- **用途**: 通过 fd 查找 DataCache
- **核心逻辑**: 查表 + 验证 `cache_fd != cache->fd()` 防止错位

### `PostCmd(cmd)` — 命令投递

- **调用流程**: 若工作线程正在重入（`async_worker_reentrant_ > 0`）→ 直接 `Process()`；否则入 `cmd_queue_` + `curl_multi_wakeup` 唤醒工作线程
- **设计意图**: 重入保护——当工作线程消费的命令内部又 `PostCmd` 时，跳过队列直接执行，避免死锁

### `LuaBeeLoggerLog(lua_State *L)` — Lua→C 日志桥接

- **用途**: Lua 脚本调用 `bee_log(level, source, line, msg)` 时桥接到 C 日志系统
- **参数**: (level:int, source:string, line:int, msg:string)
- **核心逻辑**: 参数验证后直接调用 `bee::logger::log()`

---

## 3. datacache.cpp — DataCache 数据缓存

### 生命周期与状态机

```
INIT → READY → FINISHED → SUCCESS/FAILED/LUAERR/CLOSEWAIT → DESTROY
                              ↑                │
                              └── seek ────────┘
```

| 状态 | 含义 | 允许操作 |
|---|---|---|
| `INIT` (0x01) | 刚创建 | 仅 `Startup()` |
| `READY` (0x02) | 已调用 ready() | read/send/stat/seek |
| `FINISHED` (0x04) | finish() 已调用 | read/stat/seek |
| `SUCCESS` (0x08) | bee_open 返回 true | 同上 |
| `FAILED` (0x10) | bee_open 返回 false | 通知上层失败 |
| `CLOSEWAIT` (0x20) | bee_close 已调用 | 等待清理 |
| `LUAERR` (0x40) | Lua 运行错误 | 同上 |
| `DESTROY` (0x80) | 析构中 | 无 |

### 构造函数 `DataCache(L, fd)`
- **缓冲区**: `recv_buffer_(2*1024*1024)` — **2MB 初始容量**，用于 HTTP 大文件接收
- **Lua 引用**: `luaL_ref(LUA_REGISTRYINDEX)` 注册自身，防止被 GC 回收
- 所有 Lua 回调引用初始化为 `LUA_NOREF`

### `Startup(lua_function, url, opaque)` — 启动连接
- **调用流程**:
  1. `lua_newthread(lua_main_)` → 创建主协程
  2. 压栈参数：`[C闭包, self(userdata), datacache_metatable, url, opaque]`
  3. `lua_resume(co, nullptr, 4)` → 启动协程执行 Lua `bee_open`
  4. 若立即完成 → `CacheFinished(err)`；若 yield → 等待后续 resume
- **元表设置**: DataCache 在 Lua 侧的 userdata 绑定了 `datacache` 元表，使其可调用成员方法

### `CacheFinished(err)` — 完成通知
- **调用流程**:
  1. 置状态为 `SUCCESS`/`FAILED`/`LUAERR` 或 `CLOSEWAIT`
  2. 调用 `on_open_or_close` 回调（同步命令的 promise 或异步命令的回调）
  3. `ref_co_main_` 的 Lua 协程被 unref 清理
  4. 若 `CLOSEWAIT` → `BeeManager::RemoveCache(fd)` 最终销毁
- **安全保护**: 若 `stage_ == DESTROY` → 直接返回（防止重复通知）

### `GetDataAsync(onrecv, expected_bytes, userptr)` — 异步读取
- **调用流程**:
  1. 若 buffer 中数据 ≥ expected_bytes → 立即回调
  2. 若数据不足 + `FINISHED/SUCCESS/FAILED` → 回调所有剩余数据，`buffer=nullptr` 表示数据结束
  3. 否则 → 保存回调到 `onread_*` 字段，等待 `LuaAppendData` 追加数据后触发
- **特殊返回**:
  - `data != nullptr, len > 0` → 读取的数据
  - `data == nullptr, len == 0` → 数据结束（流关闭）
  - `data == nullptr, len != 0` → 错误码

### `Stat(onstat, userp)` / `Seek(onseek, offset, whence, userp)` / `Send(onsent, msg, len, userp)`
- **共同模式**: 检查状态 → 调用 Executer 对应方法 → 通过回调返回结果

---

### Lua 绑定方法（通过 `LuaOpenDataCacheLib` 注册）

#### `ready()` — 标记就绪
- **用途**: Lua 脚本调用，表示 `bee_open` 已准备好接收数据
- **调用流程**: `INIT → READY`，返回 true；若已在 `READY/SUCCESS` 状态则直接返回

#### `fail()` — 标记失败
- **用途**: Lua 脚本调用，表示 `bee_open` 失败
- **调用流程**: `INIT → FAILED`，调用 `CacheFinished(MKERR('E','F','A','L'))`

#### `finish()` — 标记结束
- **用途**: Lua 脚本调用，表示数据接收完毕
- **调用流程**: `READY → FINISHED`

#### `append(data)` — 追加接收数据
- **用途**: Lua 脚本调用，追加收到的 body 数据到 `recv_buffer_`
- **调用流程**:
  1. `recv_buffer_.concat(data, len)`
  2. 若有等待的 read 回调 → 立即调用
  3. 若 `onmessage` 回调已设置 → 调用（WebSocket/WebRTC 场景）
  4. 若有等待的协程 → resume

#### `position(offset)` — 设置偏移
- **用途**: 设置当前流偏移量（用于 HTTP Range 请求追踪）
- **调用流程**: `base_offset_ = offset`

#### `seek(offset, whence)` — 流定位
- **用途**: Lua 脚本调用，实现 HTTP seek
- **调用流程**: 通知 Executer 发起新的 Range 请求，当前协程 yield 等待数据到达

#### `wait()` — 协程等待数据
- **用途**: Lua 脚本调用，当前协程 yield 直到有新数据
- **调用流程**: 将当前协程注册到等待队列 → `lua_yield`

#### `wakeup()` — 唤醒等待协程
- **用途**: 数据到达后 Lua 侧调用，唤醒之前 wait 的协程
- **调用流程**: 从 `wakeup_q_` 弹出一个协程 → resume

#### `flush(action)` — 消费数据
- **用途**: 清空已读数据（`release`），支持 "all" 和 "ondemand" 两种模式
- **调用流程**: "all" → 释放全部已读数据 → `bee_read` 若可继续则 resume；"ondemand" → 仅标记

#### `bufferedAmount()` — 查询缓冲数据量
- **用途**: 返回 `recv_buffer_.size()`

#### `set_callback(event, func)` — 设置回调
- **用途**: 设置 `onstat`/`onseek`/`onmessage`/`onclose` 回调
- **调用流程**: `luaL_ref` 注册到 LUA_REGISTRYINDEX，防止重复设置

#### `destroy()` — 销毁（__gc）
- **用途**: Lua GC 回调，标记为 DESTROY

---

## 模块3 设计要点总结

| 要点 | 说明 |
|---|---|
| **统一命令模板** | 所有 C API 函数遵循「创建命令 → 填充参数 → PostCmd → 阻塞等待」模式 |
| **Promise 桥接** | 同步 API 用 `std::promise/future` 将异步命令结果阻塞返回给调用者 |
| **2MB 缓冲区** | DataCache 默认缓冲区 2MB，适配大文件 HTTP 下载 |
| **协程协作模式** | Lua `yield`/`resume` 实现阻塞式语法的异步 I/O |
| **fd 重用安全** | 释放后生成新 fd（加 1024 偏移），防止 ABA 问题 |
| **Lua 内存管理** | DataCache 通过 `lua_newuserdata` 分配在 Lua 堆中，生命周期由 Lua GC 和 C 双重管理 |
| **重入保护** | `PostCmd` 检测工作线程重入，直接执行命令而非入队 |
| **热替换 Lua** | `ChangeLuaContext` 原子交换 VM，旧 VM 延迟释放等待协程结束 |
| **脚本加密链** | RC4 → gzip 解压 → xxtea 解密 → Lua 字节码，三层保护 |
| **双命令队列** | `urge_cmd_queue_` 优先处理（如 init_args 修改），`cmd_queue_` 处理常规操作 |
