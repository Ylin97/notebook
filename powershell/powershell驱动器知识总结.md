## 一、基本概念

- **PSDrive（PowerShell Drive）** 是 PowerShell 的一种抽象机制，用于以统一方式访问各种数据源。
- 这些“驱动器”不仅仅是文件系统，还包括：
    - 注册表
    - 环境变量
    - 证书存储
    - 函数、变量、命令别名等
- 所有驱动器由 **Provider（提供程序）** 支持，Provider 决定了驱动器如何与底层系统交互。

---

## 二、查看驱动器与提供程序

### 1️⃣ 查看当前所有驱动器

```powershell
Get-PSDrive
```

示例输出：

```
Name      Used (GB)     Free (GB) Provider      Root
----      ---------     --------- --------      ----
C             98.23        130.11 FileSystem    C:\
Env                                   Environment
HKCU                                  Registry    HKEY_CURRENT_USER
HKLM                                  Registry    HKEY_LOCAL_MACHINE
Variable                              Variable
Function                              Function
Alias                                 Alias
Cert                                  Certificate
```
### 2️⃣ 查看所有 Provider

```powershell
Get-PSProvider
```

常见 Provider：

| Provider 名称 | 功能   | 示例驱动器        |
| ----------- | ---- | ------------ |
| FileSystem  | 文件系统 | C:, D:       |
| Environment | 环境变量 | Env:         |
| Registry    | 注册表  | HKCU:, HKLM: |
| Variable    | 变量表  | Variable:    |
| Function    | 函数表  | Function:    |
| Alias       | 命令别名 | Alias:       |
| Certificate | 系统证书 | Cert:        |

---

## 三、常见驱动器用途速览

| 驱动器             | Provider    | 示例用途           | 示例命令                                               |
| --------------- | ----------- | -------------- | -------------------------------------------------- |
| `C:`、`D:`       | FileSystem  | 文件操作           | `ls C:\Windows`                                    |
| `Env:`          | Environment | 系统环境变量         | `ls Env:`、`$env:PATH`                              |
| `HKCU:`、`HKLM:` | Registry    | 注册表访问          | `ls HKLM:\SOFTWARE`                                |
| `Cert:`         | Certificate | 系统证书           | `ls Cert:\CurrentUser\My`                          |
| `Variable:`     | Variable    | PowerShell 变量表 | `ls Variable:`                                     |
| `Function:`     | Function    | 已定义函数          | `ls Function:`、`cat function:mkdir`（查看`mkdir`函数定义） |
| `Alias:`        | Alias       | 命令别名表          | `ls Alias:`                                        |

---

## 四、自定义驱动器

PowerShell 支持通过 `New-PSDrive` 命令创建自定义驱动器。语法：

`New-PSDrive -Name <驱动器名> -PSProvider <类型> -Root <目标路径> [-Persist]`

|参数|说明|
|---|---|
|`-Name`|驱动器标识（如 `P`、`Work`，建议简短，避免与内置驱动器重名）|
|`-PSProvider`|资源类型，常用 `FileSystem`（文件路径）、`Registry`（注册表路径）|
|`-Root`|驱动器映射的实际目标路径（如 `C:\Work\Projects`、`HKLM:\Software`）|
|`-Persist`|可选参数，使驱动器**跨会话有效**（仅支持 `FileSystem` 类型，需管理员权限）|

### 1️⃣ 创建临时驱动器（当前会话有效）

```powershell
New-PSDrive -Name MyData -PSProvider FileSystem -Root "D:\Projects"
```

使用：

```powershell
cd MyData:
ls
```

删除：

```powershell
Remove-PSDrive MyData
```

---

### 2️⃣ 创建持久化驱动器（系统级映射）

仅 FileSystem Provider 支持 `-Persist` 参数，用于创建持久驱动器（类似“网络映射盘”）：

```powershell
New-PSDrive -Name Share -PSProvider FileSystem -Root "\\server\share" -Persist
```

查看系统映射盘：

```powershell
net use
```

删除：

```powershell
Remove-PSDrive Share
```

---

### 3️⃣ 创建注册表驱动器

```powershell
New-PSDrive -Name AppConfig -PSProvider Registry -Root "HKCU:\Software\MyApp"
cd AppConfig:
```

---

## 五、常用管理命令汇总

|操作|命令|说明|
|---|---|---|
|查看当前驱动器|`Get-PSDrive`|显示所有已挂载的驱动器|
|查看 Provider|`Get-PSProvider`|显示支持的 Provider 类型|
|创建驱动器|`New-PSDrive -Name X -PSProvider FileSystem -Root "路径"`|临时挂载|
|创建持久驱动器|`New-PSDrive -Name X -PSProvider FileSystem -Root "\\server\share" -Persist`|映射网络盘|
|删除驱动器|`Remove-PSDrive X`|卸载驱动器|
|进入驱动器|`cd X:`|切换到驱动器命名空间|
|列出内容|`ls X:` 或 `Get-ChildItem X:`|查看驱动器中的项|

---

## 六、变量命名空间 vs 驱动器

|表达式|含义|说明|
|---|---|---|
|`$env:PATH`|访问单个环境变量|`$` 表示变量|
|`ls env:`|列出所有环境变量|`env:` 是驱动器名|
|`$variable:foo`|访问 PowerShell 变量||
|`ls variable:`|枚举所有变量||

💡 **记忆口诀**：

- `$` 表示变量；
- `:` 表示驱动器或命名空间。

---

## 七、进阶：自定义 Provider（高级用法）

- 你可以用 C# 或 PowerShell 模块自定义 Provider。
- 自定义 Provider 继承自 `System.Management.Automation.Provider.NavigationCmdletProvider`。
- 可实现访问任意数据源（如 JSON、数据库、HTTP API 等）。

示例思路：

```powershell
New-PSDrive -Name MyAPI -PSProvider RestApiProvider -Root "https://example.com/api"
ls MyAPI:\users
```

---

## 八、总结要点

|概念|说明|
|---|---|
|**PSDrive**|PowerShell 中的“虚拟驱动器”，用于访问各种数据源。|
|**Provider**|提供访问机制的底层实现（文件系统、注册表、环境变量等）。|
|**统一操作接口**|所有驱动器都可使用相同命令：`cd`、`ls`、`Get-Item`、`Remove-Item` 等。|
|**可扩展性**|用户可以创建、删除、自定义驱动器，甚至开发自定义 Provider。|

---

## 九、参考命令一览

```powershell
# 查看所有驱动器
Get-PSDrive

# 查看所有 Provider
Get-PSProvider

# 创建驱动器
New-PSDrive -Name MyData -PSProvider FileSystem -Root "D:\Projects"

# 删除驱动器
Remove-PSDrive MyData

# 创建持久驱动器（映射网络路径）
New-PSDrive -Name Share -PSProvider FileSystem -Root "\\Server\Share" -Persist

# 切换驱动器
cd Env:
ls

# 查看注册表内容
ls HKLM:\SOFTWARE
```
