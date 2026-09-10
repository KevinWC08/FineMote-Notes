
# 前言

本文档是笔者学习工程 C++ 构建流程的笔记，旨在提供 C++ 构建的最简流程地图，帮助读者快速理解 C++ 工程的构建过程。

# C++ 工具链指南

在了解构建工具之前，我们要先了解 C++ 工具链的组成，即源文件到可执行文件到底经历了什么。

## C++ 构建流程

对了解 C++ 的人来说，编译器的存在或许是显而易见的，但实际上，编译器只是构建流程中的一环。  
**一个完整的 C++ 工具链至少包含编译器、汇编器与链接器。**  
- 编译器（Compiler）将 C++ 源文件（.cpp）编译为汇编代码（.s）。  
- 汇编器（Assembler）将汇编代码汇编为目标文件（.o）。  
- 链接器（Linker）将目标文件链接为可执行文件（.exe/.elf）。

接下来，我们将说明 C++ 构建的具体过程。

### 工具链是什么

工具链（Toolchain）是一组共同完成软件构建的工具集合，一个完整的工具链通常包含：
- Preprocessor
- Compiler
- Assembler
- Linker

在实际工具链中，编译器通常集成了这四者的入口，这也是在一个命令就能从源文件生成可执行文件的原因。  
但是，区分这四者的职责，对理清 C++ 构建流程是十分必要的。

值得一提的是，工具链不仅决定了构建行为，也是 CPU 架构支持、操作系统支持和标准库的提供者。  
因此，尤其在嵌入式的交叉编译中，选择并配置合适的工具链是非常重要的。

### Preprocess

预处理阶段发生在真正编译之前，主要处理文件包含、宏展开、条件编译等，即 `#` 开头的指令。

#### #include 的实质

如果你还不知道的话，`#include` 的实质是将被包含文件的内容直接插入到当前文件中，因此妥善使用 `extern` 关键字是非常重要的。

#### 什么是宏

`#define`，即所谓宏定义，是 C++ 的一种文本替换机制。宏的经典用途是处理游离常量，维持代码可读性，例如定义圆周率：

```cpp
#define PI 3.14159265358979323846
```

宏也可以用于赋予函数别名，如 HAL 库中常用宏代替赋值函数，实现语义逻辑替换：

```cpp
#define __HAL_TIM_SET_PRESCALER(__HANDLE__, __PRESC__)       ((__HANDLE__)->Instance->PSC = (__PRESC__))
```

#### 条件编译

条件编译根据编译时的条件来决定是否包含某段代码。常用的条件编译指令有 `#if`、`#ifdef`、`#ifndef`、`#elif` 和 `#else`。

在头文件中，条件编译常用于防止重复包含：

```cpp
#ifndef __HELLO_H__
#define __HELLO_H__
void hello();
#endif
```

在嵌入式开发中，条件编译也可用于选择不同硬件平台。

#### 预处理器的工作

在正式编译之前，预处理器

### Compile

#### 编译器看到的是什么？

#### 编译器如何处理未知？

### Assemble

#### 汇编发生了什么？

### Link

#### 链接器如何组合各段？（scatter是什么？）

#### 链接器怎么记录与替换符号？

#### 链接器怎么找到程序入口？


# *CMake* 速通指南

现在，我们已经知道 C++ 源文件如何经过编译、汇编、链接，最后成为可执行文件的。  
不过，手动管理大量的源文件和依赖关系是非常繁琐的，尤其是在跨平台开发中。  
因此，CMake 作为一个跨平台的**构建系统生成器**应运而生，它可以自动生成 Ninja 等构建文件，从而简化构建流程。  
在本章中，我们从 CMake 执行时的**真实行为**出发，记录各类文件的作用与规范，并指出其在实际开发中的使用方法或潜在功能。

值得注意的是，*CMake 只是一个构建系统生成器，其本身不以任何形式参与编译过程*，功能仅限于根据需求自动生成编译参数。  
请记住这一点，在学习 CMake 时产生的许多疑惑都源于对 CMake 的误解，明确其职责才能避免误入歧途。

## *CMake* 的执行阶段

已知的 CMake 的执行阶段：  
- **CMake Configure 阶段，确认构建配置与环境信息：**  
        CMake 结合命令行参数，Preset 设置和已有的 `CMakeCache.txt` 文件，取得 Configure 所需的信息。    
        CMake 执行 `CMakeLists.txt` 文件，生成 Target 依赖树。  
        CMake 加载 `Toolchain` 文件，确认编译器和工具链信息。
        CMake 会将解析结果存储在内存中，并重新写入 `CMakeCache.txt`。  
- **CMake Generate 阶段，根据配置生成构建文件：**  
        CMake 对 Target 树上的每个 Target 里的每个源文件都生成构建规则，连同 Target 依赖关系一起写入构建文件 `build.ninja`。  
- **CMake Build 阶段，调用构建工具进行编译：**  
        CMake 调用 Ninja 等构建工具完成编译，此时 CMake 只作为传话筒工作。

## CMake Configure Stage

在不携带 `--build` 参数启动 CMake 时，默认进入 Configure 阶段，CMake 在此时收集构建配置、环境信息和 Target 树。

### 预解析 CMakePresets

当携带参数 `--preset` 启动 CMake时，CMake 会首先解析 `CMakePresets.json` 和 `CMakeUserPresets.json` 文件中的预设配置。值得注意的是，*指定预设配置与直接使用命令行参数启动 CMake 的效果完全等同*，预设的作用是方便管理和复用配置。

#### CMakePresets 格式规则

以 FineMote 中[某配置](../../CMakePresets.json)为例（已人工递归展开）：
```json
{
    "version": 5,
    "cmakeMinimumRequired": {
    "major": 3,
    "minor": 22,
    "patch": 0
    },
    "include": [],
    "configurePresets": []
}
```

- `version` 字段表示 CMake 采用的 Preset 版本，不同版本的 Preset 有不同的字段和语法要求。
- `cmakeMinimumRequired` 字段指定了所需的最低 CMake 版本。
- `include` 字段用于指定子 Preset 文件，子文件也需遵循CMakePresets规则。
- `configurePresets` 字段定义了配置预设，*CMake 预解析 CMakePresets 的结果完全从此字段产生*。

在阅读 CMakePresets 文件时，我们一般关注到 `include` 和 `configurePresets` 字段即可，前者用于组织各 Preset 文件，后者定义了具体的配置项。

#### configurePresets 配置格式

以 FineMote 中[某配置](../../BSP/MC_Board/CMakePresets.json)为例（已人工递归展开）：
```json
{
    "name": "arm-none-eabi-gcc-MC_Board-debug",
    "displayName": "arm-none-eabi-gcc / MC_Board / Debug",
    "inherits": [],
    "sourceDir": ".",
    "binaryDir": "${sourceDir}/build/${presetName}",
    "generator": "Ninja",
    "toolchainFile": "${sourceDir}/cmake/arm-none-eabi-gcc/toolchain.cmake",
    "cacheVariables": {
        "BOARD_NAME": "MC_Board",
        "CMAKE_BUILD_TYPE": "Debug"
    }
}
```

- `name` 字段是 Preset 的**唯一标识符**，CMake 会根据命令行参数 `--preset` 来匹配。
- `displayName` 字段是 Preset 的友好名称（显示名称），会在 IDE 等界面中显示，无效果。
- `inherits` 字段是 Preset 的继承链，CMake 会根据此字段递归展开父 Preset 的配置。
- `sourceDir` 字段是源代码目录，CMake 会在此目录下查找 `CMakeLists.txt` 文件，缺省为当前目录（运行 CMake 的目录）。
- `binaryDir` 字段是构建目录，CMake 会在此目录下生成构建文件，Ninja 等构建工具会在此目录下生成中间文件和最终产物。
- `generator` 字段指定构建工具，CMake 会根据此字段选择生成 Ninja、Makefile 等构建文件。
- `toolchainFile` 字段是工具链文件，CMake 会在此文件中查找交叉编译器和相关工具的路径。
- `cacheVariables` 字段是 CMake 预定义的缓存变量，CMake 会在解析 `CMakeLists.txt` 时使用这些变量。

#### CMake 解析 CMakePresets 的顺序

1. CMake 首先读取并合并 `CMakePresets.json` 和 `CMakeUserPresets.json` 文件。
2. CMake 根据 `include` 字段的顺序，递归读取子 Preset 文件，形成一个有向无环图。
3. CMake 找到指定 `name` 字段与命令行参数 `--preset` 匹配的配置预设，开始解析。
4. CMake 根据 `inherit` 字段，递归展开继承链，从父 Preset 开始解析，遵从以下规则：
    - `cacheVariables` 与 `environment` 字段会被合并。
    - 子 Preset 的字段会覆盖父 Preset 的同名字段。
    - `inherit` 字段为数组时，左侧 Preset 的字段会覆盖右侧 Preset 的同名字段~~视同其父~~。

### 正式的 Configure 阶段

通过命令行参数/预设配置获得 Configure 信息后，CMake 才会正式进入 Configure 阶段。  
这时，CMake 会先加载 `Toolchain` 文件，确认编译器和工具链信息，然后执行 `CMakeLists.txt` 文件，生成 Target 依赖树。

#### Target 是什么

要理解 CMake 的构建系统与包依赖，必须先知道什么是 Target。  
本部分参考[CMake 官方文档](https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html)的解释，旨在尽快建立对 Target 概念的理解。

>A CMake-based buildsystem is organized as a set of high-level logical targets. Each target corresponds to an executable or library, or is a custom target containing custom commands. Dependencies between the targets are expressed in the buildsystem to determine the build order and the rules for regeneration in response to change.

在 CMake 构建系统中，Target 是一个逻辑概念，表示一个可执行文件、库文件或自定义命令的集合，是 CMake 的最小构建单元。  

##### Executable

>Executables are binaries created by linking object files together, one of which contains a program entry point, e.g., main.  
>The add_executable() command defines an executable target

Executable 是一个可执行文件，由多个目标文件链接而成，一般就是整个构建流程的产物。  
在嵌入式构建体系中，Executable 的源一般是 `.s` 的启动文件，其中调用了 Arm C++ 运行时的 `__main` 函数，并最终会调用用户的 `main()` 函数，由此完成程序启动。

以 FineMote 中[某配置](../../CMakeLists.txt#5)为例（已人工替换宏）：
```cmake
add_executable(MC_Board /BSP/MC_Board/MDK-ARM/startup_stm32f446xx.s)
```
其含义即为将启动文件 `startup_stm32f446xx.s` 编译为目标文件，并链接为可执行文件 `MC_Board`。

#### ToolChain 文件的作用

#### CMakeLists.txt 文件的作用

### Configure 阶段的结束

## CMake Generate Stage

## CMake Build Stage



# 嵌入式 C++ 构建备注