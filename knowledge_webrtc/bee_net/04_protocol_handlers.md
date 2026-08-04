# BeeNet 模块4：协议处理器

> 精读日期：2026-06-04  
> 覆盖文件：`executer.h/cpp`, `http_exec.h/cpp`, `ws_exec.h/cpp`, `ca_exec.h/cpp`

---

## 文件依赖关系

```
executer.h/cpp ──► manager.h (AddExecuter/RemoveExecuter), deflua.h
http_exec.h/cpp ──► executer.h, manager.h, tokenizer.h
ws_exec.h/cpp   ──► executer.h, manager.h, tokenizer.h, deflua.h
ca_exec.h/cpp   ──► executer.h, manager.h, deflua.h (OpenSSL 1.x & 3.x 双兼容)
```

---

## 1. Executer — 协议处理器基类

### 构造函数
- **初始化**: `lua_co_(nullptr)`, `curr_events_(0)`, `curr_event_id_(-1)`
- **链表初始化**: `prev_(this)`, `next_(this)` — 自循环哨兵模式
- **析构断言**: 确保已从 BeeManager 链表移除（`prev_==this && next_==this`）

### `Initialize(lua_State *co)` → bool
- **用途**: 创建 curl easy handle 并设置默认选项
- **调用流程**:
  1. 保存 `lua_co_`（关联的 Lua 协程）
  2. `curl_easy_init()` 创建 handle
  3. 设置默认选项：
     - `CURLOPT_PRIVATE` = this（用于 curl 回调中找回 Executer）
     - 内置 CA 模式：`CURLOPT_SSL_CTX_FUNCTION` = `sslctx_Init`
     - 系统 CA 模式：`CURLOPT_SSL_OPTIONS` = `CURLSSLOPT_NATIVE_CA`
     - TCP keepalive：30s 空闲/10s 间隔/7 次失败
     - `CURLOPT_NOSIGNAL` = 1L（多线程安全）
- **内置 CA 模式** (`sslctx_Init`): 从 BeeManager 获取 RC4 解密 + gz 解压后的 X509 证书栈，逐一添加到 SSL_CTX 的证书存储中

### `SetOption(optname, optval)` → bool
- **用途**: 运行时设置 curl 选项（由 Lua 脚本调用）
- **支持选项**: 使用静态 map 映射字符串到 curl 选项

| 选项名 | curl 选项 | 类型 |
|---|---|---|
| `dns_server` | `CURLOPT_DNS_SERVERS` | string |
| `doh_url` | `CURLOPT_DOH_URL` | string |
| `connect_timeout` | `CURLOPT_CONNECTTIMEOUT` | number |
| `timeout` | `CURLOPT_TIMEOUT` | number |
| `verbose` | `CURLOPT_VERBOSE` + `CURLOPT_DEBUGFUNCTION` | number |
| `redirect` | `CURLOPT_FOLLOWLOCATION` | number |
| `tcp_fastopen` | `CURLOPT_TCP_FASTOPEN` + SSL early data | number |
| `speed_max` | `CURLOPT_MAX_RECV_SPEED_LARGE` | number |
| `speed_min` | `CURLOPT_LOW_SPEED_LIMIT` + `CURLOPT_LOW_SPEED_TIME`(5s) | number |
| `proxy` | `CURLOPT_PROXY` | string |
| `proxy_user` | `CURLOPT_PROXYUSERNAME` | string |
| `proxy_pass` | `CURLOPT_PROXYPASSWORD` | string |
| `untrust_proxy` | 禁用所有 SSL 验证（proxy+自身） | — |

### `SendRequest(url)` → bool
- **用途**: 发起 HTTP 请求
- **调用流程**: `EasySetOption(CURLOPT_URL, url)` → `BeeManager::AddExecuter(this)`
- **注册后**: BeeManager 的事件循环开始对此 Executer 驱动 curl 传输

### `SetEvent(ev)` / `RemoveEvent(ev)`
- **用途**: 位掩码管理 `EV_READ`/`EV_WRITE`/`EV_CLOSE` 事件

### `HttpTrace` — curl 调试回调
- **用途**: 若 `verbose` 启用，捕获 curl 的 HTTP/SSL 数据流并打日志
- **类型**: `CURLINFO_TEXT`（WARN级别）、HEADER_OUT/DATA_OUT/DATA_IN 等（注释掉）

### `EasySetOption` / `EasyGetInfo` (模板函数)
- **用途**: 类型安全的 curl API 包装
- **核心逻辑**: 自动检查返回值，失败时打 ERROR 日志 + 返回 false

---

## 2. HTTPExecuter — HTTP 协议处理器

### 状态机
```
INIT → HEAD → BODY → COMPLETE → DESTROY
                ↘ BREAK (中断)
```

### 构造函数
- `response_(65536)` — 64KB 响应缓冲区
- 所有回调引用初始化为 `LUA_NOREF`

### `GetHeader(buffer, size, nitems, userp)` — curl 静态回调
- **用途**: curl 收到 HTTP 响应头时调用
- **调用流程**:
  1. 解析响应行（`Tokenizer`）：提取 HTTP 状态码
  2. 若 stage == INIT → 设置 `stage_ = HEAD`，缓存 `ref_cb_co_`
  3. 解析 `Content-Type` → 设置 `response_content_type`
  4. 解析 `Content-Length` → 设置 `response_content_length`
  5. 调用 Lua `onhead` 回调（通过 `lua_rawgeti(L, REGISTRYINDEX, custom_get_head_)` resume 协程）

### `GetBody(buffer, size, nitems, userp)` — curl 静态回调
- **用途**: curl 收到 HTTP body 数据时调用
- **调用流程**:
  1. 若 stage == HEAD → stage = BODY，触发 `onhead` 一次
  2. 每次收到数据 → 调用 Lua `onbody` 回调
  3. body 数据追加到 `response_` 缓冲区
  4. 若 stage == COMPLETE → `EasySetOption(CURLOPT_WRITEFUNCTION, nullptr)` 阻止继续回调

### `ProgressCallback(userp, dltotal, dlnow, ultotal, ulnow)` — 进度回调
- **用途**: 下载进度报告
- **调用流程**: 调用 Lua `onprogress` 回调，传递 dlnow/dltotal

### `OnCompleteEvent()` → bool
- **用途**: curl 传输完成时被 BeeManager 调用
- **调用流程**:
  1. `EasyGetInfo` 获取 HTTP 版本、响应码
  2. 若 stage == BREAK（中断下载）→ 获取详细性能指标
  3. 将结果写入 Lua 对象元表：`http_code`、`http_version`、`remote_addr`、`body_size`、`redirect`、各项耗时
  4. Resume Lua 协程，传递 `(err, http_self)`

**性能指标**（仅 BREAK 状态获取）:

| Lua 字段 | curl INFO |
|---|---|
| `http_code` | `CURLINFO_RESPONSE_CODE` |
| `http_version` | `CURLINFO_HTTP_VERSION` |
| `remote_addr` | `CURLINFO_PRIMARY_IP` |
| `body_size` | `CURLINFO_SIZE_DOWNLOAD_T` |
| `redirect` | `CURLINFO_REDIRECT_COUNT` |
| `nslookup_time` | `CURLINFO_NAMELOOKUP_TIME` |
| `connect_time` | `CURLINFO_CONNECT_TIME` |
| `handshake_time` | `CURLINFO_APPCONNECT_TIME` |
| `request_time` | `CURLINFO_PRETRANSFER_TIME` |
| `firstbyte_time` | `CURLINFO_STARTTRANSFER_TIME` |

### Lua 绑定 (`LuaOpenHTTPLib`)

| Lua 方法 | C 函数 | 说明 |
|---|---|---|
| `http:body()` | `LuaGetBody` | 获取已接收的所有 body 数据 |
| `http:size()` | `LuaGetBodySize` | 获取 response_content_length |
| `http.onhead = func` | `SetOption("onhead", ...)` | 设置响应头回调 |
| `http.onbody = func` | `SetOption("onbody", ...)` | 设置 body 数据回调 |
| `http.onprogress = func` | `SetOption("onprogress", ...)` | 设置下载进度回调 |

### `LuaHttpGet(L)` — 主入口
- **用途**: Lua 侧 `http.get(url, options)` 调用入口
- **调用流程**:
  1. 在 Lua 堆上 placement new HTTPExecuter
  2. 设置元表 `http`
  3. 解析 options table（headers、post form 等）
  4. 设置 curl 默认选项：`ACCEPT_ENCODING`、`ALTSVC_CTRL`（H1/H2/H3）、`LOW_SPEED_LIMIT`(1B/s)/`LOW_SPEED_TIME`(5s)、`HEADERFUNCTION`/`WRITEFUNCTION`
  5. `SendRequest(url)` → `lua_yieldk` 等待完成
  6. 完成时 resume，continuation `K` 释放协程引用

---

## 3. WSExecuter — WebSocket 协议处理器

### 状态机
```
INIT → HTTP(upgrade握手) → WEBSOCKET → DESTROY
```

连接状态（`state_` 位掩码）：
| 状态 | 含义 |
|---|---|
| `S_ERROR` (-1) | 错误 |
| `S_WS_OK` (0) | 正常连接 |
| `S_CLOSE_1` (1) | 本端主动发送 Close frame |
| `S_CLOSE_2` (2) | 收到对端 Close frame 或连接断开 |
| `S_CLOSED` (3) | 双向关闭完成 |

### 构造函数
- `send_buffer_(4096)` / `recv_buffer_(4096)` — 收发缓冲区
- 所有回调引用初始化为 `LUA_NOREF`

### `LuaCreate(L)` — 主入口
- **用途**: Lua 侧 `websocket.create(url, options)` 调用入口
- **调用流程**:
  1. Lua 堆上 placement new WSExecuter
  2. 设置元表 `websocket`
  3. 创建协程用于回调执行
  4. `Initialize(co)` — 创建 curl handle
  5. 设置 WebSocket HTTP Upgrade 头：`Connection: Upgrade`、`Upgrade: websocket`、`Sec-WebSocket-Version: 13`
  6. 解析 options table → 自定义 headers、timeout
  7. `SendRequest(url)` → `lua_yieldk` 等待握手完成

### `OnCompleteEvent()` → bool
- **用途**: HTTP Upgrade 握手完成
- **调用流程**:
  1. 检查 HTTP 状态码是否为 101
  2. 检查 `Sec-WebSocket-Accept` 头
  3. 设置 `stage_ = WEBSOCKET`
  4. 调用 Lua `onopen` 回调

### `OnReadEvent()` — 从 recv_buffer_ 读取
- **用途**: BeeManager 发现 EV_READ 事件时调用
- **调用流程**:
  1. 读取 `recv_buffer_` 中的数据
  2. 解析 opcode + payload
  3. 若 PING → `SendPong` 回复
  4. 若 CLOSE → 处理关闭、回复 CLOSE
  5. 若 TEXT/BINARY → 调用 Lua `onmessage` 回调
  6. 若 CONTINUATION → 追加数据到分片缓冲区

### `OnWriteEvent()` — 向 send_buffer_ 写入
- **用途**: BeeManager 发现 EV_WRITE 事件时调用
- **调用流程**: 从 `send_buffer_` 提取数据，通过 `curl_ws_send` 发送

### `SendMsg(opcode, msg, len, fin)` → bool
- **用途**: 构造 WebSocket 帧并加入发送队列
- **核心逻辑**:
  1. 若不是 fin → `opcode |= CURLWS_CONT`（分片）
  2. 计算 buflen（CLOSE 帧额外 2 字节状态码 1000=0x03eb）
  3. 写入 `send_buffer_`：`[opcode][buflen][payload]`
  4. `SetEvent(EV_WRITE)` 触发发送

### `SendTextMsg` / `SendPing` / `SendPong` / `SendClose`
- **用途**: 便捷发送方法，均为 `SendMsg` 的封装
- `SendClose`：先 `SendMsg(CURLWS_CLOSE, reason, len, true)`，若发送失败则直接调 `OnCloseEvent`

### Lua 绑定 (`LuaOpenWebSocketLib`)

| Lua 方法 | C 函数 | 说明 |
|---|---|---|
| `websocket:ping(msg)` | `LuaSendPing` | 发送 Ping |
| `websocket:sendmsg(msg)` | `LuaSendMsg` | 发送文本/二进制消息 |
| `websocket:close()` | `LuaDisconnect` | 关闭连接 |
| `websocket.onopen = func` | `LuaSetCallback` | 设置回调 |
| `websocket.onmessage = func` | `LuaSetCallback` | 设置回调 |
| `websocket.onclose = func` | `LuaSetCallback` | 设置回调 |
| `websocket.onerror = func` | `LuaSetCallback` | 设置回调 |
| `__gc` | `LuaDestroy` | GC 销毁 |

---

## 4. CAExecuter — 证书颁发机构 / 在线更新

### 功能概述
CAExecuter 处理两件事：
1. **在线 Lua 脚本更新**：与 CA 服务器进行 RSA 加密通信，下载新版本 Lua 脚本并热替换
2. **密钥交换**：8 阶段协议完成身份验证和脚本分发

### 协议阶段（`stage_`）

```
STAGE0 → STAGE1 → STAGE2 → STAGE3 → STAGE4 → STAGE5 → STAGE6 → STAGE7 → STAGE8
  │        │        │        │        │        │        │        │
  │        │        │        │        │        │        │        └ 新脚本加载
  │        │        │        │        │        │        └ 请求脚本内容
  │        │        │        │        │        └ 检查脚本 MD5
  │        │        │        │        └ 发送脚本名称
  │        │        │        └ 验证服务端签名
  │        │        └ 发送客户端签名
  │        └ 收到服务端公钥
  └ 发送客户端公钥
```

### 协议数据结构 (`#pragma pack(1)` 紧凑布局)

| 结构 | 字段 | 用途 |
|---|---|---|
| `CAHeader` | length(2B) + code(1B) + mark(4B) | 短头部 |
| `CABigHeader` | length(4B) + code(1B) + mark(4B) | 长头部 |
| `PlayerPubKey` | header + pub_key[272] | 客户端公钥包 |
| `CAPubKey` | header + pub_key[272] | 服务端公钥包 |
| `Signature` | sum(2B) + r[32] + hash[20] | 数字签名 |
| `CAKey` | msec(8B) + key[16] + md5[32] | 加密密钥包 |
| `CAError` | header + error(1B) | 错误响应 |

Magic: `CAMARKMAGIC = 0x53484341` ("ACHS")

### `Create(co)` — 工厂方法
- **调用流程**: `new CAExecuter` → `Initialize(co)` → `CURLOPT_CONNECT_ONLY` (裸 TCP) → `GenerateRSAKey` (1024-bit RSA)

### `GenerateRSAKey(rsa_key, pub_key)` → bool
- **用途**: 生成本地 RSA 密钥对
- **双版本兼容**:
  - **OpenSSL < 3.0**: `RSA_generate_key_ex(rsa, 1024, bn(65537), NULL)` + `PEM_write_bio_RSA_PUBKEY`
  - **OpenSSL >= 3.0**: `EVP_PKEY_CTX_new_id(EVP_PKEY_RSA)` + `EVP_PKEY_keygen_init` + params `{bits=1024, e=65537}` + `PEM_write_bio_PUBKEY`

### `DoEncrypt` / `DoDecrypt`
- **用途**: RSA 公钥加密 / 私钥解密
- **双版本兼容**: 同上，`RSA_public_encrypt` vs `EVP_PKEY_encrypt`

### `CreateSignature(signature, client_pub_key)` → bool
- **用途**: 对客户端公钥创建签名
- **调用流程**: SHA1 哈希 → `CheckSum` → `EVP_DigestSign` (ECDSA with SHA256)

### `CheckSignature(signature, sig_len, pub_key)` → bool
- **用途**: 验证服务端签名
- **调用流程**: `EVP_DigestVerify` 验证 ECDSA 签名

### `CheckSum(buffer, len)` → uint16_t
- **用途**: 计算 16 位校验和（累加每个字节）

### `GetRuntime(in, inlen, key, outlen)` — 静态方法
- **用途**: xxtea 解密 + gz 解压下载的 Lua 脚本
- **调用流程**: xxtea（key=从CAKey获取）→ gz 解压 → 返回解密后的字节码

### `OnCompleteEvent()` — 阶段驱动
- **用途**: 根据当前 `stage_` 处理服务端响应
- **阶段流转**: 每个阶段完成后 `stage_++`，设置 `EV_WRITE` 发送下一阶段数据
- **STAGE8 完成**: 调用 `BeeManager::ChangeLuaContext` 热替换 Lua 脚本

### Lua 绑定

| Lua 方法 | C 函数 | 说明 |
|---|---|---|
| `ca.update()` | `LuaBeeUpdate` | 触发在线更新流程 |
| `ca.new_runtime()` | `LuaNewRuntime` | 创建新运行时（使用新密钥） |

**注意**: `LuaBeeUpdate` 末尾有 `lua_pushnil(L); lua_pushstring(L, "online update disabled"); return 2;` — 当前版本**在线更新功能已被禁用**，直接返回失败。

---

## 模块4 设计要点总结

| 要点 | 说明 |
|---|---|
| **curl 事件驱动** | 所有协议通过 curl multi handle 的事件循环统一驱动 |
| **Lua 协程挂起** | `lua_yieldk` + continuation 实现异步 I/O 的同步化 |
| **自定义数据结构** | WebSocket 帧和 CA 协议均使用 `#pragma pack(1)` 紧凑二进制布局 |
| **双 OpenSSL 兼容** | CAExecuter 同时支持 OpenSSL 1.x (RSA_*) 和 3.0 (EVP_PKEY_*) API |
| **强制 URL 格式** | `SendRequest` 通过 URL scheme 判断协议类型 |
| **递归销毁保护** | `Destroy()` 先检查 `stage_ != DESTROY` 再操作，防止重复析构 |
| **在线更新禁用** | CAExecuter 的 `LuaBeeUpdate` 硬编码返回失败 |
