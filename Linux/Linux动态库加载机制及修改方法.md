[toc]

# Linux 动态库加载机制及修改方法

## ✅ **Linux 动态库加载机制**

当 ELF 可执行文件运行时，动态链接器会根据以下优先级加载共享库（`.so`）：

1. **`LD_PRELOAD` 环境变量**（强制预加载）
2. **`RPATH` / `RUNPATH`**（编译时嵌入的搜索路径）
3. **`LD_LIBRARY_PATH` 环境变量**
4. **`/etc/ld.so.cache`**（`ldconfig` 生成的缓存）
5. **默认系统目录**（如 `/lib`, `/usr/lib`）

**RPATH vs RUNPATH**

- **RPATH**：旧机制，在 `LD_LIBRARY_PATH` 之前
- **RUNPATH**：新机制，在 `LD_LIBRARY_PATH` 之后（现代系统主要用 RUNPATH）

------

## ✅ **查看 ELF 动态库信息**

查看动态库依赖：

```bash
ldd ./program
```

查看 RPATH/RUNPATH：

```bash
readelf -d ./program | grep -E 'RPATH|RUNPATH'
# 或
patchelf --print-rpath ./program
```

------

## ✅ **修改动态库加载路径的方法**

### **方法 1：`LD_LIBRARY_PATH`（临时生效）**

修改搜索路径顺序：

```bash
LD_LIBRARY_PATH=/usr/lib ./program
```

多个路径用 `:` 分隔：

```bash
LD_LIBRARY_PATH=/usr/lib:/opt/mylibs ./program
```

**特点：**

- 临时生效（只对当前命令有效）
- 优先级高于 RUNPATH

------

### **方法 2：`LD_PRELOAD`（强制加载指定库）**

覆盖指定库：

```bash
LD_PRELOAD=/usr/lib/libcurl.so.4 ./program
```

**特点：**

- 强制加载库并替换符号
- 用于调试或 Hook，比 LD_LIBRARY_PATH 优先

------

### **方法 3：修改 RPATH/RUNPATH（永久修改 ELF）**

使用 `patchelf`：

- 查看：

  ```bash
  patchelf --print-rpath ./program
  ```

- 删除 RPATH/RUNPATH：

  ```bash
  patchelf --remove-rpath ./program
  ```

- 修改 RPATH：

  ```bash
  patchelf --set-rpath /usr/lib ./program
  ```

**特点：**

- 永久修改，可执行文件本身的搜索路径变了
- 常用于移除 vcpkg 等工具链的路径，让程序用系统库

------

### **方法 4：修改符号链接**

例如把 vcpkg 的 `libcurl.so.4` 链接到系统的：

```bash
mv libcurl.so.4 libcurl.so.4.bak
ln -s /usr/lib/x86_64-linux-gnu/libcurl.so.4 libcurl.so.4
```

**缺点：**

- 影响其他程序
- 风险大，不推荐

------

### **方法 5：更新系统库缓存**

如果安装了新库，更新 `/etc/ld.so.cache`：

```bash
sudo ldconfig
```

------

## ✅ **验证修改是否生效**

```bash
ldd ./program | grep curl
```

如果输出系统路径：

```
libcurl.so.4 => /usr/lib/x86_64-linux-gnu/libcurl.so.4
```

说明程序加载的是系统库。

------

## ✅ **最佳实践**

- 临时测试用 `LD_LIBRARY_PATH` 或 `LD_PRELOAD`
- 永久修改用 `patchelf --remove-rpath` 或 `--set-rpath`
- 避免硬改符号链接，容易破坏环境
- 如果想最小化程序大小，优先用 **系统动态库**，而不是静态打包（尤其是 OpenSSL）

-----

> ### ✅ Linux 动态库修改命令速查表（Cheat Sheet）
> 
> ------
> 
> #### 1. 查看 ELF 动态库依赖
> 
> ```bash
> ldd ./program
> ```
> 
> 示例输出：
> 
> ```
> libcurl.so.4 => /usr/lib/x86_64-linux-gnu/libcurl.so.4
> ```
> 
> ------
> 
> #### 2. 查看 RPATH / RUNPATH
> 
> ```bash
> readelf -d ./program | grep -E 'RPATH|RUNPATH'
> # 或
> patchelf --print-rpath ./program
> ```
> 
> ------
> 
> #### 3. 删除 RPATH
> 
> ```bash
> patchelf --remove-rpath ./program
> ```
> 
> **用途**：清掉硬编码的路径，让程序走系统默认搜索顺序。
> 
> ------
> 
> #### 4. 修改 RPATH（永久修改 ELF）
> 
> ```bash
> patchelf --set-rpath /usr/lib/x86_64-linux-gnu ./program
> ```
> 
> **用途**：强制程序优先在 `/usr/lib/x86_64-linux-gnu` 查找动态库。
> 
> ------
> 
> #### 5. 使用 LD_LIBRARY_PATH（临时修改）
> 
> ```bash
> LD_LIBRARY_PATH=/usr/lib ./program
> ```
> 
> 多个路径：
> 
> ```bash
> LD_LIBRARY_PATH=/usr/lib:/opt/custom/lib ./program
> ```
> 
> ------
> 
> #### 6. 使用 LD_PRELOAD（强制加载指定库）
> 
> ```bash
> LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libcurl.so.4 ./program
> ```
> 
> **用途**：替换掉运行时加载的 `libcurl.so.4`。
> 
> ------
> 
> #### 7. 更新系统库缓存（如果安装了新库）
> 
> ```bash
> sudo ldconfig
> ```
> 
> ------
> 
> #### 8. 验证最终加载的库
> 
> ```bash
> ldd ./program | grep curl
> ```
> 
> 正确输出：
> 
> ```
> libcurl.so.4 => /usr/lib/x86_64-linux-gnu/libcurl.so.4
> ```
> 
> ------
> 
> ### ✅ 典型工作流：替换 vcpkg 的 libcurl 为系统 libcurl
> 
> 1. **查看当前 RPATH**
> 
> ```bash
> patchelf --print-rpath ./program
> ```
> 
> 1. **删除 RPATH**
> 
> ```bash
> patchelf --remove-rpath ./program
> ```
> 
> 1. **检查是否加载系统库**
> 
> ```bash
> ldd ./program | grep curl
> ```
> 
> 1. （可选）**运行时强制加载系统 libcurl**
> 
> ```bash
> LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libcurl.so.4 ./program
> ```
> 
> ------
> 
> ### ✅ 额外提示
> 
> - 如果要对 **大量程序** 批量操作，可以写脚本：
> 
> ```bash
> for exe in $(find ./target/bin -type f); do
>     patchelf --remove-rpath "$exe"
> done
> ```
> 
> - 如果有 **符号链接** 的问题，不要直接改 `/usr/lib`，优先用 `patchelf`。

