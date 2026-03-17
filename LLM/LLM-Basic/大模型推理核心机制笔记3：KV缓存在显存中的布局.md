这一块我们直接上**硬核工程视角**，讲到接近 CUDA / vLLM 实现层。
#### 先思考以下几个问题：
- [ ] Token 在显存中表示的长度是一样的吗？
    
- [ ] 为什么 KV Cache 不使用一个大连续的 Tensor？
    
- [ ] Block / Page 的设计如何避免显存碎片？
    
- [ ] 当 batch 动态变化时，KV Cache 如何高效支持？
    
- [ ] decode 时 GPU 如何访问分散的 KV Cache？
    
- [ ] coalesced memory access 为什么对 attention 性能关键？

---

# 📦 一、先打破一个误解：KV Cache ≠ 一个大连续 Tensor

很多资料写成：

```text
[batch, heads, seq_len, head_dim]
```

👉 这是 **逻辑视图（logical view）**，方便理解模型结构，但在真实显存中：

```text
❌ 不是一整块连续内存
❌ 也不是简单 append tensor
```

------

# 🧠 二、真实布局：Block / Page 结构（vLLM 思想）

KV Cache 在显存中通常是：

```text
KV Cache = Blocks（页）组成的池
```

------

## 1️⃣ Block（核心单位）

一个 block 的形状：

```text
Block:
  K: [num_heads, block_size, head_dim]
  V: [num_heads, block_size, head_dim]
```

举例：

```text
num_heads = 32
head_dim = 128
block_size = 16 tokens
```

一个 block：

```text
K: 32 × 16 × 128
V: 32 × 16 × 128
```

------

## 2️⃣ 显存中的实际结构

```text
GPU Memory:

[Block0][Block1][Block2][Block3]...[BlockN]
```

👉 类似操作系统的 **内存分页（paging）**。

------

## 3️⃣ 每个请求的 KV Cache 并非连续

而是分散在不同 block 中：

```text
Request A → [Block3, Block7, Block20]
Request B → [Block1, Block2]
```

用表记录：

```text
Block Table（非常关键）

Request A:
  token 0-15   → Block3
  token 16-31  → Block7
  token 32-47  → Block20
```

------

# 🔧 三、为什么要这样设计？

### ❌ 传统方案（连续内存）

```text
malloc(seq_len × ...)
```

问题：

1. 扩展困难（需要 realloc）
2. 删除导致碎片
3. batch 动态变化困难

------

### ✅ Block / Page 方案

优点：

```text
✔ 动态扩展（追加 block）
✔ 无碎片
✔ 支持 continuous batching
✔ 易于复用
```

------

# ⚙️ 四、CUDA 访问 KV Cache 的真实方式

## 1️⃣ Decode 时访问流程

生成第 t 个 token 时：

```text
Q_t × K_cache^T
```

但 K_cache 是分散的：

```text
K_cache = [Block3, Block7, Block20]
```

------

## 2️⃣ GPU 实际做的事情

```text
for block in block_table:
    load K_block into shared memory
    compute partial attention
```

本质是 **分块加载 + 分块计算**。

------

## 3️⃣ 关键优化点

### 🔹 coalesced memory access（合并访存）

Block 内部布局通常是：

```text
[head, token, dim]
```

优化为：

```text
[head, dim, token]
```

保证连续线程访问连续内存，从而提高带宽利用率。

------

# 🧱 五、KV Cache 在显存中的真实排布

## 1️⃣ 常见 layout（优化版）

```text
K_cache:
[block_id, head, head_dim, block_size]
```

或

```text
[block_id, head, block_size, head_dim]
```

优势：

- 方便 vectorized load（float4 / float8）
- 提高 memory throughput

------

## 2️⃣ Warp 访问模式（重点）

- 一个 warp = 32 threads
- 每个 thread 负责 head_dim 的一部分
- 布局不合理 → memory access 非连续 → 带宽浪费

------

# 🔄 六、Append 新 token 时发生什么？

1. **Block 未满** → 写入当前 block
2. **Block 满了** → 分配新 block，并更新 block table
3. **不会发生**：

```text
❌ realloc 整个 KV Cache
❌ memcpy 大块数据
```

这是性能关键。

------

# 🧠 七、Fragmentation 是怎么避免的？

### ❌ 传统

```text
申请：100 tokens
释放
申请：80 tokens
→ 内存碎片
```

### ✅ Block 方案

```text
所有 block 等大小，统一管理
```

类似 **buddy allocator / slab allocator**。

------

# 🚀 八、PagedAttention（vLLM 核心）

核心思想：

```text
Attention 不再假设 KV 是连续的
而是按 block table 去 gather
```

实现：

```text
for block in block_table:
    attention(Q, K_block, V_block)
```

优势：

```text
✔ 支持非连续 KV Cache
✔ 支持动态 batch
✔ 显存利用率极高
```

------

# ⚠️ 九、真实工程中的坑

1. **block_size 选错**：太小 → 调度开销大；太大 → 内存浪费
2. **layout 不合理**：memory 不连续 → TPOT 飙升
3. **KV Cache 跨 GPU**：需要 all-gather → decode 变慢
4. **cache miss（prefix cache）**：无法复用 block

------

# 🧩 十、一句话总结

```text
KV Cache 在显存中 ≠ 一个 tensor
而是：
👉 一组固定大小的 block
👉 通过 block table 组织
👉 按需分配、按块访问
👉 由 CUDA kernel 分块计算 attention
```

------

# 🧠 十一、源码级回答示例

> 在实际推理系统中，KV Cache 通常不会以一个连续 tensor 存储，而是采用分页或 block 形式进行管理。每个 block 存储固定数量 token 的 K/V 张量，不同请求通过 block table 记录其 KV Cache 所在的 block 列表。在 decode 阶段，attention kernel 会根据 block table 分块加载 KV 数据并计算，从而避免大规模内存拷贝并提升显存利用率。这类似操作系统分页机制，也是 vLLM PagedAttention 的核心思想。

------

# 🎯 进一步深入建议

- KV Cache 的 CUDA kernel（thread/block mapping）
- FlashAttention v2 的 tile 设计
- 为什么 decode 是 memory-bound（roofline model）
- 手写一个最简 KV Cache allocator

------

下一步可以让 AI 画一张：

👉 **“KV Cache + Block Table + CUDA kernel 访问路径图（源码级理解）”**

------

> ### Q：token 再显存中占用的长度是一样的吗？
> 是的，在模型内部每个 token 都会被表示为固定维度的向量，例如 hidden_dim 或拆分为多头后的 [num_heads, head_dim]。因此在 KV Cache 中，每个 token 的 Key 和 Value 张量大小是固定的。不过不同请求的 token 数量（序列长度）是可变的，因此整体 KV Cache 大小会随序列长度变化。详情见《[大模型推理核心机制笔记4：token在显存中的表示方式](./大模型推理核心机制笔记4：token在显存中的表示方式.md)》
