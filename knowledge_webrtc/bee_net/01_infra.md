# BeeNet 模块1：基础设施层

> 精读日期：2026-06-04  
> 覆盖文件：`iobuffer.h/cpp`, `logger.h/cpp`, `os.h/cpp`, `deflua.h/c`

---

## 文件依赖关系

```
logger.h ──► logger.cpp (独立，依赖 chrono/stdio)
os.h     ──► os.cpp    (独立，MSVC 兼容层)
deflua.h ──► deflua.c  (独立，加密 Lua 字节码)
iobuffer.h ─► iobuffer.cpp (独立，纯 C/C++ 标准库)
```

四个文件间无相互依赖，都是被上层模块引用的基础设施。

---

## 1. io_buffer — 高性能读写缓冲区

文件：`iobuffer.h` / `iobuffer.cpp`  
许可：GPL v3 (Copyright 2010-2016 Yuan Jun)

### 类结构

```
io_buffer {
  - threshold_callback   // 扩容阈值回调 (bool (*)(const io_buffer&, size_t))
  - m_data_ptr           // 数据区指针
  - m_buf_size           // 已分配缓冲区总大小 (128字节对齐)
  - m_data_off           // 写指针偏移 (当前写入位置)
  - m_curr_off           // 读指针偏移 (当前读取位置)
  - m_temp_off           // get_line 辅助偏移
}
```

### `io_buffer(const char *buf, size_t size)`
- **签名**: 基于常量只读外部缓冲区构造
- **用途**: 创建不可扩容的只读视图，适用于网络接收数据解析
- **核心逻辑**: 设置 `threshold_callback = forbidden_expand_buffer`（一个断言函数，一旦触发扩容就会抛异常），`m_data_off = size` 标记全部数据已写入

### `io_buffer(char *buf, size_t size)`
- **签名**: 基于可写外部缓冲区构造
- **用途**: 创建不可扩容的可写视图
- **核心逻辑**: 同上但 `m_data_off = 0`，允许向外部缓冲区写数据

### `io_buffer(size_t size, threshold_check_callback callback)`
- **签名**: 自管理内存构造
- **用途**: 创建可自动扩容的缓冲区，默认 4KB，128 字节对齐
- **核心逻辑**: `m_buf_size = (size + 127) & ~127`，内存延迟分配（首次 alloc 时 malloc）

### `operator =(io_buffer &&from)`
- **签名**: 移动赋值
- **用途**: 转移缓冲区所有权，释放旧内存
- **核心逻辑**: 如果旧缓冲区是自管理分配的则 free，然后逐字段拷贝

### `~io_buffer()`
- **签名**: 析构函数
- **用途**: 释放自管理分配的内存
- **核心逻辑**: 通过 `threshold_callback != forbidden_expand_buffer` 判断是否为自管理内存，避免 double free

### `alloc(size_t n)`
- **签名**: `char* alloc(size_t n)`
- **用途**: 核心内存分配函数，所有写入操作均通过此函数确保有足够空间
- **调用流程**:
  1. 计算期望大小 `expect_size = m_data_off + n`
  2. 若禁止扩容 → 调用 threshold_callback 检查，失败则抛 `ENOBUFS`
  3. 若首次分配 → malloc 对齐大小
  4. 否则进入扩容逻辑
- **核心逻辑**: **自适应扩容策略**是关键设计
  - 若剩余空间不足 `min(buf_size/4, 1MB)` 则需扩容
  - **优先尝试 memmove**：如果 `curr_off >= max(128, n/4)`（已读数据足够多），则将未读数据前移，复用已读空间，避免 realloc
  - **次选 realloc**：前移不可行时，按 `min(当前大小, 4MB)` 步进扩容（即小缓冲区翻倍、大缓冲区+4MB）
  - 扩容后 size 始终 128 字节对齐
- **设计亮点**: 读指针追踪机制使得缓冲区可以被「收紧」——已读过的数据被 memmove 覆盖，减少内存申请次数

### `release(size_t n)`
- **签名**: `io_buffer& release(size_t n)`
- **用途**: 前移读指针，标记 n 字节数据已消费
- **核心逻辑**: `m_curr_off += n`；若读指针追上写指针则三指针归零

### `chop()` / `chomp()`
- **签名**: `io_buffer& chop()` / `io_buffer& chomp()`
- **用途**: chop 回退写指针一个字节；chomp 去除尾部 `\r\n`
- **核心逻辑**: chop 递减 `m_data_off`；chomp 循环检查尾部是否为 `\r` 或 `\n`，是则 chop

### `stuff(size_t n)`
- **签名**: `size_t stuff(size_t n)`
- **用途**: 提前预留 n 字节写空间
- **核心逻辑**: 调用 `alloc(n)`，然后 `m_data_off += n`（写指针前移但不写数据，留给外部填充）

### 写入函数族

| 函数 | 签名 | 用途 |
|---|---|---|
| `put_byte` | `size_t put_byte(int8_t)` | 写入 1 字节 |
| `put_word` | `size_t put_word(int16_t)` | 写入 2 字节 |
| `put_int32` | `size_t put_int32(int32_t)` | 写入 4 字节 |
| `put_int64` | `size_t put_int64(int64_t)` | 写入 8 字节 |
| `concat` | `size_t concat(const void*, size_t)` | 追加原始内存块 |
| `put_vstring` | `size_t put_vstring(const char*, va_list)` | 格式化写入（va_list 版本） |
| `put_string` | `size_t put_string(const char*, ...)` | 格式化写入 |

**核心逻辑**: 统一模式：
1. 调用 `alloc(N)` 获取可写指针
2. 直接内存写入
3. `m_data_off += N`

`put_vstring`/`put_string` 的特殊处理：
- 先用 `vsnprintf` 尝试写入剩余空间，返回所需长度
- 若 `n < len`（缓冲区不够），调用 `alloc(n+1)` 扩容后重试
- 这是典型的「试写-扩容-重试」模式

### 读取函数族

返回值版本（抛异常）：

| 函数 | 签名 | 用途 |
|---|---|---|
| `get_byte` | `int8_t get_byte()` | 读取 1 字节 |
| `get_word` | `int16_t get_word()` | 读取 2 字节 |
| `get_int32` | `int32_t get_int32()` | 读取 4 字节 |
| `get_int64` | `int64_t get_int64()` | 读取 8 字节 |

**核心逻辑**: 检查 `m_curr_off < m_data_off`（数据足够），不够则抛 `EINVAL`；读取后若读写指针相等则三指针归零

bool 返回值版本（不抛异常）：

| 函数 | 签名 |
|---|---|
| `get_byte(int8_t*)` | 读取 1 字节，返回 bool |
| `get_word(int16_t*)` | 读取 2 字节，返回 bool |
| `get_int32(int32_t*)` | 读取 4 字节，返回 bool |
| `get_int64(int64_t*)` | 读取 8 字节，返回 bool |

**核心逻辑**: 同上但不够时返回 false 而非抛异常——更适合同步 API 中的非阻塞读取

### `get_line(char const seperator[], int n)`
- **签名**: `int get_line(char const seperator[]="\n", int n=2)`
- **用途**: 按分隔符读取一行（如 `\r\n`），返回行长度或 -1
- **核心逻辑**: 使用 `m_temp_off` 跟踪扫描位置，从上次扫描结束位置继续，逐个字符与分隔符集合比对

### 网络字节序编解码 (独立函数)

| 函数 | 用途 |
|---|---|
| `EncodeInt8/16/32/64` | 主机序 → 网络序（大端） |
| `DecodeInt8/16/32/64(const uint8_t*)` | 网络序 → 主机序 |
| `DecodeInt8/16/32/64(T val)` | 原地反转字节序 |

**核心逻辑**: 手动逐字节移位重组，不依赖 `<arpa/inet.h>` 的 `htonl` 等函数——提高跨平台可移植性

**注意**: `get_int64()` 和 `DecodeInt64(uint8_t)` 实现中存在类型不一致问题：
- `get_int64()` 中 `int32_t val = *(uint64_t*)ptr;` 应为 `int64_t`
- `DecodeInt64(uint8_t val)` 参数类型应为 `uint64_t`

### `io_buffer_view<T>` (模板内部类)
- **用途**: 轻量级只读视图，持有 `io_buffer*` + offset，提供 `->` 和 `*` 操作符
- **核心逻辑**: 零拷贝访问——通过 `alloc<T>()` 创建后，通过 view 的 `get()` 做强制类型转换即用，无需额外拷贝

---

## 2. Logger — 日志系统

文件：`logger.h` / `logger.cpp`  
命名空间：`bee::logger` (独立于 `bee::net`)

### 日志级别枚举

| 值 | 宏 | 含义 |
|---|---|---|
| -1 | `BEE_NONE` | 关闭所有日志 |
| 0 | `BEE_FATAL` | 致命错误 |
| 1 | `BEE_ERROR` | 错误 |
| 2 | `BEE_WARN` | 警告 |
| 3 | `BEE_INFO` | 信息 |
| 4 | `BEE_DEBUG` | 调试 |
| 5 | `BEE_VERBOSE` | 详细 |

### `default_logger(level, name, message)`
- **签名**: `static void default_logger(const char*, const char*, const char*)`
- **用途**: 默认日志输出实现（静态函数，不对外暴露）
- **调用流程**:
  - **Android**: `level` 首字符 → `ANDROID_LOG_*` 常量 → `__android_log_write(prio, name, message)`
  - **其他平台**: `chrono::system_clock::now()` → `strftime` + 毫秒 → `fprintf(stderr, "[%s.%d] %s [%s] %s")`
- **核心逻辑**: 时间戳含毫秒精度，格式 `[2026-06-04 15:23:53.123] name [ERROR] message`

### `set_logger(callback)`
- **签名**: `void set_logger(void(*)(const char*,const char*,const char*))`
- **用途**: 注册自定义日志回调
- **核心逻辑**: 将函数指针转换为 `void*` 存储到全局变量 `g_bee_logger_callback`

### `set_log_level(level)`
- **签名**: `void set_log_level(int)`
- **用途**: 设置日志过滤级别
- **核心逻辑**: 钳位到 `[BEE_NONE, BEE_VERBOSE]` 范围

### `get_log_level()`
- **签名**: `const char* get_log_level()`
- **用途**: 获取当前日志级别字符串
- **核心逻辑**: 通过 `g_bee_logger_level_type[]` 数组查表

### `log(level, file_name, line_number, format, ...)`
- **签名**: `void log(int level, const char*, size_t, const char*, ...)`
- **用途**: 核心日志输出函数，被所有日志宏调用
- **调用流程**:
  1. `g_bee_logger_level < level` → 直接返回（级别过滤）
  2. 拼接 `"文件名:行号"` 前缀 → 4KB 栈缓冲区
  3. `vsnprintf` → 格式化用户消息
  4. 确保以 `\n` 结尾
  5. 调用 `g_bee_logger_callback(level_str, prefix, message)`
- **核心逻辑**: 注意回调参数分为三部分——`level_str`（级别标签）、`prefix`（文件名:行号）、`message`（格式化消息）

### 日志宏 (logger.h)

| 宏 | 展开 |
|---|---|
| `BEE_LOG(LEVEL, ...)` | `bee::logger::log(BEE_##LEVEL, __FILE__, __LINE__, __VA_ARGS__)` |
| `BEE_LOG_ERROR(...)` | `BEE_LOG(ERROR, ...)` |
| `BEE_LOG_WARN(...)` | `BEE_LOG(WARN, ...)` |
| `BEE_LOG_INFO(...)` | `BEE_LOG(INFO, ...)` |
| `BEE_LOG_DEBUG(...)` | `BEE_LOG(DEBUG, ...)` |
| `BEE_STACK_TRACE(...)` | `BEE_LOG(VERBOSE, ...)` (TODO: 打印堆栈信息) |
| `BEE_ASSERT(cond, ...)` | 若 `!(cond)` → `BEE_LOG(FATAL, ...)` + `abort()` |

---

## 3. OS Platform Abstraction — 平台抽象层

文件：`os.h` / `os.cpp`

### 头文件聚合 (`os.h`)

- **用途**: 作为预编译头（PCH）的前身，一次性包含所有常用 C++ 标准库头文件：
  `cassert`, `cstddef`, `cstdint`, `cstdlib`, `csignal`, `cstring`, `cstdarg`,
  `utility`, `algorithm`, `functional`, `memory`, `string`, `vector`, `queue`,
  `list`, `map`, `set`, `array`, `thread`, `future`, `mutex`, `condition_variable`, `chrono`, `random`
- **设计意图**: 减少各 .cpp 文件中重复的 #include，使编译更快

### `MKTAG(a,b,c,d)` 宏
- **签名**: `MKTAG(a,b,c,d) → uint32_t`
- **用途**: 将 4 个字符编码为 32 位整数（小端序），常用于错误码定义
- **示例**: `MKTAG('H','2','6','4')` = `0x34363248`

### `MKERR(a,b,c,d)` 宏
- **签名**: `MKERR(a,b,c,d) → -(int)MKTAG(a,b,c,d)`
- **用途**: 生成负值错误码

### MSVC 兼容 (`os.cpp`)

仅在 `_MSC_VER` 定义时编译：

#### `strcasestr(haystack, needle)`
- **签名**: `const char* strcasestr(const char*, const char*)`
- **用途**: 大小写不敏感的子串查找（POSIX 函数，MSVC 缺失）
- **核心逻辑**: 先 `strchr` 查找首字符（同时尝试 `tolower` 和 `toupper`），匹配成功后用 `tolower` 逐字符比较

#### `memmem(haystack, n, needle, m)`
- **签名**: `const void* memmem(const void*, size_t, const void*, size_t)`
- **用途**: 内存块中查找子块（POSIX 函数，MSVC 缺失）
- **核心逻辑**: 使用 `__builtin_expect` 优化分支预测；`m>1` 时采用优化的两字节跳转算法：
  - 若前两字节相同：步进 2
  - 若前两字节不同：步进 1
  - 找到匹配的前两字节后 `memcmp` 验证剩余部分
  - `m==1` 时降级为 `memchr`

### `bee_clock_gettime` / `getTimeMicroNow` (已注释)
- 两个被 `#if 0` 包围的函数，iOS/macOS 上的 `clock_gettime` 实现和跨平台微秒时间获取，均已被废弃

---

## 4. deflua — 内嵌默认 Lua 脚本

文件：`deflua.h` / `deflua.c`

### `deflua.h`
- **内容**: 三个预处理器常量
  - `DEF_GLSB_KKK` = `"QMMV<oRXK_9RvuYl"` — 疑似加密密钥
  - `DEF_GLSB_MSEC` = `1590129572013` — 时间戳 (2020-05-22)
  - `BEE_LUA_NAME` = `"bee_net_v260318"` — Lua 模块版本标识

### `deflua.c`
- **内容**: 一个 29,156 字节的 `static uint8_t def_lua[]` 数组
- **用途**: 加密存储的默认 Lua 脚本字节码，在 `bee_set_lua_file` 未被调用时作为 fallback
- **核心逻辑**: 这是一个纯数据文件——运行时通过 `xxtea` 解密后加载到 Lua VM。数组内容为密文，无法直接阅读
- **与上层的关系**: 被 `manager.cpp` 中的 `CreateLuaContext()` 引用，在无外部 Lua 脚本时使用

---

## 模块1 设计要点总结

| 要点 | 说明 |
|---|---|
| **自适应扩容** | `alloc()` 优先 memmove 而非 realloc，利用已读空间减少内存抖动 |
| **双模式缓冲** | `io_buffer` 支持「自管理堆内存」和「外部固定缓冲」两种模式，通过 callback 判定 |
| **零拷贝视图** | `io_buffer_view<T>` 提供类型安全的零拷贝随机访问 |
| **可变参数格式化** | `put_string`/`put_vstring` 的「试写-扩容-重试」模式避免了预计算长度 |
| **日志级别过滤** | 在 `log()` 入口处做级别过滤，避免低级别日志的字符串格式化开销 |
| **Android 适配** | logger 对 Android 单独使用 `__android_log_write`，其他平台用 stderr |
| **MSVC 兼容垫片** | `strcasestr` 和 `memmem` 填补了 MSVC 缺失的 POSIX 函数 |
| **加密嵌入** | deflua 将业务逻辑脚本加密后编译进二进制，运行时解密加载，保护知识产权 |
