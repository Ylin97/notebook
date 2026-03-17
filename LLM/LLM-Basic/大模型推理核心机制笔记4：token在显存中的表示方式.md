# ✅ 一句话答案

👉 **在显存中，每个 token 占用的“长度（向量维度）是固定的”，但“数量（序列长度）是可变的”。**

---

# 一、token 在模型里的“表示长度”是固定的

无论你输入什么 token：

```text
"hello"
"你好"
"🚀"
```

在进入模型后，都会变成：

```text
embedding: [hidden_dim]
```

👉 比如：

- LLaMA-7B：hidden_dim = 4096
    
- Qwen：hidden_dim ≈ 4096 / 8192
    

---

## KV Cache 中也是一样

每个 token 在 KV Cache 中对应：

```text
K: [num_heads, head_dim]
V: [num_heads, head_dim]
```

而：

```text
hidden_dim = num_heads × head_dim
```

👉 所以：

```text
每个 token 的 KV 大小是固定的 ✔
```

---

# 二、为什么“感觉长度不一样”？（常见误解）

---

## ❗误解来源 1：文本长度 ≠ token 数量

```text
"hello" → 1 token（可能）
"internationalization" → 多个 token
```

👉 tokenizer 决定 token 数量，而不是字符数

---

## ❗误解来源 2：不同请求的 seq_len 不同

```text
请求A：10 tokens
请求B：1000 tokens
```

👉 KV Cache 总大小不同，但：

```text
单个 token 的大小仍然相同 ✔
```

---

# 三、KV Cache 里一个 token 占多少显存？

---

## 单个 token 的 KV 大小：

```text
size_per_token =
2 × num_layers × num_heads × head_dim × dtype_size
```

解释：

- 2 → K 和 V
    
- num_layers → 每层都有 KV
    
- dtype_size → FP16=2字节，FP8=1字节
    

---

## 举个真实例子（你可以面试直接说）

假设：

```text
num_layers = 32
num_heads = 32
head_dim = 128
dtype = FP16 (2 bytes)
```

那么：

```text
单 token KV =
2 × 32 × 32 × 128 × 2 bytes
≈ 524,288 bytes ≈ 512 KB
```

---

## 👉 关键结论

```text
1 token ≈ 几百 KB（非常大！）
```

👉 所以：

```text
1000 tokens ≈ 500 MB 显存
```

---

# 四、在 Block / Page 结构中的体现

回到我们刚才讲的 block：

```text
Block:
  [num_heads, block_size, head_dim]
```

👉 这里：

- 每个 token 的“宽度” = head_dim × num_heads（固定）
    
- block_size = token 数量（可变块大小，但通常固定）
    

---

# 五、真正“可变”的是什么？

---

## 1️⃣ 序列长度（seq_len）

```text
token 数量不同
→ KV Cache 总大小不同
```

---

## 2️⃣ batch size

```text
多个请求
→ KV Cache × batch
```

---

## 3️⃣ 数据类型

```text
FP16 → 2 bytes
FP8 → 1 byte
INT8 → 1 byte
```

👉 会影响：

```text
单 token 显存占用 ✔
```

---

## 4️⃣ 模型结构（MQA / GQA）

---

### 普通 attention：

```text
每个 head → 独立 KV
```

---

### MQA：

```text
所有 head → 共享 KV
```

👉 结果：

```text
单 token KV 大小 ↓↓↓
```

---

# 六、一个更底层的理解（非常重要）

---

## token 在显存中的“真实形态”

👉 不是：

```text
"hello"
```

👉 而是：

```text
float16 数组
```

例如：

```text
[0.123, -1.23, 0.98, ...]  （长度 = hidden_dim）
```

---

## KV Cache 中：

```text
K_token = float16[num_heads][head_dim]
V_token = float16[num_heads][head_dim]
```

👉 所以：

```text
每个 token = 固定大小的“向量块”
```

---

# 七、你可以这样回答面试官（标准答案）

---

👉 **Q：token 在显存中长度一样吗？**

> 是的，在模型内部每个 token 都会被表示为固定维度的向量，例如 hidden_dim 或拆分为多头后的 [num_heads, head_dim]。因此在 KV Cache 中，每个 token 的 Key 和 Value 张量大小是固定的。不过不同请求的 token 数量（序列长度）是可变的，因此整体 KV Cache 大小会随序列长度变化。

---

# 八、延伸一个“加分点”（建议掌握）

---

👉 为什么 KV Cache 会成为瓶颈？

因为：

```text
token size（固定） × token 数量（增长）
```

👉 导致：

```text
显存线性增长
```

而 attention：

```text
需要访问所有历史 KV
```

👉 导致：

```text
memory bandwidth 成为瓶颈
```

---

# 🎯 一句话总结（建议记住）

```text
token 在显存中 = 固定大小的向量（宽度固定）
KV Cache 总大小 = token 数量 × 单 token 大小（长度可变）
```
