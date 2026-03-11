#  Windows 平台 C++ 处理中文命令行参数与控制台输出 —— 最佳实践总结

> **阅读目的**：快速理解 *为什么控制台会乱码*，并掌握 **最安全、最通用** 的编码处理方案与模板代码。

------

## 💥 背景问题

Windows 平台程序编码处理异常复杂：传统控制台默认使用本地代码页（ACP），如中文系统为 `GBK/CP936`，而现代工具如 Windows Terminal 又强制使用 UTF-8。这与现代 C++ 编程和跨平台开发普遍采用的 **UTF-8 编码**习惯相冲突，导致以下常见问题：

- 命令行参数中包含中文时，可能会乱码或抛异常（例如 `std::filesystem` 报错）。
- 控制台输出中文或 emoji 时乱码。
- 直接使用 `char* argv[]` 得到的编码不是 UTF-8。

------

## ✅ 推荐解决方案

### ✅ 方案一：统一使用 UTF-8（推荐方式，跨平台友好）

#### 核心做法：

- 使用 `main` 而非 `wmain`
- 手动通过 `CommandLineToArgvW(GetCommandLineW())` 获取参数
- 将 `wchar_t*` 参数转为 UTF-8（`std::string`）
- 设置控制台为 UTF-8 模式输出

#### 🔧 工具头文件 `utf8_args.h`：

```cpp
#pragma once

#include <string>
#include <vector>

#if defined(_WIN32)
#include <windows.h>
#include <shellapi.h>
#include <iostream>

namespace utf8_args {

// 将宽字符串转换为 UTF-8 编码的 std::string。注意：返回的字符串不包含 null terminator（已由 std::string 管理）
inline std::string WideToUTF8(const wchar_t* wstr) {
    if (!wstr) return "";
    int size = WideCharToMultiByte(CP_UTF8, 0, wstr, -1, nullptr, 0, nullptr, nullptr);
    if (size <= 0) return "";
    std::string utf8(size - 1, '\0'); // 不包括末尾的'\0'
    WideCharToMultiByte(CP_UTF8, 0, wstr, -1, utf8.data(), size, nullptr, nullptr);
    return utf8;
}

// 获取 UTF-8 编码的命令行参数（替代 char* argv[]）
inline std::vector<std::string> GetUTF8CommandLineArgs() {
    int argc = 0;
    wchar_t** argv_w = CommandLineToArgvW(GetCommandLineW(), &argc);
    std::vector<std::string> utf8_args;

    if (!argv_w) {
        fprintf(stderr, "[utf8_args] Failed to parse command line.\n");
        return utf8_args;
    }

    utf8_args.reserve(argc);
    for (int i = 0; i < argc; ++i) {
        utf8_args.emplace_back(WideToUTF8(argv_w[i]));
    }

    LocalFree(argv_w);
    return utf8_args;
}

} // namespace utf8_args

#else // 非 Windows 平台

namespace utf8_args {
inline std::vector<std::string> GetUTF8CommandLineArgs(int argc, char* argv[]) {
    return std::vector<std::string>(argv, argv + argc);
}
} // namespace utf8_args

#endif
```

------

#### ✨ 示例用法：

```cpp
#include "utf8_args.h"
#include <iostream>
#include <windows.h>

int main(int argc, char* argv[]) {
    // 设置控制台输出为 UTF-8（Windows 10+ 支持）
    SetConsoleOutputCP(CP_UTF8);
    SetConsoleCP(CP_UTF8);

    auto args =
    #ifdef _WIN32
        utf8_args::GetUTF8CommandLineArgs();
    #else
        utf8_args::GetUTF8CommandLineArgs(argc, argv);
    #endif

    for (const auto& arg : args) {
        std::cout << arg << "\n";
    }

    return 0;
}
```

> 💡 请确保源代码文件本身也使用 UTF-8 编码保存（可带 BOM 或不带），否则中文字面量可能出现乱码。

> ⚠️ 如果使用 `std::filesystem::path` 来处理路径，请使用 `std::filesystem::u8path(utf8_string)`，防止因编码错误导致路径找不到。

------

### ✅ 方案二：使用 `wmain` + 宽字符流（Windows 专属）

该方式由操作系统直接传递宽字符参数，适合完全在 Windows 上运行的程序：

```cpp
#include <iostream>
#include <windows.h>
#include <fcntl.h>
#include <io.h>

int wmain(int argc, wchar_t* argv[]) {
    // 设置控制台为 UTF-8（确保显示正确）
    SetConsoleOutputCP(CP_UTF8);
    SetConsoleCP(CP_UTF8);

    // 设置 stdout 为宽字符 UTF-8 输出模式
    _setmode(_fileno(stdout), _O_U8TEXT);

    std::wcout << L"argc: " << argc << L"\n";
    for (int i = 0; i < argc; ++i) {
        std::wcout << L"argv[" << i << L"]: " << argv[i] << L"\n";
    }

    std::wcout << L"你好，世界！🌏\n";
    return 0;
}
```

> ✅ 推荐配合 `std::wcout`, `std::wcin` 使用，避免混用 `cout` 和 `wcout`，以保证输出一致性。

> ⚠️ 注意，使用 g++ 编译带`wmain`函数的程序时需要在`-o`参数前加上`-municode`参数，如 `g++ test.cpp -municode -o test`。

------

## 📝 总结建议

- Windows 下，**char\* argv[] 编码不可控**，推荐使用 `CommandLineToArgvW` 配合 `WideCharToMultiByte(CP_UTF8)` 自行处理。
- 控制台输出时，推荐使用 `SetConsoleOutputCP(CP_UTF8)` 统一输出编码。
- 如果使用宽字符流输出（如 `wcout`），应额外调用 `_setmode(_fileno(stdout), _O_U8TEXT)`。
- **保持全程序 UTF-8 编码一致性（源代码、参数、文件读写）是关键**。

------

## 附录

> # Windows命令行参数、控制台、终端等乱码原因详细分析
>
> 首先我们要明白一点，对于控制台程序，用户是通过控制台或者终端以命令行的方式与其进行交互的。
>
> **控制台程序与用户之间的交互流程：**
>
> ```
> ════════════════════════════════ 程序启动阶段 ════════════════════════════════
> 
> 　   　┌──────────────────────────────────────────┐
> 　   　│  用户输入的命令行参数由控制台子系统自动处理为　　│
>    　　│  UTF-16 编码存储在其内部供程序通过 API 读取   │
>    　　└─────────────────────┬────────────────────┘
> 　　　　　　　　          　　　│ [GetCommandLineA/GetCommandLineW 
> 　　　　　　　　          　　　▼  控制台 API 获取命令行参数]
>             ╔════════════════════════════════╗
>             ║       　 CRT 初始化代码          ║
>             ╠════════════════════════════════╣
>             ║ 调用 GetCommandLineW()        　║ ◄─ 获取原始命令行 (UTF-16)
>             ║                                ║
>             ║ IF 入口是 main():               ║
>             ║   * 使用 ACP 转换命令行      　　 ║ 
>             ║   * 分割为 char* argv[]         ║ ◄─ ACP (系统活动代码页)
>             ║   * 调用 main(argc, argv)   　  ║ 
>             ║                                ║
>             ║ IF 入口是 wmain():              ║
>             ║   * 直接分割命令行            　　║
>             ║   * 调用 wmain(argc, argv) 　   ║
>             ╚════════════════════════════════╝
> 
> 
> ════════════════════════════════ 程序输入阶段 ════════════════════════════════                               
>                    [用户键盘输入/粘贴]
>                            │
>                            │ (输入法处理后为 UTF-16，控制台缓冲区
>                            ▼  使用 UTF-16 对其进行存储)
>             ╔════════════════════════════╗
>             ║    Windows 控制台子系统      ║◄── ConsoleCP (输入代码页)
>             ║ 　 (conhost.exe 或新终端) 　 ║◄── ConsoleOutputCP (输出代码页)
>             ╚════════════════════════════╝
>                            │
>             ┌──────────────┴──────────────────────┐
>             │                                     │
> ┌───────────▼───────────┐      　 　　┌────────────▼───────────┐
> │ 　　　窄字符输入    　   │            │       宽字符输入       　│
> │  (scanf/cin/getchar)  │       　　　│ (wscanf/wcin/getwchar) │
> └───────────┬───────────┘    　   　　└────────────┬───────────┘
>             │ [程序调用 ReadConsoleA]           　　│ [程序调用 ReadConsoleW] 
>             │                                　　　│
>             ▼ (内部转换流程)               　　    　▼ (直接返回 UTF-16)  
> ╔══════════════════════╗                  [返回 wchar_t 数据]
> ║ 1. 调用 ReadConsoleW　║
> ║ 2. 使用 ConsoleCP   　║
> ║    转换为 ANSI/MBCS   ║
> ╚══════════════════════╝        
>             │
>             ▼ (返回转换后数据)
> [返回 char* 数据 (ANSI/MBCS)]
> 
> 
> ════════════════════════════════ 程序输出阶段 ════════════════════════════════
> 
> ┌─────────────────────────┐        　┌───────────────────────┐
> │ 　　　窄字符输出    　     │        　│ 　　　宽字符输出       　│
> │　(printf/cout/puts)　  　│        　│ (wprintf/wcout/putws) │
> └────────────┬────────────┘      　  └───────────┬───────────┘
>              │                                 　│
>              │ [程序调用 WriteConsoleA]           │ [程序调用 WriteConsoleW]
>              ▼                                 　│
> ╔═════════════════════════╗                    　▼ (直接输出 UTF-16)
> ║ 　使用 ConsoleOutputCP   ║              [输出 UTF-16 数据]
> ║   转换为 UTF-16          ║                    　│
> ╚═════════════════════════╝                    　│
>              │                                 　│
>              ▼ (转换后输出)                    　　│ 
>      [输出 UTF-16 数据]────────────┬──────────────┘
>                                 　▼
>                    ╔════════════════════════════╗
>                    ║     Windows 控制台子系统     ║
>                    ╚════════════════════════════╝
>                                   │
>                                   ▼ (字体渲染)
>                             [控制台显示字符]
> 
> ════════════════════════════════ 关键机制说明 ════════════════════════════════
> 
> 1. 命令行参数编码:
>    - wmain(): 直接获取 UTF-16 参数 (无转换)
>    - main(): CRT 使用 ACP 转换 UTF-16 → ANSI/MBCS (如中文系统 GBK)
>      * 乱码根源: 当程序期望 UTF-8 但系统 ACP 是 GBK 时
> 
> 2. 控制台输入:
>    - ReadConsoleW: 直接返回 UTF-16
>    - ReadConsoleA: 内部使用 ReadConsoleW + ConsoleCP 转换
>      * ConsoleCP 通过 `chcp` 命令设置
> 
> 3. 控制台输出:
>    - WriteConsoleW: 直接输出 UTF-16
>    - WriteConsoleA: 使用 ConsoleOutputCP 转换 → UTF-16
>      * ConsoleOutputCP 通过 `chcp` 命令设置
>      * 乱码根源: 程序输出 UTF-8 但 ConsoleOutputCP ≠ 65001
> 
> 4. 现代终端 (Windows Terminal) 注意事项:
>    - 默认设置 ConsoleCP=65001 和 ConsoleOutputCP=65001
>    - 但 main() 参数仍依赖系统 ACP (独立设置)
>    - 解决方案:
>      * 使用 wmain() 或 CommandLineToArgvW()
>      * 设置系统级 UTF-8 标志 (Beta 功能)
>      * 输出时始终使用 WriteConsoleW 或设置 chcp 65001
> ```
>
> 如上图所示，**控制台（Console）**和**C 运行时库（CRT, C Runtime Library）**是两个不同的系统组件。
>
> 控制台类似于类 Unix 系统中的“终端”，是用户与操作系统交互的界面。而 CRT 是程序运行时的基础库，它负责初始化程序、处理命令行参数、提供输入输出功能等。
>
> 在用户与 C/C++ 程序交互的过程中，控制台起着“翻译员”的作用：
>
> 1. 用户在控制台中输入的内容，首先会根据**控制台输入代码页（Console Input Code Page）**进行编码解析（设置函数为 `SetConsoleCP`），并以 UTF-16 的形式保存在内部。
> 2. 程序启动时，CRT 再将这段 UTF-16 编码的数据转换为 `char* argv[]` 所需的字节序列。这一步使用的是**系统活动代码页（ACP）**，通过 `WideCharToMultiByte(CP_ACP)` 实现。
>
> 如果系统 ACP 是 GBK（CP936）而用户输入的是 UTF-8 编码，就会导致转换失败或乱码（因为有些 UTF-8 字符无法用 GBK 表示）。**这正是程序收到的命令行参数乱码的根源。**
>
> 程序输出数据到控制台时，也会经过类似的“反向翻译”流程：
>  程序生成 UTF-8 或 ACP 编码的文本 → CRT 输出函数 → 控制台根据**输出代码页（`SetConsoleOutputCP`）**来解析字节 → 以 UTF-16 渲染到控制台窗口。编码不一致同样会导致乱码。
>
> 例如：
>
> - 当程序使用 `main` 而非 `wmain` 时，CRT 会假定命令行参数是系统代码页编码的，导致输入是 UTF-8 时出错。
> - 同理，如果程序通过 `std::cout` 输出 UTF-8 字符串，而控制台输出代码页是 GBK，也会乱码。
>
> 尽管我们可以通过 `_setmode` 修改 CRT 的输出行为，如将 `stdout` 设置为 `_O_WTEXT` 或 `_O_U8TEXT`，但**无法改变 CRT 在程序启动前解析命令行参数时的行为** —— 它依旧使用系统 ACP。
>
> #### 几个实用提示：
>
> - 使用 `_setmode(_fileno(stdout), _O_WTEXT)` 更稳妥，它让 `std::wcout` 直接调用 `WriteConsoleW`（支持 UTF-16 输出），避免中间转换，兼容性和性能都更好。
> - `chcp` 命令用于修改控制台代码页：
>   - 在 CMD 中，`chcp` 同时影响输入（ConsoleCP）和输出（ConsoleOutputCP）；
>   - 在 Windows Terminal 或 PowerShell 中，通常**只影响输出代码页**，输入仍使用系统默认或启动配置。
> - 控制台**输入代码页**（`SetConsoleCP`）决定用户输入如何转换为 UTF-16；
> - 控制台**输出代码页**（`SetConsoleOutputCP`）决定 CRT 输出如何被解释成最终可显示字符；
> - 为了保持前后转换还原不变，`MultiByteToWideChar` 和  `WideCharToMultiByte` 所使用的活动代码页最好**保持一致**。

> **编码存储小知识：**UTF-8 最小长度为一个字节，故适合使用 `char` 数组进行存储，而 UTF-16 最小长度为两个字节，所以适合使用 `wchar_t` 数据进行存储。这也就是为何 `GetCommandLineW` 等宽字符版 API 返回的是 UTF-16 数据的原因。