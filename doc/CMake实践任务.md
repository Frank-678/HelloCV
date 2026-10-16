## 前言
这次实践任务是完成一个含有两个 CMakeLists.txt 的单位转换器项目。动手之前对 CMake 的印象是"跨平台构建工具"，但这只是个标签。完成之后才理解这句话的具体含义：CMake 不编译代码，它生成让其他工具去编译代码的规则文件；两个 CMakeLists.txt 不是重复，它们各自负责一个构建单元，通过 `add_subdirectory` 建立联系。

第一次写的根 CMakeLists.txt 能通过配置阶段，但链接阶段报错，原因是把头文件路径传给了 `target_link_libraries`——链接器不认识头文件，它只处理符号和库。排查这个错误的过程比写正确的代码学到的东西更多。

记录里保留了完整的报错内容、每一处改动的原因，以及"为什么要这么写"的推导过程。

## 一、CMake 是什么
### 1. 三个核心问题
CMake 解决的问题可以归结为三点。

**告别手写长命令。** 单个 `.cpp` 时，`g++ main.cpp -o app` 够用。几十个源文件加几个第三方库之后，编译命令会膨胀成一大串，写进 shell 脚本也丢失了增量编译的好处——改一个文件不需要重新编译所有文件，但脚本不知道哪些文件变了。CMake 描述的是"哪些文件组成哪个目标、依赖哪些库"，由它生成的构建文件负责追踪增量。

**跨平台。** `make` 是 Unix 工具，Windows 上默认的构建系统是 MSBuild（Visual Studio 项目）。CMake 读同一份 `CMakeLists.txt`，在 Linux 生成 Makefile，在 Windows 可以生成 Visual Studio 项目、Ninja 文件或 MinGW Makefile，取决于传给 `-G` 的参数。

**依赖传播。** 这是现代 CMake 最核心的能力，也是这次任务的重点。`target_link_libraries` 不只是"我依赖这个库"，它还把库声明的头文件搜索路径、编译定义等属性自动传给依赖者。`main.cpp` 能直接写 `#include "unitlib.h"` 而不用写相对路径，完全依赖这个机制。

### 2. CMake 是生成器，不是构建器
这是最重要的定位，也是初次使用时最容易混淆的地方。

```plain
CMakeLists.txt
    ↓  cmake -S . -B build      （配置阶段）
Makefile / build.ninja
    ↓  mingw32-make              （编译阶段）
.o → .a → .exe
```

配置阶段（`cmake ..`）读取 `CMakeLists.txt`，检查语法、寻找编译器、解析依赖，在 `build/` 里生成 Makefile 或其他构建文件。这个阶段出的错是 `CMakeLists.txt` 的问题，修完要重新跑 `cmake ..`。

编译阶段（`mingw32-make`）读取配置阶段生成的 Makefile，调用 `g++` 或 `gcc` 编译每个 `.cpp`，生成 `.o`，再链接成 `.a` 或 `.exe`。这个阶段出的编译错（`error:`）是源码的问题；链接错（`undefined reference`、`cannot find -l`）大概率又是 `CMakeLists.txt` 的问题——库没链对。

这个分层对排查错误很有用。看到错误第一步先判断它是哪个阶段的，再去找对应的原因。

### 3. 源码外构建
把所有编译产物（`.o`、`.a`、`.exe`、Makefile、CMakeCache.txt 等）放进一个独立目录，不和源码混在一起：

```powershell
# 在项目根目录
mkdir build
cd build
cmake .. -G "MinGW Makefiles"
mingw32-make
```

`cmake ..` 里的 `..` 指向上一级——也就是包含根 `CMakeLists.txt` 的目录。当前目录（`build/`）是构建目录，配置阶段生成的所有文件都落在这里。

源码外构建的好处：想清理重来，直接删掉 `build/` 就行，`git clean` 也只需要忽略这一个目录。如果源码里混着 `.o` 文件，清理起来就很麻烦。

`-G "MinGW Makefiles"` 指定生成器。Windows 上 CMake 的默认生成器通常是 Ninja（如果 Ninja 在 PATH 里）或者 Visual Studio（如果检测到 MSVC）。要用 `mingw32-make`，必须显式传这个参数。如果忘了写，cmake 可能生成 Ninja 文件，`mingw32-make` 读不了，直接报错找不到 `Makefile`。

## 二、项目结构
### 1. 任务书规定的目录布局
```plain
UnitConverter/
├── CMakeLists.txt          ← 根文件：工程入口，聚合子模块，生成 exe
├── src/
│   └── main.cpp
├── unitlib/
│   ├── CMakeLists.txt      ← 库的构建文件
│   ├── include/
│   │   └── unitlib.h
│   └── src/
│       └── unitlib.cpp
└── build/                  ← 编译产物（自行创建，不纳入 git）
```

`build/` 不在任务书里，是约定俗成的源码外构建目录。`.gitignore` 里加一行 `build/` 就不会被提交。

两个 `CMakeLists.txt` 各自负责一个构建单元：

+ 根文件：定义工程名、C++ 标准，用 `add_subdirectory` 把 `unitlib` 引入，定义 `UnitConverter` 可执行文件，声明它依赖 `unitlib`
+ `unitlib/CMakeLists.txt`：把 `unitlib.cpp` 打包成静态库，声明对外暴露的头文件目录

两者之间唯一的连接是根文件里的 `add_subdirectory(unitlib)`。没有这行，`unitlib/CMakeLists.txt` 永远不会被 CMake 读到，`unitlib` 库目标不存在，后面的 `target_link_libraries` 就会报找不到目标的错误。

### 2. 根 CMakeLists.txt 完整内容
```cmake
cmake_minimum_required(VERSION 3.16)
project(UnitConverter VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

add_subdirectory(unitlib)

add_executable(${PROJECT_NAME} src/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE unitlib)

target_compile_definitions(${PROJECT_NAME} PRIVATE
    UNITCONVERTER_VERSION="${PROJECT_VERSION}"
)

install(TARGETS ${PROJECT_NAME} RUNTIME DESTINATION bin)
```

逐行说明：

`cmake_minimum_required(VERSION 3.16)` — 声明构建这个工程所需的最低 CMake 版本。3.16 是 2019 年底发布的，包含了现代 CMake 的所有主要特性，在绝大多数系统上都能满足。写 3.6 虽然更宽松，但会触发 deprecation warning，一些行为也和新版本不同。

`project(UnitConverter VERSION 1.0.0 LANGUAGES CXX)` — 定义工程。`VERSION 1.0.0` 让 CMake 自动设置 `${PROJECT_VERSION}`（以及 `${PROJECT_VERSION_MAJOR}` 等拆分变量）。`LANGUAGES CXX` 明确告诉 CMake 只需要 C++ 工具链，不需要检测 C 编译器。省略 `LANGUAGES` 时 CMake 默认同时检测 C 和 CXX，在某些只有 C++ 编译器的环境里会报错找不到 C 编译器。

`set(CMAKE_EXPORT_COMPILE_COMMANDS ON)` — 让 CMake 生成 `compile_commands.json`。这个文件记录了每个源文件编译时的完整命令，是验证"头文件路径是否正确传递"的直接证据，clangd 等语言服务器也依赖它提供代码提示。

`add_subdirectory(unitlib)` — 走进 `unitlib/` 目录，读取那里的 `CMakeLists.txt`，把 `unitlib` 库目标引入当前作用域。这行必须在 `target_link_libraries` 之前，因为后者引用的 `unitlib` 目标在这一行执行前不存在。

`add_executable(${PROJECT_NAME} src/main.cpp)` — 定义可执行文件目标。`${PROJECT_NAME}` 展开为 `UnitConverter`，所以生成的文件是 `UnitConverter.exe`。只列 `main.cpp`，不需要写 `unitlib.cpp`——那由 `unitlib` 目标自己管理。

`target_link_libraries(${PROJECT_NAME} PRIVATE unitlib)` — 声明 `UnitConverter` 依赖 `unitlib`。`PRIVATE` 表示这个依赖只是 `UnitConverter` 自己的，不传播给其他依赖 `UnitConverter` 的目标。这一行做了两件事：告诉链接器把 `libunitlib.a` 链进去，同时把 `unitlib` 声明的 `PUBLIC` 属性（头文件搜索路径等）传给 `UnitConverter`。

`target_compile_definitions` — 在编译 `UnitConverter.exe` 时定义宏 `UNITCONVERTER_VERSION`，值是 `"1.0.0"`。`main.cpp` 里用 `#ifndef UNITCONVERTER_VERSION` 做了防御，如果没有这行定义，就显示 `"unknown"`。

### 3. unitlib/CMakeLists.txt 完整内容
```cmake
add_library(unitlib STATIC
        src/unitlib.cpp
)

target_include_directories(unitlib PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
)

set_target_properties(unitlib PROPERTIES
        CXX_STANDARD 17
        CXX_STANDARD_REQUIRED ON
)

target_compile_definitions(unitlib PRIVATE UNITLIB_BUILDING)

install(TARGETS unitlib ARCHIVE DESTINATION lib)
install(FILES include/unitlib.h DESTINATION include)
```

`add_library(unitlib STATIC src/unitlib.cpp)` — 建库。目标名 `unitlib`，类型 `STATIC`，源文件 `src/unitlib.cpp`。CMake 会根据平台和编译器决定实际的文件名：Linux 是 `libunitlib.a`，Windows（MinGW）也是 `libunitlib.a`，MSVC 是 `unitlib.lib`。

`${CMAKE_CURRENT_SOURCE_DIR}` — CMake 内置变量，值是当前正在处理的 `CMakeLists.txt` 所在目录的绝对路径。在 `unitlib/CMakeLists.txt` 里，它等于 `D:/Downloads/lc/UnitConverter/unitlib`。用它拼路径比写相对路径更可靠，因为相对路径的基准是 CMake 的工作目录（构建目录），而不是源码目录，直接写 `include` 可能出现意外。

`set_target_properties` — 另一种设置 C++ 标准的方式，和根文件里的 `set(CMAKE_CXX_STANDARD 17)` 效果类似，区别是这个只作用于 `unitlib` 目标，而全局的 `CMAKE_CXX_STANDARD` 作用于整个工程的所有目标。

`target_compile_definitions(unitlib PRIVATE UNITLIB_BUILDING)` — 在编译 `unitlib.cpp` 时定义宏 `UNITLIB_BUILDING`，`PRIVATE` 表示不传播。这是个惯用法：库的头文件里可以写 `#ifdef UNITLIB_BUILDING ... #endif`，让同一个头文件在库内部编译和对外使用时有不同行为（比如 dllexport 和 dllimport 的切换）。这里只是演示，`unitlib.h` 本身没有用到这个宏。

## 三、CMakeLists.txt 的编写过程
### 1. 第一次写的版本（有问题）
在没有完整理解 CMake 概念的情况下写出的第一个版本：

```cmake
cmake_minimum_required(VERSION 3.6 FATAL_ERROR)
project(UnitConverter)
include_directories( include )
set(SOURCES src/main.cpp unitlib/src/unitlib.cpp)
set(LIBS unitlib/include/unitlib.h)
add_executable(${PROJECT_NAME} ${SOURCES})
target_link_libraries( ${PROJECT_NAME} ${LIBS} m )
```

`unitlib/CMakeLists.txt` 是 0 字节的空文件，完全没有内容。

这段写法乍看合理，但有三个根本性的问题。

**问题一：**`set(LIBS unitlib/include/unitlib.h)`

这把头文件路径存进了 `LIBS` 变量，然后传给 `target_link_libraries`。链接器（`ld.exe`）的参数应该是库文件名或库目标名，不是头文件路径。链接器处理的是编译产物（`.o`），解析符号引用，它完全不知道 `.h` 文件是什么。

**问题二：**`include_directories( include )`

这是旧式的全局命令，作用范围是当前 `CMakeLists.txt` 里定义的所有目标（以及之后 `add_subdirectory` 引入的所有子目录目标）。这里指定的 `include` 路径是根目录下的 `include/`——但根目录下根本没有这个文件夹，`unitlib.h` 在 `unitlib/include/` 里。所以这行是无效的，没有起到任何作用，也不会直接报错（CMake 不检查 `include_directories` 里的路径是否存在）。

**问题三：**`unitlib/CMakeLists.txt`** 是空文件，且根文件没有 **`add_subdirectory`

两个 `CMakeLists.txt` 之间没有任何联系。`unitlib/CMakeLists.txt` 是零字节，即使有内容也不会被读取，因为根文件没有 `add_subdirectory(unitlib)` 这行调用。

### 2. 运行后的报错
执行 `cmake .. -G "MinGW Makefiles"` 时，配置阶段没有报错（CMake 不验证头文件路径是否存在，也不验证 `target_link_libraries` 的参数是否是合法库名）。

执行 `mingw32-make` 后，链接阶段报错：

```plain
D:/mingw64/bin/../lib/gcc/x86_64-w64-mingw32/14.2.0/../../../../x86_64-w64-mingw32/bin/ld.exe:
cannot find -lunitlib/include/unitlib.h: No such file or directory
collect2.exe: error: ld returned 1 exit status
```

`cannot find -lunitlib/include/unitlib.h` 说明链接器把 `unitlib/include/unitlib.h` 当成了一个库名，加上 `-l` 前缀去找 `libunitlib/include/unitlib.h.a`，当然找不到。

这条错误信息直接指向了根本问题：`target_link_libraries` 的参数应该是库目标名，不是头文件路径。

### 3. 核心误解澄清：编译和链接是两个阶段
这是新手最容易混淆的地方，值得单独梳理一遍。

**编译阶段**：`.cpp` → `.o`

编译器（`g++`）读取 `.cpp` 文件，展开所有 `#include`，把源码翻译成机器指令，生成目标文件 `.o`。这个阶段关心的是：

+ 我调用了 `TemperatureConverter::convert()`，这个函数的声明在哪里？（头文件搜索路径 → `target_include_directories`）
+ 函数签名是什么，参数类型对不对？（由头文件提供）

编译阶段不需要函数的实现在哪里，它只需要声明。

**链接阶段**：`.o` + `.a` → `.exe`

链接器（`ld.exe`）把多个 `.o` 文件合在一起，解析所有未定义的符号引用。`main.cpp.o` 里调用了 `TemperatureConverter::convert()`，链接器要找到这个函数的实现在哪个 `.o` 或 `.a` 里。这个阶段关心的是：

+ 符号 `_ZN20TemperatureConverter7convertEi` 的定义在哪里？（库文件 → `target_link_libraries`）
+ 链接器完全不认识 `.h` 文件

所以：

+ 头文件搜索路径 → `target_include_directories`，作用于编译阶段
+ 库或库目标 → `target_link_libraries`，作用于链接阶段
+ 两者不能互换，传错了是根本性的错误

### 4. 修正过程
根据报错分析，需要做的改动：

1. 删掉 `set(LIBS unitlib/include/unitlib.h)`
2. 在 `unitlib/CMakeLists.txt` 里用 `add_library` 定义库目标，用 `target_include_directories(... PUBLIC ...)` 暴露头文件目录
3. 在根文件里加 `add_subdirectory(unitlib)` 引入子目录
4. `target_link_libraries` 的参数改为库目标名 `unitlib`，不再是路径
5. 删掉无效的 `include_directories( include )`

修正后根文件不再手动指定头文件路径——这个责任交给了 `unitlib` 目标自己，通过 `PUBLIC` 传播到所有依赖它的目标。这才是模块化的正确方式。

## 四、PUBLIC / PRIVATE / INTERFACE 的含义
### 1. 为什么要区分传播范围
`target_include_directories`、`target_link_libraries`、`target_compile_definitions` 等命令都接受 `PUBLIC`、`PRIVATE`、`INTERFACE` 三个关键字。它们控制的是"这个属性传不传给依赖我的人"。

理解这三个词的关键是把"我"和"使用我的目标"分开：

| 关键字 | 编译我自己时用 | 传给链接了我的目标 |
| --- | --- | --- |
| `PRIVATE` | ✓ | ✗ |
| `PUBLIC` | ✓ | ✓ |
| `INTERFACE` | ✗ | ✓ |


`PRIVATE`：这个属性只是我内部的细节，外部不需要知道。比如 `unitlib` 编译时需要一些内部头文件，但这些头文件里的内容只在 `unitlib.cpp` 里用，外部代码不 `#include` 它们，就用 `PRIVATE`。

`PUBLIC`：这个属性不仅我自己需要，任何使用我的目标也需要。`unitlib` 的 `include/` 目录就是这种情况——`unitlib.cpp` 编译时要用，`main.cpp` 调用 `unitlib` 的接口时也要 `#include "unitlib.h"`，所以用 `PUBLIC`，让 CMake 自动把这个目录传给所有依赖 `unitlib` 的目标。

`INTERFACE`：我自己编译时不需要，但使用我的目标需要。这在 header-only 库（只有头文件，没有 `.cpp`）里最常见：库本身没有编译单元，但消费者需要那个头文件目录。

### 2. 在 UnitConverter 里的体现
`unitlib/CMakeLists.txt` 里：

```cmake
target_include_directories(unitlib PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

`PUBLIC` 的效果链：

1. 编译 `unitlib.cpp` 时，编译器的头文件搜索路径包含 `unitlib/include/`
2. 任何 `target_link_libraries(... unitlib)` 的目标，都会自动把 `unitlib/include/` 加进自己的搜索路径
3. `UnitConverter` 的根文件里写了 `target_link_libraries(${PROJECT_NAME} PRIVATE unitlib)`，所以 `main.cpp` 的编译命令里自动带 `-I.../unitlib/include`

结果：`main.cpp` 里只需要写 `#include "unitlib.h"`，不需要写 `#include "../unitlib/include/unitlib.h"`。这个相对路径是脆弱的，移动任何一个文件夹就会失效；由 CMake 传播的搜索路径则跟着 `CMAKE_CURRENT_SOURCE_DIR` 走，始终正确。

### 3. PRIVATE 用在链接上意味着什么
```cmake
target_link_libraries(${PROJECT_NAME} PRIVATE unitlib)
```

这里 `PRIVATE` 的含义：`UnitConverter` 这个可执行文件依赖 `unitlib`，但如果有其他目标依赖 `UnitConverter`（比如测试程序），它们不需要自动链接 `unitlib`。

对可执行文件来说，`PRIVATE` 几乎是固定写法，因为 exe 不会被别人链接。对库来说，选择就更关键：

+ 库 A 的实现里用了库 B，但 A 的头文件里没有 B 的类型或函数 → `PRIVATE`
+ 库 A 的头文件里有 B 的类型 → `PUBLIC`（否则消费者 `#include` A 的头文件时找不到 B 的声明）

### 4. 一个容易忽略的细节：传播是可传递的
如果链是 `C → B → A`（C 依赖 B，B 依赖 A），A 的 `PUBLIC` 属性会先传给 B，再通过 B 传给 C。C 不需要直接 `target_link_libraries(C A)` 就能获得 A 的 `PUBLIC include` 目录。

这是"只链接直接依赖，不手动链接传递依赖"这条现代 CMake 规则成立的原因。老式 CMake 教程里经常看到把所有库都手动列在一个 `target_link_libraries` 里，那是在传播机制普及前的写法，现在不需要这么做。

## 五、构建过程详解
### 1. 配置阶段
在 `build/` 目录里执行 `cmake .. -G "MinGW Makefiles"`，CMake 做以下事情：

1. 检测编译器（扫描 PATH，找到 `g++.exe` 和 `gcc.exe`）
2. 读取根 `CMakeLists.txt`，处理 `cmake_minimum_required` 和 `project`
3. 遇到 `add_subdirectory(unitlib)`，走进 `unitlib/`，读取并处理 `unitlib/CMakeLists.txt`，注册 `unitlib` 目标
4. 回到根文件，处理 `add_executable` 和 `target_link_libraries`，计算依赖图
5. 根据 `target_include_directories(unitlib PUBLIC ...)` 的传播，确定编译 `main.cpp` 时的 `-I` 参数
6. 在 `build/` 里生成 `Makefile`、`CMakeCache.txt`、`cmake_install.cmake`，以及 `unitlib/` 子目录下对应的 Makefile
7. 如果开了 `CMAKE_EXPORT_COMPILE_COMMANDS ON`，同时生成 `compile_commands.json`

PowerShell 里的操作：

```powershell
cd D:\Downloads\lc\UnitConverter
New-Item -ItemType Directory -Name build -Force
cd build
cmake .. -G "MinGW Makefiles"
```

配置成功的末尾输出：

```plain
-- Build files have been written to: D:/Downloads/lc/UnitConverter/build
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792160750506-a17df1ed-df26-4e28-8841-180cc3943ae9.png)如果这里报错，是 `CMakeLists.txt` 的问题，修完再重跑 `cmake ..`，不需要先删掉 `build/`（CMakeCache.txt 会更新）。

如果改了 CMakeLists.txt 的结构性内容（比如改了生成器、删了 `add_subdirectory`），有时需要先删掉 `build/` 再重新配置，否则缓存的旧信息会干扰。

### 2. 编译阶段
配置完成后，在 `build/` 目录里执行 `mingw32-make`：

```powershell
mingw32-make
```

构建输出按依赖顺序排列：

```plain
[ 25%] Building CXX object unitlib/CMakeFiles/unitlib.dir/src/unitlib.cpp.obj
[ 50%] Linking CXX static library libunitlib.a
[ 50%] Built target unitlib
[ 75%] Building CXX object CMakeFiles/UnitConverter.dir/src/main.cpp.obj
[100%] Linking CXX executable UnitConverter.exe
[100%] Built target UnitConverter
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792160771873-574647c3-8ed6-4314-91c1-8c845187e7a8.png)进度百分比的含义：每个构建步骤（每个 `.cpp` 编译、每个链接）占总步骤数的等份。这里有 4 个步骤（编译 `unitlib.cpp`、链接静态库、编译 `main.cpp`、链接 exe），所以每步 25%。

`unitlib`** 先于 **`UnitConverter.exe`** 构建**，这不是偶然，是 CMake 根据 `target_link_libraries` 的依赖关系自动确定的构建顺序。链接 `UnitConverter.exe` 需要 `libunitlib.a` 已经存在，所以 `unitlib` 必须先完成。如果手动调换这个顺序，链接阶段会因为找不到 `libunitlib.a` 而失败。

### 3. 增量构建
构建完成后再执行一次 `mingw32-make`：

```plain
[100%] Built target unitlib
[100%] Built target UnitConverter
```

没有重新编译，因为所有文件都没有改变。Make 用文件时间戳判断是否需要重新构建：如果源文件比目标文件新，才重新编译。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792160794021-2d724d73-5793-4c86-88a6-ea053767366a.png)

修改 `unitlib.cpp` 后再构建：

```plain
[ 25%] Building CXX object unitlib/CMakeFiles/unitlib.dir/src/unitlib.cpp.obj
[ 50%] Linking CXX static library libunitlib.a
[ 50%] Built target unitlib
[ 75%] Building CXX object CMakeFiles/UnitConverter.dir/src/main.cpp.obj
[100%] Linking CXX executable UnitConverter.exe
```

`unitlib.cpp` 改了 → 重新编译 `unitlib.cpp.obj` → 重新链接 `libunitlib.a` → `libunitlib.a` 改了 → 重新链接 `UnitConverter.exe`（因为它依赖库）→ `main.cpp` 没改，但链接步骤还是跑了，因为 `libunitlib.a` 更新了。

只修改 `main.cpp`：

```plain
[ 75%] Building CXX object CMakeFiles/UnitConverter.dir/src/main.cpp.obj
[100%] Linking CXX executable UnitConverter.exe
```

`unitlib` 完全不重新构建，因为它没有被修改。这就是增量构建省时间的地方。

### 4. 验证 compile_commands.json
`build/compile_commands.json` 是验证头文件路径传播的最直接手段。用 PowerShell 读取：

```powershell
Get-Content build\compile_commands.json | python -c "import sys,json; [print(e['command'][:200]) for e in json.load(sys.stdin)]"
```

或者直接打开文件，找 `main.cpp` 对应的条目：

```json
{
  "directory": "D:/Downloads/lc/UnitConverter/build",
  "command": "C:\\mingw64\\bin\\g++.exe  -DUNITCONVERTER_VERSION=\\\"1.0.0\\\"
              -ID:/Downloads/lc/UnitConverter/unitlib/include
              -std=c++17 ...
              -o CMakeFiles/UnitConverter.dir/src/main.cpp.obj
              -c D:/Downloads/lc/UnitConverter/src/main.cpp",
  "file": "D:/Downloads/lc/UnitConverter/src/main.cpp"
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792160833636-57562926-88b8-4758-9ecf-58e600ef1329.png)`-ID:/Downloads/lc/UnitConverter/unitlib/include` 这个参数是 CMake 通过依赖传播自动加进去的——根文件里从来没有手动写过这个路径，但它出现在 `main.cpp` 的编译命令里。这是 `PUBLIC` 传播的直接证据。

`-DUNITCONVERTER_VERSION=\"1.0.0\"` 也在这里，是 `target_compile_definitions` 的效果。`main.cpp` 里的 `#ifndef UNITCONVERTER_VERSION` 守卫就被这个值覆盖了，程序启动时会打印 `单位转换器 v1.0.0`。

## 六、旧式写法 vs 现代写法
### 1. 全局命令的问题
CMake 2.x 时代的写法是全局命令：

```cmake
include_directories(include/)          # 影响当前目录及所有子目录的所有目标
link_directories(/usr/lib/mylib)       # 全局库搜索路径
add_definitions(-DFOO=1)               # 全局编译定义
link_libraries(mylib)                  # 所有目标都链接这个库
```

这些命令是"全局感染"——一旦调用，就会影响它之后在相同目录和子目录里定义的所有目标，无论那些目标是否需要。在大项目里，这意味着一个模块的内部需求会悄悄地污染另一个模块的构建设置，造成难以追踪的编译问题。

现代 CMake（3.0 以后）的做法是把所有属性挂在目标（target）上，通过依赖关系显式传播：

```cmake
target_include_directories(mylib PUBLIC include/)      # 只影响 mylib，以及 PUBLIC 传播的范围
target_compile_definitions(mylib PRIVATE FOO=1)        # 只影响 mylib 内部
target_link_libraries(myapp PRIVATE mylib)             # 显式声明 myapp 依赖 mylib
```

### 2. 一个具体的反例
假设有两个不相关的库 `libA` 和 `libB`，以及两个可执行文件 `appX` 和 `appY`：

```cmake
# 旧写法
include_directories(libA/include)    # 原本只想让 appX 能找到 A 的头文件
add_executable(appX src/x.cpp)
add_executable(appY src/y.cpp)       # appY 也意外获得了 libA/include 的搜索路径
```

`appY` 对 `libA` 一无所知，但因为 `include_directories` 是全局的，它的编译命令里带着 `libA/include`。这本身不会出错，但会产生误导：某天 `libA/include/` 里有个 `math.h`，`appY` 可能意外包含了它而不是系统的 `<math.h>`，出了奇怪的 bug。

现代写法不存在这个问题：

```cmake
target_include_directories(libA PUBLIC libA/include)

add_executable(appX src/x.cpp)
target_link_libraries(appX PRIVATE libA)   # 只有 appX 获得 libA/include

add_executable(appY src/y.cpp)
# appY 完全不涉及 libA
```

### 3. 识别项目里的旧式写法
看到 `CMakeLists.txt` 里有以下命令，就是在用旧式写法：

| 旧式命令 | 现代等价 |
| --- | --- |
| `include_directories(...)` | `target_include_directories(TARGET PRIVATE/PUBLIC ...)` |
| `link_directories(...)` | `target_link_directories(TARGET PRIVATE ...)` |
| `add_definitions(...)` | `target_compile_definitions(TARGET PRIVATE ...)` |
| `add_compile_options(...)` | `target_compile_options(TARGET PRIVATE ...)` |
| `link_libraries(...)` | `target_link_libraries(TARGET PRIVATE ...)` |


旧式写法在老项目里很常见，不是说它一定会出问题，但理解它们的作用范围比理解 `target_*` 更难，排查问题时更难追踪。新项目应该只用 `target_*` 系列。

## 七、为什么要有两个 CMakeLists.txt
### 1. 任务书的原话
任务书里这样解释这个要求：

> 这里限制了目录结构甚至要求大家写两个 CMakeLists.txt 是希望能在小项目中尽可能模拟大家日后会接触到的更加庞大的项目的结构。
>

这句话的重点在"模拟"。`UnitConverter` 只有两个 `.cpp` 文件，把它们都列进一个 `add_executable` 是完全可行的，用一个 CMakeLists.txt 就够了。任务书刻意要求两个，是因为它想让你在小项目里感受一次大项目的组织方式。

### 2. 大项目里这个结构意味着什么
设想一个有 20 个模块的真实项目：

```plain
bigproject/
├── CMakeLists.txt
├── core/
│   ├── CMakeLists.txt
│   └── src/ include/
├── network/
│   ├── CMakeLists.txt
│   └── src/ include/
├── storage/
│   ├── CMakeLists.txt
│   └── src/ include/
├── ui/
│   ├── CMakeLists.txt
│   └── src/ include/
... （16 个模块）
└── app/
    ├── CMakeLists.txt
    └── src/main.cpp
```

根 CMakeLists.txt 只需要：

```cmake
cmake_minimum_required(VERSION 3.16)
project(BigProject)

add_subdirectory(core)
add_subdirectory(network)
add_subdirectory(storage)
add_subdirectory(ui)
# ...
add_subdirectory(app)
```

`app` 的 CMakeLists.txt：

```cmake
add_executable(bigapp src/main.cpp)
target_link_libraries(bigapp PRIVATE core network storage ui)
```

每个模块只管好自己的 CMakeLists.txt，声明自己的源文件、头文件目录、内部依赖。根文件不需要知道每个模块内部的目录结构，只知道目标名。模块之间的依赖通过目标名声明，CMake 自动推算构建顺序、传播头文件路径、解析链接依赖。

这和软件工程里的接口隔离原则是一回事：调用者只依赖接口（目标名），不依赖实现（源文件路径、内部目录结构）。

### 3. 从 UnitConverter 看这个原则
`UnitConverter` 的根文件里：

```cmake
add_subdirectory(unitlib)
add_executable(${PROJECT_NAME} src/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE unitlib)
```

根文件不知道 `unitlib` 的源码在 `src/unitlib.cpp`，不知道头文件在 `include/`。它只知道有个叫 `unitlib` 的目标，并且 `UnitConverter.exe` 依赖它。

如果把 `unitlib` 的内部结构从 `src/` 改成 `lib/`，或者增加新的源文件，根文件完全不需要改，只改 `unitlib/CMakeLists.txt` 就好。这种封装在只有两个文件的项目里体现不出价值，但在 20 个模块的项目里，这是防止整个构建系统变成牵一发动全身的必要手段。

## 八、常见报错与排查
### 1. 链接阶段：cannot find -lXXX
```plain
ld.exe: cannot find -lunitlib/include/unitlib.h: No such file or directory
```

原因：`target_link_libraries` 的参数是一个路径或文件名，不是目标名，链接器把它当成了库名 `-lXXX`。

排查步骤：

1. 检查 `target_link_libraries` 的参数，确认它是目标名（不带路径、不带扩展名）
2. 检查产生这个目标名的 `add_library` 或 `add_executable` 是否已经被 CMake 处理（`add_subdirectory` 是否在 `target_link_libraries` 之前）

### 2. 编译阶段：No such file or directory（头文件）
```plain
fatal error: unitlib.h: No such file or directory
```

原因：编译器找不到头文件。

排查步骤：

1. 确认 `target_include_directories` 是否加在正确的目标上
2. 检查传播关键字：如果 `main.cpp` 找不到 `unitlib.h`，要么 `unitlib` 没用 `PUBLIC`，要么 `UnitConverter` 没有链接 `unitlib`
3. 检查 `compile_commands.json` 里 `main.cpp` 的编译命令，看 `-I` 参数里有没有 `unitlib/include`
4. 检查 `${CMAKE_CURRENT_SOURCE_DIR}` 是否正确——如果在根文件里写了 `target_include_directories(unitlib PUBLIC include)`，`include` 是相对于根目录的 `include/`，不是 `unitlib/include/`

### 3. 编译阶段：undefined reference（链接）
```plain
undefined reference to `TemperatureConverter::convert(int)'
```

原因：头文件里有声明，但对应的实现没有被编译进来。

排查步骤：

1. 确认包含实现的 `.cpp` 文件已经列在 `add_library` 或 `add_executable` 里
2. 确认对应的库已经用 `target_link_libraries` 链进来

### 4. 配置阶段：CMake Error: The source directory does not appear to contain CMakeLists.txt
```plain
CMake Error: The source directory "D:/Downloads/lc/UnitConverter" does not appear to contain CMakeLists.txt.
```

原因：`cmake` 命令的参数指向的目录里没有 `CMakeLists.txt`，或者在错误的目录里执行了命令。

检查：确认当前目录是 `build/`，`cmake ..` 里的 `..` 指向上一级（含 `CMakeLists.txt` 的目录）。

### 5. 配置阶段：No CMAKE_CXX_COMPILER could be found
```plain
CMake Error: No CMAKE_CXX_COMPILER could be found.
```

原因：CMake 在 PATH 里找不到 C++ 编译器。

排查步骤：

1. 确认 `g++.exe` 在 PATH 里：`g++ --version`
2. 如果用了 `-G "MinGW Makefiles"`，确认 MinGW 工具链安装完整
3. 也可以显式指定：`cmake .. -G "MinGW Makefiles" -DCMAKE_CXX_COMPILER=C:/mingw64/bin/g++.exe`

### 6. 运行时：exe 被杀毒软件删除
构建成功，但 `UnitConverter.exe` 在 2 秒内消失。这是 Windows Defender 对新生成的本地 exe 进行快速扫描，不是 CMake 问题。

解决：把 `build/` 目录加入 Windows Defender 的排除列表。路径：`Windows 安全中心 → 病毒和威胁防护 → 管理设置 → 排除项 → 添加排除项 → 文件夹`。

### 7. 一个有用的诊断思路
配置阶段（`cmake ..`）的错 → `CMakeLists.txt` 语法、路径、目标名问题  
编译阶段（`mingw32-make` 出现 `error:`）的错 → 源码问题  
链接阶段（`mingw32-make` 出现 `undefined reference` 或 `cannot find -l`）的错 → 大概率又是 `CMakeLists.txt` 的问题（库没链对、目标名写错）

## 九、调试构建与编译选项
### 1. 构建类型
CMake 内置四种构建类型：

| 类型 | 优化 | 调试信息 | 断言 |
| --- | --- | --- | --- |
| `Debug` | `-O0` | `-g` | 开 |
| `Release` | `-O3` | 无 | 关 |
| `RelWithDebInfo` | `-O2` | `-g` | 关 |
| `MinSizeRel` | `-Os`（最小体积） | 无 | 关 |


指定方式：

```powershell
cmake .. -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug
```

也可以在 `CMakeLists.txt` 里设默认值（如果命令行没传就用这个）：

```cmake
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type" FORCE)
endif()
```

### 2. 手动添加编译选项
```cmake
# 只对 Debug 构建加 -Wall -Wextra
target_compile_options(UnitConverter PRIVATE
    $<$<CONFIG:Debug>:-Wall -Wextra -Wpedantic>
    $<$<CONFIG:Release>:-O3>
)
```

`$<$<CONFIG:Debug>:...>` 是 CMake 的生成器表达式（Generator Expression），在配置阶段不展开，在生成构建文件时根据当前构建类型展开。`CONFIG:Debug` 表示"当前是 Debug 构建时"，整个表达式展开为 `-Wall -Wextra -Wpedantic`，否则展开为空字符串。

生成器表达式功能很强，但语法比较繁琐。常用的几个：

```cmake
$<CONFIG:Debug>                  # 布尔值，当前是 Debug 时为 1
$<TARGET_FILE:mylib>             # mylib 的输出文件绝对路径
$<BOOL:${MY_VARIABLE}>           # 把变量转成布尔值
$<$<PLATFORM_ID:Windows>:win32>  # Windows 平台时展开为 "win32"
```

### 3. 设置 C++ 标准
有三种方式，优先级从高到低：

```cmake
# 方式一：全局变量（影响所有目标）
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)   # 关闭 GNU 扩展（__extension__ 等）

# 方式二：目标属性（只影响这一个目标）
set_target_properties(unitlib PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
)

# 方式三：直接用 compile_features（最精确，按特性而非版本）
target_compile_features(unitlib PRIVATE cxx_std_17)
```

方式三的好处是语义更清晰——"我需要 C++17 的特性"，而不是"我需要 C++17 这个版本号"。如果用了 C++20 的某个特性，写 `cxx_std_20` 比写 `CMAKE_CXX_STANDARD 20` 更自描述。

## 十、install 规则与安装路径
### 1. install 命令的作用
`cmake --install build` 执行时，CMake 把构建产物复制到安装目录。`CMakeLists.txt` 里的 `install()` 命令描述"复制什么、放哪里"。

```cmake
# 安装可执行文件
install(TARGETS UnitConverter RUNTIME DESTINATION bin)

# 安装静态库
install(TARGETS unitlib ARCHIVE DESTINATION lib)

# 安装头文件
install(FILES include/unitlib.h DESTINATION include)

# 安装目录（递归）
install(DIRECTORY doc/ DESTINATION share/UnitConverter/doc)
```

`DESTINATION` 是相对于安装前缀的路径。默认安装前缀：

+ Linux：`/usr/local`
+ Windows（MinGW）：`C:/Program Files (x86)/UnitConverter`

### 2. 自定义安装前缀
```powershell
cmake .. -G "MinGW Makefiles" -DCMAKE_INSTALL_PREFIX=D:/myapp
mingw32-make
cmake --install .
```

安装后的目录结构：

```plain
D:/myapp/
├── bin/
│   └── UnitConverter.exe
├── lib/
│   └── libunitlib.a
└── include/
    └── unitlib.h
```

### 3. 组件安装
可以把安装分成多个组件，按需安装：

```cmake
install(TARGETS UnitConverter RUNTIME DESTINATION bin COMPONENT runtime)
install(TARGETS unitlib ARCHIVE DESTINATION lib COMPONENT dev)
install(FILES include/unitlib.h DESTINATION include COMPONENT dev)
```

只安装运行时组件（不含开发头文件）：

```powershell
cmake --install . --component runtime
```

这在打包发布时很有用：给最终用户的安装包只包含 `runtime` 组件，给开发者的 SDK 包含所有组件。

## 十一、compile_commands.json 的用途
### 1. 它是什么
`compile_commands.json` 是一个 JSON 数组，每个元素描述一个文件的编译命令：

```json
[
  {
    "directory": "D:/Downloads/lc/UnitConverter/build",
    "command": "g++.exe -DUNITCONVERTER_VERSION=\"1.0.0\" -ID:/Downloads/lc/UnitConverter/unitlib/include -std=c++17 -o CMakeFiles/UnitConverter.dir/src/main.cpp.obj -c D:/Downloads/lc/UnitConverter/src/main.cpp",
    "file": "D:/Downloads/lc/UnitConverter/src/main.cpp"
  },
  {
    "directory": "D:/Downloads/lc/UnitConverter/build/unitlib",
    "command": "g++.exe -DUNITLIB_BUILDING -ID:/Downloads/lc/UnitConverter/unitlib/include -std=c++17 -o CMakeFiles/unitlib.dir/src/unitlib.cpp.obj -c D:/Downloads/lc/UnitConverter/unitlib/src/unitlib.cpp",
    "file": "D:/Downloads/lc/UnitConverter/unitlib/src/unitlib.cpp"
  }
]
```

在 `UnitConverter` 的条目里可以看到：

+ `-DUNITCONVERTER_VERSION=\"1.0.0\"`：来自 `target_compile_definitions`
+ `-ID:/Downloads/lc/UnitConverter/unitlib/include`：由 `unitlib` 的 `PUBLIC target_include_directories` 传播而来，根文件从没手写过这个路径

在 `unitlib.cpp` 的条目里：

+ `-DUNITLIB_BUILDING`：`PRIVATE` 的编译定义，只在这个文件出现，不在 `main.cpp` 的命令里

这两处是 CMake 依赖传播机制的直接物证。

### 2. clangd 和 IDE 的依赖
`clangd` 是基于 LLVM 的 C/C++ 语言服务器，VS Code 的 C/C++ 扩展、CLion 等 IDE 都可以使用它提供代码补全、跳转、诊断等功能。

clangd 要知道"编译某个文件时用了哪些 `-I` 和 `-D`"才能正确解析头文件和宏。`compile_commands.json` 就是这份信息的标准格式。把它放在项目根目录（或者让 clangd 能找到它）：

```powershell
# 构建完成后，把 compile_commands.json 复制到根目录
Copy-Item build\compile_commands.json .
```

或者让 CMake 直接在根目录生成（通过软链接或配置）。有了这个文件，在 VS Code 里打开 `main.cpp`，`#include "unitlib.h"` 不会显示为找不到文件，`TemperatureConverter` 的方法也有代码补全。

### 3. 排查头文件问题时用它
如果编译成功但 IDE 报"找不到 unitlib.h"，或者代码补全缺失，通常是 `compile_commands.json` 不存在或者 IDE 没有找到它，而不是 CMakeLists.txt 的问题。

步骤：

1. 确认 `CMakeLists.txt` 里有 `set(CMAKE_EXPORT_COMPILE_COMMANDS ON)`
2. 重新跑 `cmake ..`，确认 `build/compile_commands.json` 生成了
3. 检查文件里 `main.cpp` 对应的 `command` 字段，确认 `-I` 路径正确
4. 配置 IDE 指向这个文件，或者复制到项目根目录

## 十二、如果要再加文件呢？
如果想在 `unitlib` 里加一个新的转换类（比如速度转换），需要改的地方：

**1. 新建 **`unitlib/include/speed.h`（类声明）

**2. 新建 **`unitlib/src/speed.cpp`（实现）

**3. 修改 **`unitlib/CMakeLists.txt`，把新文件加进去：

```cmake
add_library(unitlib STATIC
    src/unitlib.cpp
    src/speed.cpp       # 新加这一行
)
```

`target_include_directories` 不需要改——新文件也在 `include/` 里，搜索路径已经对了。

**4. 根文件和 **`main.cpp`** 完全不需要改。**

这个改动范围的局限性是两个 CMakeLists.txt 设计的最好证明：新增模块内部文件，改动只在模块自己的 CMakeLists.txt 里，不影响任何外部文件。如果所有文件都列在根文件里，就必须改根文件。

如果想加一个全新的模块（比如 `formatter/`）：

```plain
UnitConverter/
├── CMakeLists.txt
├── formatter/             ← 新目录
│   ├── CMakeLists.txt
│   ├── include/
│   │   └── formatter.h
│   └── src/
│       └── formatter.cpp
...
```

`formatter/CMakeLists.txt`：

```cmake
add_library(formatter STATIC src/formatter.cpp)
target_include_directories(formatter PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

根 `CMakeLists.txt` 加两行：

```cmake
add_subdirectory(formatter)
target_link_libraries(${PROJECT_NAME} PRIVATE unitlib formatter)
```

其他文件不需要动。`main.cpp` 里加上 `#include "formatter.h"` 就能用新库——路径由 `formatter` 目标的 `PUBLIC` 自动传播。

## 十三、清理和重新配置
### 1. 什么时候需要清理
大部分改动不需要清理，直接重新 `mingw32-make` 就行。但有些改动会让 CMakeCache.txt 里缓存的旧信息失效，必须删掉 `build/` 从头来：

+ 改了生成器（从 Ninja 换成 MinGW Makefiles）
+ 改了 `CMAKE_CXX_COMPILER` 指向的编译器
+ 删掉了某个 `add_subdirectory` 或重命名了目标
+ 升级了 CMake 大版本

这些情况如果不清理，`cmake ..` 可能报"cache is stale"或者静默地用旧的编译器路径。

### 2. 清理操作
```powershell
# 最彻底：删掉整个 build 目录
Remove-Item -Recurse -Force build
New-Item -ItemType Directory -Name build
cd build
cmake .. -G "MinGW Makefiles"
mingw32-make
```

```powershell
# 只清理编译产物，保留 CMakeCache（更快，但偶尔不够干净）
mingw32-make clean
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792160995844-ebbe3890-882f-4db4-ada0-a1a55b45b908.png)`make clean` 删掉 `.o`、`.a`、`.exe`，但保留 CMake 生成的 Makefile 和缓存。重新 `mingw32-make` 会重新编译所有文件，但跳过配置阶段。这对"改了源码想完整重编"的场景够用，对"改了 CMakeLists.txt 结构"的场景不够。

## 参考资料
+ **[T1]** [CMake 官方文档：add_library](https://cmake.org/cmake/help/latest/command/add_library.html)
+ **[T2]** [CMake 官方文档：target_include_directories](https://cmake.org/cmake/help/latest/command/target_include_directories.html)
+ **[T3]** [CMake 官方文档：target_link_libraries](https://cmake.org/cmake/help/latest/command/target_link_libraries.html)
+ **[T4]** [CMake 官方文档：add_subdirectory](https://cmake.org/cmake/help/latest/command/add_subdirectory.html)
+ **[T5]** [CMake 官方文档：target_compile_definitions](https://cmake.org/cmake/help/latest/command/target_compile_definitions.html)
+ **[T6]** [CMake 官方文档：install](https://cmake.org/cmake/help/latest/command/install.html)
+ **[T7]** [CMake 入门实战 (hahack.com)](https://www.hahack.com/codes/cmake/)
+ **[T8]** [Zero to CMake (buchanan.one)](https://buchanan.one/blog/zero-to-cmake/)
+ **[T9]** [Learn CMake in Y Minutes](https://learnxinyminutes.com/zh-cn/cmake/)
+ **[T10]** [compile_commands.json 格式说明（LLVM）](https://clang.llvm.org/docs/JSONCompilationDatabase.html)

