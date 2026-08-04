# 03 -- 模块分析：stun/

## 模块概述

`stun/` 模块是 libnice 的 STUN 协议实现，总计约 10000 行 C 代码（`.c` + `.h`），涵盖 STUN (RFC 5389)、TURN (RFC 5766) 和 ICE 协议中所需的 STUN 消息处理。模块按职责分为三层架构：

```
stun/ (约 10000 行)
├── 消息格式层 (message layer)        -- 本部分内容
│   ├── stunmessage.h / stunmessage.c  -- 消息结构、构造与解析 (~1800 行)
│   ├── constants.h                    -- 线格式常量定义 (203 行)
│   ├── stun5389.h / stun5389.c        -- RFC 5389 兼容处理 (183 行)
│   ├── utils.h / utils.c              -- 底层辅助函数 (~130 行)
│   └── stuncrc32.h / stuncrc32.c      -- CRC32 计算 (用于 FINGERPRINT)
├── 事务层 (agent layer)               -- STUN agent 事务状态机
│   ├── stunagent.h / stunagent.c
│   ├── debug.h / debug.c
│   └── stunhmac.h / stunhmac.c        -- HMAC-SHA1 消息完整性 (用于 MESSAGE-INTEGRITY)
└── 应用层 (usages layer)              -- 高层 STUN 用法
    ├── bind.h / bind.c                -- 绑定发现 (NAT 类型探测)
    ├── ice.h / ice.c                  -- ICE 连接检查
    ├── turn.h / turn.c                -- TURN 分配/刷新/发送/通道绑定
    └── timer.h / timer.c              -- STUN 重传定时器
```

**依赖关系**：消息格式层是基础层，被事务层和应用层共同依赖。`constants.h` 定义了线格式偏移和魔数常量，被所有文件使用。`utils.h/utils.c` 提供整数网络字节序转换和对齐计算等底层操作。

---

## 第一部分：消息格式层

### 1. 文件: constants.h (203 行)

`constants.h` 定义了 STUN 消息和属性在线格式上的偏移量、长度和魔数常量，是整个模块的"硬件规格书"。

#### 1.1 STUN 消息 20 字节头部结构

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Message Type         |         Message Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Magic Cookie                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                     Transaction ID (96 bits)                  |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### 1.2 消息头部偏移常量

| 常量 | 值 | 含义 |
|------|-----|------|
| `STUN_MESSAGE_TYPE_POS` | 0 | Type 字段偏移 (2 bytes) |
| `STUN_MESSAGE_TYPE_LEN` | 2 | Type 字段长度 |
| `STUN_MESSAGE_LENGTH_POS` | 2 | Length 字段偏移 (2 bytes) |
| `STUN_MESSAGE_LENGTH_LEN` | 2 | Length 字段长度 |
| `STUN_MESSAGE_TRANS_ID_POS` | 4 | Transaction ID 偏移 |
| `STUN_MESSAGE_TRANS_ID_LEN` | 16 | Transaction ID 长度 (128 bits) |
| `STUN_MESSAGE_ATTRIBUTES_POS` | 20 | 属性区域起始偏移 |
| `STUN_MESSAGE_HEADER_LENGTH` | 20 | 消息头部总长度 |

**注意**：`STUN_MESSAGE_TRANS_ID_POS = 4`，这意味着 Transaction ID 的前 4 字节覆盖了 Magic Cookie 的位置。在 RFC5389 模式下，这 4 字节会被 Magic Cookie (0x2112A442) 覆盖，剩余的 12 字节才是真正的随机 Transaction ID。

#### 1.3 TLV 属性头部偏移常量

STUN 属性采用 TLV (Type-Length-Value) 格式，每个属性由一个 4 字节头部开始：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Attribute Type        |        Attribute Length       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Value (variable)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 常量 | 值 | 含义 |
|------|-----|------|
| `STUN_ATTRIBUTE_TYPE_POS` | 0 | 属性 Type 偏移 (2 bytes) |
| `STUN_ATTRIBUTE_TYPE_LEN` | 2 | 属性 Type 长度 |
| `STUN_ATTRIBUTE_LENGTH_POS` | 2 | 属性 Length 偏移 (2 bytes) |
| `STUN_ATTRIBUTE_LENGTH_LEN` | 2 | 属性 Length 长度 |
| `STUN_ATTRIBUTE_VALUE_POS` | 4 | 属性 Value 起始偏移 |
| `STUN_ATTRIBUTE_HEADER_LENGTH` | 4 | 属性头部总长度 |

**TLV 编码规则**：Length 字段表示 Value 区域的字节数（不含头部）。属性之间必须 32 位对齐，不足时以 0 填充（RFC 5389 下可选）。

#### 1.4 网络大小与标识常量

| 常量 | 值 / 行数 | 含义 |
|------|-----------|------|
| `STUN_MAX_MESSAGE_SIZE_IPV4` | 576 | IPv4 最大 STUN 消息大小 (参考 IPv4 最小 MTU) |
| `STUN_MAX_MESSAGE_SIZE_IPV6` | 1280 | IPv6 最大 STUN 消息大小 (参考 IPv6 最小 MTU) |
| `STUN_ID_LEN` | 16 | Transaction ID 逻辑长度 |
| `STUN_AGENT_MAX_SAVED_IDS` | 200 | 同时活跃的 STUN 事务最大数 |
| `STUN_AGENT_MAX_UNKNOWN_ATTRIBUTES` | 256 | 单个错误响应可报告的未知属性最大数 |
| `STUN_MAGIC_COOKIE` | 0x2112A442 | RFC 5389 Magic Cookie |
| `TURN_MAGIC_COOKIE` | 0x72c64bc6 | TURN Magic Cookie (Google/MSN 兼容) |

---

### 2. 文件: stunmessage.h / stunmessage.c (1788 行)

#### 2.1 StunMessage 结构体

```c
struct _StunMessage {
  StunAgent *agent;              // 关联的代理（用于兼容模式判断和认证状态同步）
  uint8_t *buffer;               // 指向线格式缓冲区的指针（外部分配，外部生命周期）
  size_t buffer_len;             // 缓冲区总容量（不是当前消息长度）
  uint8_t *key;                  // 短期凭证密钥（ICE 密码，SASLprep 后的 UTF-8 密码）
  size_t key_len;                // 密钥长度
  uint8_t long_term_key[16];     // 长期凭证密钥（MD5(username:realm:password)，固定 16 字节）
  bool long_term_valid;          // long_term_key 是否有效
};
```

**设计要点**：
- `StunMessage` 不拥有 buffer 内存，buffer 由调用方分配和管理
- 消息构造直接在 buffer 中增量进行：先写头部，然后逐个追加属性 TLV
- `buffer_len` 是缓冲区容量，用于溢出检查；实际消息长度通过 `stun_message_length()` 从线格式头部读取
- `agent` 指针用于兼容模式判断（如 OC2007 下 REALM/NONCE 属性 ID 互换、MSN 下 XOR 地址 Cookie 不同）
- `key` / `key_len` 在 `finish_message()` 和 `validate()` 中用于存储和传递认证密钥
- `long_term_key[16]` 缓存长期凭证的 16 字节 MD5 派生密钥，避免重复计算

#### 2.2 核心枚举

**StunClass** -- 消息类别：
| 值 | 枚举 | 含义 |
|----|------|------|
| 0 | `STUN_REQUEST` | 请求消息（需要响应） |
| 1 | `STUN_INDICATION` | 指示消息（无需响应） |
| 2 | `STUN_RESPONSE` | 成功响应 |
| 3 | `STUN_ERROR` | 错误响应 |

消息类别编码规则：类别值存储在 Message Type 的 bit [4:7] 和 bit 0 分别为 C0 和 C1。

**StunMethod** -- 消息方法：
| 值 | 枚举 | 来源 |
|----|------|------|
| 0x001 | `STUN_BINDING` | RFC 5389 |
| 0x003 | `STUN_ALLOCATE` | TURN-12 |
| 0x004 | `STUN_REFRESH` / `STUN_SEND` | TURN-12 / TURN-00 |
| 0x006 | `STUN_IND_SEND` | TURN-12 |
| 0x007 | `STUN_IND_DATA` | TURN-12 |
| 0x008 | `STUN_CREATEPERMISSION` | TURN-12 |
| 0x009 | `STUN_CHANNELBIND` | TURN-12 |

**注意**：方法值有重叠（如 0x004 同时表示 REFRESH 和 SEND），区分依赖于消息类别（REQUEST vs INDICATION）。

**StunError** -- 错误码（RFC 5389 / TURN-12 / ICE-19 / RFC 7675）：
| 值 | 枚举 | 含义 |
|----|------|------|
| 300 | `STUN_ERROR_TRY_ALTERNATE` | 尝试备用服务器 |
| 400 | `STUN_ERROR_BAD_REQUEST` | 坏请求 |
| 401 | `STUN_ERROR_UNAUTHORIZED` | 未授权 |
| 403 | `STUN_ERROR_FORBIDDEN` | 禁止（RFC 7675 许可新鲜度） |
| 420 | `STUN_ERROR_UNKNOWN_ATTRIBUTE` | 未知属性 |
| 437 | `STUN_ERROR_ALLOCATION_MISMATCH` | 分配不匹配 |
| 438 | `STUN_ERROR_STALE_NONCE` | 过期 Nonce |
| 441 | `STUN_ERROR_WRONG_CREDENTIALS` | 凭证错误 |
| 442 | `STUN_ERROR_UNSUPPORTED_TRANSPORT` | 不支持的传输 |
| 486 | `STUN_ERROR_ALLOCATION_QUOTA_REACHED` | 分配配额用尽 |
| 487 | `STUN_ERROR_ROLE_CONFLICT` | 角色冲突（ICE） |
| 500 | `STUN_ERROR_SERVER_ERROR` | 服务器错误 |

**StunAttribute** -- 属性类型枚举（80+ 个属性值），分为：
- **强制属性** (comprehension-required)：bit 15 = 0，范围 0x0001-0x7FFF
  - 地址类：`MAPPED_ADDRESS`(0x0001), `XOR_MAPPED_ADDRESS`(0x0020), `ALTERNATE_SERVER`(0x8023)
  - 认证类：`USERNAME`(0x0006), `MESSAGE_INTEGRITY`(0x0008), `REALM`(0x0014), `NONCE`(0x0015)
  - TURN 类：`LIFETIME`(0x000D), `REQUESTED_TRANSPORT`(0x0019), `XOR_RELAYED_ADDRESS`(0x0016), `DATA`(0x0013), `XOR_PEER_ADDRESS`(0x0012)
  - ICE 类：`PRIORITY`(0x0024), `USE_CANDIDATE`(0x0025)
- **可选属性** (comprehension-optional)：bit 15 = 1，范围 0x8000-0xFFFF
  - `SOFTWARE`(0x8022), `FINGERPRINT`(0x8028)
  - ICE：`ICE_CONTROLLED`(0x8029), `ICE_CONTROLLING`(0x802A)
  - MS-ICE2：`CANDIDATE_IDENTIFIER`(0x8054), `MS_IMPLEMENTATION_VERSION`(0x8070)
  - Nomination：`NOMINATION`(0xC001)

**`STUN_ALL_KNOWN_ATTRIBUTES`** 数组列出了所有已知强制属性的 ID（以 0 结尾），供 `stun_agent_init()` 使用。OC2007 兼容模式使用单独的 `STUN_MSOC_KNOWN_ATTRIBUTES` 数组（MS-TURN 中 REALM 和 NONCE 的 ID 值互换）。

#### 2.3 核心函数

##### stun_message_init() -- 消息初始化

- **原型**:
  ```c
  bool stun_message_init (StunMessage *msg, StunClass c, StunMethod m,
      const StunTransactionId id);
  ```

- **作用**: 初始化 STUN 消息的 20 字节头部。zero-out 前 4 字节，编码类别和方法到 Type 字段，拷贝 Transaction ID。

- **关键逻辑**:
  1. 检查 `msg->buffer_len >= STUN_MESSAGE_HEADER_LENGTH (20)`
  2. `memset(msg->buffer, 0, 4)` 清零 Type + Length 字段
  3. 调用 `stun_set_type(msg->buffer, c, m)` 编码类别和方法
  4. 将 16 字节 Transaction ID 拷贝到 `msg->buffer[4..19]`
  5. **不设置 Message Length**（为 0），后续通过 `stun_message_append_*()` 累积属性时自动更新

##### stun_message_length() -- 获取消息长度

- **原型**:
  ```c
  uint16_t stun_message_length (const StunMessage *msg);
  ```

- **作用**: 从线格式头部读取 Message Length 字段，加上 20 字节头部返回总长度。

- **实现**: `return stun_getw(msg->buffer + STUN_MESSAGE_LENGTH_POS) + STUN_MESSAGE_HEADER_LENGTH;`

##### stun_message_find() -- 查找属性

- **原型**:
  ```c
  const void * stun_message_find (const StunMessage * msg, StunAttribute type,
      uint16_t *palen);
  ```

- **作用**: 扫描消息的属性区域，查找指定 `type` 的属性，返回其 Value 指针和长度。

- **关键逻辑**:
  1. **OC2007 兼容处理**：如果 agent 兼容模式为 `OC2007`，查找 `REALM` 时实际查找 `NONCE` 的线 ID，反之亦然（MS-TURN 中两属性 ID 互换）
  2. 从消息长度算出属性区域总长度
  3. 从偏移 20 开始遍历属性 TLV：读取 **2 字节 Type**、**2 字节 Length**，跳过 `4 + align(Length)` 字节
  4. Type 匹配时返回 Value 指针（`buffer + attr_offset + 4`）和 Length
  5. 遍历完所有属性未找到返回 NULL

##### stun_message_find32() / stun_message_find64() -- 提取固定长度整数

- `stun_message_find32(msg, type, &val)`: 查找指定属性，验证其长度为 4 字节，读取网络字节序 32 位整数到 host byte order
- `stun_message_find64(msg, type, &val)`: 同上，验证长度为 8 字节，读取 64 位整数

##### stun_message_find_flag() -- 查找标志属性

- **原型**:
  ```c
  StunMessageReturn stun_message_find_flag (const StunMessage *msg,
      StunAttribute type);
  ```

- **作用**: 查找一个零长度属性（如 `USE_CANDIDATE`），仅检查属性是否存在。

- **关键逻辑**: 调用 `stun_message_find()` 找到属性后，检查 Length 是否为 0。非零长度返回 `INVALID`。

##### stun_message_find_string() -- 提取字符串

- **原型**:
  ```c
  StunMessageReturn stun_message_find_string (const StunMessage *msg,
      StunAttribute type, char *buf, size_t buflen);
  ```

- **作用**: 提取 UTF-8 字符串属性，自动追加 null 终止符。

##### stun_message_find_addr() -- 提取普通地址

- **原型**:
  ```c
  StunMessageReturn stun_message_find_addr (const StunMessage *msg,
      StunAttribute type, struct sockaddr_storage *addr, socklen_t *addrlen);
  ```

- **作用**: 提取标准（非 XOR）的地址属性（`MAPPED-ADDRESS`, `ALTERNATE-SERVER` 等）。属性值格式：1 字节 reserved + 1 字节 family + 2 字节 port + 4/16 字节 address。

##### stun_message_find_xor_addr() / stun_message_find_xor_addr_full() -- 提取 XOR 地址

- `stun_message_find_xor_addr(msg, type, addr, addrlen)`: 提取 XOR 地址属性。XOR 密钥使用标准 Magic Cookie (0x2112A442)
- `stun_message_find_xor_addr_full(msg, type, addr, addrlen, magic_cookie)`: 同上但使用自定义 magic_cookie（MSN 兼容模式下使用 Transaction ID 的前 4 字节作为 XOR 密钥）

- **关键逻辑**: 先读取原始地址数据，然后调用 `stun_xor_address()` 进行 XOR 反混淆。

##### stun_message_find_error() -- 提取错误码

- **原型**:
  ```c
  StunMessageReturn stun_message_find_error (const StunMessage *msg, int *code);
  ```

- **作用**: 提取 `ERROR-CODE` 属性。属性值格式：2 字节 reserved + 1 字节 error class (百位) + 1 字节 error number (个十位) + variable reason phrase。返回 `class * 100 + number`。

##### stun_message_append() -- 预留属性空间

- **原型**:
  ```c
  void *stun_message_append (StunMessage *msg, StunAttribute type,
      size_t length);
  ```

- **作用**: 在消息末尾预留一个属性的空间（4 字节 TLV 头 + length 字节 Value），写入 Type 和 Length 字段，返回 Value 区域指针供调用方填充。

- **关键逻辑**:
  1. 计算需要对齐的 padded_length（对齐到 4 字节边界）
  2. 检查 `buffer_len` 是否足够（防止溢出）
  3. 在消息末尾写入 2 字节 Type (网络字节序) + 2 字节 Length (网络字节序)
  4. 更新消息头部的 Message Length 字段
  5. 返回 Value 区域的起始指针

##### stun_message_append_bytes() / append32() / append64() -- 追加数据

- `stun_message_append_bytes(msg, type, data, len)`: 调用 `stun_message_append()` 预留空间后将数据 memcpy 进去
- `stun_message_append32(msg, type, value)`: 追加 32 位整数（host → network byte order 转换）
- `stun_message_append64(msg, type, value)`: 追加 64 位整数
- `stun_message_append_flag(msg, type)`: 追加零长度 Flag 属性（Length=0）

##### stun_message_append_string() -- 追加字符串

- **原型**:
  ```c
  StunMessageReturn stun_message_append_string (StunMessage *msg,
      StunAttribute type, const char *str);
  ```

- **作用**: 追加 null-terminated 字符串属性。长度不含终止符。

##### stun_message_append_addr() -- 追加普通地址

- **原型**:
  ```c
  StunMessageReturn stun_message_append_addr (StunMessage * msg,
      StunAttribute type, const struct sockaddr *addr, socklen_t addrlen);
  ```

- **作用**: 追加标准地址属性（MAPPED-ADDRESS 等格式：1B reserved + 1B family + 2B port + 4B/16B address）。

##### stun_message_append_xor_addr() / stun_message_append_xor_addr_full() -- 追加 XOR 地址

- `stun_message_append_xor_addr()`: 使用标准 Magic Cookie 做 XOR 混淆后追加
- `stun_message_append_xor_addr_full()`: 使用自定义 magic_cookie 做 XOR 混淆后追加

##### stun_message_append_error() -- 追加错误码

- **原型**:
  ```c
  StunMessageReturn stun_message_append_error (StunMessage * msg,
      StunError code);
  ```

- **作用**: 追加 ERROR-CODE 属性。编码为 2B reserved + 1B (code/100) + 1B (code%100) + reason phrase 字符串。

##### stun_message_validate_buffer_length() -- 消息验证

- **原型**:
  ```c
  int stun_message_validate_buffer_length (const uint8_t *msg, size_t length,
      bool has_padding);
  ```

- **作用**: 验证缓冲区是否包含有效的 STUN 消息。

- **关键逻辑**:
  1. 检查缓冲区长度 >= 20 字节（至少包含头部）
  2. 读取 Message Type 前 2 位必须为 0（类别和方法编码的约束）
  3. 读取 Message Length，计算总长度 = Length + 20
  4. 遍历所有属性，验证每个属性的 TLV 格式完整性
  5. 返回消息总长度（成功），0（不完整），-1（无效）

##### stun_message_validate_buffer_length_fast() -- 快速验证

- **原型**:
  ```c
  ssize_t stun_message_validate_buffer_length_fast (StunInputVector *buffers,
      int n_buffers, size_t total_length, bool has_padding);
  ```

- **作用**: 矢量 I/O 版本的快速消息验证。支持多个不连续缓冲区（`StunInputVector` 数组），仅检查头部和总长度的基本有效性，不遍历属性。用于在收到网络数据后快速判断是否收到完整 STUN 消息。

##### stun_message_id() / stun_message_get_class() / stun_message_get_method() -- 消息元数据

- `stun_message_id(msg, id)`: 提取 16 字节 Transaction ID
- `stun_message_get_class(msg)`: 解码并返回消息类别 (Request/Indication/Response/Error)
- `stun_message_get_method(msg)`: 解码并返回消息方法 (Binding/Allocate/Refresh 等)

##### stun_message_has_attribute() -- 属性存在检查

- 调用 `stun_message_find()` 检查属性是否存在，忽略其值。

---

### 3. 文件: stun5389.h / stun5389.c (183 行)

`stun5389.c` 提供了 RFC 5389 和 RFC 3489 兼容模式下特有的消息级操作，主要涉及 FINGERPRINT 添加/检查、Cookie 检查和 SOFTWARE 属性。

#### 3.1 核心函数

##### stun_message_has_cookie() -- Cookie 检查

- **原型**:
  ```c
  bool stun_message_has_cookie (const StunMessage *msg);
  ```

- **作用**: 检查 Transaction ID 的前 4 字节是否为 Magic Cookie (0x2112A442)。

- **关键逻辑**: 提取 `msg->buffer[4..7]` 的 4 字节，比较 `ntohl()` 后是否等于 `STUN_MAGIC_COOKIE`。

##### stun_fingerprint() (内部函数) -- 计算 FINGERPRINT

- **原型**:
  ```c
  uint32_t stun_fingerprint (const uint8_t *buf, size_t len, bool wlm2009);
  ```

- **作用**: 计算 STUN 消息的 FINGERPRINT 值（CRC32 XOR 0x5354554E）。

- **关键逻辑**: 参见第三部分 StunCRC32 的详细分析。三个数据段：Type (2B) + 替换后的 Length (2B) + 消息体 (不含 FINGERPRINT 属性自身)。

##### stun_message_append_software() -- 追加 SOFTWARE

- **原型** (在 stunagent.c 中通过 stun_message_append_string 间接使用): 追加 `SOFTWARE` 属性，内容为 agent 的 `software_attribute` 字符串。

---

### 4. 文件: utils.h / utils.c (211 行)

提供网络字节序转换、消息类型编码/解码、地址 XOR 混淆和对齐计算等底层辅助函数。

#### 4.1 核心函数

##### stun_getw() / stun_setw() -- 16 位网络字节序

```c
static inline uint16_t stun_getw (const uint8_t *p) {
  return p[0] << 8 | p[1];
}
static inline void stun_setw (uint8_t *p, uint16_t v) {
  p[0] = v >> 8; p[1] = v & 0xFF;
}
```

直接从字节数组读取/写入网络字节序（大端）的 16 位整数，无需 `htons()`/`ntohs()` 转换。

##### stun_get_type() -- 解码 Message Type

```c
static inline int stun_get_type (const uint8_t *p,
    StunClass *c, StunMethod *m);
```

从消息头部的 2 字节 Type 字段中解码出类别 (C0, C1) 和方法 (M0-M10)。

##### stun_set_type() -- 编码 Message Type

```c
static inline void stun_set_type (uint8_t *p, StunClass c, StunMethod m);
```

将类别和方法编码为 2 字节网络字节序的 Message Type。编码规则：
```
h[0] = (c >> 1) | ((m >> 6) & 0x3e)
h[1] = ((c << 4) & 0x10) | ((m << 1) & 0xe0) | (m & 0x0f)
```

##### stun_padding() -- 对齐填充计算

```c
static inline size_t stun_padding (size_t l) {
  return ((l + 3) & ~3) - l;
}
```

返回补齐到 4 字节对齐所需的填充字节数。例如 padding(5) = 3。

##### stun_align() -- 对齐后长度

```c
static inline size_t stun_align (size_t l) {
  return (l + 3) & ~3;
}
```

返回对齐到 4 字节边界的长度。

##### stun_xor_address() -- 地址 XOR 混淆

- **原型**:
  ```c
  void stun_xor_address (const struct sockaddr *addr,
      struct sockaddr_storage *out, uint32_t magic_cookie,
      const uint8_t *transaction_id);
  ```

- **作用**: 对 `sockaddr` 执行 XOR 混淆（用于 XORMAPPED-ADDRESS、XOR-RELAYED-ADDRESS 等属性）。

- **XOR 规则**:
  - **IPv4**: `port XOR htons(magic_cookie >> 16)`，`addr XOR htonl(magic_cookie)`
  - **IPv6**: `port XOR htons(magic_cookie >> 16)`，addr 每个字节 XOR `transaction_id[i]`（即 Transaction ID 的前 16 字节，恰好前 4 字节为 Magic Cookie，后 12 字节为实际随机 ID）

##### stun_optional() -- 判断可选属性

```c
bool stun_optional (uint16_t t) {
  return t >> 15;
}
```

属性类型的 bit 15 (最高位) 为 1 表示 comprehension-optional（可选属性），为 0 表示 comprehension-required（强制属性）。

##### stun_strerror() -- 错误码转字符串

- **原型**:
  ```c
  const char *stun_strerror (StunError code);
  ```

- **作用**: 将 STUN 错误码映射为人类可读的字符串（如 401 → "Unauthorized", 487 → "Role Conflict"）。

---

### 5. 属性追加/查找函数汇总

| 函数 | 作用 |
|------|------|
| `stun_message_append()` | 预留 TLV 属性空间，返回 Value 区域指针 |
| `stun_message_append_bytes(...)` | 追加原始字节数据 |
| `stun_message_append_flag(...)` | 追加零长度 Flag 属性 |
| `stun_message_append32(...)` | 追加 4 字节整数 (host → network) |
| `stun_message_append64(...)` | 追加 8 字节整数 (host → network) |
| `stun_message_append_string(...)` | 追加字符串 |
| `stun_message_append_addr(...)` | 追加标准地址 (family + port + addr) |
| `stun_message_append_xor_addr(...)` | 追加 XOR 混淆地址 |
| `stun_message_append_xor_addr_full(...)` | 追加 XOR 混淆地址（自定义 Cookie） |
| `stun_message_append_error(...)` | 追加 ERROR-CODE 属性 |
| `stun_message_find(...)` | 查找属性，返回 Value 指针和长度 |
| `stun_message_find_flag(...)` | 查找零长度 Flag 属性 |
| `stun_message_find32(...)` | 提取 4 字节整数 (network → host) |
| `stun_message_find64(...)` | 提取 8 字节整数 |
| `stun_message_find_string(...)` | 提取字符串 |
| `stun_message_find_addr(...)` | 提取标准地址 |
| `stun_message_find_xor_addr(...)` | 提取 XOR 混淆地址 |
| `stun_message_find_xor_addr_full(...)` | 提取 XOR 混淆地址（自定义 Cookie） |
| `stun_message_find_error(...)` | 提取错误码 |
| `stun_message_has_attribute(...)` | 检查属性是否存在 |
| `stun_message_validate_buffer_length(...)` | 验证消息格式完整性 |
| `stun_message_validate_buffer_length_fast(...)` | 矢量 I/O 快速格式验证 |
| `stun_message_has_cookie(...)` | 检查 Magic Cookie |
| `stun_set_type(...)` | 编码 Message Type (类别 + 方法) |
| `stun_xor_address(...)` | 对 sockaddr 做 XOR 混淆（IPv4 端口/地址 XOR magic_cookie；IPv6 地址 XOR Transaction ID） |

---

### 6. 调用关系

```
usages/bind.c, ice.c, turn.c     (应用层 -- STUN 用法)
        |
        v
stunagent.c                      (事务层 -- 状态机、重传、认证)
        |
        v
stunmessage.c / stun5389.c      (消息格式层 -- 构造、解析、验证)
        |
        +--- constants.h          (线格式偏移和常量定义)
        +--- utils.h / utils.c    (网络字节序转换、对齐计算、XOR 地址)
        +--- stuncrc32.c          (CRC32 计算，供 FINGERPRINT 使用)
        +--- stunhmac.c           (HMAC-SHA1 计算，供 MESSAGE-INTEGRITY 使用)
```

**数据流方向**：

- **发送路径**：上层（agent/usages）→ `stun_message_init()` → `stun_message_append_*()` 系列函数逐属性追加 → `stun_5389_*()` 添加 FINGERPRINT/SOFTWARE → 直接使用 `msg->buffer + msg->buffer_len` 发送线格式数据
- **接收路径**：收到线格式数据 → `stun_message_validate_buffer_length_fast()` 快速筛查 → `stun_message_validate_buffer_length()` 完整验证 → 填充 `msg->buffer` → 上层通过 `stun_message_find_*()` 系列函数提取属性

**注意**：libnice 的 stunmessage 层**没有**显式的 `serialize()` 函数。消息直接在 buffer 中增量构造，构造完成后的 buffer 内容即是可发送的线格式数据。这种设计避免了序列化阶段的内存拷贝。

---

## 第二部分：事务代理层

### 概述

事务代理层（agent layer）构建在消息格式层之上，提供 STUN 事务的完整生命周期管理：请求构造、响应构造、错误响应构造、消息验证（含认证）和事务 ID 追踪。核心文件是 `stunagent.h` / `stunagent.c`（约 1283 行），配合 `rand.c` / `rand.h`（事务 ID 随机生成）、`debug.c` / `debug.h`（调试日志）以及底层的 `stunhmac.c` / `stunhmac.h`（HMAC-SHA1 计算）和 `stuncrc32.c` / `stuncrc32.c`（FINGERPRINT CRC32）。

---

### 1. 文件: stunagent.h / stunagent.c (1283 行)

#### 1.1 StunAgent 结构体

```c
typedef struct {
  StunTransactionId id;
  StunMethod method;
  uint8_t *key;
  size_t key_len;
  uint8_t long_term_key[16];
  bool long_term_valid;
  bool valid;
} StunAgentSavedIds;

struct stun_agent_t {
  StunCompatibility compatibility;         // 兼容模式 (RFC3489/RFC5389/MSICE2/OC2007)
  StunAgentSavedIds sent_ids[STUN_AGENT_MAX_SAVED_IDS];  // 已发送事务追踪表 (最多 200 个)
  uint16_t *known_attributes;              // 已知属性列表指针 (以 0 结尾)
  StunAgentUsageFlags usage_flags;         // 使用标志位 (凭证类型、FINGERPRINT、SOFTWARE 等)
  const char *software_attribute;          // SOFTWARE 属性值
  bool ms_ice2_send_legacy_connchecks;     // MS-ICE2 是否继续发送旧版连接检查
};
```

**结构分析**：

- **`StunAgentSavedIds`**：每条记录保存一个已发送请求的 Transaction ID、Method、认证密钥和长期凭证密钥。`valid` 标志表示该槽位是否被占用。最多追踪 200 个活跃事务（`STUN_AGENT_MAX_SAVED_IDS = 200`）。这是一个固定大小的环形数组，槽位复用通过将 `valid` 设为 `FALSE` 实现。

- **`compatibility`**：决定代理的行为差异（属性对齐、REALM/NONCE 互换、Cookie 检查等）。支持四种模式：`RFC3489`（旧版 STUN）、`RFC5389`（标准 STUN）、`MSICE2` / `WLM2009`（Microsoft ICE2）、`OC2007`（Office Communicator 2007）。

- **`known_attributes`**：指向一个以 0 结尾的 `uint16_t` 数组，列出代理识别的所有属性类型。在 `stun_agent_validate()` 中用于检测未知强制属性。注意这是一个外部指针，调用方必须保证其生命周期。

- **`usage_flags`**：位标志控制代理行为，包括：
  - `SHORT_TERM_CREDENTIALS` (bit 0)：使用短期凭证（ICE 场景，密码直接作为 HMAC 密钥）
  - `LONG_TERM_CREDENTIALS` (bit 1)：使用长期凭证（TURN 场景，密钥 = MD5(username:realm:password)）
  - `USE_FINGERPRINT` (bit 2)：添加 FINGERPRINT 属性
  - `ADD_SOFTWARE` (bit 3)：添加 SOFTWARE 属性
  - `IGNORE_CREDENTIALS` (bit 4)：完全跳过凭证验证
  - `NO_INDICATION_AUTH` (bit 5)：对 Indication 消息跳过认证
  - `FORCE_VALIDATER` (bit 6)：强制调用验证回调（即使已知密钥）
  - `NO_ALIGNED_ATTRIBUTES` (bit 7)：属性不按 32 位对齐
  - `CONSENT_FRESHNESS` (bit 8)：识别 Forbidden (403) 错误以支持 RFC 7675 许可新鲜度

- **`ms_ice2_send_legacy_connchecks`**：MS-ICE2 兼容性标志。当收到对端的 `MS_IMPLEMENTATION_VERSION` 属性后，设置为 `FALSE` 以停止发送旧版连接检查。

#### 1.2 核心函数

##### stun_agent_init() -- 初始化

- **原型**:
  ```c
  void stun_agent_init (StunAgent *agent, const uint16_t *known_attributes,
      StunCompatibility compatibility, StunAgentUsageFlags usage_flags);
  ```

- **作用**: 初始化一个 StunAgent 实例，设置兼容模式、使用标志和已知属性列表。

- **关键逻辑**:
  1. 保存 `known_attributes` 指针（外部生命周期管理）
  2. 设置 `compatibility` 和 `usage_flags`
  3. 如果兼容模式为 `MSICE2`，自动设置 `ms_ice2_send_legacy_connchecks = TRUE`
  4. 遍历 200 个 `sent_ids` 槽位，将 `valid` 全部设为 `FALSE`

##### stun_agent_init_request() -- 构造 STUN 请求

- **原型**:
  ```c
  bool stun_agent_init_request (StunAgent *agent, StunMessage *msg,
      uint8_t *buffer, size_t buffer_len, StunMethod m);
  ```

- **作用**: 创建一个新的 STUN 请求消息。生成随机 Transaction ID，初始化消息头和缓冲区关联。

- **关键逻辑**:
  1. 设置 `msg->buffer`、`msg->buffer_len`、`msg->agent`，清零 `key` 和 `long_term_valid`
  2. 调用 `stun_make_transid(id)` 生成 16 字节随机 Transaction ID
  3. 调用 `stun_message_init(msg, STUN_REQUEST, m, id)` 初始化消息头
  4. 对于 RFC5389 / MSICE2 兼容模式：将前 4 字节替换为 Magic Cookie (0x2112A442)
  5. 如果启用了 SOFTWARE 属性，调用 `stun_message_append_software()` 追加

##### stun_agent_init_indication() -- 构造 STUN 指示

- **原型**:
  ```c
  bool stun_agent_init_indication (StunAgent *agent, StunMessage *msg,
      uint8_t *buffer, size_t buffer_len, StunMethod m);
  ```

- **作用**: 创建一个 STUN Indication 消息（无需响应）。与 `init_request` 的区别：
  - 消息类别为 `STUN_INDICATION` 而非 `STUN_REQUEST`
  - **不追加 SOFTWARE 属性**（即使启用了 SOFTWARE 标志）

##### stun_agent_init_response() -- 构造 STUN 成功响应

- **原型**:
  ```c
  bool stun_agent_init_response (StunAgent *agent, StunMessage *msg,
      uint8_t *buffer, size_t buffer_len, const StunMessage *request);
  ```

- **作用**: 为收到的 STUN 请求创建成功响应消息。关键特性是**继承请求的认证密钥**，这样响应可以自动附带正确的 MESSAGE-INTEGRITY。

- **关键逻辑**:
  1. 验证 request 的类别是 `STUN_REQUEST`，否则返回 FALSE
  2. 设置 `msg->buffer`，**复制** request 的 `key`、`key_len`、`long_term_key` 和 `long_term_valid`
  3. 从 request 提取 Transaction ID，使用 `stun_message_init()` 以 `STUN_RESPONSE` 类别创建消息
  4. 如果启用了 SOFTWARE 属性，追加 SOFTWARE

##### stun_agent_init_error() -- 构造 STUN 错误响应

- **原型**:
  ```c
  bool stun_agent_init_error (StunAgent *agent, StunMessage *msg,
      uint8_t *buffer, size_t buffer_len, const StunMessage *request,
      StunError err);
  ```

- **作用**: 为收到的 STUN 请求创建错误响应。继承请求的认证密钥并追加 ERROR-CODE 属性。

- **关键逻辑**:
  1. 验证 request 类别为 `STUN_REQUEST`
  2. 复制 request 的认证密钥
  3. 以 `STUN_ERROR` 类别创建消息
  4. 如果启用 SOFTWARE，追加 SOFTWARE 属性
  5. 调用 `stun_message_append_error(msg, err)` 追加 ERROR-CODE 属性

##### stun_agent_finish_message() -- 完成消息（添加 M-I 和 FINGERPRINT）

- **原型**:
  ```c
  size_t stun_agent_finish_message (StunAgent *agent, StunMessage *msg,
      const uint8_t *key, size_t key_len);
  ```

- **作用**: 这是消息发送前的最后一步，完成以下工作：
  1. 如果需要，追加 MESSAGE-INTEGRITY 属性（HMAC-SHA1）
  2. 如果启用，追加 FINGERPRINT 属性（CRC32）
  3. 如果是请求类消息，将事务 ID 保存到 `sent_ids[]` 中供后续响应匹配

- **关键逻辑**:

  **步骤 1 -- 确定是否需要记住事务**:
  - `remember_transaction = (class == STUN_REQUEST)` -- 仅对请求类消息记录事务
  - 特殊例外：OC2007 兼容模式下 `STUN_SEND` 方法不记录（因为 TURN 服务器不对 SEND 请求发送响应）

  **步骤 2 -- 查找空槽位**:
  - 遍历 200 个 `sent_ids[]` 槽位，寻找 `valid == FALSE` 的空位
  - 如果全部占满，返回 0（丢弃消息），并输出调试警告

  **步骤 3 -- 追加 MESSAGE-INTEGRITY**:
  - 如果 `key != NULL`（从参数或 `msg->key` 获取）：
    - **长期凭证模式**: 从消息中提取 REALM 和 USERNAME，计算 `MD5(username:realm:password)` 得到 16 字节 key。如果消息中已有 `long_term_key`，直接复用（避免在 response/error 中重复计算）。
    - **短期凭证模式**: 直接使用 key 作为 HMAC 密钥。
    - 调用 `stun_message_append(msg, STUN_ATTRIBUTE_MESSAGE_INTEGRITY, 20)` 预留 20 字节空间。
    - 调用 `stun_sha1()` 计算 HMAC-SHA1，根据兼容模式决定计算范围（RFC3489/OC2007/MSICE2/RFC5389 各有不同的 len 和 msg_len 参数）。

  **步骤 4 -- 追加 FINGERPRINT**:
  - 仅在 RFC5389 / MSICE2 兼容模式且启用 `USE_FINGERPRINT` 时
  - 调用 `stun_message_append(msg, STUN_ATTRIBUTE_FINGERPRINT, 4)` 预留 4 字节
  - 调用 `stun_fingerprint()` 计算 CRC32 并 XOR 0x5354554e
  - 将计算结果写入预留的 4 字节空间

  **步骤 5 -- 保存事务 ID**:
  - 如果 `remember_transaction == TRUE`：
    - 从消息中提取 Transaction ID 存入 `sent_ids[saved_id_idx].id`
    - 保存 method、key、long_term_key、long_term_valid
    - 设置 `valid = TRUE`

  **返回值**: `stun_message_length(msg)` -- 消息总长度。返回 0 表示缓冲区不足或事务槽位已满。

  **重要使用约定**: 每次调用 `stun_agent_finish_message()` 产生一个 STUN_REQUEST 后，如果最终没有收到响应（超时），必须调用 `stun_agent_forget_transaction()` 释放槽位，否则 200 个槽位耗尽后将无法创建新请求。

##### stun_agent_validate() -- 消息验证（核心函数）

- **原型**:
  ```c
  StunValidationStatus stun_agent_validate (StunAgent *agent, StunMessage *msg,
      const uint8_t *buffer, size_t buffer_len,
      StunMessageIntegrityValidate validater, void *validater_data);
  ```

- **作用**: 验证接收到的 STUN 消息。这是整个 agent 层最核心的函数，执行了完整的验证流程：格式验证、Cookie 检查、FINGERPRINT 验证、事务匹配、凭证检查、HMAC 验证、未知属性检测。

- **返回值枚举 `StunValidationStatus`**:
  | 状态 | 含义 |
  |------|------|
  | `STUN_VALIDATION_SUCCESS` | 验证成功 |
  | `STUN_VALIDATION_NOT_STUN` | 不是 STUN 消息 |
  | `STUN_VALIDATION_INCOMPLETE_STUN` | 消息不完整 |
  | `STUN_VALIDATION_BAD_REQUEST` | 缺少 Cookie 或 FINGERPRINT 不匹配 |
  | `STUN_VALIDATION_UNAUTHORIZED_BAD_REQUEST` | 缺少认证属性 (应发 400) |
  | `STUN_VALIDATION_UNAUTHORIZED` | 认证失败 (应发 401) |
  | `STUN_VALIDATION_UNMATCHED_RESPONSE` | 响应/错误不匹配任何已发送请求 |
  | `STUN_VALIDATION_UNKNOWN_REQUEST_ATTRIBUTE` | 请求含未知强制属性 |
  | `STUN_VALIDATION_UNKNOWN_ATTRIBUTE` | 响应/指示含未知强制属性 |
  | `STUN_VALIDATION_FORBIDDEN` | 对端返回 403 Forbidden (许可新鲜度) |

- **完整验证流程**（按代码执行顺序）：

  **阶段 1: 格式验证**
  - 调用 `stun_message_validate_buffer_length()` 检查消息格式完整性
  - 如果返回 `INVALID`：返回 `STUN_VALIDATION_NOT_STUN`
  - 如果返回 `INCOMPLETE`：返回 `STUN_VALIDATION_INCOMPLETE_STUN`
  - 如果返回的长度 `!= buffer_len`：返回 `STUN_VALIDATION_NOT_STUN`（消息长度不匹配）

  **阶段 2: Cookie 检查**
  - 仅在 RFC5389 / MSICE2 模式下执行
  - 调用 `stun_message_has_cookie()` 检查 Transaction ID 前 4 字节是否为 0x2112A442
  - 缺失 Cookie：返回 `STUN_VALIDATION_BAD_REQUEST`

  **阶段 3: FINGERPRINT 检查**
  - 仅在 RFC5389 / MSICE2 模式下且启用 `USE_FINGERPRINT` 时执行
  - 调用 `stun_agent_check_fingerprint()` 计算并比较 CRC32 值
  - **MS-ICE2 兼容回退**: 如果主检查失败并且兼容模式为 MSICE2 且消息中没有 `MS_IMPLEMENTATION_VERSION` 属性，则尝试使用 WLM2009 的 CRC32 拼写错误版本再次验证
  - FINGERPRINT 不匹配：返回 `STUN_VALIDATION_BAD_REQUEST`

  **阶段 4: 响应/错误的事务 ID 匹配**
  - 仅对 `STUN_RESPONSE` 或 `STUN_ERROR` 类别的消息执行
  - 提取消息的 Transaction ID，遍历 200 个 `sent_ids[]` 槽位查找匹配
  - 找到匹配：提取保存的 `key`、`key_len`、`long_term_key`、`long_term_valid`
  - 未找到匹配：返回 `STUN_VALIDATION_UNMATCHED_RESPONSE`

  **阶段 5: 凭证忽略判断**
  - 确定是否需要跳过后续的凭证检查。以下情况忽略凭证（`ignore_credentials = TRUE`）:
    - 设置了 `IGNORE_CREDENTIALS` 标志
    - 错误响应且错误码为 400 (Bad Request)、401 (Unauthorized)、438 (Stale Nonce)、300 (Try Alternate)
    - Indication 消息且使用长期凭证或设置了 `NO_INDICATION_AUTH` 标志

  **阶段 6: 未授权请求检查 (`UNAUTHORIZED_BAD_REQUEST`)**
  - 仅对无密钥 (`key == NULL`) 且 `ignore_credentials == FALSE` 的 `STUN_REQUEST` 或 `STUN_INDICATION` 消息执行
  - 检查必需的认证属性是否缺失
  - 短期凭证 + 缺失 USERNAME 或 MESSAGE-INTEGRITY → `UNAUTHORIZED_BAD_REQUEST`
  - 长期凭证 + Request 缺失 USERNAME/MESSAGE-INTEGRITY/NONCE/REALM → `UNAUTHORIZED_BAD_REQUEST`

  **阶段 7: 用户名-密码回调**
  - 当消息包含 MESSAGE-INTEGRITY 且 (没有已知密钥 或 设置了 `FORCE_VALIDATER`):
    - 从消息中提取 USERNAME 属性
    - 调用 `validater` 回调函数，传入 username
    - 回调函数应通过 `password` 和 `password_len` 输出参数返回对应的密码
    - 回调返回 FALSE → 返回 `STUN_VALIDATION_UNAUTHORIZED`

  **阶段 8: HMAC-SHA1 验证**
  - 仅当 `ignore_credentials == FALSE` 且 `key != NULL` 且 `key_len > 0` 时执行
  - 从消息中提取 MESSAGE-INTEGRITY 属性的 20 字节 HMAC 值
  - 长期凭证: 从消息提取 REALM 和 USERNAME，调用 `stun_hash_creds()` 计算 `MD5(username:realm:password)`
  - 短期凭证: 直接使用 key 作为 HMAC 密钥
  - 调用 `stun_sha1()` 计算期望的 HMAC 值，比较（`memcmp` 20 字节）
  - 不匹配 → 返回 `STUN_VALIDATION_UNAUTHORIZED`

  **阶段 9: 许可新鲜度检查**
  - 仅当设置了 `CONSENT_FRESHNESS` 标志时执行
  - 若消息为 `STUN_ERROR` 且错误码为 403 (Forbidden) → 返回 `STUN_VALIDATION_FORBIDDEN`

  **阶段 10: 清除已匹配的事务**
  - 如果阶段 4 找到了匹配的事务 ID，将该槽位的 `valid` 设为 `FALSE`（释放槽位）

  **阶段 11: MS-ICE2 旧版连接检查处理**
  - 如果消息包含 `MS_IMPLEMENTATION_VERSION` 属性，设置 agent 停止发送旧版连接检查

  **阶段 12: 未知属性检测**
  - 调用 `stun_agent_find_unknowns()` 遍历消息中所有强制属性
  - 发现未知强制属性 → 返回对应的 `UNKNOWN_*_ATTRIBUTE` 错误

##### stun_agent_forget_transaction() -- 忘记事务

- **原型**:
  ```c
  bool stun_agent_forget_transaction (StunAgent *agent, StunTransactionId id);
  ```

- **作用**: 从 agent 的事务追踪表中删除指定的事务。当请求超时未收到响应时调用，释放事务槽位。

##### stun_agent_default_validater() -- 默认验证回调

- **原型**:
  ```c
  bool stun_agent_default_validater (StunAgent *agent,
      StunMessage *message, uint8_t *username, uint16_t username_len,
      uint8_t **password, size_t *password_len, void *user_data);
  ```

- **作用**: 一个简单的用户名-密码查表回调函数，通过遍历 `StunDefaultValidaterData` 数组匹配用户名并返回密码。`user_data` 必须是以 `username = NULL` 结尾的数组。

##### stun_agent_build_unknown_attributes_error() -- 构造未知属性错误

- **原型**:
  ```c
  size_t stun_agent_build_unknown_attributes_error (StunAgent *agent,
      StunMessage *msg, uint8_t *buffer, size_t buffer_len,
      const StunMessage *request);
  ```

- **作用**: 为包含未知强制属性的请求构造 420 (Unknown Attribute) 错误响应。收集消息中所有未知强制属性 ID，追加到 `UNKNOWN-ATTRIBUTES` 属性中。

##### stun_agent_set_software() -- 设置 SOFTWARE 属性

- **原型**:
  ```c
  void stun_agent_set_software (StunAgent *agent, const char *software);
  ```

- **作用**: 设置 SOFTWARE 属性的值。指针必须在 agent 的整个生命周期内保持有效。

---

### 2. 文件: rand.c / rand.h (125 行)

提供跨平台的密码学安全随机数生成，用于生成 STUN Transaction ID。

#### nice_RAND_nonce()

- **原型**:
  ```c
  void nice_RAND_nonce (uint8_t *dst, int len);
  ```

- **作用**: 生成 `len` 字节的密码学安全随机数到 `dst` 缓冲区。

- **多平台实现**:
  - **Windows (Win32 CryptoAPI)**: 使用 `CryptGenRandom`
  - **OpenSSL**: 使用 `RAND_bytes()`
  - **GnuTLS**: 使用 `gnutls_rnd(GNUTLS_RND_NONCE, ...)`

#### stun_make_transid() (在 stunhmac.c 中实现)

- **原型**:
  ```c
  void stun_make_transid (StunTransactionId id);
  ```

- **实现**: 简单封装 `nice_RAND_nonce(id, 16)` -- 生成 16 字节随机 Transaction ID。

- **注意**: 生成的 16 字节 ID 中前 4 字节会在 `stun_agent_init_request()` 等函数中被 Magic Cookie 覆盖（RFC5389 / MSICE2 模式下）。

---

### 3. 文件: debug.c / debug.h (229 行)

提供 STUN 模块的调试日志基础设施，所有日志通过全局开关控制。

#### 全局状态

```c
static int debug_enabled = 0;                          // 调试开关 (0=关闭, 1=开启)
static StunDebugHandler handler = default_handler;      // 当前调试回调函数
```

#### stun_debug_enable() / stun_debug_disable()

- **原型**:
  ```c
  void stun_debug_enable (void);
  void stun_debug_disable (void);
  ```

- **作用**: 全局开关 STUN 调试日志。

#### stun_debug()

- **原型**:
  ```c
  void stun_debug (const char *fmt, ...) __attribute__((__format__(__printf__, 1, 2)));
  ```

- **作用**: 输出格式化调试消息（类似 `printf`）。如果 `debug_enabled` 关闭则直接返回。

#### stun_debug_bytes()

- **原型**:
  ```c
  void stun_debug_bytes (const char *prefix, const void *data, size_t len);
  ```

- **作用**: 输出字节数组的十六进制表示，带前缀标签。

#### stun_set_debug_handler()

- **原型**:
  ```c
  void stun_set_debug_handler (StunDebugHandler _handler);
  ```

- **作用**: 设置自定义的调试回调函数。传入 NULL 恢复默认处理器（输出到 stderr）。

---

### 4. 调用关系

```
应用层 (usages/)
  bind.c / ice.c / turn.c / timer.c
       |
       | 调用: init_request / init_indication / init_response / init_error
       |       finish_message / validate / forget_transaction
       |
       v
事务代理层 (stunagent.c)
  核心: stun_agent_validate() -- 12 步验证流程
  构造: init_*() 系列函数
  完成: finish_message() -- M-I + FINGERPRINT
  追踪: sent_ids[200] -- 事务 ID 匹配

  依赖:
    ├── stunmessage.c  -- 消息构造与解析
    ├── stunhmac.c      -- HMAC-SHA1 计算
    ├── stuncrc32.c     -- FINGERPRINT CRC32
    ├── rand.c          -- 事务 ID 随机生成
    ├── debug.c         -- 调试日志输出
    └── utils.c         -- 字节序转换与对齐
```

**关键调用链**:

- **发送请求**: `stun_agent_init_request()` → `stun_make_transid()` (rand) → `stun_message_init()` → `stun_message_append_*()` (按需追加属性) → `stun_agent_finish_message()` → `stun_sha1()` (HMAC) → `stun_fingerprint()` (CRC32) → buffer 可发送

- **接收验证**: buffer → `stun_agent_validate()` → `stun_message_validate_buffer_length()` → `stun_agent_check_fingerprint()` → 事务 ID 匹配 (sent_ids[]) → `validater()` 回调 → `stun_sha1()` (HMAC 验证) → `stun_hash_creds()` (长期凭证 MD5) → `stun_agent_find_unknowns()` → 返回验证状态

---

## 第三部分：加密层

### 概述

加密层是 STUN 安全机制的基础，负责两件事：
1. **消息完整性保护**（MESSAGE-INTEGRITY）：通过 HMAC-SHA1 防止消息被篡改
2. **消息指纹保护**（FINGERPRINT）：通过 CRC32 校验防止 STUN 消息与其他协议（如 RTP）混淆

两个核心文件 `stunhmac.c` 和 `stuncrc32.c` 提供了跨平台（Win32 CryptoAPI / OpenSSL / GnuTLS）的加密原语实现，被 `stunagent.c` 的 `finish_message()` 和 `validate()` 调用。

---

### 1. 文件: stunhmac.h / stunhmac.c (373 行)

#### 概述

`stunhmac.c` 实现了 STUN 协议所需的全部加密计算：HMAC-SHA1 消息完整性校验、长期凭证 MD5 密钥派生，以及事务 ID 的随机生成。支持三种加密后端：Windows CryptoAPI（Win32 平台）、OpenSSL（Unix 通用）、GnuTLS（GNU 系统），通过编译期条件宏选择。

---

#### 1.1 核心函数

##### stun_sha1() -- 计算 HMAC-SHA1

- **原型**:
  ```c
  void stun_sha1 (const uint8_t *msg, size_t len, size_t msg_len,
      uint8_t *sha, const void *key, size_t keylen, int padding);
  ```

- **作用**: 计算 STUN 消息的 MESSAGE-INTEGRITY HMAC-SHA1 值，输出 20 字节（160 位）哈希到 `sha` 缓冲区。

- **关键逻辑** -- HMAC 输入构造:

  HMAC 的输入数据按以下三段精确构造（非简单地 HMAC 整个消息缓冲区）：

  ```
  HMAC 输入:
  msg[0..1] (Type) + fakelen (替换后的 Length) + msg[4..len-25] (从偏移 4 到 M-I 属性之前)
  ```

  具体步骤：
  1. **替换 Length 字段**: `fakelen = htons(msg_len)` -- 使用 `msg_len` 参数（调用方传入的原始消息体长度，即不包含 M-I 属性的长度），而非消息中实际存储的 Message Length
  2. **分段哈希**: 先哈希 Type 字段（2 字节），再哈希替换后的 Length（2 字节，网络字节序），最后哈希剩余部分 `msg + 4`（跳过 Type + Length）
  3. **排除 M-I 属性自身**: M-I 属性（24 字节）本身不参与 HMAC 计算
  4. **RFC 3489 填充**: 如果 `padding == TRUE` 且 `(len - 24) % 64 != 0`，则填充零字节至 64 字节对齐

- **多平台实现**:
  - **Win32 CryptoAPI**: 使用 `CryptAcquireContextW` → `CryptImportKey` → `CryptCreateHash(CALG_HMAC, ...)` → `CryptSetHashParam(HP_HMAC_INFO, CALG_SHA1)` → 分段 `CryptHashData` → `CryptGetHashParam(HP_HASHVAL)`
  - **OpenSSL**: 使用 `HMAC_Init_ex(EVP_sha1())` → 分段 `HMAC_Update` → `HMAC_Final`。兼容 OpenSSL 1.1.0 前后 API 变更
  - **GnuTLS**: 使用 `gnutls_hmac_init(GNUTLS_MAC_SHA1)` → 分段 `gnutls_hmac` → `gnutls_hmac_deinit`

##### stun_hash_creds() -- 长期凭证密钥派生

- **原型**:
  ```c
  void stun_hash_creds (const uint8_t *realm, size_t realm_len,
      const uint8_t *username, size_t username_len,
      const uint8_t *password, size_t password_len,
      unsigned char md5[16]);
  ```

- **作用**: 计算长期凭证（long-term credential）的 HMAC 密钥。STUN 长期凭证使用 SIP 的 H(A1) 派生方式：`MD5(username:realm:password)`。输出 16 字节（128 位）到 `md5` 参数。

- **关键逻辑**:
  1. **去除引号**: 调用内部的 `priv_trim_var()` 对 `username`、`realm`、`password` 分别去除首尾的双引号 `"` 和尾部 null 字符
  2. **构造 MD5 输入**: `username + ":" + realm + ":" + password`（修剪后的值）
  3. **计算 MD5**: 通过多平台后端计算 16 字节 MD5 哈希

##### stun_make_transid() -- 生成事务 ID

- **原型**:
  ```c
  void stun_make_transid (StunTransactionId id);
  ```

- **作用**: 生成 16 字节（128 位）密码学安全的随机事务 ID。封装 `nice_RAND_nonce(id, 16)`。

#### 1.2 HMAC 验证流程

**发送端（finish_message）**:
```
1. 在消息末尾追加 MESSAGE-INTEGRITY 属性（4 字节 TLV 头 + 20 字节占位空间）
2. 确定 HMAC 输入参数：msg, len, msg_len, key, padding
3. 调用 stun_sha1() 计算 20 字节 HMAC 值
4. 将 HMAC 值填入 M-I 属性的值区域
```

**接收端（validate）**:
```
1. 从消息中提取 MESSAGE-INTEGRITY 属性的 20 字节 HMAC 值
2. 使用相同的 msg、len、msg_len、key、padding 参数调用 stun_sha1()
3. 将计算结果与消息中的 HMAC 值进行 memcmp(20) 比较
4. 匹配 → 验证通过；不匹配 → 返回 STUN_VALIDATION_UNAUTHORIZED
```

**短凭证 vs 长凭证的 HMAC 密钥来源差异**:
- **短期凭证**（ICE 场景）：密钥 = SASLprep(password)，直接使用 `msg->key`（长度可变）
- **长期凭证**（TURN 场景）：密钥 = MD5(username:realm:SASLprep(password))，固定 16 字节，通过 `stun_hash_creds()` 计算

---

### 2. 文件: stuncrc32.h / stuncrc32.c (221 行)

#### 概述

`stuncrc32.c` 实现了标准 CRC32 校验计算（多项式 0xEDB88320），源自 FreeBSD 内核的 `src/sys/libkern/crc32.c`。用于 STUN FINGERPRINT 属性（RFC 5389）的计算和验证。

#### 2.1 核心数据结构

```c
typedef struct {
  uint8_t *buf;    // 数据片段指针
  size_t len;      // 数据片段长度
} crc_data;
```

`stun_crc32()` 支持向量化 I/O 风格的 CRC32 计算，通过 `crc_data` 数组将多个不连续的缓冲区片段组合为单个 CRC32 输入。

#### 2.2 核心函数

##### stun_crc32()

- **原型**:
  ```c
  uint32_t stun_crc32 (const crc_data *data, size_t n, bool wlm2009_stupid_crc32_typo);
  ```

- **作用**: 对多个数据片段计算 CRC32 校验值。

- **关键逻辑**:
  1. **初始化 CRC 寄存器**: `crc = 0xffffffff`
  2. **遍历数据片段**: 逐字节处理，使用 256 条目的预计算 CRC32 查找表
  3. **查表算法**: `lkp = crc32_tab[(crc ^ *p++) & 0xFF]` → `crc = lkp ^ (crc >> 8)`
  4. **WLM2009 兼容修正**: 如果 `wlm2009_stupid_crc32_typo == TRUE`，当查表结果 `lkp == 0x8bbeb8ea` 时，将 `lkp` 替换为 `0x8bbe8ea`（Windows Live Messenger 2009 的 CRC32 实现错误）
  5. **最终 XOR**: `return crc ^ 0xffffffff`

- **CRC32 多项式**: 0xEDB88320（反射形式），即 `X^32 + X^26 + X^23 + X^22 + X^16 + X^12 + X^11 + X^10 + X^8 + X^7 + X^5 + X^4 + X^2 + X^1 + X^0`

#### 2.3 FINGERPRINT 计算流程

FINGERPRINT 的计算在 `stun_fingerprint()`（`stun5389.c`）中实现，它调用 `stun_crc32()` 并做 XOR 转换：

```
原始 STUN 消息 buffer (总长度 len，含 FINGERPRINT 属性):
  Type(2B) + Len(2B) + msg[4..len-9] (Magic Cookie + TID + 其他属性) + FPR-TLV(8B)

CRC32 输入 (三个 crc_data 段):
  段1: msg[0..1]     = Type 字段 (2 bytes)
  段2: &fakelen       = htons(len - 20)，即消息体长度(不含头部)
  段3: msg[4..len-9]  = Magic Cookie + TID + 其他属性

最终结果: htonl(CRC32(段1+段2+段3) ^ 0x5354554E)
```

**关键细节**:
1. **替换 Length 字段**: 段 2 使用 `htons(len - 20)` 而不是消息中的实际 Length
2. **排除 FINGERPRINT 自身**: 段 3 只取 `len - 12` 字节（跳过前 4 字节 Type+Length，跳过最后 8 字节 FINGERPRINT 属性）
3. **XOR 0x5354554E**: ASCII 字符串 "STUN" 作为 32 位大端整数的值
4. **网络字节序**: 最终结果通过 `htonl()` 转换为网络字节序写入线格式

---

## 第四部分：STUN 用法层（Usages）

### 概述

`usages/` 目录是 STUN 协议栈的最顶层，在 STUN 消息格式层和事务代理层的基础上，实现了四种具体的 STUN 协议应用场景：

- **bind** (Binding Discovery)：NAT 发现和 Server Reflexive 候选获取（RFC 5389 Binding Usage）
- **ice** (ICE Connectivity Check)：ICE 连接检查（RFC 5245 / RFC 8445）
- **turn** (TURN Allocation)：TURN 中继分配、权限管理和刷新（RFC 5766）
- **timer** (STUN Retransmission Timer)：STUN 事务重传定时器（RFC 5389 重传机制）

四个子模块共计约 **2578 行** C 代码（`.c` + `.h`），直接调用 `stunagent.h` 和 `stunmessage.h` 的 API。

---

### 1. bind.c / bind.h (760 行) -- STUN Binding Usage

#### 概述

`bind.c` 实现了 STUN Binding 请求的构造、响应解析和同步阻塞模式绑定发现。核心用途是向 STUN 服务器发送 Binding Request，从响应中提取 Server Reflexive 地址（NAT 映射后的公网地址）。支持两种使用模式：
- **非阻塞模式**（`stun_usage_bind_create()` + `stun_usage_bind_process()`）：由上层负责网络 I/O 和重传
- **阻塞模式**（`stun_usage_bind_run()`）：一站式同步绑定请求，内部管理 socket、发送、接收、重传

#### 数据结构

**StunUsageBindReturn** -- 绑定操作的返回状态：
| 枚举值 | 含义 |
|--------|------|
| `STUN_USAGE_BIND_RETURN_SUCCESS` | 绑定成功，提取到了映射地址 |
| `STUN_USAGE_BIND_RETURN_ERROR` | 处理出错 |
| `STUN_USAGE_BIND_RETURN_INVALID` | 消息无效，应忽略 |
| `STUN_USAGE_BIND_RETURN_ALTERNATE_SERVER` | 服务器请求使用备用服务器 |
| `STUN_USAGE_BIND_RETURN_TIMEOUT` | 绑定超时 |

#### 核心函数

##### stun_usage_bind_create() -- 构造 Binding 请求

- **原型**:
  ```c
  size_t stun_usage_bind_create (StunAgent *agent, StunMessage *msg,
      uint8_t *buffer, size_t buffer_len);
  ```

- **作用**: 构造一个标准 STUN Binding Request（非阻塞模式使用）。

- **关键逻辑**:
  1. 调用 `stun_agent_init_request(agent, msg, buffer, buffer_len, STUN_BINDING)` 初始化 Binding Request 消息
  2. 调用 `stun_agent_finish_message(agent, msg, NULL, 0)` 完成消息（无认证，Key 为 NULL）
  3. 返回消息总长度

- **注意**: Binding 请求不带任何额外属性（无 USERNAME、无 MESSAGE-INTEGRITY），大部分公共 STUN 服务器接受无认证的 Binding 请求。

##### stun_usage_bind_process() -- 处理 Binding 响应

- **原型**:
  ```c
  StunUsageBindReturn stun_usage_bind_process (StunMessage *msg,
      struct sockaddr *addr, socklen_t *addrlen,
      struct sockaddr *alternate_server, socklen_t *alternate_server_len);
  ```

- **作用**: 解析 STUN Binding 响应消息，提取映射地址和备用服务器地址（如果有）。

- **关键逻辑**:
  1. **方法验证**: 消息方法必须为 `STUN_BINDING`，否则返回 `INVALID`
  2. **类别分支**:
     - `STUN_REQUEST` / `STUN_INDICATION`: 不是有效的 Binding 响应 → 返回 `INVALID`
     - `STUN_RESPONSE`: 成功响应，进入地址提取逻辑
     - `STUN_ERROR`:
       - 提取 ERROR-CODE
       - 如果是 3xx 类错误：检查 `ALTERNATE-SERVER` 属性（存在则返回 `ALTERNATE_SERVER`，缺失则返回 `ERROR`）
       - 其他错误码：返回 `ERROR`
  3. **地址提取优先级**（RFC 5389 标准顺序）:
     - 优先：`XOR-MAPPED-ADDRESS`（通过 `stun_message_find_xor_addr()` 提取）
     - 回退：`MAPPED-ADDRESS`（通过 `stun_message_find_addr()` 提取）
     - 都缺失：返回 `ERROR`
  4. **成功**: 返回 `SUCCESS`，addr 中填充了映射后的公网地址

##### stun_usage_bind_keepalive() -- 构造保活指示

- **原型**:
  ```c
  size_t stun_usage_bind_keepalive (StunAgent *agent, StunMessage *msg,
      uint8_t *buf, size_t len);
  ```

- **作用**: 构造一个 STUN Binding Indication（用于 NAT 保活）。

- **关键逻辑**:
  1. 调用 `stun_agent_init_indication(agent, msg, buf, len, STUN_BINDING)` 初始化 Binding Indication
  2. 调用 `stun_agent_finish_message(agent, msg, NULL, 0)` 完成消息
  3. 返回消息长度

- **关键区别 vs Binding Request**: Indication 消息不需要服务器响应，仅用于刷新 NAT 绑定（keep-alive），开销更低。适合在媒体流传输期间定期发送以维持 NAT 映射。

##### stun_usage_bind_run() -- 同步绑定请求

- **原型**:
  ```c
  StunUsageBindReturn stun_usage_bind_run (const struct sockaddr *srv,
      socklen_t srvlen, struct sockaddr_storage *addr, socklen_t *addrlen);
  ```

- **作用**: 一站式阻塞同步 Binding 请求。内部完成 socket 创建、消息发送、定时器管理、接收和重传的完整流程。

- **关键逻辑**（按代码执行顺序）:
  1. **初始化 agent**: `stun_agent_init(&agent, STUN_ALL_KNOWN_ATTRIBUTES, STUN_COMPATIBILITY_RFC3489, 0)` -- 使用 RFC 3489 兼容模式（不带 Cookie），不使用任何标志位
  2. **创建 Binding 请求**: 调用 `stun_usage_bind_create()`
  3. **创建 UDP socket**: 调用 `stun_trans_create(SOCK_DGRAM)` -- 创建非阻塞 UDP socket，连接或绑定到目标服务器
  4. **发送请求**: 调用 `stun_trans_send()` 发送消息
  5. **启动定时器**: `stun_timer_start(&timer, STUN_TIMER_DEFAULT_TIMEOUT, STUN_TIMER_DEFAULT_MAX_RETRANSMISSIONS)` -- 初始超时 500ms，最多 3 次重传
  6. **接收-重传循环**:
     - 调用 `stun_timer_remainder()` 获取剩余超时时间
     - 调用 `stun_trans_poll()` 使用 poll() 等待数据或超时
     - 如果 poll 超时：
       - 调用 `stun_timer_refresh()` 检查是否需要重传还是已彻底超时
       - 如果需要重传：重新 `stun_trans_send()` 发送请求
       - 如果彻底超时：返回 `TIMEOUT`
     - 如果收到了数据：
       - 调用 `stun_agent_validate()` 验证消息
       - 调用 `stun_usage_bind_process()` 解析响应
       - **ALTERNATE-SERVER 处理**: 如果服务器要求切换到备用服务器，断开当前连接，重新创建 socket 连接到备用服务器，重新发送请求，重新启动定时器
  7. **清理**: 关闭 socket，释放资源
  8. **返回**: `SUCCESS`（获得映射地址）、`ERROR`、`TIMEOUT`

#### 内部辅助结构

`bind.c` 内部定义了一个完整的非阻塞传输抽象层（`StunTransport`）来支持 `stun_usage_bind_run()`：

- **`StunTransport`**: 封装 socket fd、目标地址和所有权标志
- **`stun_trans_create()`**: 创建非阻塞 socket（支持 `FD_CLOEXEC`、`O_NONBLOCK`、Linux `MSG_ERRQUEUE` / `IP_RECVERR` / `IPV6_RECVERR`）
- **`stun_trans_sendto()` / `stun_trans_recvfrom()`**: 带超时和信号抑制的 sendto/recvfrom 封装（使用 `MSG_DONTWAIT | MSG_NOSIGNAL`）
- **`stun_trans_poll()`**: 基于 poll() 的等待封装

---

### 2. ice.c / ice.h (653 行) -- ICE Connectivity Check Usage

#### 概述

`ice.c` 实现了 ICE（Interactive Connectivity Establishment）中连接检查的 STUN 消息构造和解析。这是 ICE 协议最核心的 STUN 用法，负责创建和解析用于连接检查的 STUN Binding 请求/响应，处理角色冲突（Role Conflict），以及提取优先级（PRIORITY）和提名标志（USE-CANDIDATE）。

支持五种兼容模式：
| 枚举值 | 说明 |
|--------|------|
| `STUN_USAGE_ICE_COMPATIBILITY_RFC5245` | 标准 ICE（RFC 5245） |
| `STUN_USAGE_ICE_COMPATIBILITY_GOOGLE` | Google 的 ICE 实现 |
| `STUN_USAGE_ICE_COMPATIBILITY_MSN` | MSN 的 ICE 实现 |
| `STUN_USAGE_ICE_COMPATIBILITY_MSICE2` | Microsoft MS-ICE2 规范 |
| `STUN_USAGE_ICE_COMPATIBILITY_DRAFT19` / `WLM2009` | 已弃用的别名 |

#### ICE STUN 扩展属性

ICE 扩展了 STUN 协议，定义了以下 ICE 特有的属性：

| 属性 | 类型 | 含义 |
|------|------|------|
| `PRIORITY` (0x0024) | 32 位整数 | 候选对的优先级值（RFC 5245 公式计算） |
| `USE-CANDIDATE` (0x0025) | Flag (零长度) | 提名标志：表示该候选对已被提名 |
| `ICE-CONTROLLING` (0x802A) | 64 位整数 | 控制方 tie-breaker 随机值 |
| `ICE-CONTROLLED` (0x8029) | 64 位整数 | 被控方 tie-breaker 随机值 |
| `CANDIDATE-IDENTIFIER` (0x8054) | 字节串 | MS-ICE2：候选的 foundation 标识符 |
| `MS_IMPLEMENTATION_VERSION` (0x8070) | 32 位整数 | MS-ICE2：协议实现版本号 |

**角色判定**：控制方（controlling agent）发送 `ICE-CONTROLLING` 属性，被控方（controlled agent）发送 `ICE-CONTROLLED` 属性。两个角色各自生成一个 64 位随机 tie-breaker 值。当双方角色一致时（都认为自己是被控方或都认为自己是控制方），需要根据 tie-breaker 值解决冲突。

#### 数据结构

**StunUsageIceReturn** -- ICE 连接检查操作的返回状态：
| 枚举值 | 含义 |
|--------|------|
| `STUN_USAGE_ICE_RETURN_SUCCESS` | 操作成功 |
| `STUN_USAGE_ICE_RETURN_ERROR` | 未指定错误 |
| `STUN_USAGE_ICE_RETURN_INVALID` | 消息无效 |
| `STUN_USAGE_ICE_RETURN_ROLE_CONFLICT` | 检测到角色冲突 |
| `STUN_USAGE_ICE_RETURN_INVALID_REQUEST` | 消息不是 Request |
| `STUN_USAGE_ICE_RETURN_INVALID_METHOD` | 请求方法无效 |
| `STUN_USAGE_ICE_RETURN_MEMORY_ERROR` | 缓冲区过小 |
| `STUN_USAGE_ICE_RETURN_INVALID_ADDRESS` | 地址族无效 |
| `STUN_USAGE_ICE_RETURN_NO_MAPPED_ADDRESS` | 响应中没有 MAPPED-ADDRESS |

#### 核心函数

##### stun_usage_ice_conncheck_create() -- 构造 ICE 连接检查请求

- **原型**:
  ```c
  size_t stun_usage_ice_conncheck_create (StunAgent *agent, StunMessage *msg,
      uint8_t *buffer, size_t buffer_len,
      const uint8_t *username, const size_t username_len,
      const uint8_t *password, const size_t password_len,
      bool cand_use, bool controlling, uint32_t priority,
      uint64_t tie, const char *candidate_identifier,
      StunUsageIceCompatibility compatibility);
  ```

- **作用**: 构造一个 ICE 连接检查 STUN Binding Request。根据兼容模式决定添加哪些属性。

- **关键逻辑**（按属性追加顺序）:
  1. 调用 `stun_agent_init_request(agent, msg, buffer, buffer_len, STUN_BINDING)`
  2. **RFC5245 / MSICE2 兼容模式**（标准 ICE）:
     - 如果 `cand_use == TRUE`: 追加 `USE-CANDIDATE` flag 属性（零长度）
     - 追加 `PRIORITY` 属性（32 位，候选对优先级）
     - 如果 `controlling == TRUE`: 追加 `ICE-CONTROLLING` 属性（64 位 tie-breaker）
     - 否则: 追加 `ICE-CONTROLLED` 属性（64 位 tie-breaker）
  3. 如果 username 非空且长度 > 0: 追加 `USERNAME` 属性（用于 ICE 短凭证认证）
  4. **MSICE2 兼容模式**: 如果提供了 `candidate_identifier`:
     - 追加 `CANDIDATE-IDENTIFIER` 属性（含 4 字节对齐填充）
     - 追加 `MS_IMPLEMENTATION_VERSION` 属性（值 = 2）
  5. 调用 `stun_agent_finish_message(agent, msg, password, password_len)` 完成消息（添加 MESSAGE-INTEGRITY 和 FINGERPRINT）

- **注意**: GOOGLE 和 MSN 兼容模式下不添加 PRIORITY、USE-CANDIDATE、ICE-CONTROLLING/CONTROLLED 属性（它们使用不同的机制）。

##### stun_usage_ice_conncheck_process() -- 处理 ICE 连接检查响应

- **原型**:
  ```c
  StunUsageIceReturn stun_usage_ice_conncheck_process (StunMessage *msg,
      struct sockaddr_storage *addr, socklen_t *addrlen,
      StunUsageIceCompatibility compatibility);
  ```

- **作用**: 解析 ICE 连接检查的 STUN 响应，提取映射地址。同时处理角色冲突错误。

- **关键逻辑**:
  1. **方法验证**: 方法必须为 `STUN_BINDING`，否则返回 `INVALID`
  2. **类别分支**:
     - `STUN_REQUEST` / `STUN_INDICATION`: 返回 `INVALID`
     - `STUN_RESPONSE`: 成功，进入地址提取
     - `STUN_ERROR`: 提取错误码，如果是 `STUN_ERROR_ROLE_CONFLICT` (487)：返回 `ROLE_CONFLICT`，否则返回 `ERROR`
  3. **地址提取**（根据兼容模式）:
     - **MSN 模式**: 使用 `stun_message_find_xor_addr_full()` 配合 Transaction ID 的前 4 字节作为 XOR Cookie（MSN 使用不同的 Magic Cookie）
     - **其他模式**: 标准 `stun_message_find_xor_addr()` → 回退到 `stun_message_find_addr()` → 都没有返回 `NO_MAPPED_ADDRESS`
  4. 返回 `SUCCESS`

##### stun_usage_ice_conncheck_create_reply() -- 构造 ICE 连接检查响应

- **原型**:
  ```c
  StunUsageIceReturn stun_usage_ice_conncheck_create_reply (StunAgent *agent,
      StunMessage *req, StunMessage *msg, uint8_t *buf, size_t *plen,
      const struct sockaddr_storage *src, socklen_t srclen,
      bool *control, uint64_t tie,
      StunUsageIceCompatibility compatibility);
  ```

- **作用**: 为收到的 ICE 连接检查请求构造对应的 STUN 响应。这是 ICE 连接检查中最复杂的函数，负责角色冲突解决和响应构造。

- **关键逻辑**（按执行顺序）:

  1. **验证请求**:
     - 类别必须为 `STUN_REQUEST`，否则返回 `INVALID_REQUEST`
     - 方法必须为 `STUN_BINDING`，否则返回 `INVALID_METHOD`（并构造 400 Bad Request 错误响应）

  2. **角色冲突检测和处理**（ICE RFC 5245 Section 7.2.1.1）:
     - 检查请求中是否包含与本方预期相反的 `ICE-CONTROLLING` 或 `ICE-CONTROLLED` 属性
     - **四种情况**:
       - 本方是 controlling，收到了 ICE-CONTROLLING → 角色冲突
       - 本方是 controlled，收到了 ICE-CONTROLLED → 角色冲突
     - **解决方案**（基于 tie-breaker 值比较）:
       - `tie < q && controlling` 或 `tie >= q && !controlling`: **切换本地角色**（本地让步），返回 `ROLE_CONFLICT` 给上层（上层会使用新角色重新启动连接检查）
       - `tie >= q && controlling` 或 `tie < q && !controlling`: **保持角色**，发送 487 Role Conflict 错误响应，返回 `ROLE_CONFLICT`

  3. **构造成功响应**:
     - 调用 `stun_agent_init_response()` 初始化响应（继承请求的认证密钥）
     - **追加 MAPPED-ADDRESS**（根据兼容模式）:
       - MSN 模式: `stun_message_append_xor_addr_full()` 配合 Magic Cookie (Transaction ID 前 4 字节)
       - RFC5389 Cookie 存在且非 GOOGLE 模式: `stun_message_append_xor_addr()` (XOR-MAPPED-ADDRESS)
       - 否则: `stun_message_append_addr()` (MAPPED-ADDRESS)
     - **回显 USERNAME**: 从请求中提取 USERNAME 属性并回显到响应中
     - **MSICE2**: 追加 `MS_IMPLEMENTATION_VERSION` (值 = 2)
     - 调用 `stun_agent_finish_message()` 完成响应（使用请求的密码自动添加 M-I）

  4. **错误处理**: 如果任何步骤失败，根据错误类型返回 `MEMORY_ERROR`、`INVALID_ADDRESS` 或 `ERROR`

- **重要**: 函数内部维护了一个 `#define err(code)` 宏来构造错误响应并输出长度。如果返回错误，`plen` 输出的是错误响应的长度（而不是 0）。

##### stun_usage_ice_conncheck_priority() -- 提取优先级

- **原型**:
  ```c
  uint32_t stun_usage_ice_conncheck_priority (const StunMessage *msg);
  ```

- **作用**: 从 STUN 消息中提取 `PRIORITY` 属性的 32 位值（host byte order）。

- **关键逻辑**: 调用 `stun_message_find32(msg, STUN_ATTRIBUTE_PRIORITY, &value)`，返回 0 表示未找到。

##### stun_usage_ice_conncheck_use_candidate() -- 检查提名标志

- **原型**:
  ```c
  bool stun_usage_ice_conncheck_use_candidate (const StunMessage *msg);
  ```

- **作用**: 检查 STUN 消息中是否包含 `USE-CANDIDATE` flag 属性。

- **关键逻辑**: 调用 `stun_message_find_flag(msg, STUN_ATTRIBUTE_USE_CANDIDATE)`，返回 TRUE 表示消息中包含此 flag。

**USE-CANDIDATE 在 ICE 中的含义**: 当 controlling agent 决定"提名"(nominate)某个候选对时，它在连接检查请求中设置 USE-CANDIDATE flag。当 controlled agent 收到带有此 flag 的请求时，表示该候选对被提名，连接检查一旦成功，即可以用于媒体传输。

---

### 3. turn.c / turn.h (756 行) -- TURN Usage

#### 概述

`turn.c` 实现了 TURN (Traversal Using Relays around NAT, RFC 5766) 协议的 STUN 消息构造和解析。核心用途是向 TURN 服务器请求中继分配（Allocate）、刷新分配（Refresh）、创建权限（CreatePermission），以及解析 TURN 响应（提取中继地址、映射地址、生存时间、带宽等）。

支持五种兼容模式：

| 枚举值 | 说明 |
|--------|------|
| `STUN_USAGE_TURN_COMPATIBILITY_DRAFT9` | TURN Draft 09 (pre-RFC) |
| `STUN_USAGE_TURN_COMPATIBILITY_GOOGLE` | Google Talk 中继服务器 |
| `STUN_USAGE_TURN_COMPATIBILITY_MSN` | MSN TURN 服务器 |
| `STUN_USAGE_TURN_COMPATIBILITY_OC2007` | Microsoft Office Communicator 2007 |
| `STUN_USAGE_TURN_COMPATIBILITY_RFC5766` | 最终标准 RFC 5766 |

#### TURN STUN 方法和属性

TURN 使用以下 STUN 方法：

| 方法 | 值 | 类别 | 用途 |
|------|-----|------|------|
| `STUN_ALLOCATE` | 0x003 | Request | 分配中继地址 |
| `STUN_REFRESH` | 0x004 | Request | 刷新已有分配 |
| `STUN_CREATEPERMISSION` | 0x008 | Request | 创建对端权限 |
| `STUN_CHANNELBIND` | 0x009 | Request | 信道绑定 |
| `STUN_IND_SEND` | 0x006 | Indication | 通过中继发送数据 |
| `STUN_IND_DATA` | 0x007 | Indication | 从中继接收数据 |

TURN 特有的 STUN 属性：

| 属性 | 值 | 说明 |
|------|-----|------|
| `REQUESTED-TRANSPORT` | 0x0019 | 请求的传输协议 (UDP=0x11000000) |
| `LIFETIME` | 0x000D | 分配的生存时间（秒） |
| `XOR-RELAYED-ADDRESS` | 0x0016 | XOR 混淆的中继地址 |
| `XOR-MAPPED-ADDRESS` | 0x0020 | XOR 混淆的映射地址 |
| `XOR-PEER-ADDRESS` | 0x0012 | XOR 混淆的对端地址（CreatePermission） |
| `BANDWIDTH` | 0x0010 | 请求/分配的带宽 |
| `REQUESTED-PORT-PROPS` | 0x0018 | 请求的端口属性（偶数端口/端口保留） |
| `RESERVATION-TOKEN` | 0x0022 | 端口保留令牌 |
| `REALM` / `NONCE` | 0x0014 / 0x0015 | 长期凭证认证域和随机数 |
| `ALTERNATE-SERVER` | 0x8023 | 备用服务器地址 |
| `MAGIC-COOKIE` | 0x000F | TURN Magic Cookie（Google/MSN 兼容） |
| `MS-VERSION` | 0x8008 | MS-TURN 版本（OC2007 兼容） |
| `MS-ALTERNATE-SERVER` | 0x000E | MS-TURN 备用服务器（OC2007） |
| `MS-XOR-MAPPED-ADDRESS` | 0x8020 | MS-TURN XOR 映射地址（OC2007） |
| `MS-SEQUENCE-NUMBER` | 0x8050 | MS-TURN 序列号 |

内部常量：
- `REQUESTED_PROPS_E = 0x80000000`: 请求偶数端口
- `REQUESTED_PROPS_R = 0x40000000`: 保留下一个端口
- `REQUESTED_PROPS_P = 0x20000000`: 端口保留位
- `TURN_REQUESTED_TRANSPORT_UDP = 0x11000000`: 请求 UDP 传输

#### 数据结构

**StunUsageTurnRequestPorts** -- 端口请求模式：
| 枚举值 | 含义 |
|--------|------|
| `STUN_USAGE_TURN_REQUEST_PORT_NORMAL` | 请求普通端口 |
| `STUN_USAGE_TURN_REQUEST_PORT_EVEN` | 请求偶数端口 |
| `STUN_USAGE_TURN_REQUEST_PORT_EVEN_AND_RESERVE` | 请求偶数端口并保留下一端口 |

**StunUsageTurnReturn** -- TURN 操作的返回状态：
| 枚举值 | 含义 |
|--------|------|
| `STUN_USAGE_TURN_RETURN_RELAY_SUCCESS` | 成功获得中继地址 |
| `STUN_USAGE_TURN_RETURN_MAPPED_SUCCESS` | 成功获得中继地址和映射地址 |
| `STUN_USAGE_TURN_RETURN_ERROR` | 服务器返回错误 |
| `STUN_USAGE_TURN_RETURN_INVALID` | 消息无效 |
| `STUN_USAGE_TURN_RETURN_ALTERNATE_SERVER` | 服务器请求切换到备用服务器 |

#### 核心函数

##### stun_usage_turn_create() -- 构造 TURN Allocate 请求

- **原型**:
  ```c
  size_t stun_usage_turn_create (StunAgent *agent, StunMessage *msg,
      uint8_t *buffer, size_t buffer_len,
      StunMessage *previous_response,
      StunUsageTurnRequestPorts request_ports,
      int32_t bandwidth, int32_t lifetime,
      uint8_t *username, size_t username_len,
      uint8_t *password, size_t password_len,
      StunUsageTurnCompatibility compatibility);
  ```

- **作用**: 构造一个 TURN Allocate Request。根据兼容模式添加不同的属性集合。支持首次请求和重试请求（通过 `previous_response` 传入之前的 401 响应以获取 REALM/NONCE）。

- **关键逻辑**（按属性追加顺序）:

  1. 调用 `stun_agent_init_request(agent, msg, buffer, buffer_len, STUN_ALLOCATE)`

  2. **传输方式**（DRAFT9 / RFC5766）:
     - 追加 `REQUESTED-TRANSPORT` 属性：值 `TURN_REQUESTED_TRANSPORT_UDP` (0x11000000)
     - 如果 `bandwidth >= 0`: 追加 `BANDWIDTH` 属性
     - **Google / MSN 兼容**: 改为追加 `MAGIC-COOKIE` 属性（值 `TURN_MAGIC_COOKIE`）

  3. **OC2007 模式**: 追加 `MS-VERSION` 属性（值 = 1）

  4. **生存时间**: 如果 `lifetime >= 0`: 追加 `LIFETIME` 属性

  5. **端口属性**（DRAFT9 / RFC5766 + 非 NORMAL 模式）:
     - 根据 `request_ports` 设置端口属性位掩码
     - 追加 `REQUESTED-PORT-PROPS` 属性

  6. **重试/认证属性**（如果有 `previous_response`）:
     - 从之前的响应中提取 `REALM`、`NONCE`、`RESERVATION-TOKEN`
     - 将这些属性追加到新请求中

  7. **用户名**: 如果 username 非空且（使用短期凭证 或 有 previous_response）:
     - 追加 `USERNAME` 属性

  8. 调用 `stun_agent_finish_message(agent, msg, password, password_len)` 完成消息

##### stun_usage_turn_create_refresh() -- 构造 TURN Refresh 请求

- **原型**:
  ```c
  size_t stun_usage_turn_create_refresh (StunAgent *agent, StunMessage *msg,
      uint8_t *buffer, size_t buffer_len,
      StunMessage *previous_response, int32_t lifetime,
      uint8_t *username, size_t username_len,
      uint8_t *password, size_t password_len,
      StunUsageTurnCompatibility compatibility);
  ```

- **作用**: 构造 TURN Refresh Request（用于刷新已有的分配，延长其生存时间）。

- **关键逻辑**:
  - **DRAFT9 / RFC5766 模式**: 使用 `STUN_REFRESH` 方法，只添加 LIFETIME 和认证属性
  - **其他兼容模式**: 退化到 `stun_usage_turn_create()`（使用 `STUN_ALLOCATE` 方法模拟刷新，因为这些旧版协议没有独立的 Refresh 方法）
  - 从 `previous_response` 中提取 REALM 和 NONCE（长期凭证认证用）

##### stun_usage_turn_create_permission() -- 构造 CreatePermission 请求

- **原型**:
  ```c
  size_t stun_usage_turn_create_permission (StunAgent *agent, StunMessage *msg,
      uint8_t *buffer, size_t buffer_len,
      uint8_t *username, size_t username_len,
      uint8_t *password, size_t password_len,
      uint8_t *realm, size_t realm_len,
      uint8_t *nonce, size_t nonce_len,
      struct sockaddr_storage *peer,
      StunUsageTurnCompatibility compatibility);
  ```

- **作用**: 构造 TURN CreatePermission Request（用于授权特定对端通过中继发送数据）。

- **关键逻辑**:
  1. 使用 `STUN_CREATEPERMISSION` 方法
  2. 追加 `XOR-PEER-ADDRESS` 属性（对端的 Server Reflexive 地址）
  3. 追加 NONCE、REALM（长期凭证认证）
  4. 追加 USERNAME（如果有短期凭证或有 NONCE+REALM）
  5. 调用 `stun_agent_finish_message()` 完成消息

##### stun_usage_turn_process() -- 处理 TURN Allocate/Refresh 响应

- **原型**:
  ```c
  StunUsageTurnReturn stun_usage_turn_process (StunMessage *msg,
      struct sockaddr_storage *relay_addr, socklen_t *relay_addrlen,
      struct sockaddr_storage *addr, socklen_t *addrlen,
      struct sockaddr_storage *alternate_server, socklen_t *alternate_server_len,
      uint32_t *bandwidth, uint32_t *lifetime,
      StunUsageTurnCompatibility compatibility);
  ```

- **作用**: 解析 TURN Allocate 响应（或旧版兼容模式下的 Refresh 响应），提取中继地址、映射地址、备用服务器地址、带宽和生存时间。

- **关键逻辑**:

  1. **方法验证**: 方法必须为 `STUN_ALLOCATE`，否则返回 `INVALID`

  2. **类别分支**:
     - `STUN_REQUEST` / `STUN_INDICATION`: 返回 `INVALID`
     - `STUN_RESPONSE`: 成功，进入属性提取
     - `STUN_ERROR`:
       - **OC2007 模式**: 额外检查 `MS-ALTERNATE-SERVER` 属性（MS-TURN 服务器在错误响应中也返回备用服务器）
       - **3xx 错误**: 检查 `ALTERNATE-SERVER` 属性 → 返回 `ALTERNATE_SERVER`
       - 其他: 返回 `ERROR`

  3. **地址提取**（根据兼容模式选择不同策略）:

     | 兼容模式 | 映射地址属性 | 中继地址属性 |
     |----------|-------------|-------------|
     | DRAFT9 / RFC5766 | XOR-MAPPED-ADDRESS | XOR-RELAYED-ADDRESS |
     | GOOGLE | (无) | MAPPED-ADDRESS |
     | MSN | 0x8000 (MSN-MAPPED) | MAPPED-ADDRESS |
     | OC2007 | MS-XOR-MAPPED-ADDRESS (自定义 Cookie) | MAPPED-ADDRESS |

  4. **返回值**:
     - 映射地址提取成功 → `MAPPED_SUCCESS`
     - 仅中继地址提取成功 → `RELAY_SUCCESS`
     - 中继地址缺失 → `ERROR`

  5. **额外提取**: `LIFETIME` 和 `BANDWIDTH` 属性（可选，不存在时不修改输出参数）

##### stun_usage_turn_refresh_process() -- 处理 TURN Refresh 响应

- **原型**:
  ```c
  StunUsageTurnReturn stun_usage_turn_refresh_process (StunMessage *msg,
      uint32_t *lifetime, StunUsageTurnCompatibility compatibility);
  ```

- **作用**: 解析 TURN Refresh 响应，提取新的生存时间。

- **关键逻辑**:
  - **DRAFT9 / RFC5766**: 期望方法为 `STUN_REFRESH`（其他模式期望 `STUN_ALLOCATE`）
  - 从成功响应中提取 `LIFETIME` 属性
  - 返回 `RELAY_SUCCESS`（Refresh 成功不提供新的中继地址，地址保持不变）

---

### 4. timer.c / timer.h (409 行) -- STUN 重传定时器

#### 概述

`timer.c` 实现了 RFC 5389 中 STUN 事务的重传机制（RTO - Retransmission Timeout）。区别于 GLib 主循环驱动的事件系统，这是一个**独立的、基于墙上时钟的定时器**，设计用于同步阻塞模式的 STUN 事务（如 `stun_usage_bind_run()`）。

#### 数据结构

```c
struct stun_timer_s {
  struct timeval deadline;          // 下一次超时的绝对截止时间
  unsigned delay;                   // 当前重传间隔（毫秒）
  unsigned retransmissions;         // 已完成的传输次数（第 n 次，从 1 开始）
  unsigned max_retransmissions;     // 最大重传次数
};
```

**关键字段含义**:
- `deadline`: 绝对时间戳（单调时钟），用于判断当前重传间隔是否到期
- `delay`: 当前 RTO 值（毫秒），每次重传后翻倍
- `retransmissions`: 传输次数（1 = 初始传输，2 = 第一次重传，...）
- `max_retransmissions`: 超出此次数后宣告超时（默认 3，意味着初始 + 3 次重传 = 4 次传输机会）

**默认常量**:
- `STUN_TIMER_DEFAULT_TIMEOUT = 500`: 初始 RTO 为 500ms
- `STUN_TIMER_DEFAULT_MAX_RETRANSMISSIONS = 3`: 最多 3 次重传
- `STUN_TIMER_DEFAULT_RELIABLE_TIMEOUT = 2000`: 可靠传输（TCP）的初始超时

**总超时计算**（RTO 指数退避）:
```
total_timeout = initial_timeout * (2^(max_retransmissions + 1) - 1)
```
以默认参数为例: `500 * (2^4 - 1) = 500 * 15 = 7500ms`

序列: 500 → 1000 → 2000 → 4000 (最后一个是 4000/2 = 2000，见下方特殊规则)

#### 时钟获取

`stun_gettime()` 使用单调时钟避免系统时间跳变的影响：
- **Linux/Unix**: 优先使用 `CLOCK_MONOTONIC`（`clock_gettime`），回退到 `gettimeofday`
- **Windows**: 使用 `GetSystemTimeAsFileTime`（从 1601-01-01 的 100ns 单位转到 Unix epoch）

#### 核心函数

##### stun_timer_start() -- 启动定时器

- **原型**:
  ```c
  void stun_timer_start (StunTimer *timer, unsigned int initial_timeout,
        unsigned int max_retransmissions);
  ```

- **作用**: 为 UDP 传输启动 STUN 重传定时器。应在首次发送消息后立即调用。

- **关键逻辑**:
  1. `retransmissions = 1`（已发送第一次）
  2. `delay = initial_timeout`（初始 RTO）
  3. `max_retransmissions = max_retransmissions`
  4. 调用 `set_delay()` 设置 `deadline = now + delay`

##### stun_timer_start_reliable() -- 启动可靠定时器

- **原型**:
  ```c
  void stun_timer_start_reliable (StunTimer *timer, unsigned int initial_timeout);
  ```

- **作用**: 为 TCP/可靠传输启动定时器。区别是将 `max_retransmissions` 设为 0（TCP 不需要应用层重传）。

- **实现**: `stun_timer_start(timer, initial_timeout, 0)`

##### stun_timer_remainder() -- 查询剩余时间

- **原型**:
  ```c
  unsigned stun_timer_remainder (const StunTimer *timer);
  ```

- **作用**: 返回距离下一次超时的剩余毫秒数。调用方使用此值作为 poll()/select() 的超时参数。

- **关键逻辑**:
  1. 获取当前单调时间 `now`
  2. 如果 `now > deadline`: 返回 0（已过期）
  3. 否则: 计算 `deadline - now`（秒 → 毫秒转换）

##### stun_timer_refresh() -- 刷新定时器状态（核心函数）

- **原型**:
  ```c
  StunUsageTimerReturn stun_timer_refresh (StunTimer *timer);
  ```

- **作用**: 在定时器到期时被调用，更新状态并返回下一步应该做什么。

- **返回值 `StunUsageTimerReturn`**:
  | 值 | 含义 | 下一步操作 |
  |----|------|-----------|
  | `STUN_USAGE_TIMER_RETURN_SUCCESS` | 定时器未到期 | 继续等待 |
  | `STUN_USAGE_TIMER_RETURN_RETRANSMIT` | 需要重传 | 重新发送消息 |
  | `STUN_USAGE_TIMER_RETURN_TIMEOUT` | 事务超时 | 放弃等待 |

- **关键逻辑**（RTO 指数退避 + 特殊尾处理）:

  1. 调用 `stun_timer_remainder()` 检查是否到期
  2. 如果未到期（delay > 0）: 返回 `SUCCESS`（继续等待）
  3. 如果到期（delay == 0）:
     - **检查是否超过最大重传次数**: 若 `retransmissions >= max_retransmissions` → 返回 `TIMEOUT`
     - **计算新 delay（指数退避 + 特殊规则）**:
       - **倒数第二次重传**（`retransmissions == max_retransmissions - 1`）：`delay = delay / 2`（最后一次的延迟不翻倍，而是减半）
       - **其他情况**：`delay = delay * 2`（标准指数退避，每次翻倍）
     - **更新 deadline**: `deadline = now + delay`
     - `retransmissions++`
     - 返回 `RETRANSMIT`

  **特殊规则的原理**：RFC 5389 规定最后一次重传的超时只使用一半的 RTO，因为最后一次重传后不会再收到响应，不需要为后续重传留时间。

**完整的重传时间线（默认参数：initial=500ms, max=3）**:

| 传输次数 | 操作 | delay (ms) | 累计时间 (ms) |
|----------|------|-----------|--------------|
| 1 (初始) | 首次发送 | 500 | 0 |
| 2 (重传1) | timer_refresh → RETRANSMIT | 1000 | 500 |
| 3 (重传2) | timer_refresh → RETRANSMIT | 2000 | 1500 |
| 4 (重传3) | timer_refresh → RETRANSMIT | 2000 (减半: 不算指数) | 3500 |
| 超时 | timer_refresh → TIMEOUT | - | 5500 |

**重要说明**: `stun_timer_refresh()` 的设计模式是"先调用再决定"——调用方在 poll() 超时后调用此函数，根据返回值决定是重传、继续等待还是宣告超时。定时器状态在 `refresh()` 内部自动更新。

---

## stun/ 模块总结

### 文件清单与代码量

| 子模块 | 文件 | 行数 | 职责 |
|--------|------|------|------|
| message | stunmessage.c/h | 1788 | 消息格式与解析 |
| message | constants.h | 203 | 协议常量 |
| message | stun5389.c/h | 183 | RFC 5389 兼容 |
| agent | stunagent.c/h | 1283 | 事务管理 |
| crypto | stunhmac.c/h | 373 | HMAC-SHA1 |
| crypto | stuncrc32.c/h | 221 | CRC32 |
| usages | bind.c/h | 760 | Binding 用法 |
| usages | ice.c/h | 653 | ICE 连接检查 |
| usages | turn.c/h | 756 | TURN 用法 |
| usages | timer.c/h | 409 | 重传定时器 |
| utils | utils.c/h + rand.c/h + debug.c/h | 565 | 工具函数 |
| **总计** | | **~7194** | |

### 模块间调用关系

```
agent/ (ICE Conncheck, Discovery)
  └── usages/ (bind, ice, turn, timer)
        └── stunagent.c (事务管理)
              ├── stunmessage.c (消息构造/解析)
              ├── stunhmac.c (MESSAGE-INTEGRITY)
              ├── stuncrc32.c (FINGERPRINT)
              ├── constants.h
              └── random/ (事务 ID)
```

### 关键设计要点

1. **三层架构**: message 格式 → agent 事务 → usages 应用，每层独立可测试。message 层只关心字节操作，agent 层管理事务状态和认证，usages 层封装协议特定逻辑
2. **直接操作线格式缓冲区**: 无中间数据结构转换层，构造完成即可发送，避免内存拷贝
3. **TLV 属性系统可扩展**: 新增属性通过枚举定义属性 ID + 调用通用的 `stun_message_append()` / `stun_message_find()`，无需修改核心代码
4. **认证机制灵活**: 支持 short-term credential (ICE) 和 long-term credential (TURN)，密钥派生在 agent 层透明处理
5. **兼容模式设计**: 通过 `StunCompatibility` 和用法层的 `StunUsage*Compatibility` 两层兼容模式，支持 RFC 3489/5389、MS-TURN、Google TURN、MS-ICE2 等多种协议变体
6. **重传机制**: timer.c 提供独立的、基于绝对时钟的 RTO 指数退避重传，不依赖 GLib 主循环，可用于同步阻塞模式
7. **跨平台加密**: HMAC-SHA1 和 CRC32 通过条件编译支持 Win32 CryptoAPI / OpenSSL / GnuTLS 三种后端
