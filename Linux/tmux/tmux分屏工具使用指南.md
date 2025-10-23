# tmux分屏工具使用指南

tmux（Terminal Multiplexer）是一个终端复用工具，它允许你在单个终端窗口中创建多个虚拟终端会话，并能保持这些会话在后台运行。与直接使用终端相比，tmux 提供了更强大的会话管理功能。

**核心优势**：

- 会话持久化：即使网络断开，会话仍保留在服务器上
- 多窗口/窗格管理：高效组织多个工作环境
- 会话共享：多个用户可以同时连接同一个会话

------

## 安装 tmux

在大多数 Linux 发行版中，tmux 可以通过包管理器轻松安装：

```shell
# Ubuntu/Debian
sudo apt-get install tmux

# Archlinux
sudo pacman -S tmux

# CentOS/RHEL
sudo yum install tmux

# macOS (使用 Homebrew)
brew install tmux
```

安装完成后，输入 `tmux` 命令即可启动一个新会话。

------

## 基本概念

### 会话（Session）

tmux 会话是一个独立的运行环境，可以包含多个窗口。即使断开连接，会话也会继续在后台运行。

### 窗口（Window）

每个会话可以包含多个窗口，类似于浏览器中的标签页。

### 窗格（Pane）

每个窗口可以分割成多个窗格，允许同时查看和操作多个终端。

------

## 常用命令与快捷键

tmux 的所有操作都需要先按下前缀键（默认是 `Ctrl+b`），然后输入命令键。

### 会话管理

| 命令/快捷键             | 说明                             |
| :---------------------- | :------------------------------- |
| `tmux new -s <name>`    | 创建名为 name 的新会话           |
| `Ctrl+b d`              | 分离当前会话（会话继续后台运行） |
| `tmux ls`               | 列出所有会话                     |
| `tmux attach -t <name>` | 重新连接到指定会话               |
| `Ctrl+b $`              | 重命名当前会话                   |
| `Ctrl+b s`              | 切换会话                         |

### 窗口管理

| 命令/快捷键       | 说明                 |
| :---------------- | :------------------- |
| `Ctrl+b c`        | 创建新窗口           |
| `Ctrl+b &`        | 关闭当前窗口         |
| `Ctrl+b n`        | 切换到下一个窗口     |
| `Ctrl+b p`        | 切换到上一个窗口     |
| `Ctrl+b <number>` | 切换到指定编号的窗口 |
| `Ctrl+b ,`        | 重命名当前窗口       |

### 窗格管理

| 命令/快捷键      | 说明                |
| :--------------- | :------------------ |
| `Ctrl+b %`       | 垂直分割当前窗格    |
| `Ctrl+b "`       | 水平分割当前窗格    |
| `Ctrl+b <arrow>` | 在窗格间移动焦点    |
| `Ctrl+b x`       | 关闭当前窗格        |
| `Ctrl+b z`       | 最大化/恢复当前窗格 |
| `Ctrl+b Space`   | 切换窗格布局        |

------

## 配置 tmux

tmux 的配置文件位于 `~/.tmux.conf`。以下是一些常用配置示例：

```shell
# 设置前缀键为 Ctrl+a（比默认的 Ctrl+b 更容易按）
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# 启用鼠标支持（可以鼠标点击切换窗格/窗口）
set -g mouse on

# 设置状态栏颜色
set -g status-bg black
set -g status-fg white

# 设置窗格边框颜色
set -g pane-border-style fg=green
set -g pane-active-border-style fg=red

# 设置窗口从1开始编号（默认是0）
set -g base-index 1
setw -g pane-base-index 1
```

修改配置后，可以按 `Ctrl+b :` 然后输入 `source-file ~/.tmux.conf` 重新加载配置。

------

## 实用技巧

### 1. 会话恢复

即使服务器重启，也可以恢复 tmux 会话：

```shell
# 安装插件
git clone https://github.com/tmux-plugins/tmux-resurrect ~/clone/path

# 在 .tmux.conf 中添加
set -g @plugin 'tmux-plugins/tmux-resurrect'
```

### 2. 复制模式

- `Ctrl+b [` 进入复制模式
- 使用方向键移动光标
- 空格开始选择，回车复制
- `Ctrl+b ]` 粘贴

### 3. 同步输入

在多窗格中同时输入相同命令：

```shell
# 进入同步模式
Ctrl+b :setw synchronize-panes on

# 关闭同步模式
Ctrl+b :setw synchronize-panes off
```

### 4. 快速创建开发环境

```shell
# 创建一个包含3个窗格的开发会话
tmux new -s dev -d
tmux send-keys -t dev:0 "vim" C-m
tmux split-window -h -t dev:0
tmux split-window -v -t dev:0.1
tmux attach -t dev
```

------

## 常见问题解决

### 1. 无法使用鼠标滚动

在 `.tmux.conf` 中添加：

```
set -g terminal-overrides 'xterm:smcup@:rmcup@'
```

### 2. 颜色显示不正常

确保终端支持 256 色，添加：

```
set -g default-terminal "screen-256color"
```

### 3. 快捷键冲突

如果某些快捷键与你的其他工具冲突，可以在配置文件中重新绑定。

------

## 在shell脚本中使用tmux

可以将分屏命令写入到脚本中，避免每次手动输入。例如：

```shell
#!/bin/bash

SESSION_NAME="my_workspace"

# 清理会话（可选）
tmux kill-session -t "$SESSION_NAME" 2>/dev/null

# 1. 创建主会话（pane 0）, -c 参数指定新 pane 的初始工作目录
tmux new-session -d -s "$SESSION_NAME" -c "/home/yalin/workspace/code"

# 2. 水平分割：创建 pane 1（右侧）
tmux split-window -h -t "$SESSION_NAME:0" -c "/home/yalin/.local"

# 3. 在右侧（pane 1）垂直分割三次，得到 pane 2、3、4
tmux split-window -v -t "$SESSION_NAME:0.1" -c "/home/yalin"
tmux split-window -v -t "$SESSION_NAME:0.2" -c "/home/yalin"
tmux split-window -v -t "$SESSION_NAME:0.3" -c "/home/yalin"

# 此时布局完成：共 5 个 pane（0 左，1/2/3/4 右侧从上到下）

# 4. 逐个 select-pane 并发送命令

# Pane 0: code 目录。C-m 表示回车，也可以写 Enter
tmux select-pane -t "$SESSION_NAME:0.0"
tmux send-keys -t "$SESSION_NAME:0.0" "cd /home/yalin/workspace/code" C-m
tmux send-keys -t "$SESSION_NAME:0.0" "ls ." C-m
tmux send-keys -t "$SESSION_NAME:0.0" "cal" C-m

# Pane 1: date
tmux select-pane -t "$SESSION_NAME:0.1"
tmux send-keys -t "$SESSION_NAME:0.1" "date" C-m

# Pane 2: system info
tmux select-pane -t "$SESSION_NAME:0.2"
tmux send-keys -t "$SESSION_NAME:0.2" "whoami" C-m
tmux send-keys -t "$SESSION_NAME:0.2" "uname -r" C-m

# Pane 3: lsblk
tmux select-pane -t "$SESSION_NAME:0.3"
tmux send-keys -t "$SESSION_NAME:0.3" "lsblk" C-m
tmux send-keys -t "$SESSION_NAME:0.3" '请手动输入: ip addr'

# Pane 4: bashrc + pacman tip
tmux select-pane -t "$SESSION_NAME:0.4"
tmux send-keys -t "$SESSION_NAME:0.4" "cat ~/.bashrc" C-m
tmux send-keys -t "$SESSION_NAME:0.4" '请手动输入: sudo pacman -Syu'

# 可选：调整布局更均衡
#tmux select-layout -t "$SESSION_NAME:0" main-vertical

# 附加会话
tmux attach-session -t "$SESSION_NAME"
```

上面示例中采用了“先布局，后填充”的策略，这是一种**清晰、可控、可读性强**的 tmux 自动化写法。其中几个参数和选项说明如下：

- `-c`：用来指定新会话 / 窗口 / 窗格的初始工作目录；

- `C-m`：或者写全称 `Enter`。表示回车，如果不加它，则命令只会输入到终端界面而不会执行（就像手动敲入命令而没按回车键一样）；

- `$SESSION_NAME:0.1`：它的格式是 `会话名:窗口索引.窗格索引`

- `select-layout`：可以用来调整当前窗口 pane 的布局为 tmux 内置的布局方案:

  | 布局名称          | 快捷键（在 tmux 中）              | 效果说明                                             |
  | ----------------- | --------------------------------- | ---------------------------------------------------- |
  | `even-horizontal` | `Ctrl-b Alt-1` 或 `Ctrl-b Meta-1` | 所有 pane **水平均分**（从左到右）                   |
  | `even-vertical`   | `Ctrl-b Alt-2` 或 `Ctrl-b Meta-2` | 所有 pane **垂直均分**（从上到下）                   |
  | `main-horizontal` | `Ctrl-b Alt-3` 或 `Ctrl-b Meta-3` | **一个大 pane 在上**，其余小 pane **水平排列在下方** |
  | `main-vertical`   | `Ctrl-b Alt-4` 或 `Ctrl-b Meta-4` | **一个大 pane 在左**，其余小 pane **垂直排列在右侧** |
  | `tiled`           | `Ctrl-b Alt-5` 或 `Ctrl-b Meta-5` | **网格平铺**（尽可能接近正方形，自动行列排布）       |

  > 💡 注：`Alt` 在某些终端中需用 `Esc` 代替（先按 `Esc`，再按数字）。
  
  ```
     even-horizontal       even-vertical     main-horizontal          main-vertical                   tiled
  +----+----+----+----+     +----------+     +--------------+     +----------+----------+     +----------+----------+
  |    |    |    |    |     |    0     |     |      0       |     |          |    1     |     |    0     |    1     |
  | 0  | 1  | 2  | 3  |     +----------+     +----+----+----+     |          +----------+     +----------+----------+
  |    |    |    |    |     |    1     |     | 1  | 2  | 3  |     |    0     |    2     |     |    2     |    3     |
  +----+----+----+----+     +----------+     +----+----+----+     |          +----------+     +----------+----------+
                            |    2     |                          |          |    3     |
                            +----------+                          +----------+----------+
                            |    4     |
                            +----------+
  ```
  
  

