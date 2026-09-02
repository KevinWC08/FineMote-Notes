# 快速上手 *FineMote*

## *FineMote* 是什么？
FineMote 是一套针对机器人应用的嵌入式代码框架，通过抽象层设计和内置的调度器，为嵌入式开发者提供高效、易用的开发环境。  
本教程从源码获取与编译开始，基于真实开发场景，手把手指导如何快速上手 FineMote，并在机器人应用中实现具体功能。

## 获取并编译 *FineMote*

### 获取源码

FineMote 当前通过[GitHub](https://github.com/FINS-Fines/FineMote)分发，可以使用 vscode 自带的存储库功能克隆 https://github.com/FINS-Fines/FineMote.git ，或者在命令行中使用以下指令获取源码：

```console
git clone https://github.com/FINS-Fines/FineMote.git
```
```text
Cloning into 'FineMote'...
remote: Enumerating objects: 9808, done.
remote: Counting objects: 100% (477/477), done.
remote: Compressing objects: 100% (258/258), done.
remote: Total 9808 (delta 315), reused 338 (delta 218), pack-reused 9331 (from 3)
Receiving objects: 100% (9808/9808), 18.01 MiB | 2.28 MiB/s, done.
Resolving deltas: 100% (5667/5667), done.
Updating files: 100% (3875/3875), done.
```

完成后，使用 vscode 打开仓库目录（右键 -> 通过Code打开），出现以下界面即为完成：

<p align="center">
  <img src="获取源码.png" width="600">
</p>

### 配置编译环境

FineMote 支持多种工具链，本文以 Windows 平台为例，推荐使用 armclang 编译器进行编译。

#### 省流版

1. 下载并安装[CMake 4.4.2](https://github.com/Kitware/CMake/releases/download/v4.4.2/cmake-4.4.2-windows-x86_64.msi)，勾选 *Add CMake to the system PATH for all users*。  
2. 下载并解压[Ninja 1.13.2](https://github.com/ninja-build/ninja/releases/download/v1.13.2/ninja-win.zip)。  
3. 下载并安装[arm-gnu-toolchain-15.3.rel1](https://gitlab.arm.com/api/v4/projects/tooling%2Fgnu-toolchains-for-arm/packages/generic/gnu-toolchain/15.3.rel1/arm-gnu-toolchain-15.3.rel1-mingw-w64-x86_64-aarch64-none-elf.msi)。  
4. 为 Ninja 和 arm-none-eabi-gcc 添加环境变量。  
5. 在 vscode 中安装[CMake插件](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools)。
6. 快速校验环境 `cmake --version && echo. && ninja --version && echo. && arm-none-eabi-gcc --version`。

#### 安装 CMake

FineMote 使用 CMake 作为构建系统生成工具，并依此实现板卡选择。  
可以在[CMake下载页](https://cmake.org/download/)获取 CMake，选择对应的操作系统版本下载并安装，安装过程中建议勾选 *Add CMake to the system PATH for all users*，以便在命令行中直接使用 CMake。  
截至此时，CMake 的最新版本为[CMake 4.4.2](https://github.com/Kitware/CMake/releases/download/v4.4.2/cmake-4.4.2-windows-x86_64.msi)。  
完成安装后，使用命令行运行以下命令，验证 CMake 是否安装成功：

```console
cmake --version
```
```text
cmake version 4.4.2

CMake suite maintained and supported by Kitware (kitware.com/cmake).

```

记得为 vscode 安装[CMake插件](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools)。

#### 安装 Ninja

Ninja 是一个小巧的构建系统，FineMote 使用 Ninja 作为默认构建工具。  
可以在[Ninja发布页](https://github.com/ninja-build/ninja/releases)获取 Ninja，选择对应的操作系统版本，下载后将其解压到任意目录（建议使用简单的纯英文目录，如 `D:\ninja`），记得将 Ninja 的路径添加到环境变量中。  
截至此时，Ninja 的最新版本为[Ninja 1.13.2](https://github.com/ninja-build/ninja/releases/download/v1.13.2/ninja-win.zip)。  
完成安装后，使用命令行运行以下命令，验证 Ninja 是否安装成功：

```console
ninja --version
```
```text
1.13.2
```

#### 获取 armclang 编译器

我们推荐使用 armclang 编译器进行编译，armclang 是 ARM 官方提供的编译器，支持最新的 ARM 架构和优化。  
获取*可以使用的* armclang 编译器，最简单的方式是随 Keil 一同获得（如何获得 Keil 的许可建议自行搜索）。  
在 Keil 安装目录下，armclang 编译器的路径一般为 `\Keil_v5\ARM\ARMCLANG\bin\armclang.exe`，将其添加到环境变量中。

#### 获取 arm-none-eabi-gcc 编译器

若在获取 armclang 编译器时遇到困难，也可以使用 Arm GNU 工具链进行编译。  
arm-none-eabi-gcc 是 GNU 工具链的一部分，专门为 ARM 架构设计，是开源的免费编译器。  
可以在[Arm GNU发布页](https://gitlab.arm.com/tooling/gnu-toolchains-for-arm)获取 arm-none-eabi-gcc，选择对应的操作系统版本下载并安装，记得将 arm-none-eabi-gcc 的路径添加到环境变量中，一般为安装目录下的 `bin` 文件夹。  
截至此时，Arm GNU 工具链的最新版本为[arm-gnu-toolchain-15.3.rel1](https://gitlab.arm.com/api/v4/projects/tooling%2Fgnu-toolchains-for-arm/packages/generic/gnu-toolchain/15.3.rel1/arm-gnu-toolchain-15.3.rel1-mingw-w64-x86_64-aarch64-none-elf.msi)。  
完成安装后，使用命令行运行以下命令，验证 arm-none-eabi-gcc 是否安装成功：

```console
arm-none-eabi-gcc --version
```
```text
arm-none-eabi-gcc (Arm GNU Toolchain 15.3.Rel1 (Build arm-15.149)) 15.3.1 20260627
Copyright (C) 2025 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

### 编译 FineMote

使用CMake 构建 FineMote，可以选择在命令行中使用 CMake，也可以在 IDE 中使用 CMake 插件进行构建。

#### 通过 IDE 构建

以 VSCode 为例，在安装 CMake 插件后打开有效的 CMake 项目目录，插件识别后在左侧功能栏会出现 CMake 图标，点击后会显示 CMake 工具栏。

<p align="center">
  <img src="使用CMake构建.png" width="200">
</p>

在工作栏中，

#### 使用命令行构建

### 后记

如果你与笔者一样，第一次接触复杂的工程 C++ 项目，可能会觉得工程 C++ 项目的配置与构建有点复杂。  
~~的确，C++ 的构建流程不说是简洁明了，也只能说是非常复杂。~~  
并且，构建生成工具 CMake 的学习曲线相对陡峭，但掌握 CMake 的使用方法对嵌入式开发又是相对必要的。  
因此，在此推荐阅读笔者的另一篇笔记 [C++ 构建快速指南](../../Learning%20Notes/C%2B%2B%E6%9E%84%E5%BB%BA%E5%BF%AB%E9%80%9F%E6%8C%87%E5%8D%97.md)，快速了解完整构建一个 C++ 工程的流程，掌握基本的 CMake 使用方法。

## 用 *FineMote* 控制电机

