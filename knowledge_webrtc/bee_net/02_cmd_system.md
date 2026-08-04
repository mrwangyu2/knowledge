# BeeNet 模块2：命令系统

> 精读日期：2026-06-04  
> 覆盖文件：`basecmd.h`, `opencmd.h/cpp`, `initcmd.h/cpp`

---

## 文件依赖关系

```
basecmd.h (纯头文件，依赖 <atomic>/<cassert>)
    │
    ├──► opencmd.h ──► opencmd.cpp ──► manager.h
    │
    └──► initcmd.h ──► initcmd.cpp ──► manager.h
```

命令系统自身不依赖协议处理器或 WebRTC，仅依赖 `BeeManager` 的静态工厂方法。

---

## 1. BaseCmd / CmdHandler / CmdQueue — 命令框架

文件：`basecmd.h`（纯头文件，无 .cpp）

### `BaseCmd`
- **签名**: `class BaseCmd { virtual void Process() = 0; }`
- **用途**: 命令抽象基类，所有命令必须实现 `Process()` 纯虚函数
- **设计意图**: 极简接口——只定义一个执行入口，参数由子类成员变量携带

### `CmdHandler`
- **用途**: 命令句柄，是 `CmdQueue` 链表的节点
- **核心逻辑**:
  - 持有 `base_cmd_ptr_`（指向 `BaseCmd` 子类实例，用于 `BP<T>()` 转换）
  - `next`（生产者→消费者方向）和 `rnext`（回收队列方向）两个原子指针
  - `Free()` 虚函数：被 `CmdQueue::GetCmd` 消费后调用，默认清空 next 指针
  - `CmdMaker<T>` 重写 `Free()` 实现对象池回收

### `BP<T>(CmdHandler* cmd)`
- **签名**: `template<class T> T* BP(CmdHandler*)`
- **用途**: Base Pointer — 从 `CmdHandler` 恢复出具体的 `BaseCmd` 子类指针
- **核心逻辑**: `static_assert(is_base_of<BaseCmd,T>)` + `reinterpret_cast<BaseCmd*>(base_cmd_ptr_)` + `static_cast<T*>`
- **使用场景**: 工作线程 `GetCmd()` 拿到 `CmdHandler*` 后，通过 `BP<T>(cmd)->Process()` 执行

### `CmdQueue` — 无锁生产者/消费者队列

- **签名**: `class CmdQueue`
- **数据成员**:
  ```cpp
  CmdHandler dummy;                    // 哨兵节点
  atomic<CmdHandler*> head_, tail_;    // 队列头尾指针
  atomic<unsigned> reset_lock;         // Reset 并发保护
  ```

#### `PutCmd(CmdHandler* cmd)`
- **用途**: 生产者入队（工作线程外的任何线程调用）
- **调用流程**:
  1. `assert(cmd->next == nullptr)`
  2. CAS 循环更新 `tail_` 到新节点
  3. 将旧 tail 的 `next` 设为 cmd（release 语义，保证之前的写入对消费者可见）
- **核心逻辑**: Michael-Scott 队列的标准入队：先 CAS tail，再 store next

#### `GetCmd()`
- **用途**: 消费者出队（仅工作线程调用）
- **调用流程**:
  1. 读取 `head_`
  2. 若 `head == tail`（队列空）→ 返回 `nullptr`
  3. 若 `head == &dummy` → 加 `reset_lock` 防止并发 Reset
  4. 读取 `head->next`，若为 `nullptr` 则重试
  5. CAS 更新 `head_` 到 next
  6. 调用 `head->Free()` 释放旧节点
  7. 若 next 是 dummy（只剩哨兵）→ 继续循环；否则返回 next
- **核心逻辑**: 哨兵节点 `dummy` 永不出队，防止空队列时 head/tail 同时为 nullptr

#### `Reset()`
- **用途**: 清空队列（析构或异常恢复时调用）
- **调用流程**:
  1. 循环消费所有节点直到只剩 dummy
  2. 用 `reset_lock` 保护——消费期间若 `PutCmd` 并发写入，则 Reset 会检测到 `reset_lock != 0` 并放弃 CAS tail
- **核心逻辑**: 多线程安全的清空——与 `PutCmd` 通过 `reset_lock` 互斥

### `CmdMaker<T>` — 模板命令工厂 + 对象池

- **签名**: `template<class T> class CmdMaker : public T, public CmdHandler`
- **设计**: 菱形继承——同时是 `BaseCmd` 的子类（通过 T）和 `CmdHandler`（用于链表节点）

#### `Create(Args... args)` → `CmdHandler*`
- **用途**: 创建命令对象（优先从对象池取）
- **调用流程**:
  1. 若对象池未销毁 → `reuse_queue_.Allocate()` 尝试获取空闲对象
  2. 池空 → `new char[(sizeof+127)&~127]`（128 字节对齐）
  3. placement new 构造 `CmdMaker`
  4. 设置 `base_cmd_ptr_ = static_cast<BaseCmd*>(this)`
- **核心逻辑**: 每个模板特化有独立的静态 `ReuseQueue`，不同类型命令池隔离

#### `Free()` override
- **用途**: 被 `CmdQueue::GetCmd` 消费后调用
- **调用流程**:
  1. 调用析构函数 `this->~CmdMaker()`
  2. 若对象池未销毁 → `reuse_queue_.Recycle(this)` 回收（placement new 的对象不 delete，放回池中复用）
  3. 若池已销毁 → `delete[]` 释放原始内存

#### `ReuseQueue` — 类型级对象池
- **设计**: 同样基于无锁单向链表（通过 `rnext` 指针）
- `Allocate()` — CAS 出队取空闲对象
- `Recycle(cmd)` — CAS 入队归还对象
- `hold_` — 临时持有已分配但未归还的对象，防止析构时丢失

**设计亮点**: `CmdMaker` 不是简单的 `new/delete` 包装——它用 placement new 在预先分配的对齐内存块上构造，回收时不释放内存只析构，实现零内存碎片的对象池。

---

## 2. opencmd — 同步/异步操作命令

文件：`opencmd.h` / `opencmd.cpp`  
依赖：`manager.h`（通过静态方法访问 BeeManager）

### `wrap_lua_bee_open(lua_State *L)` — 辅助函数

- **签名**: `static int wrap_lua_bee_open(lua_State*)`
- **用途**: 包装 Lua 的 `bee_open` 全局函数调用，添加完成后处理
- **调用流程**:
  1. `lua_getglobal(L, "bee_open")` — 获取 Lua 函数
  2. 检查是否为 Lua 函数（非 C 函数）
  3. `lua_pcallk(L, 3, 1, 0, cache, K)` — 带 continuation 的保护调用
  4. continuation `K` 处理结果：
     - 运行错误 → `cache->CacheFinished(MKERR('E','L','U','A'))`
     - 返回非 bool → 同上
     - 返回 false → `cache->CacheFinished(MKERR('E','F','A','L'))`
     - 返回 true → `cache->CacheFinished(MKERR('E','F','I','N'))`
- **核心逻辑**: 使用 Lua 5.3+ 的 `lua_pcallk` continuation 机制——即使 `bee_open` yield，等 resume 后 continuation `K` 仍会被调用

### `BeeOpenCmd::Process()`
- **用途**: 同步 `bee_open` 的实现
- **调用流程**:
  1. `BeeManager::CreateCache()` — 分配 fd 和 DataCache
  2. 设置 `cache->on_open_or_close = bind(on_open, _1, result_)` — open 完成时通过 promise 返回结果
  3. `cache->Startup(wrap_lua_bee_open, url_, opaque_)` — 启动 Lua 处理
- **核心逻辑**: 同步转异步——`Process()` 本身在异步线程执行，但调用者通过 `result_->ret`（`std::promise`）阻塞等待

### `BeeReadCmd::Process()`
- **用途**: 同步 `bee_read` 的实现
- **调用流程**:
  1. 通过 `fd_` 查找 DataCache
  2. `cache->GetDataAsync(on_recvmsg, buflen, result_)`
  3. 回调中 `memcpy` 数据到用户 buffer，通过 promise 返回拷贝字节数
  4. 若 `buffer == nullptr`（读取结束/错误）→ 返回状态码

### `BeeSendCmd::Process()`
- **用途**: 同步 `bee_send` 的实现
- **调用流程**: 查找 cache → `cache->Send(on_sent, msg, len, result_)` → 回调通过 promise 返回 err

### `BeeStatCmd::Process()`
- **用途**: 同步 `bee_stat` 的实现
- **调用流程**: 查找 cache → `cache->Stat(on_stat, result_)` → 回调 `memcpy` 到用户 buffer

### `BeeSeekCmd::Process()`
- **用途**: 同步 `bee_seek` 的实现
- **调用流程**: 查找 cache → `cache->Seek(on_seek, offset, whence, result_)`

### `BeeCloseCmd::Process()`
- **用途**: 同步 `bee_close` 的实现
- **调用流程**: 查找 cache → 绑定 close 回调 → `cache->CacheFinished(MKERR('C','L','O','S'))`

---

### 异步命令

异步命令与同步命令结构相同，但有 **两个关键差异**：

**差异 1：回调 swap 防护**
```cpp
decltype(on_open_) onopen = nullptr;
std::swap(onopen, on_open_);  // 先取出回调，防止重入
```
命令执行前先用 `std::swap` 取出回调函数指针，防止 `Process()` 被多次调用时重复回调。

**差异 2：析构时的 KILL 通知**
每个异步命令的析构函数都检查回调是否仍存在，若存在（命令被出队但未执行即被 Reset 销毁），则调用回调通知 `MKERR('K','I','L','L')`：
```cpp
BeeAsyncReadCmd::~BeeAsyncReadCmd() {
  if (nullptr != on_recv_) on_recv_(nullptr, MKERR('K','I','L','L'), user_ptr_);
}
```
- **设计意图**: `CmdQueue::Reset()` 会消费并释放所有 pending 命令。若用户持有 `user_ptr_` 的资源引用计数（如 `AsyncBeeStream`），KILL 通知确保引用计数被正确递减，防止内存泄漏。

### 异步命令列表

| 类 | Process 核心逻辑 |
|---|---|
| `BeeAsyncOpenCmd` | swap 取出 `on_open_` → `CreateCache` → `Startup(wrap_lua_bee_open)` |
| `BeeAsyncReadCmd` | swap 取出 `on_recv_` → `cache->GetDataAsync(onrecv, read_bytes_, user_ptr_)` |
| `BeeAsyncSendCmd` | swap 取出 `on_sent_` → `cache->Send(onsent, msg, len, user_ptr_)` |
| `BeeAsyncStatCmd` | swap 取出 `on_stat_` → `cache->Stat(onstat, user_ptr_)` |
| `BeeAsyncSeekCmd` | swap 取出 `on_seek_` → `cache->Seek(onseek, offset, whence, user_ptr_)` |
| `BeeAsyncCloseCmd` | swap 取出 `on_close_` → `cache->CacheFinished(MKERR('C','L','O','S'))` |

---

## 3. initcmd — 初始化命令

文件：`initcmd.h` / `initcmd.cpp`  
依赖：`manager.h`（配置合并、Lua 上下文创建）

### `merge_config(original_json)` — 辅助函数

- **签名**: `static std::string merge_config(const std::string&)`
- **用途**: 将 BeeManager 中的全局配置（uid/app_version/sdk_version/device_type）自动合并到传入的 JSON 中
- **调用流程**:
  1. 若 original 为 `null` 或空 → 设为 `{}`
  2. 用简单字符串查找检查 JSON 中是否已有 `"uid"`、`"app_version"`、`"bee_version"`、`"sys_info"` 等键
  3. 缺失的键从 BeeManager 补齐，uid 无值时自动生成 `bee_user_<rand>`
  4. 在 JSON 最后一个 `}` 前插入补充的键值对
- **核心逻辑**: **字符串级别的 JSON 操作**——不依赖 JSON 库，只用 `find`/`substr`/`stringstream` 拼接。这是一个轻量但脆弱的实现：不支持嵌套 JSON、不支持逗号在字符串内存在的情况

### `wrap_lua_runtime_init(lua_State *L)` — 辅助函数

- **签名**: `static int wrap_lua_runtime_init(lua_State*)`
- **用途**: 包装 Lua `runtime_init()` 全局函数调用
- **参数传递**: 通过 Lua 栈传递 `ref_co`（协程引用）、`drm_keyid`、`drm_key`（加密密钥）、`init_args`（合并后的配置 JSON）
- **continuation K**: 无论成功与否都 `luaL_unref` 释放协程引用，防止 Lua 内存泄漏

### `RuntimeInitCmd::Process()`
- **用途**: 执行 Lua 运行时初始化（`bee_env_init` 之后的第二步）
- **调用流程**:
  1. `lua_newthread(lua_)` — 在主 Lua 状态下创建协程
  2. `luaL_ref(L, LUA_REGISTRYINDEX)` — 注册协程防止被 GC
  3. 向协程栈压入：C 闭包 + ref_co + keyid + key + merged_config
  4. `lua_resume(co, nullptr, 4)` — 启动协程执行（适配 Lua 5.3/5.4 API 差异）
  5. 不管 Lua 执行结果，调用 `post_execute()` 完成后续操作
- **设计意图**: 初始化在独立协程中执行，允许 Lua 脚本在其中 yield（如等待网络请求完成后再继续初始化）

### `wrap_lua_bee_env_init(lua_State *L)` — 辅助函数

- **签名**: `static int wrap_lua_bee_env_init(lua_State*)`
- **用途**: 包装 Lua `bee_env_init()` 全局函数调用
- **参数传递**: 通过 lightuserdata 传递 `result_` 指针，以及合并后的配置 JSON
- **continuation K**: 解析返回值 → 设置 `result_->ret` promise → `luaL_unref` 释放协程

### `EnvInitCmd::Process()`
- **用途**: 执行 Lua 环境初始化（第一步，在 RuntimeInitCmd 之前）
- **调用流程**: 类似 RuntimeInitCmd，但使用 `result_->ret`（promise）返回成功/失败给同步调用者
- **Lua API 差异处理**: `#if LUA_VERSION_NUM == 503` vs `>= 504` — Lua 5.4 中 `lua_resume` 的 `nresult` 参数从返回值变为输出参数

---

## 模块2 设计要点总结

| 要点 | 说明 |
|---|---|
| **无锁队列** | `CmdQueue` 基于 Michael-Scott 算法，生产者/消费者均无锁，仅用 CAS |
| **类型级对象池** | `CmdMaker<T>::ReuseQueue` 为每种命令类型独立池化，避免跨类型竞争 |
| **Placement New** | 对象池用 placement new 在预分配对齐内存上构造，回收时仅析构 |
| **同步转异步桥** | 同步命令在 `Process()` 中设置 promise 回调，调用者通过 `future.get()` 阻塞等待 |
| **KILL 保护** | 异步命令析构时通知 `MKERR('K','I','L','L')`，防止 CmdQueue::Reset 导致资源泄漏 |
| **Swap 防护** | `Process()` 入口 swap 取出回调，防止重复执行 |
| **Lua Continuation** | 使用 `lua_pcallk` 的 continuation 机制，即使 Lua yield 也能在 resume 后处理结果 |
| **协程引用管理** | 初始化协程通过 `luaL_ref` / `luaL_unref` 防止 GC 提前回收 |
| **配置自动合并** | `merge_config` 将 SDK 全局设置自动注入到每个操作的 JSON 参数中 |
