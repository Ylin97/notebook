# Git 使用总结

#### 1. git 的三个存储区

按优先级排列：**工作区（工作树）**> **暂存区（索引）** > **本地仓库**。

#### 2. reset命令的三个常用参数

- `--soft`：将 HEAD 指针回退到指定版本，并将该指定版本之后的文件或代码放回到**暂存区**中。详细行为如下：
  - 对于执行回退之前暂存区已有文件，它们会被原样保留，不会被撤销差异内容覆盖；
  - 对于撤销差异中那些执行回退之前暂存区没有的文件，它们会被添加到暂存区；
  - 对于工作区，软重置不会对它进行任何修改。
- `--mixed`（默认）：将 HEAD 指针回退到指定版本，并将该指定版本之后的文件或代码放回到**工作区**中。详细行为如下：
  - 清空暂存区；
  - 对于执行回退之前工作区已修改未暂存的文件，它们会被原样保留，不会被撤销差异内容覆盖；
  - 对于撤销差异中那些执行回退之前工作区没有被修改的文件，它们会被添加到工作区作为“未暂存”的修改。
- `--hard`：将 HEAD 指针回退到指定版本，并将工作区和暂存区都清空，所有的撤销差异都会被丢弃。

> 撤销差异：回退的目标提交到当前 HEAD 指针指向的提交之间的所有提交差异总和。

#### 3. git自动切换换行符

git 仓库和 github 默认都使用 `\n`（LF）作为换行符，但 Windows 中的换行符是 `\r\n`（CRLF）两个字符的组合。所以为了不必要的 git 冲突，建议对 git 的自动转换换行符功能进行设置：
首先查看当前的设置值：
```shell
git config --global core.autocrlf       # 查看全局值
git config core.autocrlf                # 查看本地仓库值
git config --show-origin core.autocrlf  # 查看最终生效的值（包括系统级 + 全局 + 本地）
```
设置新的值：
- **Windows 平台：**
   如果项目只在 Windows 平台中执行，建议将全局的 `core.autocrlf` 的值设置为 `true`，让 git 在检出时转换为 CRLF，提交时转换为 LF。
   ```shell
     git config --global core.autocrlf true
     ```
- **Linux/MacOS 或跨平台：**
  如果项目是在 Linux/MacOS 平台或者有跨平台需求，则建议将全局 `core.autocrlf` 的值设置为 `input`，让 git 在提交时转换为 LF，检出不转换。
  ```shell
    git config --global core.autocrlf input
    ```
  同时建议在仓库根目录添加 `.gitattributes` 文件：
  ```shell
    # 其他文本自动处理
    * text=auto
    # 所有 Markdown 文件统一 LF
    *.md text eol=lf
    # 二进制文件不做换行处理
    *.svg -text
    *.pdf -text
    ```

💡 **提示：**
如果仓库中已经有换行符不一致的文件，可以执行如下命令来将所有换行符一致化：
```shell
git add .gitattributes
git add --renormalize .
git commit -m "Normalize line endings for markdown files"
```