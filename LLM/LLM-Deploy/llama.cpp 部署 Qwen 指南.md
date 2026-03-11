# llama.cpp 部署 Qwen 指南

> [!Caution]
>
> 请确保操作系统中已经安装了 **NVIDIA 驱动** 和 **CUDA Toolkits**。对于 50 系显卡，推荐安装驱动版本 ≥ 575，CUDA Tookit 版本 ≥ 12.8。

## 一、安装 llama.cpp（开启 CUDA）

### 1.1 安装依赖

```shell
# 安装依赖并编译（有 GPU 用 CUDA=ON，纯 CPU 改成 OFF）
apt-get update
apt-get install pciutils build-essential cmake curl libcurl4-openssl-dev libssl-dev -y
```

### 1.2 编译 llama.cpp

```shell
mkdir -p ~/Workspace && cd ~/Workspace
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build \
  -DBUILD_SHARED_LIBS=OFF \ 
  -DGGML_CUDA=ON \
  -DLLAMA_BUILD_TOOLS=ON
cmake --build build --config Release --clean-first  \
  # --target llama-cli llama-mtmd-cli llama-server llama-gguf-split llama-quantize \ # (可选)
  -j
```

## 二、准备 Python 环境

使用 [conda](https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/)、[uv](https://hellowac.github.io/uv-zh-cn/getting-started/installation/) 或 [micromamba](https://mamba.readthedocs.io/en/latest/installation/micromamba-installation.html) 创建虚拟环境（这里以 conda 为例）：

```shell
conda create -n qwen_deploy python=3.10 -y
conda activate qwen_deploy
# 安装必要依赖包
pip install torch transformers sentencepiece safetensors openai
```

## 三、运行模型

### 3.1 方案一：先手动下载模型再运行

#### 3.1.1 下载模型

按照 [Hugging Face](https://huggingface.co/) 国内镜像站 https://hf-mirror.com/ 中提供的下载方式，进行配置（推荐使用 `hfd.sh`）:

**1) 下载 hfd**

```shell
wget https://hf-mirror.com/hfd/hfd.sh
chmod a+x hfd.sh
```

**2) 设置环境变量**
*Linux*

```shell
export HF_ENDPOINT=https://hf-mirror.com
```

> [!TIP]
>
> 可以将环境变量配置命令添加到 `.bashrc` 中：
>
> ```shell
> echo "export HF_ENDPOINT=https://hf-mirror.com" >> ~/.bashrc
> ```

**3) 下载模型**

> [!Important]
>
> llama.cpp 使用的模型文件格式是 GGUF 格式，所以优先在 Hugging Face 上面搜索 GGUF 格式的模型。如果没有找到，可以先下载其他格式，然后通过 *3.1.2* 的方法进行格式转换。

```shell
mkdir -p ~/Workspace/models
./hfd.sh hfd Qwen/Qwen3.5-27B-FP8 --hf_username YOUR_HF_USERNAME --hf_token hf_*** --local-dir ~/Workspace/models/Qwen
```

#### 3.1.2 转换模型为 GGUF 格式

如果下载的是 **Transformers 格式（FP8 safetensors 分片）**，而 **llama.cpp 不能直接加载 safetensors**，必须先转换为 **GGUF 格式** 才能在 llama.cpp 中运行。

1）进入 llama.cpp 目录：

```
cd ~/Workspace/llama.cpp
```

2）执行转换，27B-F16 非常大（>50GB），建议**量化**：

```shell
python3 convert_hf_to_gguf.py \
  ~/Workspace/models/Qwen/Qwen3.5-27B-FP8 \
  --outfile ~/Workspace/models/Qwen/qwen3.5-27b-q5_k_m.gguf \
  --outtype q5_k_m   # 这里进行了量化
```

> 常用量化等级：
>
> | 量化   | 显存需求 | 推荐       |
> | ------ | -------- | ---------- |
> | q8_0   | 高       | 质量最好   |
> | q6_k   | 中高     | 很稳       |
> | q5_k_m | 中       | 推荐       |
> | q4_k_m | 较低     | 性价比最高 |

> [!Tip]
>
> **也可以先转换后量化：**
>
> 1. 先转换成 FP16:
>
>    ```shell
>    python3 convert_hf_to_gguf.py \
>      ~/Workspace/Qwen3.5-27B-FP8 \
>      --outfile qwen3.5-27b-f16.gguf \
>      --outtype f16
>    ```
>
> 2. 再进行量化：
>
>    ```shell
>    ./build/bin/llama-quantize \
>      qwen3.5-27b-f16.gguf \
>      qwen3.5-27b-q4_k_m.gguf \
>      q4_k_m
>    ```

#### 3.1.3 运行模型

这里我们打算通过 API 调用模型，所以启动将 llama 作为服务器运行，执行命令：

```shell
./build/bin/llama-server \
  -m ~/Workspace/models/Qwen/qwen3.5-27b-q5_k_m.gguf \
  -ngl 999 \
  -c 8192 \
  -b 1024 \
  --threads 16 \
  --chat-template chatml \
  --host 0.0.0.0 \
  --port 8000 \
  --parallel 2
```

**参数说明：**

- `-ngl 999` → 全部层上 GPU
- `-c 8192` → 上下文长度
- `-b 1024` → 批大小，影响吞吐
- `--threads 16` → 线程数量（≤ CPU 核心数）
- `--chat-template chatml` → 指定对话格式模板
- `--host 0.0.0.0` → 局域网可访问
- `--port 8000` → API 端口
- `--parallel 2` → 并发请求

启动成功后会看到：

```shell
HTTP server listening at http://0.0.0.0:8000
```

> [!TIP]
>
> 可以将上面的命令保存为一个 bash 脚本（`start_qwen3.5-27B.sh`），脚本建议放置在 `~/Workspace/my_scripts/` 路径中：
>
> ```bash
> #!/usr/bin/env bash
> ~/Workspace/llama.cpp//build/bin/llama-server \
>   -m ~/Workspace/models/Qwen/qwen3.5-27b-q5_k_m.gguf \
>   -ngl 999 \
>   -c 8192 \
>   -b 1024 \
>   --threads 16 \
>   --chat-template chatml \
>   --host 0.0.0.0 \
>   --port 8000 \
>   --parallel 2
> ```
>
> 更改脚本权限：
>
> ```bash
> chmod a+x ~/Workspace/my_scripts/start_qwen3.5-27B.sh
> ```
>
> 执行脚本：
>
> ```bash
> ~/Workspace/my_scripts/start_qwen3.5-27B.sh
> ```

#### 3.1.4 测试 API：

1）通过网页访问：

打开浏览器，输入 `http://localhost:8000`（本地访问）或 `http://你的电脑IP:8000`（局域网访问），进入 llama.cpp 的网页交互界面。

2）使用 curl：

```shell
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen",
    "messages": [
      {"role": "user", "content": "你好，请介绍一下你自己"}
    ],
    "temperature": 0.7
  }'
```

3）使用 Python 调用（OpenAI SDK 兼容）：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="sk-no-key-required"
)

resp = client.chat.completions.create(
    model="qwen",
    messages=[
        {"role": "user", "content": "写一个快速排序"}
    ],
    temperature=0.7,
)

print(resp.choices[0].message.content)
```

4）可以使用下面的 Python 脚本对模型速度进行测试：

```bash
python speed_test.py -q "请详细解释PagedAttention原理" -m 1024 -t 0.6
```

### 3.2 方案二：直接运行，在线下载（要求网络稳定）

#### 3.2.1 Thinking 模式

```text
# 精确编码任务用这个（temperature=0.6，更稳定）
export LLAMA_CACHE="~/Workspace/models/unsloth/Qwen3.5-35B-A3B-GGUF"
./llama.cpp/llama-cli \\
  -hf unsloth/Qwen3.5-35B-A3B-GGUF:MXFP4_MOE \\
  --ctx-size 8192 \\
  --temp 0.6 \\
  --top-p 0.95 \\
  --top-k 20 \\
  --min-p 0.00

# 通用任务用这个（temperature=1.0，更有创意）
export LLAMA_CACHE="~/Workspace/models/unsloth/Qwen3.5-35B-A3B-GGUF"
./llama.cpp/llama-cli \\
  -hf unsloth/Qwen3.5-35B-A3B-GGUF:MXFP4_MOE \\
  --ctx-size 8192 \\
  --temp 1.0 \\
  --top-p 0.95 \\
  --top-k 20 \\
  --min-p 0.00
```

#### 3.2.2 非思考模式（更快响应）

```text
# 不需要深度推理时，关掉 thinking 模式
export LLAMA_CACHE="~/Workspace/models/unsloth/Qwen3.5-35B-A3B-GGUF"
./llama.cpp/llama-cli \\
  -hf unsloth/Qwen3.5-35B-A3B-GGUF:MXFP4_MOE \\
  --ctx-size 8192 \\
  --temp 0.7 \\
  --top-p 0.8 \\
  --top-k 20 \\
  --min-p 0.00 \\
  --chat-template-kwargs "{\\"enable_thinking\\": false}"
```

## 四、做成后台服务（推荐）

用 `tmux`：

```
tmux new -s qwen
```

或者写 systemd 服务。

## 附录

### 1. 模型速度测试脚本

```python
import time
import sys
import argparse
from openai import OpenAI

# ======== 默认配置 ========
BASE_URL = "http://localhost:8000/v1"
MODEL_NAME = "qwen"
DEFAULT_PROMPT = "请详细解释一下什么是 Transformer 架构。"
DEFAULT_TEMPERATURE = 0.7
DEFAULT_MAX_TOKENS = 512
# ============================


def main():
    parser = argparse.ArgumentParser(description="LLM Speed Test Script")
    parser.add_argument(
        "-q", "--question",
        type=str,
        help="输入要提问的问题"
    )
    parser.add_argument(
        "-t", "--temperature",
        type=float,
        default=DEFAULT_TEMPERATURE,
        help="采样温度 (默认 0.7)"
    )
    parser.add_argument(
        "-m", "--max_tokens",
        type=int,
        default=DEFAULT_MAX_TOKENS,
        help="最大生成 token 数 (默认 512)"
    )

    args = parser.parse_args()

    # 如果没有传 question，则进入交互模式
    if args.question:
        prompt = args.question
    else:
        prompt = input("请输入问题: ").strip()
        if not prompt:
            prompt = DEFAULT_PROMPT

    client = OpenAI(
        base_url=BASE_URL,
        api_key="sk-no-key-required"
    )

    print("\n===== 开始请求 =====\n")

    start_time = time.time()
    first_token_time = None
    generated_text = ""

    stream = client.chat.completions.create(
        model=MODEL_NAME,
        messages=[{"role": "user", "content": prompt}],
        temperature=args.temperature,
        max_tokens=args.max_tokens,
        stream=True
    )

    for chunk in stream:
        delta = chunk.choices[0].delta
        if delta.content:
            if first_token_time is None:
                first_token_time = time.time()
            token = delta.content
            generated_text += token
            print(token, end="", flush=True)

    end_time = time.time()

    print("\n\n===== 统计信息 =====")

    total_time = end_time - start_time
    ttft = first_token_time - start_time if first_token_time else 0

    approx_tokens = len(generated_text.split())
    speed = approx_tokens / total_time if total_time > 0 else 0

    print(f"Time to First Token : {ttft:.2f} s")
    print(f"Total Time          : {total_time:.2f} s")
    print(f"Approx Tokens       : {approx_tokens}")
    print(f"Approx Speed        : {speed:.2f} tokens/s")


if __name__ == "__main__":
    main()
```

