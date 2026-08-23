
# *CMake*速通指南

## 记录格式

本文档是笔者学习CMake的笔记，从CMake执行时的 **真实行为** 出发，记录各类文件的作用与规范，并指出其在实际开发中的使用方法或潜在功能。

## *CMake*的执行阶段和内存模型


已知的 CMake 的执行阶段：  
~~- 预解析 preset 并生成 `CMakeCache.txt`~~
~~- 加载 `CMakeCache.txt`~~
~~- 解析 `CMakeLists.txt` 并生成 `build.ninja` 或 `Makefile`~~
~~- 调用 `ninja` 或 `make` 执行构建~~


## 跟我变身*CMake*

### 预解析 CMakePresets

当携带参数 `--preset` 启动 CMake时，CMake 会首先解析 `CMakePresets.json` 和 `CMakeUserPresets.json` 文件中的预设配置。

#### CMakePresets 格式规则

以 FineMote 中某配置为例（已人工递归展开）：
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

在阅读 CMakePresets 文件时，我们一般关注到 `include` 和 `configurePresets` 字段即可，前者用于组织各 Preset 文件，后者定义了具体的配置项，在下一节中说明。

#### configurePresets 配置格式

以 FineMote 中某配置为例（已人工递归展开）：
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

当预解析 CMakePresets 完成后，CMake 已将预设配置加载至内存中，值得注意的是，*指定预设配置与直接使用命令行参数启动 CMake 的效果完全等同*，预设用于方便管理和复用配置。  
通过命令行参数/预设配置获得 Configure 信息后，CMake 才会开始正式的 Configure 阶段。  