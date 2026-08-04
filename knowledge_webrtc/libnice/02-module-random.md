# 02 -- 模块分析：random/

## 概述

`random/` 模块是对 GLib 随机数生成设施（`GRand` / `g_random_int_range`）的轻量封装，为 libnice 提供统一的随机数生成接口。其主要用途是为 STUN 事务 ID、ICE ufrag/password、TURN 凭证等协议字段生成密码学上不需要强随机的随机数据。

**总代码量**：4 个源文件，约 375 行（不含注释和空行）。

**文件列表**：

| 文件 | 行数 | 作用 |
|------|------|------|
| `random.h` | 79 | 公共头文件，定义 `NiceRNG` 虚表结构体及公共 API |
| `random.c` | 132 | 分发层实现，将调用委托给可替换的后端实现 |
| `random-glib.h` | 56 | GLib 后端实现的公共头文件 |
| `random-glib.c` | 108 | 基于 GLib `g_random_*` 的默认 RNG 后端实现 |

**依赖关系**：

- 上游依赖：`glib-2.0`、`gio`、`gthread`
- 被依赖方：`agent/`（通过静态库 `librandom` 间接链接到 `libnice.so`）

---

## 文件: random.h / random.c

### 数据结构

#### NiceRNG

```c
typedef struct _NiceRNG NiceRNG;

struct _NiceRNG {
  void (*seed) (NiceRNG *src, guint32 seed);
  void (*generate_bytes) (NiceRNG *src, guint len, gchar *buf);
  guint (*generate_int) (NiceRNG *src, guint low, guint high);
  void (*free) (NiceRNG *src);
  gpointer priv;
};
```

`NiceRNG` 采用**虚表（vtable）模式**：结构体中直接内嵌四个函数指针和一个私有数据指针。这种设计允许在运行时替换随机数生成算法（例如测试时替换为确定性生成器），而无需修改调用方代码。

字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `seed` | 函数指针 | 设置随机数种子 |
| `generate_bytes` | 函数指针 | 生成指定长度的随机字节序列 |
| `generate_int` | 函数指针 | 生成 `[low, high)` 范围内的随机整数 |
| `free` | 函数指针 | 释放 RNG 实例 |
| `priv` | `gpointer` | 私有数据指针，供后端实现存储状态（当前 GLib 后端未使用） |

### 全局状态

```c
static NiceRNG * (*nice_rng_new_func) (void) = NULL;
```

模块级全局函数指针，保存当前的 RNG 工厂函数。默认为 `NULL`，此时 `nice_rng_new()` 回退到 `nice_rng_glib_new()`。

### 函数分析

#### nice_rng_new()

- **原型**：`NiceRNG * nice_rng_new (void);`
- **作用**：创建新的随机数生成器实例。
- **关键逻辑**：
  1. 检查全局工厂函数指针 `nice_rng_new_func` 是否为 `NULL`。
  2. 若为 `NULL`（默认情况），调用 `nice_rng_glib_new()` 返回 GLib 后端的实例。
  3. 若已通过 `nice_rng_set_new_func()` 设置了自定义工厂，则调用该工厂函数。
  4. 这是一个典型的**策略模式**入口，允许在不修改调用代码的情况下全局切换 RNG 实现。

#### nice_rng_set_new_func()

- **原型**：`void nice_rng_set_new_func (NiceRNG * (*func) (void));`
- **作用**：设置全局的 RNG 工厂函数，替换默认的 GLib 后端。
- **关键逻辑**：直接将传入的函数指针赋值给模块级静态变量 `nice_rng_new_func`。该函数主要在**测试环境**中使用，例如 `random/test.c` 中将其设置为 `nice_rng_glib_new_predictable` 以获得可复现的随机序列，从而进行断言验证。

#### nice_rng_generate_bytes()

- **原型**：`void nice_rng_generate_bytes (NiceRNG *rng, guint len, gchar *buf);`
- **作用**：生成指定长度的随机字节序列，写入调用方提供的缓冲区。
- **关键逻辑**：直接通过虚表调用 `rng->generate_bytes(rng, len, buf)`。在默认的 GLib 后端中，该函数循环调用 `g_random_int_range(0, 256)` 逐字节填充缓冲区（生成 0-255 的字节值）。

#### nice_rng_generate_int()

- **原型**：`guint nice_rng_generate_int (NiceRNG *rng, guint low, guint high);`
- **作用**：生成范围在 `[low, high)` 内的随机无符号整数。
- **关键逻辑**：通过虚表调用 `rng->generate_int(rng, low, high)`。在默认的 GLib 后端中，该函数直接委托给 `g_random_int_range(low, high)`。

#### nice_rng_generate_bytes_print()

- **原型**：`void nice_rng_generate_bytes_print (NiceRNG *rng, guint len, gchar *buf);`
- **作用**：生成仅包含可打印 ASCII 字符（符合 ICE 规范定义的 `ice-char`）的随机字节序列。
- **关键逻辑**：
  1. 定义合法字符集 `chars`，包含：
     - `A-Z` (0x41-0x5A)
     - `a-z` (0x61-0x7A)
     - `0-9` (0x30-0x39)
     - `+` (0x2B) 和 `/` (0x2F)
  2. 循环 `len` 次，每次调用 `nice_rng_generate_int(rng, 0, strlen(chars))` 获取随机索引，从 `chars` 中选取字符写入 `buf`。
  3. 这符合 **ICE 规范（RFC 8445, Section 15.1）** 中对 `ice-char` 的定义。生成的输出可用于 ufrag 和 password。

#### nice_rng_seed()

- **原型**：`void nice_rng_seed (NiceRNG *rng, guint32 seed);`
- **作用**：设置随机数种子。
- **关键逻辑**：通过虚表调用 `rng->seed(rng, seed)`。**注意**：该函数在 `random.h` 中声明，但在 `random.c` 中**没有对应的分发实现**（没有函数体）。当前通过 `nice_rng_glib_new_predictable()` 间接使用此接口。

#### nice_rng_free()

- **原型**：`void nice_rng_free (NiceRNG *rng);`
- **作用**：释放随机数生成器实例。
- **关键逻辑**：通过虚表调用 `rng->free(rng)`。在默认的 GLib 后端中，该函数使用 `g_slice_free(NiceRNG, rng)` 释放通过 `g_slice_new0` 分配的内存。

---

## 文件: random-glib.h / random-glib.c

### 概述

`random-glib.c` 是 `NiceRNG` 虚表的**默认后端实现**，内部完全基于 GLib 的全局随机数 API（`g_random_int_range`、`g_random_set_seed`）。它不是基于 `GRand` 实例（每个 GRand 有独立状态），而是使用 GLib 的进程级全局随机状态。这意味着所有通过此后端创建的 `NiceRNG` 实例实际上共享同一个随机数生成器进程状态。

`NiceRNG` 结构体在此后端中通过 `g_slice_new0` 分配（零初始化），所有函数指针指向 `static` 函数。

### 函数分析

#### nice_rng_glib_new()

- **原型**：`NiceRNG * nice_rng_glib_new (void);`
- **作用**：创建基于 GLib 全局随机数 API 的 `NiceRNG` 实例。
- **关键逻辑**：
  1. 调用 `g_slice_new0(NiceRNG)` 分配并零初始化 `NiceRNG` 结构体。
  2. 设置虚表函数指针：
     - `ret->seed = rng_seed`
     - `ret->generate_bytes = rng_generate_bytes`
     - `ret->generate_int = rng_generate_int`
     - `ret->free = rng_free`
  3. 返回构造好的 `NiceRNG` 实例。
  4. 注意：`priv` 字段保持为 `NULL`，因为 GLib 后端使用全局状态，无需私有数据。

#### nice_rng_glib_new_predictable()

- **原型**：`NiceRNG * nice_rng_glib_new_predictable (void);`
- **作用**：创建使用固定种子（seed = 0）的确定性 `NiceRNG` 实例，用于测试。
- **关键逻辑**：
  1. 调用 `nice_rng_glib_new()` 创建标准实例。
  2. 立即调用 `rng->seed(rng, 0)` 将 GLib 的进程级随机数种子重置为 0。
  3. 这意味着后续通过该实例（以及任何其他使用 GLib 全局随机状态的代码）生成的随机数序列是**完全可复现的**。这使得测试可以验证具体的随机输出值。

#### rng_seed() [static]

- **原型**：`static void rng_seed (NiceRNG *rng, guint32 seed)`
- **作用**：设置 GLib 进程级随机数生成器的种子。
- **关键逻辑**：忽略 `rng` 参数（标记为 `G_GNUC_UNUSED`），直接调用 `g_random_set_seed(seed)`。由于 GLib 的随机数 API 是进程全局的，此操作会影响整个进程中的所有随机数调用。

#### rng_generate_bytes() [static]

- **原型**：`static void rng_generate_bytes (NiceRNG *rng, guint len, gchar *buf)`
- **作用**：生成随机字节序列。
- **关键逻辑**：忽略 `rng` 参数，循环 `len` 次，每次调用 `g_random_int_range(0, 256)` 生成一个 `[0, 255]` 的字节值写入 `buf[i]`。之所以用 256 而非 255，是因为 `g_random_int_range` 的上界是开区间 `[begin, end)`。

#### rng_generate_int() [static]

- **原型**：`static guint rng_generate_int (NiceRNG *rng, guint low, guint high)`
- **作用**：生成指定范围内的随机整数。
- **关键逻辑**：忽略 `rng` 参数，直接返回 `g_random_int_range(low, high)`。返回值范围是 `[low, high)`。

#### rng_free() [static]

- **原型**：`static void rng_free (NiceRNG *rng)`
- **作用**：释放 `NiceRNG` 实例。
- **关键逻辑**：调用 `g_slice_free(NiceRNG, rng)` 释放通过 `g_slice_new0` 分配的内存。由于 GLib 后端不使用 `priv` 字段，无需额外清理。

---

## 调用关系

### 哪些模块依赖 random/

```
tests/test-common.c ──► nice_rng_new() ──► random/
agent/agent.c        ──► nice_rng_new(), nice_rng_free(), nice_rng_generate_bytes(), nice_rng_generate_int() ──► random/
agent/discovery.c    ──► nice_rng_generate_bytes(), nice_rng_generate_bytes_print() ──► random/
agent/stream.c       ──► nice_rng_generate_bytes_print() ──► random/
stun/usages/turn.c   ──► (间接通过 agent 的 rng 字段)
```

### random/ 依赖哪些模块/库

```
random/ ──► glib-2.0 (g_random_int_range, g_random_set_seed, g_slice_new0, g_slice_free)
random/ ──► config.h (HAVE_CONFIG_H)
```

`random/` 是一个**底层模块**，不依赖 libnice 的任何其他内部模块。它只依赖 GLib 系统库。

### 编译产物

`random/` 编译为静态库 `librandom`，被上层模块（主要是 `agent/`）链接。其符号通过 `libnice.so` 对外不可见（受 `nice/libnice.sym` 版本脚本控制）。

---

## 在 libnice 中的使用

### 1. STUN 事务 ID 生成

虽然 STUN 事务 ID（96 bits / 12 bytes）不在 `random/` 模块中直接生成，但它为 TURN 分配等场景提供了随机数据基础。

### 2. ICE ufrag 和 password 生成（agent/stream.c）

```c
nice_rng_generate_bytes_print (rng, NICE_STREAM_DEF_UFRAG - 1, stream->local_ufrag);
nice_rng_generate_bytes_print (rng, NICE_STREAM_DEF_PWD - 1, stream->local_password);
```

- **ufrag**：4 字节可打印字符（`NICE_STREAM_DEF_UFRAG = 4 + 1`，含 NULL 终止符）
- **password**：22 字节可打印字符（`NICE_STREAM_DEF_PWD = 22 + 1`，含 NULL 终止符）

使用 `generate_bytes_print` 确保输出仅包含 ICE 规范允许的 `ice-char` 字符集（字母数字和 `+/`）。

### 3. TURN 凭证生成（agent/discovery.c）

```c
nice_rng_generate_bytes (agent->rng, 32, (gchar *)username);   // TURN 用户名
nice_rng_generate_bytes (agent->rng, 16, (gchar *)password);   // TURN 密码
nice_rng_generate_bytes_print (agent->rng, 16, (gchar *)username); // STUN 用户名
```

TURN 的 username 和 password 使用原始随机字节（非可打印字符），而 STUN 的 username 使用可打印字符集。

### 4. Tie-breaker 生成（agent/agent.c）

```c
nice_rng_generate_bytes (agent->rng, 8, (gchar*)&agent->tie_breaker);
```

ICE tie-breaker 是一个 64 位无符号整数（8 字节），用于 ICE 角色冲突解决。这里使用原始随机字节填充。

### 5. 随机端口选择（agent/agent.c）

```c
start_port = nice_rng_generate_int(agent->rng, component->min_port, component->max_port+1);
```

当需要从端口范围中随机选择一个起始端口时使用。`max_port+1` 因为上界是开区间。

### 6. 测试中的确定性随机数（random/test.c）

```c
nice_rng_set_new_func (nice_rng_glib_new_predictable);
rng = nice_rng_new ();
```

测试代码通过 `nice_rng_set_new_func()` 将全局工厂函数替换为 `nice_rng_glib_new_predictable`（seed=0），然后验证生成的随机序列与预期值完全一致。这种设计使得随机数相关的测试变为确定性的。

---

## 设计特点

1. **策略模式**：通过函数指针虚表和全局工厂函数实现可替换的 RNG 后端。生产环境使用 GLib 随机数，测试环境可替换为确定性生成器。

2. **GLib 后端使用进程全局状态**：`random-glib.c` 使用 `g_random_int_range()` 而非独立的 `GRand` 实例。这意味着多个 `NiceRNG` 实例实际上共享同一个随机状态。这在 ICE 场景中没有问题，因为 agent 通常只有一个 RNG 实例。

3. **两层分发**：`random.c` 是公共 API 层（`nice_rng_*`），将调用分发到实际的后端虚表。这种设计使得未来可以轻松添加新的后端实现（如基于 OpenSSL 或内核 `/dev/urandom` 的后端），只需实现虚表接口即可。

4. **ice-char 合规**：`generate_bytes_print` 的字符集严格遵循 ICE 规范（RFC 8445, Section 15.1），确保生成的 ufrag/password 在 SDP 中传输时不会出现兼容性问题。

5. **无强密码学要求**：该模块注释表明生成的随机数据用于 STUN 事务 ID 和 ICE 凭证，这些不需要密码学强度的随机性。GLib 的 Mersenne Twister 伪随机数生成器对此场景已足够。
