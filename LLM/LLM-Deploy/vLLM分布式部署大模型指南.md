# vLLM分布式部署大模型指南

## 一、准备工作

### 1.1 Linux系统安装

为了达到最佳性能，建议使用 Linux 系统部署大模型，可以选择 Ubuntu、Fedora、Debian、ArchLinux、OpenSUSE、RHEL 等任意发行版。这里以 Ubuntu 24.04 LTS 版为例，具体安装教程见《[手把手教你安装 Ubuntu 24.04 LTS 桌面操作系统](https://zhuanlan.zhihu.com/p/695298037)》。安装系统时需要注意以下几点：

- [x] `Install recommended proprietary software?` 这一步建议勾选 `Install third-party software for graphics and Wi-Fi hardware`，这样在系统安装时会自动安装显卡驱动和 Wi-Fi 驱动。
- [x] 分区时建议创建交换分区，分区大小设置为与物理内存大小相同。

### 1.2 固定主机 IP

为了保持集群稳定，需要对集群中所有主机的 IP 进行固定，具体方法如下：

获取当前 IP、网关、网卡信息：

```shell
ip route get 1.1.1.1

# 输出结果类似于下面，其中 10.60.11.1 为默认网关，10.60.11.51 为主机当前 IP
# 1.1.1.1 via 10.60.11.1 dev enp130s0 src 10.60.11.51 uid 1000
```

获取子网掩码：

```shell
ip addr

# 输出结果类似于下面，这里是通过有线网连接（enp130s0 网卡），从 10.60.11.51/24 可知子网掩码占 24 位，所以为 255.255.255.0
# 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
#     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
#     inet 127.0.0.1/8 scope host lo
#        valid_lft forever preferred_lft forever
#     inet6 ::1/128 scope host noprefixroute
#        valid_lft forever preferred_lft forever
# 2: enp130s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
#     link/ether bc:fc:e7:eb:d5:1e brd ff:ff:ff:ff:ff:ff
#     inet 10.60.11.51/24 brd 10.60.11.255 scope global noprefixroute enp130s0
#        valid_lft forever preferred_lft forever
# 3: wlp129s0f0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
#     link/ether 80:c0:1e:73:98:94 brd ff:ff:ff:ff:ff:ff
```

综上，我们得到如下数据：

- Address: `10.60.11.51`
- Netmask: `255.255.255.0`
- Gateway: `10.60.11.1`

接下来，把这些数据填入 Ubuntu 设置：

进入 `Settings → Network → ⚙️ → IPv4`，将 `Automatic (DHCP)` 改为 `Manual`，然后填入下列内容：

| 项      | 填写                       |
| ------- | -------------------------- |
| Address | 当前IP或新的IP             |
| Netmask | 255.255.255.0              |
| Gateway | 10.60.11.1                 |
| DNS     | 114.114.114.114 或 8.8.8.8 |

点击 `Apply` 后，需要执行下面的操作来应用设置：

- 断开网络
- 再重新连接

运行下面的命令来查看 IP 是否设置成功。如果没有设置成功，可能需要重启电脑：

```shell
ip addr
```

> [!Tip]
>
> 如果要做 **多机器推理节点**，最好：
>
> - 每台机器固定 IP
> - IP 按节点编号
>
> 例如：
>
> ```
> 192.168.31.101  llm-node1
> 192.168.31.102  llm-node2
> 192.168.31.103  llm-node3
> ```
>
> 再写进：
>
> ```
> /etc/hosts
> ```
>
> 这样 **Ray 集群通信会稳定很多**。

### 1.3 配置 ssh 和防火墙

在集群中的 **每台电脑** 上执行下面的步骤，这里以两台主机为例：`Master (10.60.11.209)` 和 `Worker (10.60.11.51)`。

#### 1）配置 ssh

首先安装 openssh-client 和 openssh-server：

```shell
sudo apt update && sudo apt install openssh-client openssh-server -y
```

然后修改 ssh 默认端口号：

```shell
# 编辑 /etc/ssh/sshd_config
nano /etc/ssh/sshd_config
```

找到其中 `#Port 22` 所在的行，去掉前面的 `#`，并将默认端口号改成 `20222`（其他端口号也可以）：

```
#   systemctl restart ssh.socket
#
Port 20222
#AddressFamily any
#ListenAddress 0.0.0.0
```

开启 ssh 服务并设置开机自启动：

```shell
# 设置开机自启动并立即启动服务
sudo systemctl enable ssh --now

# 检查 ssh 是否已经启动成功，如果启动成功会显示： Active: active (running)
sudo systemctl status ssh

# 如果服务是状态是 deactive，则尝试执行下面的命令：
sudo systemctl start ssh
```

#### 2）配置防火墙

启用 ufw 防火墙（注意！其他 Linux 发行版可能是不同的防火墙，具体需上网查阅）：

```shell
sudo ufw enable
```

开放 ssh 端口号和 Ray 所需的端口号：

```shell
# 开始前面设置的 ssh 端口号
sudo ufw allow 20222/tcp
sudo ufw allow 20222/udp

# 1. 允许 Ray 的主控制端口 (Redis)
sudo ufw allow 6379/tcp

# 2. 允许 Ray 和 vLLM 的内部通信端口范围 (注意使用冒号 : 表示范围)
# vLLM 在分布式推理时需要这些端口来交换张量数据
sudo ufw allow 10000:15000/tcp
sudo ufw allow 10000:15000/udp

# 3. (可选) 如果你还需要通过网页查看 Ray 的仪表盘 (Dashboard)
sudo ufw allow 8265/tcp
```

> [!Tip]
>
> 集群通常是通过网线直连或在同一个私有路由器下，为了性能和减少麻烦，也可以**只针对另一台机器的 IP 开放所有权限**，而不是逐个开端口：
>
> **在 Master (10.60.11.209) 上执行：**
>
> ```shell
> sudo ufw allow from 10.60.11.51
> ```
>
> **在 Worker (10.60.11.51) 上执行：**
>
> ```shell
> sudo ufw allow from 10.60.11.209
> ```
>
> 这样两台机器之间就像“亲兄弟”一样完全互信，不会有任何端口阻碍。

开放端口后需要重启防火墙：

```shell
# 重载配置以确保生效
sudo ufw reload

# 验证配置是否生效
sudo ufw status numbered
```

输出应该包含类似下面的内容：

```shell
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 20222/tcp                  ALLOW IN    Anywhere
[ 2] 8000/tcp                   ALLOW IN    Anywhere
[ 3] 6379/tcp                   ALLOW IN    Anywhere
[ 4] 10000:15000/tcp            ALLOW IN    Anywhere
[ 5] 10000:15000/udp            ALLOW IN    Anywhere
[ 6] 8265/tcp                   ALLOW IN    Anywhere
[ 7] Anywhere                   ALLOW IN    10.60.11.209
[ 8] 20222/tcp (v6)             ALLOW IN    Anywhere (v6)
[ 9] 8000/tcp (v6)              ALLOW IN    Anywhere (v6)
[10] 6379/tcp (v6)              ALLOW IN    Anywhere (v6)
[11] 10000:15000/tcp (v6)       ALLOW IN    Anywhere (v6)
[12] 10000:15000/udp (v6)       ALLOW IN    Anywhere (v6)
[13] 8265/tcp (v6)              ALLOW IN    Anywhere (v6)
```

#### 3）配置集群内头节点主机与工作节点主机免密登录 ssh

分布式部署需要 Master 节点能够自动在 Worker 节点上启动进程，因此必须配置免密登录。

**a. 在 Master (10.60.11.209) 上生成密钥对：**

```shell
ssh-keygen -t rsa -b 4096  # 一路回车即可
```

**b. 将公钥发送给 Worker (10.60.11.51)：**

由于我们修改了 SSH 端口为 `20222`，需指定端口：

```shell
ssh-copy-id -p 20222 user@10.60.11.51  # 将 user 替换为真实用户名
```

**c. 验证登录：**

```shell
ssh -p 20222 user@10.60.11.51  # 如果无需密码直接进入，则成功
```

#### 4）修改 hosts 主机映射

```shell
# 1. 打开 Master 和 Worker 主机系统中的 hosts 文件
sudo nano /etc/hosts

# 2. 在末尾添加
10.60.11.209 master  # 将 master 替换为头节点的主机名
10.60.11.51  worker  # 将 worker 替换为工作节点的主机名

# 3. 测试
ping -c4 master  # 将 master 替换为头节点的主机名
ping -c4 worker  # 将 worker 替换为工作节点的主机名
```



### 1.4 安装显卡驱动和 CUDA Toolkit

> [!Warning]
>
> Ubuntu 24.04 在安装系统的时候可以选择启动第三方库，该情况下系统会自动安装最新的 NVIDIA Driver，所以可以直接跳过手动安装显卡驱动的步骤。可以运行命令 `nvidia-smi` 或 `sudo apt list *nvidia-driver*` 检查系统是否已经存在显卡驱动。

如果系统没有自动安装显卡驱动，则执行下面的命令安装：

```shell
# 更新软件源
sudo apt update

# 自动安装推荐的驱动
sudo ubuntu-drivers autoinstall

# 安装完成后重启系统
sudo reboot
```

如果上面的命令不能执行，则可以手动指定版本安装（**50 系显卡建议安装版本 ≥575+**）：

```shell
# 查看可用驱动
sudo ubuntu-drivers devices

# 从上面列出的可用驱动中选择一个安装
sudo apt update && sudo apt install nvidia-driver-590-open

# 安装完成后重启系统
sudo reboot
```

执行命令 `nvidia-smi` 检查是否能输出显卡信息，如果报错，则重新选择一个版本的驱动进行安装。

重启完成后开始安装 CUDA Toolkit:

打开英伟达官网 https://developer.nvidia.com/cuda-toolkit-archive，选择一个与已装显卡驱动匹配的版本，这里选择与 590 驱动兼容的 `12.8.1`：

> [!Tip]
>
> 安装 CUDA Toolkit 时，通常选择与 `nvidia-smi | grep CUDA` 输出结果相同的版本，但是一些高版本的驱动也兼容较低版本的 CUDA Toolkit。

**注意！安装的版本要与自己的系统相匹配！！！**这里我们选择在线安装：

![image-20260306114632453](https://markdown-gallery.oss-cn-shenzhen.aliyuncs.com/images/image-20260306114632453.png)

安装完成后，先检查 CUDA 可执行文件路径和库路径是否已存在 `~/.bashrc` 中：

```shell
# 请将命令中 cuda-12.8 替换成自己安装的 CUDA Toolkit 版本
grep -n "cuda-12.8" ~/.bashrc | grep -E "PATH|LD_LIBRARY_PATH|CUDA_HOME"
```

如果 `~/.bashrc` 中不存在，则执行下面的命令将其添加进去：

```shell
# 配置CUDA环境变量（永久生效，添加CUDA_HOME避免依赖报错）
# 请将命令中 `cuda-12.8` 替换成自己安装的 CUDA Toolkit 版本
cuda_home="/usr/local/cuda-12.8"
echo "export CUDA_HOME='$cuda_home'" >> ~/.bashrc
echo "export PATH=\"$cuda_home/bin\${PATH:+:\${PATH}}\"" >> ~/.bashrc
echo "export LD_LIBRARY_PATH=\"$cuda_home/lib64\${LD_LIBRARY_PATH:+:\${LD_LIBRARY_PATH}}\"" >> ~/.bashrc

# 立即生效环境变量
source ~/.bashrc

# 验证CUDA版本（显示release 12.8即为成功）
nvcc -V
# 示例正确输出：Cuda compilation tools, release 12.8, V12.8.89
```

因为我们已经提前安装了显卡驱动，所以 CUDA Tookit 安装说明下方的驱动安装部分可以**直接跳过**。

### 1.5 安装 Python 虚拟环境管理器

可以选择 Miniconda、uv、micromanba 等作为虚拟环境管理器。这里以 Miniconda 为例。

从[清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/)下载最新的 Miniconda：

```shell
cd ~/Downloads
wget https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash ./Miniconda3-latest-Linux-x86_64.sh
# 按提示进行安装即可，但是注意最后一步提示是否初始化 shell，最好输入 yes
```

安装完成后进行 conda 换源，执行命令：

```shell
cat << 'EOF' > ~/.condarc
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
EOF
```

关闭当前终端并重新打开一个终端窗口，执行下列命令清除旧的 conda  索引：

```shell
conda clean -i
```

### 1.6 配置模型下载工具

模型权重文件可从 [Hugging Face](https://huggingface.co/) 国内镜像站 https://hf-mirror.com/ 进行下载。按照 https://hf-mirror.com/ 中提供的下载方式，进行配置（推荐使用 `hfd.sh`）:

#### 1）下载 hfd

**a. 获取 hfd.sh 脚本，执行命令：**

```shell
wget https://hf-mirror.com/hfd/hfd.sh -O ~/.local/bin/hfd.sh
```

**b. 为脚本添加可执行权限，执行命令：**

```shell
chmod a+x ~/.local/bin/hfd.sh
```

**c. 定义 hfd 函数封装 hfd.sh 脚本，方便调用，执行命令：**

```shell
cat << 'EOF' >> ~/.bashrc
# >>> hfd function definition <<<
hfd() {
    local DEFAULT_DIR="/data/models"
    local HAS_LOCAL_DIR=0

    # 检查用户是否自己传入了 --local-dir
    for arg in "$@"; do
        if [[ "$arg" == "--local-dir" ]]; then
            HAS_LOCAL_DIR=1
            break
        fi
    done

    # 如果没有指定 --local-dir，则自动添加
    if [[ $HAS_LOCAL_DIR -eq 0 ]]; then
        ~/.local/bin/hfd.sh "$@" --local-dir "$DEFAULT_DIR"
    else
        ~/.local/bin/hfd.sh "$@"
    fi
}
# <<< hfd function definition <<<
EOF
```

#### 2）设置环境变量

*Linux*

```shell
# 将环境变量配置命令添加到 `~/.bashrc` 中
echo "export HF_ENDPOINT=https://hf-mirror.com" >> ~/.bashrc

# 重新载入 ~/.bashrc
source ~/.bashrc
```

### 1.7 创建工作目录

为了方便管理模型，创建如下工作目录：

```shell
# 模型存放目录
sudo mkdir -p /data/models && sudo chmod -R 777 /data

# 脚本存放目录
mkdir -p ~/Workspace/tool_scripts
```

## 二、部署方案

### 方案一：真机部署（性能最佳）

#### 1）安装最新版本的 vLLM

```shell
# 创建虚拟环境
conda create -n vllm python=3.12 -y 

# 激活虚拟环境
conda activate vllm 

# 安装 pytorch, vllm 和 transformers
pip install vllm transformers torch torchvision
```

> [!Note]
>
> 如果模型无法运行，可尝试安装 nightly 版本的 vLLM 和 Transformers：
>
> ```shell
> # 安装最新的 nightly 版本
> pip install -U vllm --pre --index-url https://pypi.org/simple \
>   --extra-index-url https://wheels.vllm.ai/nightly
> 
> # 更新 transformers
> pip install git+https://github.com/huggingface/transformers.git
> # 可以使用 github 加速链接：https://gh-proxy.org/https://github.com/huggingface/transformers.git
> ```

安装完成后测试 torch 是否可使用：

```shell
python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
# 输出如下内容则说明 torch 正常：
# True
# NVIDIA GeForce RTX 5090 Laptop GPU
```

#### 2）下载模型权重文件

打开  [Hugging Face](https://huggingface.co/) 或国内镜像站 https://hf-mirror.com/，搜索需要的模型，然后使用 hfd 进行下载：

```shell
hfd marksverdhei/GLM-4.7-Flash-FP8 --hf_username YOUR_HF_USERNAME --hf_token hf_*** --local-dir /data/models/marksverdhei/GLM-4.7-Flash-FP8
```

#### 3）启动 Ray 头节点和工作节点

**a. 启动头节点：**

```shell
ray start --head \
  --node-ip-address='10.60.11.209' \
  --port=6379 \
  --num-gpus=1 \
  --disable-usage-stats

# 注释说明（移到脚本开头/结尾，避免干扰参数）
# --node-ip-address：主节点内网IP（10.60.11.209）
# --port：Ray默认端口（6379）
# --num-gpus：主节点GPU数量（1张RTX 5090）
# --disable-usage-stats：关闭 Ray 的匿名使用统计（这些数据会上传给 Ray 组织）
```

**b. 启动工作节点：**

```shell
ray start --address='10.60.11.209:6379' \
  --node-ip-address='10.60.11.51' \
  --num-gpus=1
```

**c. 检查节点状态：**

```shell
ray status
```

输出结果应与下面类似：

``` 
======== Autoscaler status: 2026-03-09 20:56:08.067470 ========
Node status
---------------------------------------------------------------
Active:
 1 node_b6125412b2f9f8f51c5a3bd83ba5f746ae2daad561615d9fa38aa676
 1 node_c1e98a7ccd326177aa93475f9d7e19fbb4ef9b595c34a460f31f4dc8
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Total Usage:
 0.0/48.0 CPU
 2.0/2.0 GPU (2.0 used of 2.0 reserved in placement groups)
 0.0/1.0 TPU
 0B/81.92GiB memory
 0B/35.11GiB object_store_memory

From request_resources:
 (none)
Pending Demands:
 (no resource demands)
```

#### 4）加载模型

**设置必要的环境变量：**

```shell
# 关闭 Pytorch 可扩展显存段避免显存分配 bug，因为我们的显存已经快被占满了，所以取消扩展显存段的显存管理方式 
# （该方式会对显存段进行对齐，这样虽然能提高显存访问效率，但是可能会占用更多的显存）
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:False 
```

**启动模型：**

```shell
python -m vllm.entrypoints.openai.api_server \
    --model /data/models/marksverdhei/GLM-4.7-Flash-FP8 \
    --served-model-name GLM-4.7-Flash-FP8 \
    --tensor-parallel-size 2 \
    --distributed-executor-backend ray \
    --max-model-len 65536 \
    --max-num-batched-tokens 8192 \
    --max-num-seqs 2 \
    --gpu-memory-utilization 0.92 \
    --swap-space 20 \
    --seed 3407 \
    --host 0.0.0.0 \
    --port 8000 \
    #--kv-cache-dtype fp8 \
    
    # 可选参数：
    #--tool-call-parser glm47 \
    #--reasoning-parser glm45 \
    #--enable-auto-tool-choice 
```

> ##### 💡参数说明
>
> | 参数名  | 解释 |
> | ------- | ---- |
> | `--model` | 指定模型路径或 HuggingFace 仓库名 |
> | `--served-model-name` | API 对外暴露的模型名字，即模型 id |
> | `--tensor-parallel-size` | 张量并行数量，即模型会被拆分到 2 张 GPU 上运行 |
> | `--distributed-executor-backend` | 指定分布式执行框架 |
> | `--max-model-len` | 模型最大上下文长度，prompt + generation ≤ 32768 tokens |
> | `--max-num-batched-tokens` | 一次调度 step 最多允许处理多少 token（所有请求加起来），通常设置为  max-model-len / 8 |
> | `--max-num-seqs` | 最大并发序列数量，即同时支持多少条请求 |
> | `--gpu-memory-utilization` | 限制 GPU 显存使用比例 |
> | `--swap-space` | CPU 内存作为 KV Cache 的溢出空间，如果报错请去掉 |
> | `--seed` | 控制随机数种子，让模型推理结果可复现（使用采样策略时才有用，即 `temperature` 或 `top_p` 大于零） |
> | `--host` | API 监听地址（127.0.0.1 仅本地访问，0.0.0.0 所有机器都能访问） |
> | `--port` | API 服务端口 |
> | `--kv-cache-dtype` | FP8 加速，需要显卡支持，RTX 5090 最新的驱动已支持，但是 vllm 还不支持 `MLA Attention` |
> | `--enable-auto-tool-choice` | 开启自动工具选择，开启后模型可以自动决定是否调用工具，而不是强制调用 |
> | `--tool-call-parser` | 指定工具调用解析器 |
> | `--reasoning-parser` | 指定推理内容解析器 |
>

#### 5）测试 API

*通过 curl 测试：*

```shell
curl -X POST http://10.60.11.209:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"/home/jsti/GLM-4.7-Flash-FP8","messages":[{"role":"user","content":"你好"}]}'
```

*通过 Python 测试（OpenAI SDK 兼容）：*

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

*可以使用下面的 Python 脚本对模型速度进行测试（脚本见附件）：*

```python
python speed_test.py \
    --base-url "http://master:8000/v1" \
    --model "GLM-4.7-Flash-FP8" \
    -q "请详细解释PagedAttention原理" \
    -m 1024 \
    -t 0.6
```

### 方案二：Docker 部署

---

Docker 可以避免复杂的依赖冲突，且方便在多台主机间快速迁移环境。

#### 一、安装 NVIDIA Container Toolkit

```shell
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo systemctl restart docker
```

#### 二、启动 vLLM 容器

```shell
docker run --gpus all \
    -v ~/Workspace/models:/root/.cache/huggingface \
    -p 8000:8000 \
    --ipc=host \
    vllm/vllm-openai:latest \
    --model /root/.cache/huggingface/marksverdhei/GLM-4.7-Flash-FP8 \
    --gpu-memory-utilization 0.95
```

## 附件

### 1. GLM-4.7 建议参数配置

基于 Unsloth 对 **GLM-4.7** 系列的建议:

```python
glm_4_7_config = {
    "temperature": 0.8,
    "top_p": 0.6,  # 智谱AI推荐
    "top_k": 2,     # 智谱AI推荐
    "max_tokens": 16384,
    "repetition_penalty": 1.0
}
```

#### 不同用例的 GLM-4.7

##### 使用GLM-4.7编码

```python
# 代码生成的最佳设置
coding_config = {
    "temperature": 0.2,  # 降低以获得确定性代码
    "top_p": 0.9,
    "max_tokens": 4096
}
```

##### 使用GLM-4.7创意写作

```python
# 创意任务的最佳设置
creative_config = {
    "temperature": 1.0,  # 提高以增加创造力
    "top_p": 0.95,
    "max_tokens": 8192
}
```

##### 使用GLM-4.7进行工具使用

```python
# 启用工具调用
tool_config = {
    "temperature": 0.7,
    "tools": [...],  # 您的工具定义
    "tool_choice": "auto"
}
```

##### GLM-4.7上下文管理

通过 `MLA` 让 GLM-4.7 高效处理长上下文:

```python
# 示例: 使用GLM-4.7处理大型代码库
def analyze_codebase_with_glm(files):
    context = "\n\n".join([f"文件: {f.name}\n{f.content}" for f in files])
    
    response = glm_client.chat.completions.create(
        model="glm-4.7-flash",
        messages=[
            {"role": "system", "content": "你是一个代码审查员"},
            {"role": "user", "content": f"审查这个代码库:\n{context}"}
        ],
        max_tokens=4096
    )
    
    return response.choices[0].message.content
```

> 更多技巧可参考文章：
>
> https://www.cnblogs.com/sing1ee/p/19504922/2026-gml-4-7-flash-guide
>
> https://ascendai.csdn.net/697883d57c1d88441d8ff3f6.html

### 2. 模型速度测试脚本

```python
import time
import sys
import argparse
from openai import OpenAI

# ======== 默认配置 ========
BASE_URL = "http://localhost:8000/v1"
MODEL_NAME = "GLM-4.7-Flash-FP8"
API_KEY = "sk-no-key-required"
DEFAULT_PROMPT = "请详细解释一下什么是 Transformer 架构。"
DEFAULT_TEMPERATURE = 0.7
DEFAULT_MAX_TOKENS = 1024
# ============================


def main():
    parser = argparse.ArgumentParser(description="LLM Speed Test Script")
    parser.add_argument(
        "--base-url",
        type=str,
        default=BASE_URL,
        help="base_url"
    )
    parser.add_argument(
        "--model",
        type=str,
        default=MODEL_NAME,
        help="模型名(ID)"
    )
    parser.add_argument(
        "--api-key",
        type=str,
        default=API_KEY,
        help="模型名(ID)"
    )
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
       "--max_tokens",
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

### 3.模型速度测试脚本（异步）

```python
import aiohttp
import asyncio
import time
import json
from datetime import datetime
from typing import Dict, List

# ===================== 核心配置（适配你的环境）=====================
API_URL = "http://10.60.11.209:8000/v1/chat/completions"  # 模型服务地址
MODEL_NAME = "GLM-4.7-Flash-FP8"                        # 模型名称
MAX_TOKENS = 2048                                       # 生成最大长度
TEMPERATURE = 0.1                                       # 低随机性，保证结果可对比
CONCURRENT_NUM = 7                                      # 并发测试用例数
# ==============================================================

# 多维度测试用例
TEST_CASES: List[Dict] = [
    {
        "category": "基础问答",
        "prompt": "你好，请用100字以内介绍一下自己的核心能力",
        "description": "验证模型基础响应和自我认知"
    },
    {
        "category": "数学推理",
        "prompt": "一个水池有进水管和出水管，进水管每小时进水10立方米，出水管每小时出水6立方米。空水池先开进水阀3小时，再同时打开进出水阀，问再过几小时水池能存满80立方米水？",
        "description": "验证数学应用题推理能力"
    },
    {
        "category": "代码生成",
        "prompt": "用Python写一个函数，实现批量读取指定目录下所有.csv文件，并将其合并为一个大的DataFrame，要求处理空值和重复行，最后保存为新的.csv文件。添加详细注释。",
        "description": "验证代码生成和工程化能力"
    },
    {
        "category": "文本创作",
        "prompt": "以‘人工智能与未来工作’为主题，写一篇200字左右的短文，观点明确，语言流畅",
        "description": "验证文本创作和逻辑表达能力"
    },
    {
        "category": "指令遵循",
        "prompt": "请完成以下指令：1. 将‘大语言模型的量化部署可以显著降低显存占用，提升推理速度’翻译成英文；2. 标注出翻译中的专业术语；3. 简要说明量化部署的核心优势。",
        "description": "验证多指令遵循和细节处理能力"
    },
    {
        "category": "逻辑推理",
        "prompt": "有A、B、C、D四个人，其中一人是小偷：1. A说：不是我；2. B说：是D；3. C说：是B；4. D说：不是我。已知只有一人说了真话，请问谁是小偷？请详细说明推理过程。",
        "description": "验证逻辑推理和分析能力"
    },
    {
        "category": "多轮对话",
        "prompt": "先告诉我Python中列表和元组的核心区别，然后基于你的回答，再说明在什么场景下优先使用元组",
        "description": "验证多轮上下文理解能力"
    }
]

def print_separator(title: str):
    """打印分隔符，美化输出"""
    print(f"\n{'='*20} {title} {'='*20}")

async def test_single_case(session: aiohttp.ClientSession, case: Dict) -> Dict:
    """异步测试单个用例，返回详细结果"""
    headers = {"Content-Type": "application/json"}
    data = {
        "model": MODEL_NAME,
        "messages": [{"role": "user", "content": case["prompt"]}],
        "max_tokens": MAX_TOKENS,
        "temperature": TEMPERATURE,
        "stream": False
    }
    # 记录单个用例的开始时间
    case_start = time.time()
    try:
        # 异步发送POST请求
        async with session.post(API_URL, headers=headers, json=data) as response:
            response.raise_for_status()  # 抛出HTTP错误
            res_json = await response.json()
            case_end = time.time()
            # 提取结果和性能数据
            content = res_json["choices"][0]["message"]["content"]
            usage = res_json.get("usage", {})
            prompt_tokens = usage.get("prompt_tokens", 0)
            completion_tokens = usage.get("completion_tokens", 0)
            total_tokens = usage.get("total_tokens", 0)
            return {
                "success": True,
                "category": case["category"],
                "description": case["description"],
                "prompt": case["prompt"],
                "response": content,
                "单个用例耗时(秒)": round(case_end - case_start, 2),
                "输入tokens": prompt_tokens,
                "输出tokens": completion_tokens,
                "总tokens": total_tokens,
                "tokens/秒": round(completion_tokens / (case_end - case_start), 2) if (case_end - case_start) > 0 else 0
            }
    except Exception as e:
        case_end = time.time()
        return {
            "success": False,
            "category": case["category"],
            "description": case["description"],
            "prompt": case["prompt"],
            "response": f"测试失败：{str(e)}",
            "单个用例耗时(秒)": round(case_end - case_start, 2),
            "输入tokens": 0,
            "输出tokens": 0,
            "总tokens": 0,
            "tokens/秒": 0
        }

async def test_stream_response(session: aiohttp.ClientSession) -> bool:
    """异步测试流式响应能力"""
    print_separator("异步流式响应测试")
    data = {
        "model": MODEL_NAME,
        "messages": [{"role": "user", "content": "请简要说明GLM-4模型的核心优势，分点列出"}],
        "max_tokens": 512,
        "temperature": 0.7,
        "stream": True
    }
    try:
        async with session.post(API_URL, headers={"Content-Type": "application/json"}, json=data) as response:
            response.raise_for_status()
            print("流式回复内容：")
            full_content = ""
            async for chunk in response.content.iter_any():
                if chunk:
                    chunk_str = chunk.decode("utf-8").lstrip("data: ")
                    if chunk_str == "[DONE]":
                        break
                    try:
                        delta = json.loads(chunk_str)["choices"][0]["delta"]
                        if "content" in delta:
                            content = delta["content"]
                            full_content += content
                            print(content, end="", flush=True)
                    except:
                        continue
        print(f"\n✅ 异步流式响应测试成功，完整内容长度：{len(full_content)}字")
        return True
    except Exception as e:
        print(f"\n❌ 异步流式响应测试失败：{str(e)}")
        return False

async def main():
    """异步测试主流程"""
    # 1. 基础信息打印
    print(f"开始异步测试GLM-4.7-Flash-FP8模型（适配新启动参数）")
    print(f"模型服务地址：{API_URL}")
    print(f"并发测试用例数：{CONCURRENT_NUM}")
    print(f"测试开始时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    # 2. 创建异步HTTP会话（复用连接，提升性能）
    async with aiohttp.ClientSession(
        timeout=aiohttp.ClientTimeout(total=300)  # 设置超时时间（5分钟）
    ) as session:
        # 3. 先测试基础连通性
        print_separator("基础连通性测试")
        try:
            async with session.get(f"{API_URL.replace('/chat/completions', '/models')}") as model_res:
                if model_res.status == 200:
                    models = (await model_res.json())["data"]
                    model_names = [m["id"] for m in models]
                    if MODEL_NAME in model_names:
                        print(f"✅ 模型服务连通正常，已加载模型：{MODEL_NAME}")
                    else:
                        print(f"⚠️ 模型服务连通，但未找到{MODEL_NAME}，已加载模型：{model_names}")
                else:
                    print(f"❌ 模型列表接口调用失败，状态码：{model_res.status}")
        except Exception as e:
            print(f"❌ 基础连通性测试失败：{str(e)}")
            return
        # 4. 异步批量执行测试用例
        print_separator(f"开始异步执行{CONCURRENT_NUM}个测试用例")
        total_start = time.time()
        # 创建所有异步任务
        tasks = [test_single_case(session, case) for case in TEST_CASES[:CONCURRENT_NUM]]
        # 并行执行所有任务（异步核心）
        results = await asyncio.gather(*tasks)
        total_end = time.time()
        total_elapsed = round(total_end - total_start, 2)
        # 5. 测试流式响应
        await test_stream_response(session)
        # 6. 生成测试报告
        print_separator("异步测试报告汇总")
        # 统计数据
        success_cases = len([r for r in results if r["success"]])
        fail_cases = len(results) - success_cases
        avg_single_time = sum([r["单个用例耗时(秒)"] for r in results]) / len(results) if results else 0
        avg_tokens_per_second = sum([r["tokens/秒"] for r in results if r["success"]]) / success_cases if success_cases > 0 else 0
        # 打印核心统计
        print(f"测试结束时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"模型启动参数适配：显存利用率0.85 + enforce-eager + max-num-batched-tokens 8192")
        print(f"总并发用例数：{len(results)}")
        print(f"成功用例数：{success_cases}")
        print(f"失败用例数：{fail_cases}")
        print(f"异步总耗时：{total_elapsed}秒（同步版本约264秒，效率提升{round((264-total_elapsed)/264*100, 1)}%）")
        print(f"单个用例平均耗时：{round(avg_single_time, 2)}秒")
        print(f"平均生成速度：{round(avg_tokens_per_second, 2)} tokens/秒")
        # 打印详细结果
        print_separator("详细测试结果")
        for i, res in enumerate(results, 1):
            print(f"\n【{i}】{res['category']}")
            print(f"描述：{res['description']}")
            print(f"单个用例耗时：{res['单个用例耗时(秒)']}秒")
            print(f"Tokens：输入{res['输入tokens']} | 输出{res['输出tokens']} | 总计{res['总tokens']}")
            print(f"生成速度：{res['tokens/秒']} tokens/秒")
            print(f"回复：{res['response'][:500]}..." if len(res['response'])>500 else f"回复：{res['response']}")
            print("-" * 50)
    print("\n✅ 所有异步测试完成！")

        avg_tokens_per_second = sum([r["tokens/秒"] for r in results if r["success"]]) / success_cases if success_cases > 0 else 0
        # 打印核心统计
        print(f"测试结束时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"模型启动参数适配：显存利用率0.85 + enforce-eager + max-num-batched-tokens 8192")
        print(f"总并发用例数：{len(results)}")
        print(f"成功用例数：{success_cases}")
        print(f"失败用例数：{fail_cases}")
        print(f"异步总耗时：{total_elapsed}秒（同步版本约264秒，效率提升{round((264-total_elapsed)/264*100, 1)}%）")
        print(f"单个用例平均耗时：{round(avg_single_time, 2)}秒")
        print(f"平均生成速度：{round(avg_tokens_per_second, 2)} tokens/秒")
        # 打印详细结果
        print_separator("详细测试结果")
        for i, res in enumerate(results, 1):
            print(f"\n【{i}】{res['category']}")
            print(f"描述：{res['description']}")
            print(f"单个用例耗时：{res['单个用例耗时(秒)']}秒")
            print(f"Tokens：输入{res['输入tokens']} | 输出{res['输出tokens']} | 总计{res['总tokens']}")
            print(f"生成速度：{res['tokens/秒']} tokens/秒")
            print(f"回复：{res['response'][:500]}..." if len(res['response'])>500 else f"回复：{res['response']}")
            print("-" * 50)
    print("\n✅ 所有异步测试完成！")

if __name__ == "__main__":
    # 仅保留Linux兼容的异步运行逻辑（移除Windows专属代码）
    asyncio.run(main())
                                                                                                                       if __name__ == "__main__":
    # 仅保留Linux兼容的异步运行逻辑（移除Windows专属代码）
    asyncio.run(main())

```
