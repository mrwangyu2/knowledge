# llama.cpp 完整使用指南

## 目录
- [简介](#简介)
- [核心参数配置](#核心参数配置)
- [性能优化](#性能优化)
- [多卡协同](#多卡协同)
- [MoE 支持](#moe-支持)
- [生产实践](#生产实践)

---

## 简介

**llama.cpp** 是一个高效的 C++ 推理引擎，专为在消费级硬件上运行大型语言模型优化。核心特点：
- 🚀 纯 C++ 实现，性能高效
- 💾 支持量化模型（GGUF 格式）
- 🎯 支持 CPU、GPU（CUDA/Metal/Vulkan）推理
- ⚡ 支持多卡并行、混合精度、MoE 模型
- 📊 完整的服务器 API（兼容 OpenAI 接口）

---

## 核心参数配置

### 1. 基础推理参数

#### 模型加载
```bash
./main -m model.gguf \
  --n-gpu-layers 35 \          # GPU 层数（0=纯 CPU，-1=自动）
  --tensor-split 0.5,0.5 \     # 多卡张量分割比例
  --main-gpu 0                  # 主 GPU 设备
```

| 参数                | 说明            | 默认值 | 建议值         |
| ----------------- | ------------- | --- | ----------- |
| `-m, --model`     | 模型路径（GGUF 格式） | -   | 必需          |
| `-n, --n-predict` | 最大生成 token 数  | 128 | 512-2048    |
| `-n-gpu-layers`   | 加载到 GPU 的层数   | 0   | -1（自动）或设置层数 |
| `--tensor-split`  | **多卡权重分割**：把模型的权重参数按比例分配到不同 GPU。例如 `0.5,0.5` = 2 张卡各承载 50% 的权重；`0.33,0.33,0.33` = 3 张卡各占 33%。推理时每张卡只计算分配给它的部分，最后合并结果。**分割对象是模型的矩阵权重**（不是数据）。**分割粒度是整个 Transformer 层的权重**（例如 7B 模型有 32 层，2 卡分割则每卡约 16 层的权重）。| -   | 2卡均分: `0.5,0.5`；3卡均分: `0.33,0.33,0.33`；4卡均分: `0.25,0.25,0.25,0.25`；按卡能力加权: `0.4,0.3,0.2,0.1` |
| `--main-gpu`      | 主 GPU ID      | 0   | 根据需要        |

#### 上下文和序列长度
```bash
./main -m model.gguf \
  -c 4096 \                     # 上下文长度（KV cache）
  --rope-scale 1.0 \            # RoPE 缩放（扩展上下文）
  --rope-freq-base 500000       # RoPE 基础频率
```

| 参数 | 说明 | 影响 |
|------|------|------|
| `-c, --ctx-size` | **KV 缓存大小**：模型需要记住的最长历史对话长度（单位：token）。例如 `-c 4096` 表示最多能处理 4096 个 token 的对话历史。超过这个长度的信息会被遗忘。每增加 1K tokens，VRAM 增加约 20-30MB（取决于模型大小和精度）。**直接决定 VRAM 占用和支持的最大序列长度** | 内存占用、支持的序列长度 |
| `--rope-scale` | **位置编码缩放因子**：用来扩展模型能理解的最大序列长度。原模型训练在 4K，设置 `--rope-scale 2.0` 后能理解 8K；`4.0` 能理解 16K。**本质是改变位置编码的频率尺度**，让模型"相信"序列更长。但质量会下降，可能需要微调 | 需要与 `-c` 配合使用以获得最佳效果 |
| `--rope-freq-base` | **RoPE 基础频率**：位置编码的频率基数（默认 10000）。调大这个值（如 500000）能更好地支持超长序列。**主要在极长上下文（>64K）时需要调整** | 对超长序列重要；通常保持默认即可 |

### 2. 采样参数

#### 温度和采样
```bash
./main -m model.gguf \
  --temp 0.7 \                  # 温度（0=贪心，高=随机）
  --top-p 0.9 \                 # 核采样
  --top-k 40 \                  # Top-K 采样
  --repeat-penalty 1.1 \        # 重复惩罚
  --min-p 0.05                  # 最小概率
```

| 参数 | 范围 | 说明 |
|------|------|------|
| `--temp` | 0.0-2.0 | **温度**：控制模型输出的"随机性"。`0.0` = 完全贪心（每次选择概率最高的词，完全确定性）；`0.7` = 平衡（既有多样性又相对稳定，推荐日常使用）；`1.0` = 标准随机采样；`>1.0` = 更加"疯狂"和多样（适合创意写作）。**影响每个 token 的选择概率分布** |
| `--top-p` | 0.0-1.0 | **核采样（Nucleus Sampling）**：只从概率累积达到 p 值的词表中采样。例如 `0.9` 表示只考虑概率累积到 90% 的词。`0.95` 更保守，`0.8` 更激进。**用来自动过滤"尾部"的低概率词** |
| `--top-k` | 1-100+ | **Top-K 采样**：只从概率最高的 K 个词中采样。`40` 表示每次只从前 40 个高概率词中选择。`1` = 贪心；`100+` = 考虑更多词。**限制词表大小** |
| `--repeat-penalty` | 1.0+ | **重复惩罚**：避免模型重复相同的词组。`1.0` = 无惩罚；`1.1` = 轻度惩罚（推荐）；`1.5` = 强惩罚。**通过降低已生成词的概率来实现** |
| `--min-p` | 0-1.0 | **最小概率阈值**：过滤掉概率低于这个值的词。`0.05` 表示只考虑概率 ≥5% 的词。**相当于一个简单的概率下界** |

### 3. 批处理参数

```bash
./main -m model.gguf \
  -n-batch 512 \                # 批大小
  --ubatch-size 512 \           # 逻辑批大小
  -ngl 35                       # GPU 层数（同 -n-gpu-layers）
```

| 参数 | 说明 | 内存/速度 |
|------|------|----------|
| `-n-batch` | **物理批大小**：GPU 一次性处理的 token 数量。例如 `-n-batch 512` 表示 GPU 每次处理 512 个 token。**越大越快**（GPU 利用率高），但**需要更多 VRAM**。典型值：128-2048。对延迟敏感的应用用 256-512；对吞吐量敏感的用 1024-2048 | 大→快 but 高内存 |
| `--ubatch-size` | **逻辑批大小**：统一批大小，用于多个序列并行处理时的分割。例如有 4 个序列，`--ubatch-size 256` 表示每个序列分成 256 token 的块来处理。**不直接影响 VRAM，主要影响计算调度**。通常设置为 `-n-batch` 的一半 | 影响调度灵活性 |
| `-ngl` | **加载到 GPU 的 Transformer 层数**。例如 7B 模型有 32 层，`-ngl 35` 表示全部加载到 GPU（35 > 32）；`-ngl 20` 表示只加载前 20 层到 GPU，剩余 12 层在 CPU 上计算。`-ngl 0` = 纯 CPU；`-ngl -1` = 自动全部加载。**越多越快，但越占 VRAM** | 核心性能参数 |

---

## 性能优化

### 1. 内存优化（VRAM 受限）

**策略 1: 量化**
```bash
# 使用低精度量化模型
./main -m model.Q4_K_M.gguf  # Q4_K_M 是最常用的平衡方案
# Q3_K_M: 最小化 VRAM（质量有所下降）
# Q4_K_M: 平衡（推荐）
# Q5_K_M: 接近原始质量
# Q6_K: 高质量（需要更多 VRAM）
```

**策略 2: 降低 KV Cache**
```bash
./main -m model.gguf \
  -c 2048 \                     # 从 4096 降到 2048
  --mlock                       # 锁定内存以加快访问
```

**策略 3: CPU + GPU 混合**
```bash
./main -m model.gguf \
  -ngl 30 \                     # 只加载 30 层到 GPU
  -c 4096                       # 其余在 CPU 上计算
```

### 2. 推理速度优化

**启用 GPU 加速**
```bash
# CUDA（NVIDIA）
./main -m model.gguf -ngl 35

# Metal（Apple Silicon）
./main -m model.gguf -ngl 35

# Vulkan（AMD/Intel）
./main -m model.gguf -ngl 35 --device vulkan
```

**批处理优化**
```bash
./main -m model.gguf \
  -n-batch 2048 \               # 大批次
  --ubatch-size 512 \           # 逻辑批
  -ngl 35
```

**并行处理**
```bash
# 启用多线程
export GGML_NUM_THREADS=8       # CPU 线程数
export GGML_GPU_THREADS=2       # GPU 线程数（如果支持）

./main -m model.gguf -ngl 35
```

### 3. 吞吐量优化（多个请求）

```bash
# 服务器模式
./server -m model.gguf \
  -ngl 35 \
  -n-batch 2048 \
  --ubatch-size 512 \
  --parallel 4 \                # 并行处理 4 个请求
  --slots-endpoint-disable      # 简化 API
```

---

## 多卡协同

### 1. 张量分割（Tensor Parallel）

**核心概念**：把模型的权重矩阵按比例分割到多张 GPU 上并行计算。

**分割原理**：
- **分割对象**：模型的 Transformer 层中的权重矩阵（如 Q、K、V 投影矩阵、FFN 权重等）
- **分割粒度**：整层权重（7B 模型有 32 个 Transformer 层；2 卡均分则每卡约 16 层）
- **计算流程**：输入 token → 卡 1 处理自己的权重层 → 结果传给卡 2 → 卡 2 继续处理 → 合并输出
- **通信开销**：层与层之间需要 GPU 间通信，多卡越多通信越多

```bash
# 2 卡：各占 50%（均匀分割，推荐）
./main -m model.gguf \
  --tensor-split 0.5,0.5 \
  -ngl 35

# 4 卡：均匀分配（每卡 25%）
./main -m model.gguf \
  --tensor-split 0.25,0.25,0.25,0.25 \
  -ngl 35

# 4 卡：按计算能力加权（e.g., RTX 4090 + 3x RTX 3090）
# 4090 性能强分 40%，其他卡各分 20%
./main -m model.gguf \
  --tensor-split 0.4,0.2,0.2,0.2 \
  -ngl 35
```

| 配置 | 优点 | 缺点 | 适用场景 |
|------|------|------|--------|
| 均匀分割（0.5,0.5） | 简单、负载均衡 | 受限最慢卡速度 | 2 张同型号卡 |
| 按能力加权（0.4,0.2,0.2,0.2） | 更高效、速度快 | 需手动调整比例 | 混合卡型号 |
| 极端分割（0.9,0.1） | 少数卡承载更多 | 严重不平衡，总体慢 | 不推荐 |

### 2. 主-从（RPC）架构

**用途**：跨机器分布式推理。把模型的不同部分放在不同的**物理服务器**上。

**工作原理**：
- **主机**：运行推理引擎，加载部分模型权重或全部权重
- **从机**：运行 RPC 服务器，提供额外的 GPU 算力
- **通信**：主机通过网络 RPC 调用从机，从机返回计算结果
- **适用场景**：单机 GPU 内存不足，但有多台服务器的场景

```bash
# 机器 1（主）：运行推理服务器，连接远程 RPC
./main -m model.gguf \
  --tensor-split 0.5 \           # 本地卡占 50%
  -ngl 35

# 机器 2（从）：运行 RPC 服务器，提供远程 GPU 算力
./rpc-server \
  --n-threads 8 \                # CPU 线程数
  --host 192.168.1.100 \         # 监听 IP
  --port 50052                   # RPC 端口
```

**配置说明**：
- **本地部分**：主机加载部分或全部模型权重（`--tensor-split 0.5` 表示只加载 50%）
- **远程部分**：RPC 服务器在从机上接收计算任务，返回结果
- **通信延迟**：网络延迟（跨机器通常 1-10ms），比 GPU 间通信（< 1ms）慢

**vs 张量分割**：
- **张量分割**：卡在同一机器（PCIe 连接，高带宽、低延迟）
- **RPC**：机器间通信（网络连接，低带宽、高延迟）
- **RPC 适用**：长序列推理（通信比例小）；**不适用**：短序列、实时应用

配置文件 (`rpc.txt`)：
```
rpc_servers = [
  "127.0.0.1:50052",   # 本地 GPU
  "remote-ip:50052"    # 远程机器
]
```

### 3. 管道并行（Pipeline Parallel）

**原理**：把模型的不同**阶段**或**层**分配到不同卡上，形成流水线。

**vs 张量分割**：
- **张量分割**：同一层的权重分割到多卡，所有卡同时工作（层级并行）
- **管道并行**：不同层分别在不同卡，前卡处理完传给后卡（层级流水线）

**优缺点**：
- ✅ 卡之间依赖关系明确，易于调试
- ✅ 通信量相对少（只在卡间传递激活值，不是权重）
- ❌ 卡之间存在依赖，难以完全并行，存在"气泡"（某些卡空闲等待）
- ❌ llama.cpp 官方支持有限

**llama.cpp 当前状态**：
管道并行目前不是 llama.cpp 的主要优化方向。**推荐优先用张量分割**。

```bash
# 目前 llama.cpp 的官方支持有限
# 推荐使用张量分割 + 批处理替代
./main -m model.gguf \
  --tensor-split 0.5,0.5 \       # 张量分割（比管道更高效）
  -n-batch 2048 \                # 大批次提高并行度
  -ngl 35
```

### 4. 多卡最佳实践

#### 案例 1：高端双卡（2x RTX 4090）

```bash
# 配置示例：2x RTX 4090 + 大模型
./main -m model.gguf \
  --tensor-split 0.5,0.5 \      # 均匀分割
  -ngl 35 \                      # 全部层到 GPU
  -n-batch 2048 \                # 大批次
  --ubatch-size 1024 \           # 大逻辑批
  -c 4096 \                      # 完整上下文
  --temp 0.7
```

#### 案例 2：中端双卡（2x RTX 2080 Ti 22GB）- 详细对比

**场景**：有一个 18GB 的模型，两张 RTX 2080 Ti（各 22GB VRAM）

**方案 A：全部加载到第一张卡（❌ 不推荐）**

```bash
./main -m model-18g.gguf \
  --main-gpu 0 \           # 所有权重都在 GPU 0
  -ngl 35                  # 全部层到 GPU 0
```

| 指标 | 情况 |
|------|------|
| GPU 0 占用 | 18GB（模型）+ 4GB（KV Cache）= **22GB**（刚好满） |
| GPU 1 占用 | 0GB（完全闲置） |
| 内存余量 | 0GB（无余量，生成长序列易 OOM） |
| 推理速度 | **1.0x**（基准，单卡） |
| GPU 利用率 | GPU 0: 100% / GPU 1: 0% |

**问题**：
- ⚠️ VRAM 刚好满，没有缓冲空间
- 生成长序列时 KV Cache 增长 → 超过 22GB → **OOM 崩溃**
- GPU 1 完全浪费

---

**方案 B：张量分割 0.5,0.5（✅ 推荐）**

```bash
./main -m model-18g.gguf \
  --tensor-split 0.5,0.5 \       # 模型权重 50% 分给每张卡
  -ngl 35 \                       # 全部层到 GPU
  -n-batch 1024 \                 # 大批次
  --ubatch-size 512 \
  -c 2048 \                       # KV cache（两卡各 1GB）
  --temp 0.7
```

| 指标 | 情况 |
|------|------|
| GPU 0 占用 | 9GB（权重）+ 2GB（KV Cache）= **11GB**（余 11GB） |
| GPU 1 占用 | 9GB（权重）+ 2GB（KV Cache）= **11GB**（余 11GB） |
| 内存余量 | 每卡 11GB，总计 22GB 可用余量 |
| 推理速度 | **~1.9x**（接近 2 倍加速 ⚡） |
| GPU 利用率 | GPU 0: ~95% / GPU 1: ~95% |

**为什么快接近 2 倍**：

```
单卡推理：GPU 0 依次处理 32 层
  第 1 层：50ms
  第 2 层：50ms
  ...
  第 32 层：50ms
  总耗时：32 × 50ms = 1600ms

双卡分割推理：GPU 0 和 GPU 1 流水线协作
  GPU 0 处理第 1-16 层（800ms）
  ↓ (同时) ↓
  GPU 1 处理第 17-32 层（800ms）
  + 层间通信开销（~20ms）
  总耗时：≈ 820ms

加速比 = 1600 / 820 ≈ 1.95x ⚡
```

**优势**：
- ✅ 每卡还有 11GB 余量，安全处理长序列
- ✅ 推理速度接近 2 倍，吞吐量大幅提升
- ✅ 两张卡都充分利用
- ✅ 通信开销仅占 ~1-2%（PCIe 高带宽）

---

**对比总结**：

| 方面 | 方案 A | 方案 B |
|------|--------|--------|
| **速度** | 1.0x | **1.9x** ⚡ |
| **内存安全** | ⚠️ 危险 | ✅ 安全 |
| **余量** | 0GB | 11GB/卡 |
| **长序列支持** | ❌ 易崩溃 | ✅ 可靠 |
| **推荐度** | ❌ 不推荐 | ✅✅ 强烈推荐 |

---

**监控多卡状态**
```bash
# 查看 GPU 使用率
nvidia-smi -l 1                 # NVIDIA

# 查看 llama.cpp 的 VRAM 分配
# 输出中会显示每卡的内存占用
```

---

## MoE 支持

### 1. 什么是 MoE？

**混合专家模型**：
- 使用多个专家网络，动态路由输入
- 参数多但激活参数少→高效推理
- 例：Mixtral 8x7B（8 个 7B 专家）

### 2. 参数配置

```bash
./main -m mixtral-8x7b.gguf \
  -ngl 35 \                      # GPU 加速
  --moe-top-k 2 \                # 激活前 K 个专家（通常 2）
  -n-batch 512 \                 # MoE 对批大小敏感
  --ubatch-size 256
```

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--moe-top-k` | 激活的专家数 | 2 |
| `--moe-aux-loss` | 辅助损失（微调用）| 0 |

### 3. 性能特性

| 指标 | 说明 |
|------|------|
| **内存占用** | 类似 7B（激活参数多），参数总数大 |
| **推理速度** | 比密集模型快 2-3x（激活参数少） |
| **VRAM** | 需要加载所有专家，通常需要 16GB+ |
| **批处理** | 批大小→更好的负载均衡 |

### 4. 多卡 + MoE

```bash
# 2 卡 × Mixtral 8x7B
./main -m mixtral-8x7b.gguf \
  --tensor-split 0.5,0.5 \       # 分割模型
  -ngl 35 \
  --moe-top-k 2 \
  -n-batch 1024 \
  --ubatch-size 512
```

**最佳实践**：
- 启用批处理以提高专家利用率
- 多卡分割时，确保张量分割适配专家数
- 量化 MoE 模型以节省 VRAM

---

## 生产实践

### 1. 服务器模式

```bash
./server -m model.gguf \
  -ngl 35 \
  -n-batch 2048 \
  --ubatch-size 512 \
  --parallel 4 \
  -c 4096 \
  --log-format text \
  --host 0.0.0.0 \
  --port 8000
```

**API 调用**（OpenAI 兼容）：
```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "model.gguf",
    "prompt": "Hello",
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

### 2. 内存管理

```bash
# 避免 OOM：监控内存
./main -m model.gguf \
  -ngl 30 \                      # 只加载 30 层（保留余量）
  -c 2048 \                      # 较小 KV cache
  --mlock                        # 锁定 RAM
```

### 3. 推理稳定性

```bash
./main -m model.gguf \
  -ngl 35 \
  --seed 42 \                    # 固定随机种子
  --temp 0.0 \                   # 确定性输出（如需）
  -n-batch 512 \                 # 保守的批大小
```

### 4. 模型量化

```bash
# 量化 fp32 模型为 Q4_K_M
./quantize model.gguf model-q4.gguf Q4_K_M

# 加载量化模型
./main -m model-q4.gguf -ngl 35
```

**量化对比**：
| 格式 | VRAM | 速度 | 质量 |
|------|------|------|------|
| FP32 | 100% | 1x | ⭐⭐⭐⭐⭐ |
| Q8 | 50% | 1.2x | ⭐⭐⭐⭐⭐ |
| Q6 | 35% | 1.5x | ⭐⭐⭐⭐ |
| Q5 | 30% | 2x | ⭐⭐⭐⭐ |
| Q4 | 25% | 2.5x | ⭐⭐⭐ |
| Q3 | 20% | 3x | ⭐⭐ |

---

## 运行状态监控与 OOM 检测

### 1. 实时 GPU 监控

#### NVIDIA GPU 监控

```bash
# 实时监控（每秒刷新）
nvidia-smi -l 1

# 监控特定进程（llama.cpp）
nvidia-smi -l 1 | grep main

# 持续监控 GPU 使用率和内存
watch -n 1 nvidia-smi

# 详细模式（包括进程信息）
nvidia-smi dmon -s pucvmet
```

**关键指标说明**：
- **Memory-Usage**：显存占用（MB），例如 `22000MiB / 24576MiB` 表示用了 22GB，总共 24GB
- **Processes**：运行在 GPU 上的进程，应该看到 `main` 进程
- **Volatile GPU-Util**：GPU 计算利用率（%），80%+ 表示充分利用

#### 输出示例

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.104.05             Driver Version: 535.104.05                |
|-------------------------------+----------------------+----------------------+
| GPU  Name                 TCC/FC Bus-Id        Memory-Usage | GPU-Util  |
|===============================+======================+======================|
|   0  NVIDIA RTX 4090      On  | 00:1F.0     22000MiB / 24576MiB |    85%   |
|   1  NVIDIA RTX 4090      On  | 01:00.0     12000MiB / 24576MiB |    80%   |
+-------------------------------+----------------------+----------------------+
```

### 2. OOM（显存溢出）的迹象和检测

#### 如何识别 OOM 错误

**症状 1：进程立即崩溃**
```bash
./main -m model.gguf -ngl 35
# 立即输出：
# CUDA error: out of memory
# 或 cudaMalloc failed: out of memory
```

**症状 2：运行一段时间后崩溃**
```
正常输出几个 token...然后：
CUDA error 2: out of memory
或显示一个断言失败 (Aborted)
```

**症状 3：进程挂起（未真正崩溃）**
```
- 输出停止，但进程没有退出
- GPU 内存占用达到 100%
- CPU 占用率降到 0%
```

#### 检测方法

**方法 1：查看错误日志**
```bash
# 运行时重定向 stderr 到文件
./main -m model.gguf -ngl 35 2> error.log

# 查看错误信息
cat error.log | grep -i "out of memory\|cuda error\|malloc"
```

**方法 2：监控 GPU 内存**
```bash
# 开两个终端
# 终端 1：启动推理
./main -m model.gguf -ngl 35 -c 4096

# 终端 2：实时监控
while true; do
  echo "=== $(date) ==="
  nvidia-smi --query-gpu=memory.used,memory.total --format=csv,noheader
  sleep 1
done
```

**方法 3：运行参数检查**
```bash
# 根据 KV Cache 占用表计算理论占用
# 例如 7B Q4 + 128K 上下文 = 28GB（单卡不够！）

# 改用多卡或降低上下文长度
./main -m model-7b-q4.gguf \
  --tensor-split 0.5,0.5 \     # 多卡分割
  -c 65536 \                    # 降到 64K
  -ngl 35
```

### 3. 预防 OOM 的最佳实践

#### 策略 1：事前计算

```bash
# 根据公式计算最大可支持的上下文长度
# KV Cache ≈ (model_params / num_gpus) × context_length × 0.1GB

# 例如：7B 单卡，22GB VRAM，减去 4GB 模型 = 18GB 可用
# 18GB ÷ (7B × 0.1/7000) ≈ 18GB ÷ 0.0001 = 最大约 128K tokens

# 保守估计：取 70% 的理论值以留余量
# 128K × 0.7 ≈ 90K tokens
./main -m model-7b-q4.gguf -c 90000 -ngl 35
```

#### 策略 2：启用内存监控参数

```bash
./main -m model.gguf \
  -c 4096 \
  -ngl 35 \
  --mlock \                     # 锁定已分配内存（防止页面交换）
  --verbose                     # 启用详细日志
```

**--mlock 作用**：
- 防止 RAM 页面交换到磁盘
- 提高推理速度（内存访问不经过交换）
- 但消耗更多 RAM

#### 策略 3：渐进式增加

```bash
# 从保守配置开始
./main -m model.gguf \
  -c 2048 \                     # 2K 上下文
  -ngl 20 \                     # 只加载 20 层
  -n-batch 256

# 逐步增加参数
# 第 1 步：增加 GPU 层数
# -ngl 20 → -ngl 30 → -ngl 35

# 第 2 步：增加上下文长度
# -c 2048 → -c 4096 → -c 8192

# 第 3 步：增加批大小
# -n-batch 256 → -n-batch 512 → -n-batch 1024

# 每步之间测试，观察是否 OOM
```

### 4. OOM 后的恢复和降级方案

#### 快速修复

```bash
# 如果 OOM 了，立即使用以下配置（必定不会 OOM）
./main -m model.gguf \
  -ngl 0 \                      # 纯 CPU 推理（慢但安全）
  -c 2048 \                     # 最小上下文
  -n-batch 128 \                # 小批次
  --temp 0.7
```

#### 渐进式恢复

```bash
# 配置 1：CPU 推理（完全安全，但极慢）
./main -m model.gguf -ngl 0 -c 2048

# 配置 2：部分 GPU（20 层）
./main -m model.gguf -ngl 20 -c 4096 -n-batch 512

# 配置 3：更多 GPU（30 层）
./main -m model.gguf -ngl 30 -c 8192 -n-batch 1024

# 配置 4：全 GPU（35 层）
./main -m model.gguf -ngl 35 -c 16384 -n-batch 2048

# 配置 5：多卡分割（必须成功）
./main -m model.gguf --tensor-split 0.5,0.5 -ngl 35 -c 32768
```

### 5. 服务器模式下的监控

#### 带内存限制的服务器启动

```bash
# 设置环境变量限制内存
export GGML_GPU_MEMORY=20000000000  # 20GB 限制

./server -m model.gguf \
  -ngl 35 \
  -n-batch 2048 \
  --log-format text \
  --verbose
```

#### 监控服务器内存泄漏

```bash
# 方法 1：监控内存增长趋势
while true; do
  echo "$(date '+%H:%M:%S') - $(nvidia-smi --query-gpu=memory.used --format=csv,noheader | head -1)"
  sleep 10
done > memory_trend.log

# 方法 2：监控进程信息
ps aux | grep server | grep -v grep

# 方法 3：分析日志中的错误
tail -f server_log.txt | grep -i "error\|warning\|oom"
```

### 6. 多卡监控

#### 卡间负载均衡检查

```bash
# 双卡监控（实时对比）
watch -n 1 'nvidia-smi --query-gpu=index,memory.used,memory.total,utilization.gpu --format=csv,noheader'

# 输出示例
0, 12000MiB, 24576MiB, 95%
1, 12000MiB, 24576MiB, 95%  # 均衡 ✅

# 不均衡的例子
0, 22000MiB, 24576MiB, 98%  # GPU 0 爆满
1, 2000MiB, 24576MiB, 5%    # GPU 1 空闲
```

#### 调整张量分割

```bash
# 如果不均衡，调整分割比例
# 原来：0.5,0.5（不均衡）
# 改为：0.7,0.3（按卡能力）

./main -m model.gguf \
  --tensor-split 0.7,0.3 \
  -ngl 35
```

---

## API 接口配置与 base_url 管理

### 1. llama.cpp 的两种 API 接口

llama.cpp 内置服务器支持两种 API 接口风格：

#### 接口 1：OpenAI 兼容接口（推荐）

**Base URL**：`http://localhost:8000/v1`

**支持的端点**：
- `/v1/chat/completions` - 对话接口（兼容 OpenAI）
- `/v1/completions` - 补全接口（兼容 OpenAI）
- `/v1/models` - 列出可用模型
- `/v1/embeddings` - 生成嵌入向量

**启动服务器**：
```bash
./server -m model.gguf \
  -ngl 35 \
  --host 0.0.0.0 \
  --port 8000
```

**使用示例**：
```bash
# Chat Completions
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2",
    "messages": [{"role": "user", "content": "Hello"}],
    "temperature": 0.7,
    "max_tokens": 100
  }'
```

**Python 客户端**：
```python
import openai

openai.api_base = "http://localhost:8000/v1"
openai.api_key = "not-needed"  # llama.cpp 不需要 API key

response = openai.ChatCompletion.create(
    model="llama-2",
    messages=[{"role": "user", "content": "Hello world"}],
    temperature=0.7,
    max_tokens=100
)
print(response.choices[0].message.content)
```

#### 接口 2：llama.cpp 原生接口

**Base URL**：`http://localhost:8000`

**支持的端点**：
- `/completion` - 文本补全（原生参数）
- `/tokenize` - 文本分词
- `/embedding` - 生成嵌入
- `/slots` - 查看推理槽位状态
- `/health` - 健康检查

**优势**：
- 支持所有 llama.cpp 特有参数
- 更细粒度的控制（`mirostat`、`top-k`、`top-p` 等）
- 性能更优

**使用示例**：
```bash
curl http://localhost:8000/completion \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Hello",
    "n_predict": 100,
    "temperature": 0.7,
    "top_k": 40,
    "top_p": 0.9,
    "mirostat": 2,
    "mirostat_tau": 5.0,
    "mirostat_eta": 0.1
  }'
```

### 2. 配置多个 base_url 的方案

#### 方案 A：单机多端口（推荐用于开发）

**启动多个服务器实例**：
```bash
# 服务器 1：7B 模型，高性能
./server -m model-7b.gguf \
  -ngl 35 \
  --host 0.0.0.0 \
  --port 8000 &

# 服务器 2：13B 模型，中等性能
./server -m model-13b.gguf \
  -ngl 30 \
  --host 0.0.0.0 \
  --port 8001 &

# 服务器 3：70B 模型，多卡
./server -m model-70b.gguf \
  --tensor-split 0.5,0.5 \
  -ngl 35 \
  --host 0.0.0.0 \
  --port 8002 &
```

**Python 客户端**：
```python
import openai

# 定义多个 base_url
BASE_URLS = {
    "fast": "http://localhost:8000/v1",      # 7B
    "balanced": "http://localhost:8001/v1",  # 13B
    "powerful": "http://localhost:8002/v1"   # 70B
}

# 根据需求选择
openai.api_base = BASE_URLS["balanced"]
openai.api_key = "not-needed"

response = openai.ChatCompletion.create(
    model="llama-2",
    messages=[{"role": "user", "content": "Hello"}]
)
```

#### 方案 B：负载均衡（用于生产）

```python
import openai
import random
from typing import List

class LlamaCppRouter:
    def __init__(self, servers: List[str]):
        self.servers = servers
        self.current_idx = 0
    
    def get_base_url(self, strategy="round-robin"):
        """获取下一个可用的 base_url"""
        if strategy == "round-robin":
            url = self.servers[self.current_idx]
            self.current_idx = (self.current_idx + 1) % len(self.servers)
            return url
        elif strategy == "random":
            return random.choice(self.servers)
    
    def request(self, messages, model="llama-2", **kwargs):
        """发送请求到任意可用服务器"""
        for attempt in range(len(self.servers)):
            base_url = self.get_base_url()
            try:
                openai.api_base = base_url
                openai.api_key = "not-needed"
                response = openai.ChatCompletion.create(
                    model=model,
                    messages=messages,
                    **kwargs
                )
                return response
            except Exception as e:
                print(f"Failed on {base_url}: {e}")
                continue
        raise Exception("All servers failed")

# 使用
router = LlamaCppRouter([
    "http://localhost:8000/v1",
    "http://localhost:8001/v1",
    "http://localhost:8002/v1"
])

response = router.request(
    messages=[{"role": "user", "content": "Hello"}],
    temperature=0.7,
    max_tokens=100
)
```

#### 方案 C：Docker Compose 编排（推荐用于部署）

**docker-compose.yml**：
```yaml
version: '3.8'

services:
  # 快速推理服务（7B）
  llama-fast:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    volumes:
      - ./models:/models
    environment:
      - MODEL_PATH=/models/model-7b-q4.gguf
      - N_GPU_LAYERS=35
      - N_THREADS=8
    command: >
      ./server 
      -m /models/model-7b-q4.gguf 
      -ngl 35 
      --host 0.0.0.0 
      --port 8000

  # 平衡推理服务（13B）
  llama-balanced:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8001:8000"
    volumes:
      - ./models:/models
    environment:
      - MODEL_PATH=/models/model-13b-q4.gguf
      - N_GPU_LAYERS=30
    command: >
      ./server 
      -m /models/model-13b-q4.gguf 
      -ngl 30 
      --host 0.0.0.0 
      --port 8000

  # 高性能推理服务（70B 多卡）
  llama-powerful:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8002:8000"
    volumes:
      - ./models:/models
    environment:
      - MODEL_PATH=/models/model-70b-q4.gguf
      - TENSOR_SPLIT=0.5,0.5
    command: >
      ./server 
      -m /models/model-70b-q4.gguf 
      --tensor-split 0.5,0.5
      -ngl 35 
      --host 0.0.0.0 
      --port 8000
```

**启动**：
```bash
docker-compose up -d

# 查看日志
docker-compose logs -f

# 测试
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "llama", "messages": [{"role": "user", "content": "Hi"}]}'
```

### 3. 环境变量配置

**方案 1：.env 文件管理**

**.env**：
```
# OpenAI 兼容接口
LLAMA_CPP_BASE_URL_1=http://localhost:8000/v1
LLAMA_CPP_BASE_URL_2=http://localhost:8001/v1
LLAMA_CPP_BASE_URL_3=http://localhost:8002/v1

# 原生接口
LLAMA_CPP_NATIVE_URL=http://localhost:8000

# API 密钥（llama.cpp 无需，但兼容标准）
OPENAI_API_KEY=not-needed
```

**Python 代码**：
```python
import os
from dotenv import load_dotenv
import openai

load_dotenv()

# 读取第一个 base_url
openai.api_base = os.getenv("LLAMA_CPP_BASE_URL_1")
openai.api_key = os.getenv("OPENAI_API_KEY")

response = openai.ChatCompletion.create(...)
```

**方案 2：配置文件（config.yaml）**

**config.yaml**：
```yaml
llama_cpp:
  servers:
    - name: "fast"
      base_url: "http://localhost:8000/v1"
      model: "model-7b"
      gpu_layers: 35
    
    - name: "balanced"
      base_url: "http://localhost:8001/v1"
      model: "model-13b"
      gpu_layers: 30
    
    - name: "powerful"
      base_url: "http://localhost:8002/v1"
      model: "model-70b"
      gpu_layers: 35
      tensor_split: "0.5,0.5"
```

**Python 加载**：
```python
import yaml
import openai

with open("config.yaml") as f:
    config = yaml.safe_load(f)

# 使用第一个服务器
server = config["llama_cpp"]["servers"][0]
openai.api_base = server["base_url"]
openai.api_key = "not-needed"

response = openai.ChatCompletion.create(...)
```

### 4. 与 DeepSeek 的对比

| 特性 | DeepSeek | llama.cpp |
|------|---------|----------|
| **Base URL 形式** | `https://api.deepseek.com/v1` | `http://localhost:PORT/v1` |
| **多 base_url 方式** | 同一服务，参数切换 | 多个实例，不同端口 |
| **API 兼容性** | OpenAI 兼容 | OpenAI 兼容 + 原生 |
| **认证方式** | API Key 必需 | 不需要（可选） |
| **部署方式** | 云服务 | 本地 / 容器化 |
| **多模型支持** | 需要 base_url 区分 | 多服务器实例 |

### 5. 快速切换 base_url 的工具函数

```python
from enum import Enum
from typing import Optional
import openai

class ModelSize(Enum):
    FAST = "http://localhost:8000/v1"      # 7B
    BALANCED = "http://localhost:8001/v1"  # 13B
    POWERFUL = "http://localhost:8002/v1"  # 70B

def set_model(size: ModelSize, api_key: str = "not-needed"):
    """快速切换模型"""
    openai.api_base = size.value
    openai.api_key = api_key

def query(prompt: str, model_size: ModelSize = ModelSize.BALANCED):
    """使用指定模型查询"""
    set_model(model_size)
    response = openai.Completion.create(
        model="llama-2",
        prompt=prompt,
        max_tokens=100
    )
    return response.choices[0].text

# 使用
print(query("Hello", ModelSize.FAST))       # 用快速模型
print(query("Hello", ModelSize.POWERFUL))   # 用强力模型
```

---

### Q1: 推理很慢？
1. 检查是否启用 GPU：`-ngl 35`
2. 增加批大小：`-n-batch 2048`
3. 使用量化模型：`model-q4.gguf`
4. 检查 CPU 线程：`GGML_NUM_THREADS=8`

### Q2: 爆显存？
1. 降低 KV cache：`-c 2048`
2. 使用量化模型：`Q4_K_M`
3. 只加载部分层：`-ngl 20`
4. 启用 CPU 辅助：混合 GPU + CPU

### Q3: 多卡不平衡？
1. 调整张量分割：`--tensor-split 0.4,0.6`（按卡能力）
2. 增加批大小以提高利用率
3. 使用 `nvidia-smi` 监控均衡性

### Q4: MoE 模型推理慢？
1. 启用批处理：`-n-batch 1024`
2. 多卡分割：`--tensor-split 0.5,0.5`
3. 减小 KV cache：`-c 2048`

---

## 上下文长度配置详解

### 不同 -c 参数的具体配置需求

| 上下文长度 | 单次推理 VRAM | KV Cache 大小 | 推荐硬件 | 完整配置示例 | 使用场景 |
|-----------|-------------|-------------|--------|-----------|--------|
| **32K** `-c 32768` | +6-8GB | ~12GB | RTX 3090/4080 | `./main -m model-q4.gguf -c 32768 -ngl 35 -n-batch 512 --rope-scale 2.0` | 长文档分析、代码审查 |
| **64K** `-c 65536` | +12-16GB | ~24GB | RTX 4090/A100 | `./main -m model-q4.gguf -c 65536 -ngl 35 -n-batch 256 --rope-scale 4.0 --rope-freq-base 500000` | 书籍总结、多轮对话 |
| **96K** `-c 98304` | +18-24GB | ~36GB | 2x RTX 4090 | `./main -m model-q4.gguf -c 98304 --tensor-split 0.5,0.5 -ngl 35 -n-batch 256 --rope-scale 6.0` | 项目级文档、技术规范 |
| **128K** `-c 131072` | +24-32GB | ~48GB | A100 / 2x RTX 4090 | `./main -m model-q4.gguf -c 131072 --tensor-split 0.5,0.5 -ngl 35 -n-batch 128 --rope-scale 8.0 --mlock` | 整本书分析、多文档融合 |
| **256K** `-c 262144` | +48-64GB | ~96GB | 4x RTX 4090 / H100 | `./main -m model-q4.gguf -c 262144 --tensor-split 0.25,0.25,0.25,0.25 -ngl 35 -n-batch 64 --rope-scale 16.0 --mlock` | 超长文本、实时数据库 |

### 详细参数说明

#### 32K 配置
```bash
# RTX 3090 / RTX 4080 上运行
./main -m model-q4.gguf \
  -c 32768 \                     # 32K tokens
  -ngl 35 \                      # GPU 加速
  -n-batch 512 \                 # 较大批次
  --ubatch-size 256 \
  --rope-scale 2.0 \             # RoPE 2倍扩展
  --temp 0.7
```

**资源需求**：
- 基础模型（7B Q4）：~4GB
- KV Cache（32K）：~6GB
- **总计**：~10-12GB VRAM

#### 64K 配置
```bash
# RTX 4090 / A6000 上运行
./main -m model-q4.gguf \
  -c 65536 \                     # 64K tokens
  -ngl 35 \
  -n-batch 256 \                 # 中等批次
  --ubatch-size 128 \
  --rope-scale 4.0 \             # RoPE 4倍扩展
  --rope-freq-base 500000 \      # 调整频率基数
  --temp 0.7
```

**资源需求**：
- 基础模型（7B Q4）：~4GB
- KV Cache（64K）：~12GB
- **总计**：~16-18GB VRAM

#### 96K 配置
```bash
# 2x RTX 4090 或单 A100
./main -m model-q4.gguf \
  -c 98304 \                     # 96K tokens
  --tensor-split 0.5,0.5 \       # 双卡均衡
  -ngl 35 \
  -n-batch 256 \
  --ubatch-size 128 \
  --rope-scale 6.0 \             # RoPE 6倍扩展
  --rope-freq-base 500000 \
  --mlock                        # 锁定内存
```

**资源需求**（单卡）：
- 基础模型（7B Q4）：~4GB
- KV Cache（96K）：~18GB
- **总计**：~22-24GB VRAM
- **多卡**：2x 12GB = 24GB 足够

#### 128K 配置
```bash
# A100 或 2x RTX 4090（推荐）
./main -m model-q4.gguf \
  -c 131072 \                    # 128K tokens
  --tensor-split 0.5,0.5 \       # 张量分割
  -ngl 35 \
  -n-batch 128 \                 # 保守批次
  --ubatch-size 64 \
  --rope-scale 8.0 \             # RoPE 8倍扩展
  --rope-freq-base 500000 \
  --mlock \
  --parallel 2                   # 并行请求
```

**资源需求**（单卡）：
- 基础模型（7B Q4）：~4GB
- KV Cache（128K）：~24GB
- **总计**：~28-32GB VRAM（不可行）
- **多卡**：2x 24GB = 48GB（推荐）

#### 256K 配置
```bash
# H100 / 4x RTX 4090（必需多卡）
./main -m model-q4.gguf \
  -c 262144 \                    # 256K tokens
  --tensor-split 0.25,0.25,0.25,0.25 \  # 4卡均衡
  -ngl 35 \
  -n-batch 64 \                  # 小批次
  --ubatch-size 32 \
  --rope-scale 16.0 \            # RoPE 16倍扩展
  --rope-freq-base 500000 \
  --mlock \
  --parallel 1                   # 单请求模式
```

**资源需求**：
- 基础模型（7B Q4）：~4GB
- KV Cache（256K）：~48GB
- **总计**：~52-56GB VRAM
- **推荐**：4x 24GB = 96GB（安全余量）

### 不同模型大小的 KV Cache 占用

| 模型 | 量化 | 基础 VRAM | 32K | 64K | 128K | 256K |
|------|------|----------|-----|-----|------|------|
| 7B | Q4 | 4GB | 10GB | 16GB | 28GB | 52GB |
| 7B | Q5 | 6GB | 12GB | 18GB | 30GB | 54GB |
| 7B | Q8 | 8GB | 14GB | 20GB | 32GB | 56GB |
| 13B | Q4 | 7GB | 13GB | 19GB | 31GB | 55GB |
| 13B | Q5 | 10GB | 16GB | 22GB | 34GB | 58GB |
| **27B** | **Q4** | **14GB** | **20GB** | **26GB** | **38GB** | **62GB** |
| **27B** | **Q5** | **19GB** | **25GB** | **31GB** | **43GB** | **67GB** |
| **27B** | **Q8** | **24GB** | **30GB** | **36GB** | **48GB** | **72GB** |
| **35B** | **Q4** | **18GB** | **24GB** | **30GB** | **42GB** | **66GB** |
| **35B** | **Q5** | **25GB** | **31GB** | **37GB** | **49GB** | **73GB** |
| **35B** | **Q8** | **32GB** | **38GB** | **44GB** | **56GB** | **80GB** |
| 70B | Q4 | 37GB | 43GB | 49GB | 61GB | 85GB |

**说明**：
- **基础 VRAM** = 模型权重占用（不含 KV Cache）
- **32K/64K/128K/256K** = 模型 + 对应上下文长度的 KV Cache 总占用
- 27B 和 35B 是中等规模模型，常见于：
  - 27B：Llama 2 27B、Mistral-medium 等
  - 35B：Llama 3 35B、Qwen 35B 等
- Q4（4-bit）是最常用的平衡方案
- 如果单卡 VRAM 不足，使用多卡张量分割（`--tensor-split`）

### RoPE 参数配置对照

| 上下文 | rope-scale | rope-freq-base | 说明 |
|-------|-----------|----------------|------|
| 4K (默认) | 1.0 | 10000 | 原始设置 |
| 32K | 2.0 | 10000 | 2倍扩展 |
| 64K | 4.0 | 500000 | 高频基数 |
| 96K | 6.0 | 500000 | 更高频率 |
| 128K | 8.0 | 500000 | 深度外推 |
| 256K | 16.0 | 500000 | 极限外推 |

### 性能影响

#### 推理速度衰减（相对于 4K）
```
32K  : 0.95x (几乎无影响)
64K  : 0.85x (约 15% 减速)
96K  : 0.70x (约 30% 减速)
128K : 0.50x (约 50% 减速)
256K : 0.30x (约 70% 减速)
```

#### 内存扩展策略

**内存不足时的降级方案**：
```bash
# 原计划 128K，改为 96K + 更小批次
./main -m model-q4.gguf \
  -c 98304 \
  -ngl 35 \
  -n-batch 128 \
  --ubatch-size 64

# 或改为 CPU 辅助（64K + CPU）
./main -m model-q4.gguf \
  -c 65536 \
  -ngl 20 \                      # 只加载部分到 GPU
  -n-batch 256
```

---

### 6. 支持 Anthropic API 接口（原生内置，无需适配层）

> ⚠️ **重要更新**：llama.cpp 从 2025 年初（PR #17570 / #17425）起已**原生内置** Anthropic Messages API 支持，无需任何第三方适配层或代理。

#### 原生接口概览

llama.cpp 服务器同时暴露三种 API 接口，使用**同一个端口**：

| 接口 | Base URL | 端点 | 说明 |
|------|----------|------|------|
| **OpenAI 兼容** | `http://localhost:8000/v1` | `/chat/completions` | 兼容 OpenAI SDK |
| **Anthropic 兼容** | `http://localhost:8000` | `/v1/messages` | 兼容 Anthropic SDK ⭐ |
| **llama.cpp 原生** | `http://localhost:8000` | `/completion` | 最底层控制 |

> 注意：Anthropic 接口的端点是 `/v1/messages`，不需要额外端口或服务。

#### 启动服务器（无需特殊参数）

```bash
# 一个命令，同时支持所有三种 API
./server -m model.gguf \
  -ngl 35 \
  --host 0.0.0.0 \
  --port 8000
```

启动后即可使用：
- `http://localhost:8000/v1/chat/completions` → OpenAI 接口
- `http://localhost:8000/v1/messages` → Anthropic 接口 ⭐

#### 使用方式

**Python - Anthropic SDK**：
```python
import anthropic

# 指向本地 llama.cpp 服务器
client = anthropic.Anthropic(
    api_key="not-needed",          # llama.cpp 不需要真实 key
    base_url="http://localhost:8000"  # 直接指向 llama.cpp，无需适配层
)

message = client.messages.create(
    model="llama-2",               # 模型名称（与服务器保持一致）
    max_tokens=1024,
    system="You are a helpful assistant.",
    messages=[
        {"role": "user", "content": "Hello, how are you?"}
    ]
)

print(message.content[0].text)
```

**Python - OpenAI SDK**：
```python
import openai

client = openai.OpenAI(
    api_key="not-needed",
    base_url="http://localhost:8000/v1"
)

response = client.chat.completions.create(
    model="llama-2",
    messages=[{"role": "user", "content": "Hello"}]
)
print(response.choices[0].message.content)
```

**cURL - Anthropic 接口**：
```bash
curl http://localhost:8000/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: not-needed" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "llama-2",
    "max_tokens": 1024,
    "system": "You are a helpful assistant.",
    "messages": [
      {"role": "user", "content": "Hello"}
    ]
  }'
```

**cURL - OpenAI 接口**：
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

#### 与 DeepSeek 的对比

| 特性 | DeepSeek | llama.cpp |
|------|---------|----------|
| **OpenAI Base URL** | `https://api.deepseek.com/v1` | `http://localhost:8000/v1` |
| **Anthropic Base URL** | `https://api.deepseek.com/anthropic` | `http://localhost:8000`（同端口） |
| **OpenAI 接口** | 原生支持 | ✅ 原生支持 |
| **Anthropic 接口** | 原生支持 | ✅ 原生支持（PR #17570） |
| **API Key** | 必需 | 不需要 |
| **部署方式** | 云服务 | 本地/Docker |

#### 相关资源

- [PR #17570 - Anthropic Messages API 支持](https://github.com/ggml-org/llama.cpp/pull/17570)
- [PR #17425 - Anthropic Messages API 支持（初版）](https://github.com/ggml-org/llama.cpp/pull/17425)
- [HuggingFace 官方博客说明](https://huggingface.co/blog/ggml-org/anthropic-messages-api-in-llamacpp)
- [官方单元测试](https://github.com/ggml-org/llama.cpp/blob/7cadbfce/tools/server/tests/unit/test_compat_anthropic.py)

---

#### 完整的适配层代码（adapter.py）

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse, StreamingResponse
import httpx
import json
import os

app = FastAPI()

# 从环境变量读取 llama.cpp 服务器地址
LLAMA_CPP_BASE_URL = os.getenv("LLAMA_CPP_BASE_URL", "http://localhost:8000/v1")

def convert_anthropic_to_openai(payload: dict) -> dict:
    """
    将 Anthropic 格式请求转换为 OpenAI 格式
    
    Anthropic 格式：
    {
        "model": "claude-3-opus",
        "max_tokens": 1024,
        "messages": [{"role": "user", "content": "..."}],
        "system": "You are a helpful assistant",
        "temperature": 0.7
    }
    
    OpenAI 格式：
    {
        "model": "gpt-4",
        "max_tokens": 1024,
        "messages": [{"role": "system", "content": "..."}, {"role": "user", "content": "..."}],
        "temperature": 0.7
    }
    """
    openai_payload = {}
    
    # 映射基础参数
    openai_payload["model"] = payload.get("model", "llama-2")
    openai_payload["max_tokens"] = payload.get("max_tokens", 1024)
    openai_payload["temperature"] = payload.get("temperature", 0.7)
    openai_payload["top_p"] = payload.get("top_p", 1.0)
    
    # 处理消息
    messages = payload.get("messages", [])
    
    # 如果有 system 参数，将其作为第一条消息插入
    if "system" in payload:
        system_message = {
            "role": "system",
            "content": payload["system"]
        }
        messages = [system_message] + messages
    
    openai_payload["messages"] = messages
    
    # 流式标志
    if "stream" in payload:
        openai_payload["stream"] = payload["stream"]
    
    return openai_payload

def convert_openai_to_anthropic(openai_response: dict) -> dict:
    """
    将 OpenAI 格式响应转换为 Anthropic 格式
    
    OpenAI 格式：
    {
        "choices": [{"message": {"role": "assistant", "content": "..."}, "finish_reason": "stop"}],
        "usage": {"prompt_tokens": 10, "completion_tokens": 20},
        "model": "gpt-4"
    }
    
    Anthropic 格式：
    {
        "content": [{"type": "text", "text": "..."}],
        "usage": {"input_tokens": 10, "output_tokens": 20},
        "model": "claude-3-opus",
        "stop_reason": "end_turn"
    }
    """
    anthropic_response = {}
    
    # 提取内容
    if "choices" in openai_response and len(openai_response["choices"]) > 0:
        message_content = openai_response["choices"][0].get("message", {}).get("content", "")
        anthropic_response["content"] = [
            {
                "type": "text",
                "text": message_content
            }
        ]
    
    # 映射用量
    if "usage" in openai_response:
        anthropic_response["usage"] = {
            "input_tokens": openai_response["usage"].get("prompt_tokens", 0),
            "output_tokens": openai_response["usage"].get("completion_tokens", 0)
        }
    
    # 其他字段
    anthropic_response["model"] = openai_response.get("model", "llama-2")
    
    # 映射完成原因
    finish_reason = openai_response.get("choices", [{}])[0].get("finish_reason", "stop")
    if finish_reason == "stop":
        anthropic_response["stop_reason"] = "end_turn"
    elif finish_reason == "length":
        anthropic_response["stop_reason"] = "max_tokens"
    else:
        anthropic_response["stop_reason"] = finish_reason
    
    return anthropic_response

@app.post("/anthropic/v1/messages")
async def anthropic_messages(request: Request):
    """
    Anthropic API 兼容端点
    
    使用示例：
    curl http://localhost:8001/anthropic/v1/messages \\
      -H "Content-Type: application/json" \\
      -H "x-api-key: not-needed" \\
      -d '{
        "model": "claude-3-opus",
        "max_tokens": 100,
        "messages": [{"role": "user", "content": "Hello"}]
      }'
    """
    try:
        anthropic_payload = await request.json()
        
        # 转换为 OpenAI 格式
        openai_payload = convert_anthropic_to_openai(anthropic_payload)
        
        # 发送到 llama.cpp
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{LLAMA_CPP_BASE_URL}/chat/completions",
                json=openai_payload,
                timeout=300.0
            )
            
            openai_response = response.json()
            
            # 转换为 Anthropic 格式
            anthropic_response = convert_openai_to_anthropic(openai_response)
            
            return JSONResponse(content=anthropic_response)
    
    except Exception as e:
        return JSONResponse(
            status_code=500,
            content={"error": str(e)}
        )

@app.get("/anthropic/v1/models")
async def list_models():
    """列出可用模型"""
    return JSONResponse(content={
        "models": [
            {"id": "llama-2", "type": "text"},
            {"id": "llama-2-70b", "type": "text"},
            {"id": "mistral", "type": "text"}
        ]
    })

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
```

#### 启动方式

**方式 1：直接运行**

```bash
# 终端 1：启动 llama.cpp 服务器（OpenAI 兼容）
./server -m model.gguf -ngl 35 --port 8000

# 终端 2：启动适配层（Anthropic 兼容）
python adapter.py

# 现在你有两个 API 接口可用
# OpenAI:    http://localhost:8000/v1
# Anthropic: http://localhost:8001/anthropic/v1
```

**方式 2：Docker Compose（推荐）**

**Dockerfile.adapter**：
```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN pip install fastapi uvicorn httpx

COPY adapter.py .

EXPOSE 8001

CMD ["python", "adapter.py"]
```

**docker-compose.yml**：
```yaml
version: '3.8'

services:
  # llama.cpp 服务器（OpenAI 兼容接口）
  llama-cpp-openai:
    build: ./llama.cpp
    ports:
      - "8000:8000"
    volumes:
      - ./models:/models
    command: >
      ./server 
      -m /models/model.gguf 
      -ngl 35 
      --host 0.0.0.0 
      --port 8000

  # Anthropic 适配层
  anthropic-adapter:
    build:
      context: .
      dockerfile: Dockerfile.adapter
    ports:
      - "8001:8001"
    depends_on:
      - llama-cpp-openai
    environment:
      - LLAMA_CPP_BASE_URL=http://llama-cpp-openai:8000/v1
```

**启动**：
```bash
docker-compose up -d

# 查看日志
docker-compose logs -f

# 验证两个接口都在运行
curl http://localhost:8000/v1/models      # OpenAI
curl http://localhost:8001/anthropic/v1/models  # Anthropic
```

#### 客户端使用示例

**Python - OpenAI 接口**：
```python
import openai

openai.api_base = "http://localhost:8000/v1"
openai.api_key = "not-needed"

response = openai.ChatCompletion.create(
    model="llama-2",
    messages=[{"role": "user", "content": "Hello, how are you?"}],
    temperature=0.7,
    max_tokens=100
)

print(response.choices[0].message.content)
```

**Python - Anthropic 接口**：
```python
import anthropic

client = anthropic.Anthropic(
    api_key="not-needed",
    base_url="http://localhost:8001/anthropic/v1"
)

message = client.messages.create(
    model="llama-2",
    max_tokens=100,
    messages=[
        {"role": "user", "content": "Hello, how are you?"}
    ]
)

print(message.content[0].text)
```

**cURL - OpenAI 接口**：
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2",
    "messages": [{"role": "user", "content": "Hi"}],
    "max_tokens": 100
  }'
```

**cURL - Anthropic 接口**：
```bash
curl http://localhost:8001/anthropic/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: not-needed" \
  -d '{
    "model": "llama-2",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Hi"}]
  }'
```

#### 参数映射对照表

| 功能 | Anthropic | OpenAI | 说明 |
|------|-----------|--------|------|
| **Base URL** | `/anthropic/v1` | `/v1` | 适配层处理转换 |
| **模型** | `model` | `model` | 直接传递 |
| **消息** | `messages` | `messages` | 格式兼容 |
| **系统提示** | `system` 参数 | `messages[0]` role 为 system | 插入为第一条消息 |
| **Token 限制** | `max_tokens` | `max_tokens` | 直接传递 |
| **温度** | `temperature` | `temperature` | 直接传递 |
| **核采样** | `top_p` | `top_p` | 直接传递 |
| **输入 Token** | `input_tokens` | `prompt_tokens` | 字段名映射 |
| **输出 Token** | `output_tokens` | `completion_tokens` | 字段名映射 |
| **完成原因** | `stop_reason` | `finish_reason` | 值映射（end_turn ↔ stop） |

#### 与 DeepSeek 的完整对比

| 特性 | DeepSeek | llama.cpp |
|------|---------|----------|
| **OpenAI Base URL** | `https://api.deepseek.com/v1` | `http://localhost:8000/v1` |
| **Anthropic Base URL** | `https://api.deepseek.com/anthropic` | `http://localhost:8001/anthropic/v1` |
| **OpenAI 接口** | 原生支持 | 原生支持 |
| **Anthropic 接口** | 原生支持 | 适配层支持 |
| **API Key** | 必需 | 不需要 |
| **部署方式** | 云服务 | 本地/Docker |
| **灵活性** | 服务器端管理 | 客户端可组合 |

---

## 参考资源

- 官方 GitHub：https://github.com/ggerganov/llama.cpp
- 文档：https://github.com/ggerganov/llama.cpp#description
- 模型库：https://huggingface.co/models?search=gguf
- API 文档：http://localhost:8000/docs（本地服务器）

---

**最后更新**：2026-07-21
