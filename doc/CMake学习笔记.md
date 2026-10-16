## 前言
这份笔记是完成 UnitConverter 实践任务过程中的知识整理。主要参考了四份材料，它们各自的定位不同，正好互补：

| 材料 | 视角 | 补的是哪一块 |
| --- | --- | --- |
| [Zero to CMake](https://buchanan.one/blog/zero-to-cmake/)（Buchanan，2024-12） | 底层还原 | 从手工敲 g++ 开始，到手写 Makefile，再到 CMake 生成 Makefile，讲清"为什么需要 CMake" |
| [CMake 入门实战](https://www.hahack.com/codes/cmake/)（hahack.com，2013） | 案例递进 | Demo1 到 Demo8，逐级加上多源文件、库、编译选项、安装测试、打包 |
| [Learn CMake in Y Minutes](https://learnxinyminutes.com/zh-cn/cmake/) | 语言速查 | CMake 脚本语言本身：语法、变量作用域、控制流、message 等级 |
| [CMake 官方文档](https://cmake.org/cmake/help/latest/) | 权威定义 | 现代 targets 模型与 PUBLIC/PRIVATE/INTERFACE 传播 |


Zero to CMake 那篇的组织方式很有意思：它**倒着讲**。先给你看最原始的 `g++ -o hello main.cpp`，然后是手写的 Makefile，最后才是生成 Makefile 的 CMake。这样每一层的存在理由都从上一层的痛点里自然长出来，而不是先给结论再补理由。

这份笔记沿用这个顺序。前三节是"底下发生了什么"，中间是"CMake 的语言和用法"，后面才是"现代 CMake 应该怎么写"。

凡是项目里实际踩过的坑，都写了报错原文和排查思路。

## 一、CMake 是什么
### 1. 从一个具体问题开始
假设有一个跨平台项目要在 Linux 和 Windows 上编译。历史上各个平台各有自己的构建工具：

| 工具 | 平台 / 生态 | Makefile 格式 |
| --- | --- | --- |
| GNU Make | Unix / Linux | 自己的语法 |
| nmake | Windows MSVC | 兼容但扩展不同 |
| qmake | Qt 项目 | 自己的 `.pro` 格式 |
| pmake | BSD | 自己的约定 |
| Makepp | Perl 生态 | 自己的扩展 |


它们的语法、变量约定、内置规则都不同，互相不兼容。**要为两个平台编译同一份代码，就得维护两份构建脚本，改一处要同步改两处。**

Wheeler 那句被反复引用的话在这里很贴切：计算机科学里的任何问题都可以通过再加一层间接来解决。CMake 就是那一层——写一份和平台无关的 `CMakeLists.txt` 描述"要构建什么"，由 CMake 生成各平台本地的构建文件：

```plain
        一份描述
     CMakeLists.txt
            │
      ┌─────┼─────┬──────────┐
      ↓     ↓     ↓          ↓
  Makefile  .sln  build.ninja  Xcode
  (Linux) (Windows) (Ninja)   (macOS)
```

这就是"Write once, run everywhere"在构建领域的版本。用 CMake 的知名项目包括 VTK、ITK、KDE、OpenCV、OSG——它们都需要在多个平台上构建，也正是 CMake 最早被设计出来要解决的问题。

### 2. CMake 是生成器，不是构建器
这是最关键的定位，也是初学者最容易搞混的一点。

**CMake 本身一行代码都不编译。** 它读取 `CMakeLists.txt`，检查语法、检测编译器、解析依赖关系，然后生成让别的工具去干活的文件。真正调用 `g++` 的是 make、ninja 或 MSBuild。

```plain
CMakeLists.txt
      │
      │  cmake -S . -B build       ← 配置阶段 configure
      ↓
build/Makefile 或 build/build.ninja
      │
      │  mingw32-make 或 ninja      ← 构建阶段 build
      ↓
   .o  →  .a  →  .exe
```

这个分层不只是概念洁癖，它直接决定了排查错误的方向：

| 出错阶段 | 报错的样子 | 先怀疑什么 |
| --- | --- | --- |
| 配置阶段 | `CMake Error at CMakeLists.txt:12 (add_executable):` | `CMakeLists.txt` 的语法、路径、目标名 |
| 编译阶段 | `main.cpp:15:10: error: expected ';' before '}' token` | 源码 |
| 链接阶段 | `undefined reference to 'TemperatureConverter::convert(int)'` | 大概率还是 `CMakeLists.txt`：库没链对 |
| 运行阶段 | 启动即崩溃，或提示找不到 `.dll` | 运行时依赖与路径 |


Zero to CMake 的结尾把这条总结成一句话：**配置错是 CMake 脚本的问题，编译错是源码的问题，链接错多半还是 CMake 的问题。** 这条判断规则贯穿整份笔记。

### 3. 三步工作流
Linux 上 CMake 的标准流程是三步：写 `CMakeLists.txt`、运行 `cmake PATH`（或交互式的 `ccmake PATH`）生成 Makefile、运行 `make` 编译。

Windows + MinGW 下对应的流程：

```powershell
# 1. 写 CMakeLists.txt（编辑器里完成）

# 2. 配置：生成 Makefile
cd D:\Downloads\lc\UnitConverter
New-Item -ItemType Directory -Name build -Force
cd build
cmake .. -G "MinGW Makefiles"

# 3. 构建
mingw32-make
```

`cmake ..` 里的 `..` 是相对于当前目录（`build/`）的路径，指向含根 `CMakeLists.txt` 的目录。当前目录 `build/` 是构建目录，所有生成的文件落在这里。

也可以用 `-S` / `-B` 参数，不依赖当前工作目录：

```powershell
cmake -S . -B build -G "MinGW Makefiles"
cmake --build build
```

`cmake --build build` 是跨平台的构建命令，它会自动调用对应的构建工具，不需要知道底下是 make 还是 ninja 还是 MSBuild。

### 4. 源码外构建（out-of-source build）
上面的 `build/` 目录就是"源码外构建"：编译产物和源码分开。

```powershell
cmake -S . -B build -G "MinGW Makefiles"
```

好处有三条：

+ 想彻底清理，`Remove-Item -Recurse -Force build` 一下就行，源码树始终干净
+ `.gitignore` 里加一行 `build/` 就不会误提交编译产物
+ 可以同时存在多个构建目录（`build-debug/`、`build-release/`），用不同选项各构建一次，互不干扰

hahack 那篇教程是 2013 年写的，它的演示都是在源码目录里直接 `cmake .`（源码内构建），生成的 `CMakeCache.txt`、`Makefile`、`.o` 文件会和 `.cpp` 混在一起。**现在的做法一律用源码外构建**，那份教程里的 `cmake .` 换成 `cmake -S . -B build` 就是现代的写法。

## 二、第一层：手工调用编译器
这一节从最原始的地方开始——不用 make，不用 CMake，直接敲 `g++`。理解这一层，后面每一层为什么要存在就都清楚了。

### 1. 单个文件
一个 `main.cpp`：

```cpp
#include <iostream>

void say_hello(const std::string& name) {
    std::cout << "Hello, " << name << "!\n";
}

int main(int argc, char** argv) {
    say_hello(argc > 1 ? argv[1] : "world");
    return 0;
}
```

直接编译：

```powershell
g++ -o hello main.cpp
.\hello.exe
```

`-o` 指定输出文件名。只有一个源文件时这样最省事，但程序会变长。

### 2. 多个文件：声明与定义之分
把 `say_hello()` 拆到单独的 `say_hello.cpp` 里，然后：

```powershell
g++ -o hello main.cpp say_hello.cpp
```

报错了：

```plain
main.cpp: In function 'int main(int, char**)':
main.cpp:4:5: error: 'say_hello' was not declared in this scope
```

**注意错误的措辞：not declared（未声明），不是 not defined（未定义）。** 这是 C++ 里一个核心区分：

+ **声明**告诉你"有这么一个东西，它长什么样"——函数名、参数类型、返回类型
+ **定义**告诉你"它是怎么实现的"

`main.cpp` 里调用 `say_hello()` 时，编译器完全不知道有这么一个函数存在，因为它在 `main.cpp` 里从没出现过。解决办法是在调用之前加上声明：

```cpp
#include <iostream>

void say_hello(const std::string& name);   // 声明

int main(int argc, char** argv) {
    say_hello(argc > 1 ? argv[1] : "world");
    return 0;
}
```

一个函数可以有多个声明（每个用它的 `.cpp` 里都可以声明一次），但**只能有一个定义**。定义重复了会报 `multiple definition of 'say_hello(...)'`。

### 3. 头文件：为什么它不在编译命令行上
把声明放进 `say_hello.h`：

```cpp
#pragma once

#include <string>

void say_hello(const std::string& name);
```

`main.cpp` 里 `#include "say_hello.h"`，`say_hello.cpp` 里也 include 一次。

**关键观察：头文件从不出现在编译命令行上。**

```powershell
g++ -o hello main.cpp say_hello.cpp    # 只列 .cpp，没有 .h
```

因为**头文件不能被单独编译**。它没有产生任何机器码，只是一段文本。`#include` 做的事是**文本替换**，由预处理器在编译器看到代码之前完成：

```plain
预处理：#include "say_hello.h" → 把文件内容原样粘进来
编译：  读展开后的完整文本 → 生成机器码
```

`#pragma once` 保证同一个头文件在一次编译里只被粘进去一次（即使它被间接 include 了多次）。老式的写法是 include guard（`#ifndef SAY_HELLO_H` / `#define` / `#endif`），作用相同。

系统头文件（`<string>`、`<iostream>`）放在系统 include 目录里，路径随发行版和编译器版本变化。在 MinGW 上大概是：

```plain
C:\mingw64\include\c++\14.2.0\string
C:\mingw64\include\c++\14.2.0\iostream
```

编译器的默认搜索路径里已经包含了这些目录，所以写 `#include <string>` 就能找到。

**在定义 **`say_hello()`** 的那个文件里也 include 一次头文件**，这是一个好习惯：编译器会拿头文件里的声明和 `.cpp` 里的定义做比对，如果签名写歪了（漏了个 `const`、参数类型不同），会立刻报错。不 include 的话，写错的函数会变成一个没人调用的新函数，静默地不生效。

### 4. 分步编译：-c 和目标文件
每次改动都要把两个 `.cpp` 都重新编译一遍，在文件多起来之后很浪费时间。C++ 的解析本来就慢（头文件展开后动辄几万行），所以值得把编译拆成两步。

`-c` 参数让编译**只生成目标文件**，不链接：

```powershell
g++ -c -o say_hello.o say_hello.cpp
g++ -c -o main.o main.cpp
```

产物是 `say_hello.o` 和 `main.o`。目标文件里装的是机器码，加上符号表——记录"这个文件里定义了哪些符号、引用了哪些符号"。

然后链接：

```powershell
g++ -o hello main.o say_hello.o
```

现在的收益很直接：只改了 `main.cpp`（比如把 "world" 换成别的），只需要

```powershell
g++ -c -o main.o main.cpp
g++ -o hello main.o say_hello.o
```

`say_hello.o` 完全不用重编。文件越多，省下的时间越多。

### 5. 符号与名字修饰
用 `objdump` 看一下目标文件里的符号：

```powershell
objdump -d say_hello.o
```

输出里能看到类似这样的符号名：

```plain
0000000000000000 <_Z9say_helloRKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE>:
```

这就是 `void say_hello(const std::string&)` 被**名字修饰**（name mangling）之后的样子。原始函数名被编码成一串字符：

| 片段 | 含义 |
| --- | --- |
| `_Z` | 修饰名的起始标记（Itanium C++ ABI） |
| `9say_hello` | 名字长度 9，名字本身 |
| `RK` | 引用（R）+ const（K） |
| `NSt...` | 嵌套名（N），`std::basic_string<...>` |
| `c` `i` | char、int（模板参数 char_traits / allocator 的实际类型） |


更简单的例子：`int add(int, int)` 修饰成 `_Z3addii`——`_Z` + `3` + `add` + `i` + `i`。

**为什么需要修饰？** 因为 C++ 支持重载。`add(int, int)` 和 `add(double, double)` 名字都叫 `add`，必须编码成不同的符号链接器才能区分。

想看到原始名字可以加 `-C`：

```powershell
objdump -d -C say_hello.o
```

C 语言没有重载，所以符号名基本就是函数名本身（可能加一个下划线前缀），这也是 C 库能被各种语言直接调用的原因。

### 6. 链接到底是什么
看链接前后的差别。`main.o` 里对 `say_hello` 的调用，在链接前是一个悬空的引用——地址是 0，等着被填上：

```plain
# main.o（未链接）
0000000000000055:  e8 00 00 00 00    call  000000000000005a    ← 目标地址是占位符
```

链接之后：

```plain
# hello.exe（已链接）
0000000000401234:  e8 26 03 00 00    call  000000000040155f    ← 指向真实地址
```

`say_hello` 的实现现在待在 `0x40155f`，调用点的地址被改写成了它。

所以**链接 = 把多个目标文件的符号合并到一起，然后把所有引用都改成正确的地址**。链接器做三件事：

1. 收集所有 `.o`（和 `.a`、`.so`）里定义的符号
2. 检查有没有被引用但没找到定义的符号 → 报 `undefined reference`
3. 把每个引用处都填上真实地址

这一步很值得拆开，因为"分步编译"正是本次 UnitConverter 实践任务里 `add_library` 要做的事——把 `unitlib.cpp` 编成一个 `.a`，单独维护，不再和 `main.cpp` 一起编。

### 7. 小结：这一层的三个痛点
手工敲 g++ 到这一步，暴露出来的问题是：

| 痛点 | 表现 |
| --- | --- |
| 命令要手敲 | 改一个文件也要记得重新编译哪些目标文件、按什么顺序链接 |
| 依赖靠脑子记 | `main.cpp` 只改了 `main.o`；但 `say_hello.h` 改了，两个 `.o` 都得重编——很容易漏 |
| 完全没有跨平台能力 | `g++`、`.o`、`objdump` 是 GNU 工具链的，Windows MSVC 下这些命令全都不存在 |


第二行的痛点尤其隐蔽：**头文件的依赖关系不在命令行上，只在 **`#include`** 里**。手写命令时没人帮你追踪它。

下一节的 make 解决了前两条，第三条留给 CMake。

## 三、第二层：make 与 Makefile
### 1. make 的模型
make 的核心模型只有三行：

```makefile
output: dependencies
    command
```

读作：**当任何一个 dependency 比 output 新的时候，运行 command 重新生成 output。**

三个要素：

+ **目标（target）**：要生成的东西，通常是文件名
+ **依赖（prerequisites）**：生成它需要哪些东西
+ **命令（recipe）**：怎么生成

判断"新不新"靠文件的**修改时间戳**（mtime）。make 建一张依赖图，从最终目标倒着递归：任何一环的依赖比目标新，就重新跑那一环的命令。

注意：命令那一行**必须是 Tab 开头**，不能用空格。这是 make 最容易踩的格式坑，报错是 `Makefile:3: *** missing separator. Stop.`（大多数编辑器里会显示成 `^I` 或者提示 tab/space 混用）。

### 2. 手写一个 Makefile
针对前面的项目（`main.cpp`、`say_hello.cpp`、`say_hello.h`）：

```makefile
hello: main.o say_hello.o
    g++ -o hello main.o say_hello.o

main.o: main.cpp
    g++ -c -o main.o main.cpp

say_hello.o: say_hello.cpp
    g++ -c -o say_hello.o say_hello.cpp
```

运行：

```powershell
make hello
```

三条规则会按依赖顺序执行：先编译两个 `.o`，再链接成 `hello`。

### 3. 看它怎么增量
把 `main.cpp` 里的一个字符串改一下，再跑 `make hello`：

```plain
g++ -c -o main.o main.cpp
g++ -o hello main.o say_hello.o
```

只有 `main.o` 重新编译了，`say_hello.o` 没动。因为 make 检查发现：

+ `hello` 依赖 `main.o` 和 `say_hello.o`
+ `main.o` 依赖 `main.cpp` —— `main.cpp` 的时间戳比 `main.o` 新，所以要重编
+ `say_hello.o` 依赖 `say_hello.cpp` —— `say_hello.cpp` 没变，跳过
+ `hello` 的某个依赖（`main.o`）更新了，所以重新链接

这个判断是自动的，比手敲命令时靠脑子记可靠得多。这就是 make 解决的第一个痛点。

### 4. 变量的冗余问题
上面这个 Makefile 有个刺眼的地方：文件名重复了三遍。加一个源文件要改四处。用变量可以改善一点：

```makefile
CXX = g++
CXXFLAGS = -Wall -std=c++17
TARGET = hello
OBJS = main.o say_hello.o

$(TARGET): $(OBJS)
    $(CXX) -o $(TARGET) $(OBJS)

main.o: main.cpp
    $(CXX) $(CXXFLAGS) -c -o main.o main.cpp

say_hello.o: say_hello.cpp
    $(CXX) $(CXXFLAGS) -c -o say_hello.o say_hello.cpp
```

`$(VAR)` 展开变量，`=` 是延迟展开（用的时候才求值），`:=` 是立即展开（定义时就求值）。

`CXX` 和 `CXXFLAGS` 是 make 的内置变量名，有默认值，写在这里是覆盖它们。遵循这个约定可以让别人用 `make CXX=clang++` 临时换编译器，不用改 Makefile。

### 5. 自动变量与模式规则
每个 `.o` 的规则长得几乎一样，只有文件名不同。make 提供了自动变量和模式规则来消除这种重复：

```makefile
CXX = g++
CXXFLAGS = -Wall -std=c++17
TARGET = hello
OBJS = main.o say_hello.o

$(TARGET): $(OBJS)
    $(CXX) -o $@ $^

%.o: %.cpp
    $(CXX) $(CXXFLAGS) -c -o $@ $<
```

**常用自动变量：**

| 变量 | 含义 |
| --- | --- |
| `$@` | 当前规则的目标名（这里是 `main.o`） |
| `$<` | 第一个依赖（这里是 `main.cpp`） |
| `$^` | 所有依赖，去重，空格分隔（这里是 `main.o say_hello.o`） |
| `$?` | 比目标新的那些依赖 |
| `$*` | 模式规则里 `%` 匹配到的部分（`main.o` 里的 `main`） |


**模式规则 **`%.o: %.cpp` 的意思是"任何 `.o` 都从同名的 `.cpp` 生成"，`%` 是通配的部分。这样就不用为每个源文件写一条规则了。

### 6. 伪目标与 .PHONY
```makefile
.PHONY: clean all

all: $(TARGET)

clean:
    rm -f $(TARGET) $(OBJS)
```

`clean` 不是一个文件，而是"要执行的动作"。如果目录里恰好有个叫 `clean` 的文件，make 会以为目标已经是最新的，跳过执行。`.PHONY: clean` 告诉 make "`clean` 永远是过期的，无条件执行"。

`all` 是传统上的默认目标——直接运行 `make` 不带参数时执行第一个规则，把 `all` 放最前面就能让 `make` 等价于 `make all`。

Windows 下 `rm -f` 要换成 `del /f /q`，或者用 MinGW 提供的 `rm`：

```makefile
clean:
    -rm -f $(TARGET) $(OBJS)
```

前面的 `-` 表示忽略这条命令的失败（文件本来就不存在时 `rm` 会返回非零）。

### 7. make 解决不了的三件事
到这一步，Makefile 已经能很好地完成增量编译了。但把它当成"通用做法"用下去，会遇到三个绕不过去的问题。

**问题一：头文件依赖完全没有追踪。**

上面那份 Makefile 有一个隐藏的严重缺陷：

```makefile
main.o: main.cpp          # 没有列 main.cpp 里 include 的头文件
```

如果改了 `say_hello.h`，make 完全没有反应——因为 `main.o` 的依赖列表里只有 `main.cpp`，而 `main.cpp` 本身没变。**结果是用旧的 **`main.o`** 链接出一个行为不对的 **`hello`**，而且不报错。**

手工补上可以：

```makefile
main.o: main.cpp say_hello.h
say_hello.o: say_hello.cpp say_hello.h
```

但头文件的依赖是**递归**的——`say_hello.h` 里 include 了 `<string>`，`<string>` 里又 include 了别的。真要把依赖写全，得把整个包含树展开。没人手工做这件事。

GNU make 提供了 `-MMD` 类的机制配合编译器生成依赖文件（下面讲 CMake 生成的 Makefile 时会看到），但那是绕开 make 本身、借编译器之力，配置起来也不轻松。

**问题二：没有跨平台能力。**

这份 Makefile 在 Linux 上能用，在 Windows MSVC 上完全不能用：

+ 编译器名字不同（`g++` vs `cl`）
+ 编译参数不同（`-c -o file.o` vs `/c /Fo:file.obj`）
+ 目标文件扩展名不同（`.o` vs `.obj`）
+ 可执行文件扩展名不同（无扩展名 vs `.exe`）
+ 清理命令不同（`rm -f` vs `del /f /q`）

要支持两个平台，得维护两份内容不同但意图相同的 Makefile。改动一处要同步两处，时间一长必然漂移。

**问题三：不支持查找库和依赖传播。**

用到一个第三方库时要手工找到它的头文件目录和库文件路径，写成 `-I` 和 `-L`：

```makefile
CXXFLAGS = -I/usr/include/opencv4 -I/usr/local/include/fmt
LDFLAGS = -L/usr/lib/x86_64-linux-gnu -lopencv_core -lfmt
```

路径是硬编码的，换一台机器、换一个发行版就可能失效。而且 OpenCV 依赖的一些库，也要手动列全（否则链接期报 `undefined reference`）——这个依赖关系其实存在 OpenCV 自己的元数据里，但 make 读不到。

**这三点正是 CMake 存在的理由**，也是 Zero to CMake 那篇文章的转折点：手写 Makefile 在单平台小项目上完全够用，但它止步于此。

### 8. make 的其他变体和传统用法
`make` 本身只是一个程序，读 Makefile 执行。历史上各家都实现过自己的版本（前面表格里的 nmake、pmake、qmake 都是），它们在语法扩展上互不兼容。GNU Make 是 Unix 世界里最主流的实现。

除编译之外，make 常被当作通用的任务运行器，这种用法至今还很常见：

```makefile
.PHONY: test lint fmt

test:
    python -m pytest tests/ -v

lint:
    ruff check .

fmt:
    ruff format .
```

`make test`、`make lint` 比记住一长串命令方便，写进 CI 脚本也清楚。CMake 项目里经常保留一个包装用的 Makefile 或 shell 脚本，把 `cmake -S . -B build && cmake --build build` 这类长命令收成 `make build`。

**在 CMake 项目里，前面那些手写编译规则都不需要了**——CMake 会生成 Makefile。但理解它们仍然有用，因为 CMake 生成出来的正是这些结构的复杂版本，看懂原始形态才看得懂生成物。

## 四、第三层：CMake 生成的 Makefile 长什么样
这一节回答一个具体问题：**用 CMake 之后，那些 Makefile 是谁写的、写成什么样子、为什么不让你改。**

### 1. 手写的 CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.16)
project(hello VERSION 1.0 LANGUAGES CXX)

add_executable(hello main.cpp say_hello.cpp)
```

配置：

```powershell
cmake -S . -B build -G "MinGW Makefiles"
```

输出（Linux 上类似）：

```plain
-- The CXX compiler identification is GNU 14.2.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Configuring done
-- Generating done
-- Build files have been written to: D:/Downloads/lc/hello/build
```

这几行输出就是"配置阶段"的全部工作：检测编译器、确认 ABI、生成构建文件。

### 2. 生成出来的 Makefile 在哪里
`build/` 目录下会有：

```plain
build/
├── CMakeCache.txt              缓存变量
├── CMakeFiles/
│   ├── hello.dir/
│   │   ├── main.cpp.obj
│   │   ├── main.cpp.obj.d      ← 头文件依赖记录
│   │   ├── say_hello.cpp.obj
│   │   └── flags.make          ← 编译参数
│   ├── build.make              ← 各文件的构建规则
│   └── ...
├── Makefile                    顶层，转发到 CMakeFiles/build.make
└── compile_commands.json       （如果开了 CMAKE_EXPORT_COMPILE_COMMANDS）
```

顶层那个 `Makefile` 只是一个薄壳，内容是"把所有目标转发给 `CMakeFiles/Makefile2`"。真正的规则在 `CMakeFiles/build.make` 里。

### 3. 一条生成的规则
打开 `build/CMakeFiles/build.make`，找到 `main.cpp.obj` 的那条规则，大致是这样：

```makefile
CMakeFiles/hello.dir/main.cpp.obj: CMakeFiles/hello.dir/flags.make
CMakeFiles/hello.dir/main.cpp.obj: D:/Downloads/lc/hello/main.cpp
CMakeFiles/hello.dir/main.cpp.obj: CMakeFiles/hello.dir/compiler_depend.ts
    @$(CMAKE_COMMAND) -E cmake_echo_color --switch=$(COLOR) --green --progress-dir=... --progress-num=$(CMAKE_PROGRESS_1) "Building CXX object CMakeFiles/hello.dir/main.cpp.obj"
    cd D:/Downloads/lc/hello/build/CMakeFiles/hello.dir && C:/mingw64/bin/g++.exe \
      $(CXX_DEFINES) $(CXX_INCLUDES) $(CXX_FLAGS) \
      -MD -MT CMakeFiles/hello.dir/main.cpp.obj \
      -MF CMakeFiles/hello.dir/main.cpp.obj.d \
      -o CMakeFiles/hello.dir/main.cpp.obj \
      -c D:/Downloads/lc/hello/main.cpp
```

对比前面手写的那条 `main.o: main.cpp`，多了这些东西：

| 多出来的部分 | 作用 |
| --- | --- |
| 依赖里包含 `flags.make` | 编译参数一变（比如换了构建类型），这个文件也重编 |
| `compiler_depend.ts` | 编译器依赖信息的时间戳文件 |
| `$(CXX_DEFINES) $(CXX_INCLUDES) $(CXX_FLAGS)` | 从 `flags.make` 读出来的参数，由 CMake 计算 |
| `-MD -MT ... -MF ...obj.d` | **让编译器额外产出一份头文件依赖清单** |
| `cmake_echo_color ... --progress-num=` | 那个 `[ 33%] Building CXX object ...` 的输出 |
| `@` 前缀 | 不回显命令本身，只显示那行彩色进度 |


### 4. 关键一着：-MD 解决头文件依赖
这是前面手写 Makefile 完全做不到、而 CMake 生成物天然具备的能力。

`-MD` 让 g++ 在编译 `main.cpp` 时**顺便扫一遍所有 **`#include`，把完整的头文件依赖树写进 `main.cpp.obj.d`：

```makefile
CMakeFiles/hello.dir/main.cpp.obj: D:/Downloads/lc/hello/main.cpp \
  D:/Downloads/lc/hello/say_hello.h \
  C:/mingw64/include/c++/14.2.0/string \
  C:/mingw64/include/c++/14.2.0/bits/basic_string.h \
  ...（几十行）...
```

这个 `.d` 文件会被 Makefile `include` 进来，于是**头文件的依赖关系变成了 make 能看到的真实依赖**。

效果：改了 `say_hello.h` 之后，`main.cpp.obj` 会被正确判定为过期并重新编译——手工写依赖列表时那个"改了头文件不重编"的陷阱，在这里消失了。

**这不是 CMake 发明的技巧**，是 GNU make 社区很早就在用的方案（`.d` 文件 + `include`），CMake 把它做成了默认行为，每个文件都自动带上。

### 5. flags.make 里有什么
```makefile
# CMAKE generated file: DO NOT EDIT!
CXX_DEFINES = -DUNITCONVERTER_VERSION=\"1.0.0\"
CXX_INCLUDES = -ID:/Downloads/lc/UnitConverter/unitlib/include
CXX_FLAGS = -std=gnu++17
```

**这三行是理解 CMake 传播机制的关键。** 注意：

+ `CXX_DEFINES` 里的宏来自 `target_compile_definitions`
+ `CXX_INCLUDES` 里的路径来自 `target_include_directories`，**包括通过 **`PUBLIC`** 从依赖库传播过来的那些**
+ 都是 CMake 在配置阶段算好的

文件头的注释说明了态度：`DO NOT EDIT!`。这个文件每次配置时会被重新生成，手改的内容会被覆盖。改构建参数的正确做法是改 `CMakeLists.txt`，然后重新配置。

**这也解释了 CMake 项目的一个工作习惯**：`build/` 目录里的任何文件都是生成物，不值得读、不应该改、可以随时删。要理解构建参数是怎么来的，看 `CMakeLists.txt` 和 `compile_commands.json`（后者是 CMake 主动导出给人看的）。

## 五、CMake 的语言基础
这一节转到 CMake 脚本语言本身。CMake 用的是自己的一套语言，不是 shell，也不是 Python。

### 1. 语法规则
```cmake
# 井号开始的是注释
cmake_minimum_required(VERSION 3.16)     # 命令名（不区分大小写）+ 括号 + 空格分隔的参数
project(hello VERSION 1.0 LANGUAGES CXX)
```

三条规则：

+ **命令名不区分大小写**：`project(...)`、`PROJECT(...)`、`Project(...)` 等价。但社区约定全小写。
+ `#`** 之后到行尾是注释**，没有多行注释语法（`#[[ ... ]]` 块注释是 3.0 之后加的，很少用）。
+ **命令的形态是「名字 + 括号 + 参数」**，参数之间用空格或换行分隔。参数可以用引号包起来（含空格时必须），也可以不用（不含空格时）。

换行不是语句分隔符——参数可以跨多行写，直到括号闭合：

```cmake
project(hello
        VERSION 1.0
        LANGUAGES CXX
)
```

分号也可以当参数分隔符，`set(SOURCES a.cpp b.cpp)` 和 `set(SOURCES a.cpp;b.cpp)` 等价。

### 2. 变量
```cmake
set(CMAKE_CXX_STANDARD 17)              # 内置变量：影响所有目标的默认 C++ 标准
set(SOURCES src/main.cpp src/util.cpp)  # 自定义变量：一个列表
```

取值用 `${VAR}`：

```cmake
message(STATUS "源文件：${SOURCES}")
```

**未设置的变量展开成空字符串，不报错。** 这是很多"路径拼出来是空的"问题的根源。排查时先确认变量有值：

```cmake
message(STATUS "SOURCES=[${SOURCES}]")   # 用方括号包住，空值能看出来
```

`set` 可以设置三种变量：

```cmake
set(NORMAL_VAR "value")                              # 普通变量
set(CACHE_VAR "value" CACHE STRING "说明文字")        # 缓存变量，写入 CMakeCache.txt
set(ENV{PATH_VAR} "/some/path")                      # 环境变量
```

### 3. 变量的三个作用域
**普通变量**：作用范围是当前 `CMakeLists.txt`，以及它 `add_subdirectory` 进来的子目录。**子目录能读到父目录的值，但子目录里的修改不会影响父目录。**

```cmake
# 根 CMakeLists.txt
set(MY_VAR "root")
add_subdirectory(sub)
message(STATUS "根里看到：${MY_VAR}")     # → root
```

```cmake
# sub/CMakeLists.txt
message(STATUS "子里看到：${MY_VAR}")     # → root（能读到）
set(MY_VAR "sub")                        # 只在这个子目录里生效
```

想让子目录的修改影响父目录，显式加 `PARENT_SCOPE`：

```cmake
# sub/CMakeLists.txt
set(RESULT "computed" PARENT_SCOPE)      # 写进父目录的作用域
```

**缓存变量**：存进 `build/CMakeCache.txt`，跨配置持久化，可以用命令行 `-D` 覆盖。

```cmake
set(MY_OPTION "default" CACHE STRING "我的选项")
```

```powershell
cmake -S . -B build -DMY_OPTION=changed
```

第二次运行 `cmake -S . -B build` 时，`MY_OPTION` 保持 `changed`——缓存变量已经存在，`set` 的默认值不会再生效。想强制覆盖要加 `FORCE`：

```cmake
set(MY_OPTION "default" CACHE STRING "我的选项" FORCE)
```

这个"缓存值优先"的行为是 CMake 配置出问题时的一个常见原因：改了 `CMakeLists.txt` 里的默认值，但重新配置后没生效，因为缓存里还是旧值。**解决办法是删掉 **`build/`** 重新配置。**

### 4. 条件判断
```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    message(STATUS "Windows 平台")
elseif(UNIX)
    message(STATUS "Unix 平台")
else()
    message(STATUS "其他平台")
endif()
```

`endif()` 现在可以不带参数，老式写法是 `endif(CMAKE_SYSTEM_NAME STREQUAL "Windows")`，两行必须重复条件。新版本的 CMake 不再要求重复。

**比较运算符：**

| 运算符 | 用途 |
| --- | --- |
| `STREQUAL` | 字符串相等 |
| `STRLESS` / `STRGREATER` | 字符串字典序 |
| `EQUAL` / `LESS` / `GREATER` | 数值比较 |
| `MATCHES` | 正则匹配 |
| `VERSION_LESS` / `VERSION_GREATER` | 版本号比较（按段比较，`3.10` > `3.9`） |


**逻辑运算：**`AND`、`OR`、`NOT`，以及括号分组：

```cmake
if(FALSE AND (FALSE OR TRUE))
    message(STATUS "不会显示")
endif()
```

`FALSE AND (FALSE OR TRUE)` 求值为假，所以这条 `message` 不执行。

**单独一个变量名时，CMake 判断它的真假**：

| 假 | 真 |
| --- | --- |
| `OFF`、`NO`、`FALSE`、`N`、`IGNORE`、`NOTFOUND` | 其他值 |
| `0`、空字符串、未定义变量 |  |


所以 `if(USE_MYMATH)` 这种写法是在判断"这个选项是不是开着"。

### 5. 循环
`foreach` 有三种形式：

```cmake
# 形式一：遍历字面量列表
foreach(item alpha beta gamma)
    message(STATUS "item=${item}")
endforeach()

# 形式二：数值区间
foreach(i RANGE 1 5)          # 1 2 3 4 5
    message(STATUS "i=${i}")
endforeach()

foreach(i RANGE 0 10 2)       # 0 2 4 6 8 10（start stop step）
    message(STATUS "i=${i}")
endforeach()

# 形式三：遍历变量里的列表
set(SOURCES src/a.cpp src/b.cpp)
foreach(src IN LISTS SOURCES)
    message(STATUS "源文件：${src}")
endforeach()
```

形式三的 `IN LISTS` 和 `IN ITEMS` 可以混用：

```cmake
foreach(x IN LISTS HEADERS ITEMS extra.cpp stub.cpp)
endforeach()
```

`IN LISTS` 后面跟的是**变量名**，`IN ITEMS` 后面跟的是**字面值**。直接写 `foreach(src ${SOURCES})` 也能工作，但在元素含空格或分号时会出问题，`IN LISTS` 更安全。

`while` 循环：

```cmake
set(i 0)
while(i LESS 5)
    message(STATUS "i=${i}")
    math(EXPR i "${i} + 1")    # 数值运算要用 math(EXPR ...)
endwhile()
```

写循环时注意：CMake 里没有 `i++`，数值自增要用 `math(EXPR i "${i} + 1")`。

### 6. 函数与宏
```cmake
# 函数：自己的作用域，能 return
function(my_func arg1)
    set(LOCAL "只在函数内可见")
    message(STATUS "arg1=${arg1}")
    # 要影响外面得用 PARENT_SCOPE
    set(OUTPUT "结果" PARENT_SCOPE)
endfunction()

my_func(hello)
message(STATUS "OUTPUT=${OUTPUT}")   # → 结果
```

```cmake
# 宏：文本替换，没有自己的作用域
macro(my_macro arg1)
    set(INNER "在宏里设置")          # 会污染调用处的作用域
    message(STATUS "arg1=${arg1}")
endmacro()
```

**函数与宏的区别**：

|  | function | macro |
| --- | --- | --- |
| 作用域 | 独立，需要 `PARENT_SCOPE` 才能输出 | 无独立作用域，`set` 直接影响调用处 |
| 参数 | `ARGN`、`ARGV0` 等变量可用 | 文本替换，注意重名 |
| 调试 | 出错时栈信息清楚 | 出错时位置信息指向调用处 |
| 适用 | 一般情况 | 需要"就地展开"的场合 |


**除了极少数情况，优先用 **`function`**。** 宏的文本替换语义容易出意外。

参数还可以是关键字形式，用 `cmake_parse_arguments` 解析：

```cmake
function(my_add_target name)
    cmake_parse_arguments(ARG "REQUIRED;VERBOSE" "OUTPUT_DIR" "SOURCES" ${ARGN})
    if(ARG_REQUIRED)
        message(STATUS "${name} 是必需的")
    endif()
    message(STATUS "源文件：${ARG_SOURCES}")
    message(STATUS "输出目录：${ARG_OUTPUT_DIR}")
endfunction()

my_add_target(mylib REQUIRED OUTPUT_DIR "out" SOURCES a.cpp b.cpp)
```

`cmake_parse_arguments` 的第一个参数是前缀，后面三组分别是：选项（布尔标志）、单值参数、多值参数。解析结果放进 `ARG_<名字>` 变量。

### 7. message 的五个等级
```cmake
message(STATUS "普通进度信息")           # 加 -- 前缀
message(WARNING "警告，继续执行")         # 加 WARNING 前缀
message(AUTHOR_WARNING "只在开发者模式显示")  # 加 AUTHOR_WARNING 前缀
message(SEND_ERROR "错误，继续处理但跳过生成")  # 报错，配置完但不生成构建文件
message(FATAL_ERROR "致命错误，立刻停止")      # 报错并立即中断
```

在配置阶段的输出里它们长这样：

```plain
-- 普通进度信息
CMake Warning at CMakeLists.txt:5 (message):
  警告，继续执行
CMake Error at CMakeLists.txt:7 (message):
  错误，继续处理但跳过生成
```

`STATUS` 用于正常的进度和调试输出。`SEND_ERROR` 会继续处理剩下的 `CMakeLists.txt`，好处是能一次暴露出多个错误；`FATAL_ERROR` 立刻停。

`FATAL_ERROR`** 的典型用法**是发现前置条件不满足时立刻停：

```cmake
if(NOT CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    message(FATAL_ERROR "本项目只支持 GCC 和 Clang，当前是 ${CMAKE_CXX_COMPILER_ID}")
endif()
```

第二个参数 `FATAL_ERROR` 在 `cmake_minimum_required` 里是老式写法（新版本不写也是这个行为），但在 `message` 里它仍然是有意义的等级区分。

### 8. 常用内置变量
| 变量 | 含义 |
| --- | --- |
| `CMAKE_CXX_COMPILER` | C++ 编译器的完整路径 |
| `CMAKE_CXX_COMPILER_ID` | 编译器标识：`GNU`、`Clang`、`MSVC`、`AppleClang` |
| `CMAKE_CXX_COMPILER_VERSION` | 编译器版本号 |
| `CMAKE_CXX_STANDARD` | 默认 C++ 标准 |
| `CMAKE_BUILD_TYPE` | 构建类型（`Debug`/`Release`/…） |
| `CMAKE_SOURCE_DIR` | **最外层**的源码目录 |
| `CMAKE_CURRENT_SOURCE_DIR` | **当前** `CMakeLists.txt` 所在目录 |
| `CMAKE_BINARY_DIR` | 最外层的构建目录 |
| `CMAKE_CURRENT_BINARY_DIR` | 当前对应的构建目录 |
| `PROJECT_SOURCE_DIR` | 最近一次 `project()` 所在目录 |
| `PROJECT_BINARY_DIR` | 上面那个的构建目录 |
| `PROJECT_NAME` / `PROJECT_VERSION` | 工程名 / 版本号 |
| `CMAKE_SYSTEM_NAME` | 操作系统名（`Windows`、`Linux`、`Darwin`） |
| `WIN32` / `UNIX` / `APPLE` | 平台布尔值 |
| `CMAKE_EXPORT_COMPILE_COMMANDS` | 是否导出 `compile_commands.json` |
| `CMAKE_INSTALL_PREFIX` | `install` 的目标前缀 |


`CMAKE_CURRENT_SOURCE_DIR`** 和 **`CMAKE_SOURCE_DIR`** 的区别最容易出错**：前者随 `add_subdirectory` 变化，后者始终是最外层。在 `unitlib/CMakeLists.txt` 里：

```cmake
# 正确：unitlib/include
target_include_directories(unitlib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

# 错误：指向根目录的 include/，那个目录不存在
target_include_directories(unitlib PUBLIC ${CMAKE_SOURCE_DIR}/include)
```

写自己的头文件目录时一律用 `CMAKE_CURRENT_SOURCE_DIR`。

### 9. 检查编译器与平台
```cmake
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    add_definitions(-std=c++17)
    message(STATUS "使用 GCC 编译器")
endif()

if(UNIX)
    set(PLATFORM_DEFS -Wall -Wextra -Werror -Wno-unused-parameter)
elseif(WIN32)
    set(PLATFORM_DEFS /W4)
endif()
```

不过**平台相关的编译参数更推荐用生成器表达式或目标属性来处理**，而不是 `if` + 全局 `add_definitions`，理由和前面"旧式 vs 现代"里说的一样：全局命令会污染其他目标。

`CMAKE_CXX_COMPILER_ID` 比字符串匹配编译器路径可靠——路径可能带各种前后缀，标识符是 CMake 归一化过的标准值。

### 10. 组织：include、CMAKE_MODULE_PATH
项目变大之后，公共的 CMake 代码可以抽成 `.cmake` 文件放在 `cmake/` 目录里：

```cmake
# 添加自己的模块搜索路径
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

# 引入一个自定义模块
include(MyHelpers)

# 引入 CMake 自带的模块
include(CheckFunctionExists)
include(CTest)
```

`include()` 的搜索顺序是：先在 `CMAKE_MODULE_PATH` 里找，再在 CMake 安装目录的 `Modules/` 里找。所以 `include(CTest)` 找到的是 CMake 自带的 `CTest.cmake`，`include(MyHelpers)` 找到的是项目自己的 `cmake/MyHelpers.cmake`。

`CMAKE_MODULE_PATH` 的赋值要写成 `${CMAKE_MODULE_PATH} 新路径` 的形式追加，不是直接覆盖——否则会丢掉之前设置的值。

## 六、案例递进：从单文件到完整项目
hahack 那篇教程的组织方式很值得学：它用一个"求幂"的小程序，从最简单的版本开始，每加一个需求就引入一批新命令。这种递进方式让每条命令都有一个具体的、正在解决的问题作背景。

下面按同样的顺序走一遍，每条命令都标注在 UnitConverter 项目里的对应位置。

### 1. Demo1：单个源文件
一个 `main.cc`，从命令行读底数和指数，算幂：

```cmake
cmake_minimum_required(VERSION 2.8)

project(Demo1)

add_executable(Demo main.cc)
```

三条命令，构成最小学的 CMake 工程：

| 命令 | 作用 |
| --- | --- |
| `cmake_minimum_required` | 声明所需的最低 CMake 版本 |
| `project` | 定义工程名 |
| `add_executable` | 把 `main.cc` 编译成名为 `Demo` 的可执行文件 |


构建（教程里是源码内构建）：

```powershell
cmake .
make
.\Demo.exe 5 4      # → 625
```

**注意 **`cmake_minimum_required(VERSION 2.8)`：这是 2013 年的教程，当时 2.8 是最低要求。现在的项目应该写 `3.16` 或更高——版本号决定了 CMake 采用哪一套 policy，写低了会继承很多已废弃的行为。

### 2. Demo2：同目录多个源文件
`power` 函数拆到 `MathFunctions.cc`，声明放 `MathFunctions.h`。目录结构：

```plain
Demo2/
├── CMakeLists.txt
├── main.cc
├── MathFunctions.cc
└── MathFunctions.h
```

最简单的改法是直接补齐源文件列表：

```cmake
add_executable(Demo main.cc MathFunctions.cc)
```

文件多了以后，手写列表容易漏。`aux_source_directory` 可以自动收集某个目录下的所有源文件：

```cmake
aux_source_directory(. DIR_SRCS)
add_executable(Demo ${DIR_SRCS})
```

`aux_source_directory(<目录> <变量>)` 把指定目录下所有能被识别的源文件（`.c`、`.cpp`、`.cc`、`.cxx` 等）收集进变量。

**这个命令在现代 CMake 里不推荐用**，原因和 `file(GLOB)` 一样：新增源文件之后 CMake 不会自动重新配置（因为 `CMakeLists.txt` 没变），必须手删缓存或重新 `cmake .`。**显式列出源文件更可靠**，也是现在的通行做法。

UnitConverter 项目里就是显式列出的：

```cmake
add_library(unitlib STATIC
        src/unitlib.cpp
)
```

### 3. Demo3：多目录，构建静态库
`MathFunctions` 移到 `math/` 子目录，做成一个库：

```plain
Demo3/
├── CMakeLists.txt          ← 根
├── main.cc
└── math/
    ├── CMakeLists.txt      ← 子
    ├── MathFunctions.cc
    └── MathFunctions.h
```

`math/CMakeLists.txt`**：**

```cmake
aux_source_directory(. DIR_LIB_SRCS)
add_library(MathFunctions ${DIR_LIB_SRCS})
```

`add_library` 第一个参数是目标名 `MathFunctions`，省略类型时默认是静态库（`STATIC`），产出一个 `.a` 文件。

**根 **`CMakeLists.txt`**：**

```cmake
add_subdirectory(math)

add_executable(Demo main.cc)
target_link_libraries(Demo MathFunctions)
```

两行关键：

+ `add_subdirectory(math)` —— 走进 `math/` 目录，读取那里的 `CMakeLists.txt`。**这是两个 **`CMakeLists.txt`** 之间唯一的联系。** 没有这行，`MathFunctions` 目标根本不存在，后面的 `target_link_libraries` 会报"找不到目标"。
+ `target_link_libraries(Demo MathFunctions)` —— 让 `Demo` 链接 `MathFunctions`。参数是**目标名**，不是文件路径。

**这正是 UnitConverter 项目的结构**，只是那里还多了 `target_include_directories` 来把头文件目录传给主程序。

**关于头文件路径的一处遗留问题**：Demo3 的根文件里没有处理"`main.cc` 怎么找到 `MathFunctions.h`"这件事。hahack 的教程在 Demo4 里用 `include_directories` 解决了，但那是全局命令。UnitConverter 里用的是更精确的 `target_include_directories(... PUBLIC ...)`，放在这一节的第七部分讲。

### 4. Demo4：可选的编译选项
需求：让 `MathFunctions` 变成可选的。开着就用项目自己的 `power()`，关掉就用标准库的 `pow()`。

`CMakeLists.txt`** 的关键片段：**

```cmake
# 定义一个选项，默认打开
option(USE_MYMATH "Use provided math implementation" ON)

# 把配置生成到代码里
configure_file(
    "${PROJECT_SOURCE_DIR}/config.h.in"
    "${PROJECT_BINARY_DIR}/config.h"
)

# 根据选项决定是否加子目录和链接库
if(USE_MYMATH)
    include_directories("${PROJECT_SOURCE_DIR}/math")
    add_subdirectory(math)
    set(EXTRA_LIBS ${EXTRA_LIBS} MathFunctions)
endif()

aux_source_directory(. DIR_SRCS)
add_executable(Demo ${DIR_SRCS})
target_link_libraries(Demo ${EXTRA_LIBS})
```

`config.h.in`**（模板文件，只有一行）：**

```c
#cmakedefine USE_MYMATH
```

`main.cc`** 里：**

```cpp
#include "config.h"

#ifdef USE_MYMATH
    result = power(base, exponent);      // 用自己的实现
#else
    result = pow(base, exponent);        // 用标准库
#endif
```

**这套机制的原理：**

`option(VAR "描述" 默认值)` 定义一个布尔型缓存变量。它会在 `CMakeCache.txt` 里出现，可以在配置时用 `-DUSE_MYMATH=OFF` 覆盖，也可以用 `ccmake` 交互式修改。

`configure_file(输入 输出)` 把模板文件里的 `${VAR}` 和 `#cmakedefine VAR` 替换成实际值，生成一个新的头文件放到构建目录。生成结果：

| `USE_MYMATH` | `config.h` 的内容 |
| --- | --- |
| `ON` | `#define USE_MYMATH` |
| `OFF` | `/* #undef USE_MYMATH */` |


`#cmakedefine X` 是个条件指令：变量为真时展开成 `#define X`，否则展开成注释掉的 `/* #undef X */`。这样 `#ifdef` 两种状态都能正确处理。

`#cmakedefine01 X` 是另一个变体，无论真假都展开成 `#define X 0` 或 `#define X 1`，适合需要数值的场合。

**为什么要把配置生成到构建目录而不是源码目录？** 因为源码目录应该是只读的（尤其在多配置构建时，同一个源码目录对应多个构建目录，配置值可能不同）。`${PROJECT_BINARY_DIR}` 指向构建目录，生成的文件落在那里。

**交互式配置**：hahack 的教程推荐用 `ccmake`（或老的 `cmake -i`）来改选项：

```plain
$ ccmake .
```

界面里用方向键在字段间移动，Enter 编辑，`c` 执行配置，`g` 生成并退出。屏幕上会列出所有可配置项和当前值。这在选项很多、记不住变量名时比命令行方便。

现在的替代方案是 `cmake-gui`（图形界面）或者直接在命令行传 `-D`。

### 5. Demo5：安装与测试
到这一步才出现 `make install` 和 `make test`。这两个目标在 `CMakeLists.txt` 里没有定义，是 CMake 生成 Makefile 时自动加上的——这正是"CMake 是生成器"的一个直观体现：它替你写了 Makefile 里的伪目标和规则。

**安装规则：**

```cmake
# math/CMakeLists.txt
install(TARGETS MathFunctions DESTINATION bin)
install(FILES MathFunctions.h DESTINATION include)

# 根 CMakeLists.txt
install(TARGETS Demo DESTINATION bin)
install(FILES "${PROJECT_BINARY_DIR}/config.h" DESTINATION include)
```

`DESTINATION` 是相对于安装前缀的路径。默认前缀是 `/usr/local`（Linux），所以：

| 内容 | 安装到 |
| --- | --- |
| `Demo` 可执行文件 | `/usr/local/bin/` |
| `libMathFunctions.a` | `/usr/local/bin/`（因为 `TARGETS MathFunctions` 也写了 `bin`） |
| `MathFunctions.h` | `/usr/local/include/` |
| 生成的 `config.h` | `/usr/local/include/` |


安装前缀可以用 `CMAKE_INSTALL_PREFIX` 改变：

```powershell
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=D:/myapp
cmake --build build
cmake --install build
```

`sudo make install` 的输出大致是：

```plain
Install the project...
-- Install configuration: ""
-- Installing: /usr/local/bin/Demo
-- Installing: /usr/local/bin/libMathFunctions.a
-- Installing: /usr/local/include/MathFunctions.h
-- Installing: /usr/local/include/config.h
```

**测试规则：**

```cmake
# 启用测试
enable_testing()

# 加一个测试：跑 Demo，检查输出里有没有特定内容
add_test(NAME test_power_5_2 COMMAND Demo 5 2)
set_tests_properties(test_power_5_2 PROPERTIES
    PASS_REGULAR_EXPRESSION "is 25")

add_test(NAME test_power_10_5 COMMAND Demo 10 5)
set_tests_properties(test_power_10_5 PROPERTIES
    PASS_REGULAR_EXPRESSION "is 100000")

add_test(NAME test_power_2_10 COMMAND Demo 2 10)
set_tests_properties(test_power_2_10 PROPERTIES
    PASS_REGULAR_EXPRESSION "is 1024")
```

`add_test(NAME 名字 COMMAND 可执行文件 参数...)` 注册一个测试。`set_tests_properties(... PROPERTIES PASS_REGULAR_EXPRESSION "...")` 定义"通过"的判据：程序的标准输出里匹配到这个正则就算通过。

跑测试：

```powershell
cd build
ctest
# 或
make test
```

输出：

```plain
Test project D:/Downloads/lc/Demo5/build
    Start 1: test_power_5_2
1/4 Test #1: test_power_5_2 ...................   Passed
    Start 2: test_power_10_5
2/4 Test #2: test_power_10_5 ..................   Passed
...
100% tests passed, 0 tests failed out of 4
```

**用宏消除重复**：三个测试的写法几乎一样，可以抽成一个宏：

```cmake
macro(do_test arg1 arg2 result)
    add_test(NAME test_power_${arg1}_${arg2} COMMAND Demo ${arg1} ${arg2})
    set_tests_properties(test_power_${arg1}_${arg2} PROPERTIES
        PASS_REGULAR_EXPRESSION "is ${result}")
endmacro()

do_test(5 2 "25")
do_test(10 5 "100000")
do_test(2 10 "1024")
```

这里用宏而不是函数，是因为宏的文本替换语义让 `add_test` 在当前作用域直接生效，不需要额外处理作用域问题。这是"宏适合就地展开的场合"的一个例子。

更完整的 CTest 用法可以看 `man 1 ctest`。现代项目更常用 `include(CTest)` 加上直接调用 gtest/catch2 之类的测试框架，`add_test` 只是把测试可执行文件接进 CTest 的登记表。

### 6. Demo6：环境检查
需求：检查平台是否提供 `pow`，有就用它，没有就用项目自己的实现。

```cmake
include(${CMAKE_ROOT}/Modules/CheckFunctionExists.cmake)

# 必须在 configure_file 之前
check_function_exists(pow HAVE_POW)

configure_file(
    "${PROJECT_SOURCE_DIR}/config.h.in"
    "${PROJECT_BINARY_DIR}/config.h"
)
```

`check_function_exists(函数名 结果变量)` 让链接器试着解析这个函数符号，成功就把结果变量设为真。它做的是**链接期检查**，比"某个头文件存在吗"更准确。

`config.h.in` 里加上：

```c
#cmakedefine HAVE_POW
```

`main.cc` 里：

```cpp
#ifdef HAVE_POW
    result = pow(base, exponent);
#else
    result = power(base, exponent);
#endif
```

**这个模式在实际项目里用得很多**：检测平台有没有某个函数、某个头文件、某个类型，然后生成条件编译的宏。CMake 自带一批这类模块：

| 模块 | 检查什么 |
| --- | --- |
| `CheckFunctionExists` | 函数能否链接 |
| `CheckIncludeFile` | 头文件能否编译进来 |
| `CheckSymbolExists` | 符号（宏或函数）是否存在 |
| `CheckTypeSize` | 类型的大小 |
| `CheckCXXSourceCompiles` | 一段 C++ 代码能否编译 |


`include(${CMAKE_ROOT}/Modules/xxx.cmake)` 里的 `${CMAKE_ROOT}` 是 CMake 安装目录。实际上直接写 `include(CheckFunctionExists)` 就行——`include` 会自动在 CMake 的 `Modules/` 目录里找，不需要写全路径。hahack 那篇写成全路径是当时的习惯，现在更简洁。

### 7. Demo7：版本号
版本号的作用是让用户判断维护状态和向后兼容性。

```cmake
project(Demo7)

set(Demo_VERSION_MAJOR 1)
set(Demo_VERSION_MINOR 0)
```

注意变量名必须是 `<工程名>_VERSION_MAJOR` 的形式——CMake 会按这个约定识别并生成相应的变量，配合 `config.h.in` 传给代码：

```c
#define Demo_VERSION_MAJOR @Demo_VERSION_MAJOR@
#define Demo_VERSION_MINOR @Demo_VERSION_MINOR@
```

`@VAR@` 是 `configure_file` 的另一种替换语法（`${VAR}` 也可以，但 `@VAR@` 在 `.in` 文件里更常用，因为它和 shell 的变量语法不冲突）。

代码里就能打印版本号：

```cpp
std::cout << "Demo Version " << Demo_VERSION_MAJOR
          << "." << Demo_VERSION_MINOR << "\n";
```

**更现代的写法**是用 `project()` 的 `VERSION` 参数：

```cmake
project(Demo VERSION 1.0 LANGUAGES CXX)
```

这样 CMake 自动设置 `PROJECT_VERSION`、`PROJECT_VERSION_MAJOR`、`PROJECT_VERSION_MINOR`、`PROJECT_VERSION_PATCH`，不需要手动 `set`。UnitConverter 用的就是这个：

```cmake
project(UnitConverter VERSION 1.0.0 LANGUAGES CXX)

target_compile_definitions(${PROJECT_NAME} PRIVATE
    UNITCONVERTER_VERSION="${PROJECT_VERSION}"
)
```

这里没用 `configure_file`，而是用 `target_compile_definitions` 把版本号直接作为编译宏传进去——对单个项目的版本号这一种用途来说，比生成头文件更省事。

### 8. Demo8：打包
最后一步是用 CPack 生成安装包。CPack 是 CMake 自带的打包工具。

在 `CMakeLists.txt` 末尾追加：

```cmake
include(InstallRequiredSystemLibraries)

set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")
set(CPACK_PACKAGE_VERSION_MAJOR "${Demo_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${Demo_VERSION_MINOR}")

include(CPack)
```

**打包命令：**

```powershell
# 二进制包
cpack -C CPackConfig.cmake

# 源码包
cpack -C CPackSourceConfig.cmake
```

`include(CPack)` 会生成 `CPackConfig.cmake` 和 `CPackSourceConfig.cmake` 两个文件，`cpack` 读它们来打包。

一次运行会生成多种格式的包（生成器由 `CPACK_GENERATOR` 控制，默认是当前平台所有可用的）：

```plain
CPack: Create package using STGZ
CPack: - package: D:/.../Demo8-1.0.1-Linux.sh generated.
CPack: Create package using TGZ
CPack: - package: D:/.../Demo8-1.0.1-Linux.tar.gz generated.
CPack: Create package using TZ
CPack: - package: D:/.../Demo8-1.0.1-Linux.tar.Z generated.
```

`.sh` 那个是自解压的交互式安装脚本，运行过程：

```plain
Demo8 Installer Version: 1.0.1, Copyright (c) Demo8
This is a self-extracting archive...
The default target directory is: /home/user/Demo8-1.0.1-Linux

Do you accept the license? [yN]: y
By saying yes, you agree to the terms of the license.

Should I create a subdirectory? [yN]: y

Extracting...
```

安装完就能运行：

```powershell
.\Demo8-1.0.1-Linux\bin\Demo.exe 5 2
```

**CPack 支持的格式**（部分）：

| 生成器 | 产物 |
| --- | --- |
| `TGZ` / `TBZ2` / `TXZ` | `.tar.gz` / `.tar.bz2` / `.tar.xz` |
| `STGZ` | 自解压 shell 脚本 |
| `ZIP` | `.zip` |
| `DEB` / `RPM` | Linux 包管理器格式 |
| `NSIS` / `WIX` | Windows 安装程序 |
| `DragNDrop` | macOS 的 `.dmg` |


更完整的用法看 `man 1 cpack` 或官方文档的 CPack 章节。UnitConverter 项目用不到打包，这一节了解即可。

## 七、现代目标模型
前面五、六两节讲的是 CMake 的语言和命令的用法。这一节讲的是**这些命令应该怎么组织**——现代 CMake 和 2013 年那批教程最大的分歧就在这里。

### 1. 目标是中心
**一个目标（target）就是一个构建产物**：可执行文件、静态库、动态库，或者一个纯粹用于传递属性的接口集合。

```cmake
add_executable(UnitConverter src/main.cpp)    # 可执行文件目标
add_library(unitlib STATIC src/unitlib.cpp)   # 静态库目标
add_library(headeronly INTERFACE)             # 接口目标（不产生文件）
```

目标身上挂着一组**属性**（property）：头文件搜索路径、编译宏、编译选项、要链接的库、C++ 标准……这些属性通过 `target_*` 系列命令设置：

```cmake
target_include_directories(unitlib PUBLIC include/)
target_compile_definitions(unitlib PRIVATE UNITLIB_BUILDING)
target_link_libraries(UnitConverter PRIVATE unitlib)
```

也可以直接读写属性：

```cmake
set_target_properties(unitlib PROPERTIES
        CXX_STANDARD 17
        CXX_STANDARD_REQUIRED ON
        OUTPUT_NAME "unit"          # 产出的文件名改成 libunit.a
)

get_target_property(lib_type unitlib TYPE)
message(STATUS "unitlib 类型：${lib_type}")    # STATIC_LIBRARY
```

其实 `target_include_directories`、`target_compile_definitions` 这些命令本质上也是往目标的属性上追加值——CMake 内部存成 `INCLUDE_DIRECTORIES`、`COMPILE_DEFINITIONS` 这些属性。之所以有专门的命令，是因为它们能区分 `PUBLIC`/`PRIVATE`/`INTERFACE`，把值分开存到不同的属性上（`INTERFACE_INCLUDE_DIRECTORIES` 等）。

**重要的是"属性挂在目标上"这件事**：`target_include_directories(unitlib ...)` 只影响 `unitlib` 这一个目标，以及它主动传播出去的部分。它不会影响项目里其他任何东西。

### 2. 旧式命令 vs 现代命令
这是 2013 年那批教程和现在的最大分歧。旧式 CMake（2.x 时代）用全局命令：

```cmake
include_directories(include/)     # 影响从这里往后、当前目录及所有子目录里的所有目标
link_libraries(mylib)             # 同上
add_definitions(-DFOO=1)          # 同上
add_compile_options(-Wall)        # 同上
link_directories(/usr/lib/mylib)  # 同上
```

现代 CMake（3.x 以后）用目标级命令：

```cmake
target_include_directories(myapp PRIVATE include/)
target_link_libraries(myapp PRIVATE mylib)
target_compile_definitions(myapp PRIVATE FOO=1)
target_compile_options(myapp PRIVATE -Wall)
target_link_directories(myapp PRIVATE /usr/lib/mylib)
```

**新旧对照表：**

| 旧式（全局） | 现代（目标级） |
| --- | --- |
| `include_directories(...)` | `target_include_directories(TARGET SCOPE ...)` |
| `link_directories(...)` | `target_link_directories(TARGET SCOPE ...)` |
| `link_libraries(...)` | `target_link_libraries(TARGET SCOPE ...)` |
| `add_definitions(...)` | `target_compile_definitions(TARGET SCOPE ...)` |
| `add_compile_options(...)` | `target_compile_options(TARGET SCOPE ...)` |


看到 `CMakeLists.txt` 里有左列的命令，就是在用旧式写法。不是用了一定出问题，但它们的**作用范围更难推断**。

**一个具体的反例：**

```cmake
# 旧写法
include_directories(libA/include)    # 原本只想让 appX 能找到 A 的头文件
add_executable(appX src/x.cpp)
add_executable(appY src/y.cpp)       # appY 也意外获得了 libA/include
```

`appY` 对 `libA` 一无所知，但它的编译命令里带着 `libA/include`。某天 `libA/include/` 里出现一个和系统头文件同名的文件，`appY` 可能意外包含它，出一个很难查的 bug。

现代写法不存在这个问题，因为传播必须显式声明：

```cmake
target_include_directories(libA PUBLIC libA/include)

add_executable(appX src/x.cpp)
target_link_libraries(appX PRIVATE libA)   # 只有 appX 获得 libA/include

add_executable(appY src/y.cpp)
# appY 完全不涉及 libA
```

**hahack 那篇教程里 Demo4 的 **`include_directories("${PROJECT_SOURCE_DIR}/math")`** 就是这个旧式写法**——它是全局的，在 `if(USE_MYMATH)` 块里调用，一旦调用过，后面所有目标都受影响。UnitConverter 里把这行换成了目标级的 `target_include_directories(unitlib PUBLIC ...)`，放在 `unitlib/CMakeLists.txt` 里，作用范围精确到 `unitlib` 和它的使用者。

**新项目应该只用 **`target_*`** 系列。**

### 3. PUBLIC / PRIVATE / INTERFACE
这三个关键字控制"属性传不传给依赖我的目标"。理解的关键是把**"我自己"**和**"使用我的目标"**分开：

| 关键字 | 编译我自己时用 | 传给链接了我的目标 |
| --- | --- | --- |
| `PRIVATE` | ✓ | ✗ |
| `PUBLIC` | ✓ | ✓ |
| `INTERFACE` | ✗ | ✓ |


`PRIVATE`**：内部实现细节。**

```cmake
target_include_directories(unitlib PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src/internal)
```

`unitlib.cpp` 里 `#include "helper.h"`（在 `src/internal/` 下）需要这个路径，但外部代码不会碰这些内部头文件，所以标 `PRIVATE`。

`PUBLIC`**：我自己需要，使用者也需要。**

```cmake
target_include_directories(unitlib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

`unitlib.cpp` 编译时需要 `unitlib.h`，`main.cpp` 调用 `unitlib` 接口时也需要。标 `PUBLIC` 让 CMake 自动把这个目录传给所有依赖 `unitlib` 的目标——这就是 `main.cpp` 里能直接写 `#include "unitlib.h"` 的原因。

`INTERFACE`**：我自己不需要，使用者需要。**

```cmake
add_library(myheaders INTERFACE)
target_include_directories(myheaders INTERFACE include/)
target_compile_definitions(myheaders INTERFACE MYLIB_VERSION=1)
```

header-only 库的典型写法。库本身没有任何需要编译的源文件（所以它自己在编译期"不需要"任何东西），但消费者需要头文件目录和宏定义。

`target_link_libraries`** 里也能用这三个关键字**，含义是"这个依赖要不要往下传"：

```cmake
target_link_libraries(${PROJECT_NAME} PRIVATE unitlib)
```

`PRIVATE` 表示：`UnitConverter` 依赖 `unitlib`，但如果有别的目标依赖 `UnitConverter`，它们不会自动链接 `unitlib`。

对可执行文件来说 `PRIVATE` 几乎是固定写法（exe 不会被别人链接）。对库来说选择更关键：

+ 库 A 的实现里用了库 B，但 A 的头文件里没出现 B 的类型或函数 → `PRIVATE`
+ 库 A 的头文件里有 B 的类型（比如 `void f(const B::Thing&)`）→ `PUBLIC`，否则消费者 include A 的头文件时找不到 B 的声明

### 4. 传播链是可传递的
假设依赖链是 `C → B → A`（C 链接 B，B 链接 A）：

```plain
A 的 PUBLIC 属性
    ↓  B 链接了 A，B 继承 A 的 PUBLIC
B 的全部 PUBLIC 属性（含从 A 传来的）
    ↓  C 链接了 B，C 继承 B 的 PUBLIC
C 编译时拿到：A 的 PUBLIC 目录 + B 的 PUBLIC 目录
```

C 不需要写 `target_link_libraries(C ... A)` 就能获得 A 的 `PUBLIC include` 目录。**这就是"只链接直接依赖"这条现代 CMake 规则成立的原因。**

老式教程里常见把所有库都手动塞进一个 `target_link_libraries` 的写法，那是传播机制普及前的做法。现在不需要，而且手动列传递依赖反而容易出错：漏一个就链接失败，多一个就引入不必要的耦合。

### 5. 编译和链接是两个阶段
这一节前面（二、6）已经从天真的 g++ 命令角度讲过。这里换成 CMake 命令的角度再看一遍，因为**混淆这两个阶段是新手最常犯的错误**。

**编译阶段**：`.cpp` + `.h` → `.o`

编译器读取 `.cpp`，展开所有 `#include`，翻译成机器指令。这个阶段只关心：

+ 我调用的函数/类，**声明**在哪里？→ 头文件搜索路径 → `target_include_directories`
+ 签名匹配吗？→ 由头文件提供

编译阶段不需要知道函数的实现在哪里。

**链接阶段**：`.o` + `.a` → `.exe`

链接器把多个 `.o` 合并，解析所有未定义的符号引用，把每个引用处填上真实地址。这个阶段只关心：

+ 符号的**定义**在哪里？→ 库 → `target_link_libraries`
+ **链接器完全不认识 **`.h`** 文件**

**对应关系：**

| 需求 | 命令 | 阶段 |
| --- | --- | --- |
| 让编译器找到头文件 | `target_include_directories` | 编译 |
| 让链接器找到符号实现 | `target_link_libraries` | 链接 |
| 传递编译宏 | `target_compile_definitions` | 编译 |
| 传递编译选项 | `target_compile_options` | 编译 |


把头文件路径传给 `target_link_libraries` 是典型错误，会得到 `cannot find -lxxx.h` 这类报错（报错原文见十一节）。

### 6. 用 compile_commands.json 验证传播
CMake 可以把每个文件的实际编译命令导出成 JSON，这是**验证属性传播是否按预期生效的直接证据**。

开启：

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

配置后生成 `build/compile_commands.json`：

```json
[
  {
    "directory": "D:/Downloads/lc/UnitConverter/build",
    "command": "C:\\mingw64\\bin\\g++.exe -DUNITCONVERTER_VERSION=\\\"1.0.0\\\" -ID:/Downloads/lc/UnitConverter/unitlib/include -std=c++17 -o CMakeFiles/UnitConverter.dir/src/main.cpp.obj -c D:/Downloads/lc/UnitConverter/src/main.cpp",
    "file": "D:/Downloads/lc/UnitConverter/src/main.cpp"
  },
  {
    "directory": "D:/Downloads/lc/UnitConverter/build/unitlib",
    "command": "C:\\mingw64\\bin\\g++.exe -DUNITLIB_BUILDING -ID:/Downloads/lc/UnitConverter/unitlib/include -std=c++17 -o CMakeFiles/unitlib.dir/src/unitlib.cpp.obj -c D:/Downloads/lc/UnitConverter/unitlib/src/unitlib.cpp",
    "file": "D:/Downloads/lc/UnitConverter/unitlib/src/unitlib.cpp"
  }
]
```

**从 **`main.cpp`** 的条目能验证两件事：**

+ `-ID:/Downloads/lc/UnitConverter/unitlib/include` —— 这个路径**从来没在根 **`CMakeLists.txt`** 里手写过**，它来自 `unitlib` 的 `PUBLIC target_include_directories`，通过 `target_link_libraries(... unitlib)` 传播过来
+ `-DUNITCONVERTER_VERSION=\"1.0.0\"` —— 来自根文件的 `target_compile_definitions`

**从 **`unitlib.cpp`** 的条目能验证 **`PRIVATE`** 确实不传播：**

+ `-DUNITLIB_BUILDING` 只出现在这个条目里，`main.cpp` 的命令里没有它——因为 `unitlib` 的 `target_compile_definitions` 用的是 `PRIVATE`

**这就是这份文件的用处**：它记录的是 CMake **实际传给编译器**的参数，不是你以为的参数。排查"头文件找不到"这类问题时，比反复读 `CMakeLists.txt` 有效得多。

PowerShell 里查看某个文件的编译命令：

```powershell
$db = Get-Content build\compile_commands.json | ConvertFrom-Json
$db | Where-Object { $_.file -like "*main.cpp" } | Select-Object -ExpandProperty command
```

`compile_commands.json` 还有另一个用途：它是不依赖 CMake 的标准格式，clangd、VS Code 的 C/C++ 扩展、CLion 都靠它提供代码补全和跳转。把它放到项目根目录（或让 IDE 指向它），`#include "unitlib.h"` 就不会显示成错误了。

## 八、库与模块化构建
### 1. add_library 的库类型
```cmake
add_library(unitlib STATIC src/unitlib.cpp)
```

第一个参数是目标名，第二个是库类型：

| 类型 | 产物 | 特点 |
| --- | --- | --- |
| `STATIC` | `libunitlib.a`（GNU/MinGW）   `unitlib.lib`（MSVC） | 链接时直接合入 exe，运行时不需要额外文件 |
| `SHARED` | `libunitlib.so` / `unitlib.dll` | 运行时加载，可被多个程序共享 |
| `MODULE` | 同 `SHARED`，但设计为用 `dlopen`/`LoadLibrary` 运行时加载 | 插件 |
| `INTERFACE` | 无产物 | 只传播属性（header-only 库） |
| 省略 | 由 `BUILD_SHARED_LIBS` 决定（默认 `STATIC`） |  |


**UnitConverter 用 **`STATIC`：只有一个程序用它，静态链接省去部署时复制 `.dll` 的麻烦，也没有 DLL 搜索路径的问题。这一点在本次任务的实测里也得到验证——`libunitlib.a` 直接被链进 `UnitConverter.exe`，拷走 exe 就能运行。

`add_library` 还可以用 `ALIAS` 创建别名：

```cmake
add_library(unitlib::unitlib ALIAS unitlib)
```

别名目标不能用于 `set_target_properties` 等修改性操作，只能用于引用。它的用途是统一命名风格，让项目内的目标名和对外导出的名字一致（`unitlib::unitlib`），后面十一节导出目标时会用到这个形式。

### 2. 用 file(GLOB) 收集源文件
前面 Demo2 提到的 `aux_source_directory` 有一个现代等价物：

```cmake
file(GLOB UNITLIB_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")
add_library(unitlib STATIC ${UNITLIB_SOURCES})
```

**这两者都有同一个陷阱**：新增源文件后 CMake 不会自动重新配置，因为 `CMakeLists.txt` 本身没变，CMake 不会重新运行。结果新建的 `.cpp` 静默地不参与构建。

要强制重新扫描，得手动重跑 `cmake -S . -B build` 或者删掉 `build/CMakeCache.txt`。CMake 官方文档明确建议**不要用 **`GLOB`** 收集源文件**，就是这个原因。

**显式列出源文件是默认做法**：

```cmake
add_library(unitlib STATIC
        src/unitlib.cpp
        # 新增文件时在这里加一行
)
```

多写一行，换来"构建内容和 `CMakeLists.txt` 一一对应"这个确定性。

### 3. add_subdirectory
```cmake
add_subdirectory(unitlib)
```

这一行走进 `unitlib/` 目录，读取那里的 `CMakeLists.txt`，把它定义的目标引入当前工程。**这是两个 **`CMakeLists.txt`** 之间唯一的联系。** 没有这行，`unitlib/CMakeLists.txt` 永远不会被 CMake 读取，`unitlib` 目标不存在，后面的 `target_link_libraries` 会直接报错。

相对路径是相对于当前 `CMakeLists.txt` 所在目录。也可以指定第二个参数作为二进制目录：

```cmake
add_subdirectory(unitlib libs/unitlib)   # 构建产物放在 build/libs/unitlib/
```

默认情况下，子目录的构建产物放在构建目录下的同名子目录里（`build/unitlib/`）。

### 4. UnitConverter 的完整结构
`unitlib/CMakeLists.txt`**（库）：**

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

**根 **`CMakeLists.txt`**（总指挥）：**

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

**这个结构里最值得注意的一点**：根文件完全不知道 `unitlib` 的源码在 `src/unitlib.cpp`，不知道头文件在 `include/`。它只知道有个叫 `unitlib` 的目标，并且 `UnitConverter.exe` 依赖它。把 `unitlib` 内部目录从 `src/` 改成 `lib/`，根文件一行都不用改。

### 5. 构建输出与顺序
```plain
[ 25%] Building CXX object unitlib/CMakeFiles/unitlib.dir/src/unitlib.cpp.obj
[ 50%] Linking CXX static library libunitlib.a
[ 50%] Built target unitlib
[ 75%] Building CXX object CMakeFiles/UnitConverter.dir/src/main.cpp.obj
[100%] Linking CXX executable UnitConverter.exe
[100%] Built target UnitConverter
```

构建顺序由 CMake 根据依赖关系确定，不需要手动排列。`unitlib` 先于 `UnitConverter.exe`，因为链接 exe 需要静态库已经存在。

百分比是每个构建步骤占总步骤的等份。这里有 4 步（编译 `unitlib.cpp`、链接静态库、编译 `main.cpp`、链接 exe），所以每步 25%。

### 6. 增量构建的实际表现
不改任何文件，再跑一次：

```plain
[100%] Built target unitlib
[100%] Built target UnitConverter
```

没有重新编译。make 用文件时间戳判断，源文件比目标文件新才重新编译。

只改 `main.cpp`：

```plain
[ 75%] Building CXX object CMakeFiles/UnitConverter.dir/src/main.cpp.obj
[100%] Linking CXX executable UnitConverter.exe
```

`unitlib` 完全不重新构建。

改 `unitlib.cpp`：

```plain
[ 25%] Building CXX object unitlib/CMakeFiles/unitlib.dir/src/unitlib.cpp.obj
[ 50%] Linking CXX static library libunitlib.a
[ 50%] Built target unitlib
[ 75%] Building CXX object CMakeFiles/UnitConverter.dir/src/main.cpp.obj
[100%] Linking CXX executable UnitConverter.exe
```

`libunitlib.a` 更新了 → 依赖它的 `UnitConverter.exe` 需要重新链接。这就是目标划分带来的收益：改动局限在目标内部时，其他目标的编译被跳过。

**对比一下手工 g++ 的情形**：改 `unitlib.cpp` 之后要记得重新编译它、重新打包 `.a`、重新链接——三步都得手动做，顺序错了还会失败。这里只需要 `mingw32-make`。

### 7. 扩展：加一个新模块
想在项目里加一个 `formatter/` 模块：

```plain
UnitConverter/
├── CMakeLists.txt
├── formatter/
│   ├── CMakeLists.txt
│   ├── include/
│   │   └── formatter.h
│   └── src/
│       └── formatter.cpp
└── ...
```

`formatter/CMakeLists.txt`：

```cmake
add_library(formatter STATIC src/formatter.cpp)
target_include_directories(formatter PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

根文件加两行：

```cmake
add_subdirectory(formatter)
target_link_libraries(${PROJECT_NAME} PRIVATE unitlib formatter)
```

其他文件都不用动。`main.cpp` 里加 `#include "formatter.h"` 就能用——路径由 `formatter` 的 `PUBLIC` 自动传播。

**在 **`unitlib`** 内部加一个新的转换类**则连根文件都不用改：新建 `speed.h` 和 `speed.cpp` 放进对应目录，只在 `unitlib/CMakeLists.txt` 的 `add_library` 里加一行源文件即可。头文件目录不用动（新文件也在 `include/` 下，搜索路径已经对了）。

这两个改动的范围差异，正好说明模块化结构里"边界"划在哪里的价值。

### 8. 为什么要有两个 CMakeLists.txt
任务书里这样解释这个要求：

> 这里限制了目录结构甚至要求大家写两个 CMakeLists.txt 是希望能在小项目中尽可能模拟大家日后会接触到的更加庞大的项目的结构。
>

这句话的重点在"模拟"。UnitConverter 只有两个 `.cpp`，把它们都列进一个 `add_executable` 完全可行。任务书刻意要求两个，是想让人在小项目里体验一次大项目的组织方式。

设想一个有 20 个模块的项目：

```plain
bigproject/
├── CMakeLists.txt
├── core/       ├── CMakeLists.txt  └── src/ include/
├── network/    ├── CMakeLists.txt  └── src/ include/
├── storage/    ├── CMakeLists.txt  └── src/ include/
├── ui/         ├── CMakeLists.txt  └── src/ include/
│   ...（还有 16 个）
└── app/        ├── CMakeLists.txt  └── src/main.cpp
```

根文件只需要：

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

每个模块只管好自己的 `CMakeLists.txt`：声明源文件、头文件目录、内部依赖。根文件不需要知道每个模块的内部结构，只知道目标名。模块之间通过目标名声明依赖，CMake 自动推算构建顺序、传播头文件路径、解析链接依赖。

这和软件工程里的接口隔离是同一件事：**调用者只依赖接口（目标名），不依赖实现（源文件路径、内部目录结构）。**

## 九、依赖管理
### 1. find_package
`find_package` 用来查找系统里已安装的第三方库：

```cmake
find_package(OpenCV REQUIRED)
find_package(Threads REQUIRED)
find_package(fmt 9.0 REQUIRED COMPONENTS header-only)
```

参数有三部分：

+ **包名**：`OpenCV`
+ **版本**（可选）：`9.0`，不满足会报错
+ `REQUIRED`：找不到就报错停止。不写的话找不到了只会设一个 `<包名>_FOUND` 为假，静默继续——除非你后面自己检查，否则会在链接期才发现问题
+ `COMPONENTS`：需要这个包的哪些子组件

**hahack 那篇 2013 年的教程完全没讲 **`find_package`，它用的是 `check_function_exists` 做更基础的环境检查。`find_package` 在现代 CMake 里是接入第三方库的标准方式，必须掌握。

### 2. 两种查找模式
`find_package` 有两种工作模式，理解它们的区别有助于排查"找不到包"的问题。

**Config 模式**：库自带一个配置文件，名字是 `<PackageName>Config.cmake` 或 `<package-name>-config.cmake`。这个文件由库的作者提供，里面声明了库提供了哪些目标、每个目标的头文件路径和依赖。

现代库（fmt、spdlog、protobuf、Catch2、gRPC……）都提供这个文件。**这是首选模式。**

**Module 模式**：CMake 自带的 `Find<PackageName>.cmake` 脚本，位于 CMake 安装目录的 `Modules/` 下。适用于一些标准库和常见系统库：

| 模块 | 查找 |
| --- | --- |
| `FindThreads.cmake` | pthread |
| `FindZLIB.cmake` | zlib |
| `FindPython.cmake` | Python 解释器和库 |
| `FindBoost.cmake` | Boost |
| `FindOpenMP.cmake` | OpenMP |


查找顺序默认是先 Module 再 Config（所以 `FindZLIB.cmake` 会赢过 zlib 自己带的 config），可以用 `MODULES` 或 `CONFIG` 关键字强制指定：

```cmake
find_package(OpenCV REQUIRED CONFIG)     # 只用 Config 模式
find_package(ZLIB REQUIRED MODULES)      # 只用 Module 模式
```

### 3. 两种使用方式
**方式一：导入目标（推荐）**

现代库会提供导入目标（imported target）：

```cmake
find_package(fmt REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)

find_package(OpenCV REQUIRED)
target_link_libraries(myapp PRIVATE opencv_core opencv_imgproc)
```

导入目标自带所有属性：头文件路径、编译定义、它自己依赖的其他库。`target_link_libraries` 一步到位，不需要手动处理 `-I` 和 `-L`。

**注意 **`fmt::fmt`** 里的双冒号**。这是导入目标的命名约定（`<命名空间>::<目标名>`），一眼就能区分"这是从外部找来的目标"和"这是本项目定义的目标"。CMake 会强制检查：带 `::` 的目标名如果不是导入目标或别名，配置阶段直接报错。这个机制能防止拼错名字——写 `fmt::fnt` 会立刻报错，而写 `fmt` 会当成"本项目里有个叫 fmt 的目标"去找。

**方式二：变量**

老式库只提供变量：

```cmake
find_package(OpenCV REQUIRED)
target_include_directories(myapp PRIVATE ${OpenCV_INCLUDE_DIRS})
target_link_libraries(myapp PRIVATE ${OpenCV_LIBS})
```

变量名没有统一标准，不同库不一样（`OpenCV_INCLUDE_DIRS`、`ZLIB_INCLUDE_DIR`、`Python_INCLUDE_DIRS`……），需要查对应库的文档。**优先用方式一。**

### 4. 找不到包怎么排查
**第一步：看报错里给出的搜索路径。**

```plain
CMake Error at CMakeLists.txt:5 (find_package):
  Could not find a package configuration file provided by "OpenCV" with any
  of the following names:

    OpenCVConfig.cmake
    opencv-config.cmake

  Add the installation prefix of "OpenCV" to CMAKE_PREFIX_PATH or set
  "OpenCV_DIR" to a directory containing one of the above files.
```

报错本身就告诉你怎么办了。

**第二步：显式指定搜索位置。**

```powershell
# 直接指向含 Config 文件的目录
cmake -S . -B build -DOpenCV_DIR=D:/opencv/build

# 或者指定搜索根目录，CMake 会在它下面的标准位置找
cmake -S . -B build -DCMAKE_PREFIX_PATH=D:/opencv/build
```

`<PackageName>_DIR` 指向**含有** `OpenCVConfig.cmake` 的目录（不是它的上级）。`CMAKE_PREFIX_PATH` 更通用：CMake 会在它下面的 `lib/cmake/`、`share/cmake/`、`lib/cmake/<名字>/` 等标准位置搜索。

**第三步：确认库真的装了。**

有些系统上库的头文件和运行库是分开装的（Linux 上的 `libfoo-dev` 和 `libfoo`），只装了运行库就找不到 `.cmake` 配置。

**诊断命令：**

```powershell
cmake --find-package -DNAME=OpenCV -DCOMPILER_ID=GNU -DLANGUAGE=CXX -DMODE=EXIST
```

### 5. 引入本地第三方库：add_subdirectory
如果第三方库是源码形式的（比如放在 `third_party/` 下的 git submodule），可以直接加进来：

```cmake
add_subdirectory(third_party/fmt)
target_link_libraries(myapp PRIVATE fmt::fmt)
```

这种方式把第三方库和项目一起编译。好处是不依赖系统里装了特定版本，构建结果可复现；缺点是编译时间变长，而且要求第三方库的 `CMakeLists.txt` 写得规范（不用全局命令污染上层作用域）。

### 6. FetchContent：自动下载
更现代的方式是 `FetchContent`，能在配置阶段自动下载并引入：

```cmake
include(FetchContent)

FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 10.1.1              # 指定 tag 或 commit，不要用分支
)
FetchContent_MakeAvailable(fmt)

target_link_libraries(myapp PRIVATE fmt::fmt)
```

`FetchContent_MakeAvailable(fmt)` 做了三件事：下载、`add_subdirectory`、允许用它的目标。

几个实践要点：

+ `GIT_TAG`** 用固定的 tag 或 commit，不要用分支名**。用分支意味着构建结果随时间变化，同一个 `CMakeLists.txt` 在不同时间配置出不同的依赖版本
+ 下载的内容放在构建目录的 `_deps/` 下，不会污染源码树
+ 它和 `find_package` 可以组合：先 `find_package` 找系统里装好的，找不到再 `FetchContent` 下载（需要 `FIND_PACKAGE_ARGS` 参数）

`FetchContent` 是 CMake 3.11 引入的，3.14 加了 `MakeAvailable`，实际使用建议 3.14 以上。

### 7. 依赖选择建议
| 场景 | 选择 |
| --- | --- |
| 系统里装好了、版本合适 | `find_package(Pkg REQUIRED CONFIG)` |
| 系统里可能没装，希望自动搞定 | `FetchContent` |
| 项目自己维护的第三方源码 | `add_subdirectory(third_party/xxx)` |
| 只是几个头文件 | 直接 `target_include_directories`，不用建目标 |


## 十、配置、调试与安装
### 1. 构建类型
CMake 内置四种构建类型：

| 类型 | 优化 | 调试信息 | 典型用途 |
| --- | --- | --- | --- |
| `Debug` | `-O0` | `-g` | 开发调试 |
| `Release` | `-O3` | 无 | 发布 |
| `RelWithDebInfo` | `-O2` | `-g` | 性能分析 |
| `MinSizeRel` | `-Os` | 无 | 体积敏感 |


指定方式：

```powershell
cmake -S . -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug
```

**单配置生成器 vs 多配置生成器**，这是容易踩的一个坑：

| 生成器 | 类型 | 特点 |
| --- | --- | --- |
| Unix Makefiles / MinGW Makefiles / Ninja | 单配置 | `build/` 里只能有一种构建类型，换类型要重新配置 |
| Visual Studio / Xcode | 多配置 | 一个 `build/` 里同时存在 Debug 和 Release，用 `--config` 切换 |
| Ninja Multi-Config | 多配置 | 同上，但用 Ninja |


用 MinGW Makefiles 时，`build/` 目录绑定一种构建类型。要同时有 Debug 和 Release，得开两个构建目录：

```powershell
cmake -S . -B build-debug   -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug
cmake -S . -B build-release -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release
cmake --build build-debug
cmake --build build-release
```

在 `CMakeLists.txt` 里设默认值（只在命令行没传时生效）：

```cmake
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type" FORCE)
endif()
```

### 2. 调试构建的编译选项
hahack 那篇教程里给的做法：

```cmake
set(CMAKE_BUILD_TYPE "Debug")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -Wall -g -ggdb")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3 -Wall")
```

+ `-g` 生成调试符号
+ `-ggdb` 生成 GDB 专用的扩展调试信息（比 `-g` 更完整）
+ 追加而不是覆盖：`${CMAKE_CXX_FLAGS_DEBUG}` 是 CMake 已有的默认值，直接 `set` 会丢掉它

**更推荐的做法**是用目标级的 `target_compile_options` 配合生成器表达式，避免全局污染：

```cmake
target_compile_options(UnitConverter PRIVATE
    $<$<CONFIG:Debug>:-Wall -Wextra -Wpedantic -g>
    $<$<CONFIG:Release>:-O3 -Wall>
)
```

### 3. 生成器表达式
生成器表达式用 `$<...>` 语法，**在生成构建文件时才展开**，而不是配置阶段。这允许根据构建类型、平台、编译器做条件展开。

```cmake
target_compile_options(UnitConverter PRIVATE
    $<$<CONFIG:Debug>:-Wall>
)
```

`$<$<CONFIG:Debug>:-Wall>` 读作：如果当前构建类型是 Debug，展开为 `-Wall`；否则展开为空。配置阶段它是一整块文本，生成 Makefile 时才决定。

**常用生成器表达式：**

| 表达式 | 含义 |
| --- | --- |
| `$<CONFIG:Debug>` | 布尔值，当前是 Debug 时为 1 |
| `$<PLATFORM_ID:Windows>` | 平台是否匹配 |
| `$<TARGET_FILE:mylib>` | 目标的产出文件绝对路径 |
| `$<TARGET_OBJECTS:mylib>` | 目标的对象文件列表 |
| `$<BOOL:${VAR}>` | 变量转布尔 |
| `$<IF:cond,a,b>` | 三元表达式 |
| `$<STREQUAL:a,b>` | 字符串比较 |


嵌套起来可读性会变差。能用普通 `if()` 解决的问题不必用它，但**区分构建类型这个场景只能用生成器表达式**——因为 `if(CMAKE_BUILD_TYPE STREQUAL "Debug")` 在配置阶段判断，而多配置生成器在配置阶段根本不知道最终会构建哪个类型。

### 4. C++ 标准的三种设置方式
```cmake
# 方式一：全局变量，影响所有目标
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 方式二：目标属性，只影响一个目标
set_target_properties(unitlib PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
)

# 方式三：按特性声明（最精确）
target_compile_features(unitlib PRIVATE cxx_std_17)
```

三个变量的区别：

+ `CMAKE_CXX_STANDARD_REQUIRED ON`：编译器不支持该标准时直接报错，而不是静默降级。不加这个，老编译器可能悄悄用 C++14 编译，然后在运行时出奇怪的问题
+ `CMAKE_CXX_EXTENSIONS OFF`：让 CMake 传 `-std=c++17` 而不是 `-std=gnu++17`。保证代码在不同编译器上行为一致（GNU 扩展不是标准的一部分）

方式三的语义是"我需要 C++17 的特性"，比"我需要 C++17 这个版本号"更自描述。用了 C++20 的某个特性就写 `cxx_std_20`。

### 5. install 规则
```cmake
install(TARGETS UnitConverter RUNTIME DESTINATION bin)
install(TARGETS unitlib ARCHIVE DESTINATION lib)
install(FILES include/unitlib.h DESTINATION include)
install(DIRECTORY doc/ DESTINATION share/UnitConverter/doc)
```

`install` 命令在 `cmake --install` 时生效。`DESTINATION` 是相对于安装前缀的路径。

`TARGETS`** 之后的关键字指定产物类型**（不同平台上是不同种类的文件，必须分开）：

| 关键字 | 对应产物 |
| --- | --- |
| `RUNTIME` | 可执行文件、`.dll`（Windows） |
| `LIBRARY` | 共享库（`.so`、`.dylib`） |
| `ARCHIVE` | 静态库（`.a`、`.lib`）、导入库 |
| `PUBLIC_HEADER` | 声明为 `PUBLIC_HEADER` 属性的头文件 |
| `INCLUDES DESTINATION` | 写成导出目标的 `INTERFACE_INCLUDE_DIRECTORIES` |


静态库必须用 `ARCHIVE`；直接写 `install(TARGETS unitlib DESTINATION lib)` 会让 CMake 报错要求区分产物类型（或者用 `RUNTIME LIBRARY ARCHIVE` 一次给三个都指定）。

默认安装前缀：

+ Linux：`/usr/local`
+ Windows（MinGW）：`C:/Program Files (x86)/<项目名>`

自定义前缀：

```powershell
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=D:/myapp
cmake --build build
cmake --install build
```

安装后的结构：

```plain
D:/myapp/
├── bin/
│   └── UnitConverter.exe
├── lib/
│   └── libunitlib.a
└── include/
    └── unitlib.h
```

**注意 **`include/`** 的布局问题**：这样装完之后，`unitlib.h` 直接躺在 `include/` 下面。如果以后装了第二个库，它的头文件也放 `include/`，就可能撞名。更稳妥的做法是加一层子目录：

```cmake
install(FILES include/unitlib.h DESTINATION include/unitlib)
```

这样消费者要写 `#include "unitlib/unitlib.h"`。装系统目录的库通常都这么做（`include/opencv4/`、`include/fmt/`）。

### 6. 组件（COMPONENT）
把安装内容分组，按需安装：

```cmake
install(TARGETS UnitConverter RUNTIME DESTINATION bin COMPONENT runtime)
install(TARGETS unitlib ARCHIVE DESTINATION lib COMPONENT dev)
install(FILES include/unitlib.h DESTINATION include/unitlib COMPONENT dev)
```

只装运行时部分：

```powershell
cmake --install build --component runtime
```

打包发布时很有用：给最终用户的安装包只含 `runtime`（可执行文件加动态库），给开发者的 SDK 含所有组件。

### 7. 导出目标：让别的项目能找到你的库
前面 `install` 只是把文件复制到某处。如果希望别的 CMake 项目能用 `find_package(unitlib)` 找到它，还需要**导出一份 CMake 配置**：

```cmake
# 把 unitlib 目标登记进一个导出集合
install(TARGETS unitlib
    EXPORT unitlibTargets
    ARCHIVE DESTINATION lib
    INCLUDES DESTINATION include          # 写进导出目标的头文件路径
)

install(FILES include/unitlib.h DESTINATION include/unitlib)

# 把这个导出集合写成 CMake 配置文件
install(EXPORT unitlibTargets
    FILE unitlibConfig.cmake
    NAMESPACE unitlib::
    DESTINATION lib/cmake/unitlib
)
```

装完之后，另一个项目就能：

```cmake
find_package(unitlib REQUIRED)
target_link_libraries(otherapp PRIVATE unitlib::unitlib)
```

`NAMESPACE unitlib::` 给导出的目标名加上前缀，避免和消费者项目里自己的目标名冲突。这也解释了前面 `fmt::fmt` 那种写法的来源——那是 fmt 库在导出时用的命名空间。

`INCLUDES DESTINATION include` 的作用是把这个相对路径换算成绝对路径，写进导出目标的 `INTERFACE_INCLUDE_DIRECTORIES`，这样消费者才知道去哪里找头文件。

**这是发布库时应该做的事**，本次任务用不到，但接触第三方库时会经常从消费者这一侧看到它（`fmt::fmt`、`opencv_core` 都是这样来的）。

### 8. 优化构建速度
**并行编译：**

```powershell
mingw32-make -j4           # 4 个并行任务
cmake --build build -j 4   # 跨平台写法
```

`-j` 让 make 同时编译多个文件。一般设成 CPU 核心数，或核心数 +1。**注意 **`-j`** 和错误输出的关系**：并行时多个任务的输出会交错，报错信息不一定连贯。调试时可以先不加 `-j`。

**换用 Ninja：**

```powershell
cmake -S . -B build -G Ninja
cmake --build build
```

Ninja 的增量检测比 Make 更精确，启动开销更小，在大项目上通常更快。Ninja 不支持自定义规则（Makefile 的灵活语法在 Ninja 里被砍掉了），这正是它快的原因。

**预编译头（PCH）**：如果所有源文件都包含同一块大东西，可以预编译：

```cmake
target_precompile_headers(unitlib PRIVATE
    <vector>
    <string>
    <iostream>
)
```

CMake 会把这块头文件编译一次，后续每个源文件复用结果。对包含标准库头文件较多的项目效果明显。

**ccache / sccache**：缓存编译结果，重复编译同一份代码时直接命中缓存。切分支、清理重构建时收益很大。Linux 上配 `ccache`，Windows 上可以用 `sccache`。

### 9. 清理与重新配置
大部分改动不需要清理，直接重新 `mingw32-make` 就行。以下改动会让 `CMakeCache.txt` 里的缓存失效，需要删掉 `build/` 从头配置：

+ 换了生成器（从 Ninja 改成 MinGW Makefiles）
+ 换了 `CMAKE_CXX_COMPILER` 指向的编译器
+ 删掉了某个 `add_subdirectory`，或重命名了目标
+ 升级了 CMake 大版本

```powershell
# 彻底清理
Remove-Item -Recurse -Force build
cmake -S . -B build -G "MinGW Makefiles"
cmake --build build

# 只清理编译产物，保留配置缓存
mingw32-make clean
```

`make clean` 删掉 `.o`、`.a`、`.exe`，但保留 CMake 生成的 Makefile 和缓存。重新构建会重新编译所有文件，但跳过配置阶段。适合"改了源码想完整重编"，不适合"改了 `CMakeLists.txt` 结构"。

`CMakeCache.txt`** 是什么**：纯文本，记录所有缓存变量：

```plain
CMAKE_CXX_COMPILER:FILEPATH=C:/mingw64/bin/g++.exe
CMAKE_BUILD_TYPE:STRING=Debug
CMAKE_INSTALL_PREFIX:PATH=C:/Program Files (x86)/UnitConverter
```

格式是 `变量名:类型=值`。**配置行为诡异时，删掉 **`build/`** 重新来一遍是最省事的排查手段**——它排除了所有缓存残留的可能。

## 十一、常见报错速查
### 1. 配置阶段
`CMake Error: The source directory ... does not appear to contain CMakeLists.txt`

`cmake` 的参数指向的目录里没有 `CMakeLists.txt`。检查当前目录是否是 `build/`，或者 `-S` 后面的路径对不对。

`CMake Error at CMakeLists.txt:12 (add_subdirectory): add_subdirectory given source "unitlib" which is not an existing directory`

`add_subdirectory` 的路径不对，或对应目录里没有 `CMakeLists.txt`。`add_subdirectory` 的自变量必须是**含 **`CMakeLists.txt`** 的目录**，不能指向文件。

`CMake Error: No CMAKE_CXX_COMPILER could be found.`

CMake 在 PATH 里找不到 C++ 编译器。先确认 `g++ --version` 能用；或者显式指定：

```powershell
cmake -S . -B build -G "MinGW Makefiles" -DCMAKE_CXX_COMPILER=C:/mingw64/bin/g++.exe
```

`CMake Error: Could not find a package configuration file provided by "XXX"`

`find_package` 找不到。用 `-DXXX_DIR=...` 或 `-DCMAKE_PREFIX_PATH=...` 告诉 CMake 去哪找（见九、4）。

`CMake Error: The current CMakeCache.txt directory ... is different than the directory ... where CMakeCache.txt was created`

构建目录被移动或复制过，缓存里记录的绝对路径失效。删掉 `build/` 重新配置。

`CMake Warning: Manually-specified variables were not used by the project: XXX`

命令行传的 `-DXXX=...` 在整个构建里没有任何地方读取它。通常是变量名拼错了，或者它本该被 `option()`/`set(... CACHE ...)` 声明但没写。这个警告很有用，值得留意。

`CMake Error at ... (target_link_libraries): Cannot specify link libraries for target "xxx" which is not built by this project.`

`target_link_libraries` 的第一个参数是个不存在的目标名。常见原因是拼错了，或者 `add_subdirectory` 漏了（子目录里的目标还没被引入）。

### 2. 编译阶段
`fatal error: unitlib.h: No such file or directory`

编译器找不到头文件。排查顺序：

1. `target_include_directories` 是否加在了正确的目标上
2. 传播关键字对不对——`main.cpp` 找不到时，检查 `unitlib` 用的是不是 `PUBLIC`，以及根文件有没有 `target_link_libraries(... unitlib)`
3. 看 `compile_commands.json` 里 `main.cpp` 的 `-I` 参数，确认 `unitlib/include` 在不在
4. 检查用的是 `CMAKE_CURRENT_SOURCE_DIR` 还是 `CMAKE_SOURCE_DIR`——在 `unitlib/CMakeLists.txt` 里错用后者会指向根目录

`error: 'to_string' is not a member of 'std'`**（或类似的标准库特性找不到）**

C++ 标准版本太低或没设置。检查 `CMAKE_CXX_STANDARD` 或 `CXX_STANDARD`。

`Makefile:3: *** missing separator. Stop.`

Makefile 里命令行的缩进用了空格而不是 Tab。手写 Makefile 时容易遇到；CMake 生成的 Makefile 不会。

### 3. 链接阶段
`undefined reference to 'TemperatureConverter::convert(int)'`

头文件里有声明，但实现没被编译进来。检查：

1. 实现所在的 `.cpp` 是否列在 `add_library` / `add_executable` 里
2. 该库是否用 `target_link_libraries` 链进来了
3. 符号名对不上时（比如改了签名但没重编），可以先清理重构建排除缓存问题

`ld.exe: cannot find -lunitlib/include/unitlib.h: No such file or directory`

**最典型的错误**：把头文件路径传给了 `target_link_libraries`。链接器的参数应该是库名或库目标名，不是头文件。改成目标名 `unitlib` 即可。

这是本次任务实际踩到的坑，完整分析见实践任务文档的第三节。

`multiple definition of 'xxx'`

同一个符号在多个 `.o` 里被定义。常见原因：头文件里定义了非 `inline` 的函数或全局变量，而它被多个 `.cpp` 包含。解法是头文件里只放声明，定义放 `.cpp`；或者用 C++17 的 `inline`（变量也可以 `inline`）。

`undefined reference to 'pthread_create'`**（Linux）**

用了 pthread 但没链接。用 `find_package(Threads REQUIRED)` 加 `target_link_libraries(myapp PRIVATE Threads::Threads)`，比手写 `-lpthread` 跨平台。

`undefined reference`** 指向标准库符号（比如 **`std::cout`**）**

链接顺序问题，或者用 `gcc` 而不是 `g++` 去链接 C++ 代码。`gcc` 不会自动链接 `libstdc++`，链接 C++ 程序要用 `g++`。

### 4. 运行阶段
**exe 被 Windows Defender 杀掉**

编译成功但生成的 exe 在几秒内消失或变成零字节。这是 Defender 在扫描新生成的本地 exe，和 CMake 无关。把 `build/` 加入排除列表：`Windows 安全中心 → 病毒和威胁防护 → 管理设置 → 排除项 → 添加排除项 → 文件夹`。

**提示找不到 **`.dll`**（用了 SHARED 库时）**

动态链接的库需要运行时能找到。把 `.dll` 放在 exe 同目录，或者把它的目录加进 `PATH`。

### 5. 诊断思路总结
| 出错阶段 | 判断依据 | 先怀疑什么 |
| --- | --- | --- |
| 配置阶段 | 报错带 `CMake Error at CMakeLists.txt:N` | `CMakeLists.txt` 的语法、路径、目标名 |
| 编译阶段 | `error:` 带源文件名和行号 | 源码 |
| 链接阶段 | `undefined reference` / `cannot find -l` | `CMakeLists.txt`（库没链对） |
| 运行阶段 | 启动即崩溃或找不到文件 | 运行时依赖、路径 |


**验证 **`CMakeLists.txt`** 是否正确的可靠手段**：看 `compile_commands.json`。它反映的是 CMake **实际传给编译器**的参数，不是你以为的。

## 十二、一页速查
### 1. 最小可用工程
```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

add_executable(myapp src/main.cpp)
```

```powershell
cmake -S . -B build -G "MinGW Makefiles"
cmake --build build
.\build\myapp.exe
```

### 2. 一个库 + 一个程序（现代写法）
```cmake
# unitlib/CMakeLists.txt
add_library(unitlib STATIC src/unitlib.cpp)
target_include_directories(unitlib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

add_subdirectory(unitlib)

add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE unitlib)
```

### 3. 三条最重要的规则
1. **只用 **`target_*`** 系列命令，不用全局命令**（`include_directories` 等）
2. **头文件路径用 **`target_include_directories`**，库用 **`target_link_libraries`，两者不能互换
3. **只链接直接依赖**，传递依赖由 `PUBLIC` 传播自动处理

### 4. 命令速查表
| 命令 | 作用 |
| --- | --- |
| `cmake_minimum_required(VERSION X)` | 声明最低版本，切换 policy |
| `project(NAME VERSION X LANGUAGES CXX)` | 定义工程和一批变量 |
| `add_executable(TARGET srcs...)` | 可执行文件目标 |
| `add_library(TARGET STATIC srcs...)` | 库目标 |
| `add_subdirectory(dir)` | 引入子目录 |
| `target_include_directories(TARGET SCOPE dirs)` | 头文件搜索路径 |
| `target_link_libraries(TARGET SCOPE libs)` | 链接依赖 |
| `target_compile_definitions(TARGET SCOPE defs)` | 编译宏 |
| `target_compile_options(TARGET SCOPE opts)` | 编译选项 |
| `target_compile_features(TARGET SCOPE cxx_std_17)` | 按特性要求 C++ 标准 |
| `set_target_properties(TARGET PROPERTIES ...)` | 设置目标属性 |
| `set(VAR v)` / `set(VAR v CACHE STRING "d")` | 普通变量 / 缓存变量 |
| `option(VAR "描述" ON)` | 布尔选项 |
| `configure_file(in out)` | 生成配置文件 |
| `find_package(Pkg REQUIRED CONFIG)` | 查找第三方库 |
| `install(TARGETS ... DESTINATION ...)` | 安装规则 |
| `enable_testing()` / `add_test(NAME n COMMAND c)` | 测试 |
| `message(STATUS ...)` | 输出信息 |
| `include(Module)` | 引入模块 |
| `foreach` / `if` / `function` / `macro` | 控制流与封装 |
| `$<CONFIG:Debug>` | 生成器表达式 |


### 5. 命令速查表（构建命令）
| 命令 | 作用 |
| --- | --- |
| `cmake -S . -B build -G "MinGW Makefiles"` | 配置 |
| `cmake --build build` | 构建（跨平台） |
| `cmake --build build -j 4` | 并行构建 |
| `cmake --install build` | 安装 |
| `cmake --install build --component runtime` | 按组件安装 |
| `ctest --test-dir build` | 跑测试 |
| `cmake --build build --clean-first` | 清理后重建 |
| `cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug` | 指定构建类型 |
| `cmake -S . -B build -DCMAKE_INSTALL_PREFIX=D:/myapp` | 指定安装前缀 |
| `cmake -S . -B build -DVAR=value` | 传缓存变量 |
| `cmake --help` | 列出可用生成器 |
| `cmake --version` | 查看 CMake 版本 |


## 参考资料
**主要参考的四份材料：**

+ [Zero to CMake（buchanan.one）](https://buchanan.one/blog/zero-to-cmake/) —— 本文第二、三、四节的框架来源，从编译器到 make 到 CMake 的倒序讲法
+ [CMake 入门实战（hahack.com）](https://www.hahack.com/codes/cmake/) —— 第六节案例递进（Demo1 到 Demo8）的来源
+ [Learn CMake in Y Minutes](https://learnxinyminutes.com/zh-cn/cmake/) —— 第五节语言基础的来源
+ [CMake 官方文档](https://cmake.org/cmake/help/latest/) —— 现代目标模型、`PUBLIC`/`PRIVATE`/`INTERFACE`、`find_package`、`install`/`export` 的权威定义

**延伸阅读：**

+ [CMake 官方教程](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)
+ [An Introduction to Modern CMake (cliutils.gitlab.io)](https://cliutils.gitlab.io/modern-cmake/) —— 现代 CMake 最系统的入门材料
+ [Effective Modern CMake (gist)](https://gist.github.com/mbinna/c61dbb39bca0e4fb7d1f73d0669e5a5c) —— 实践建议清单
+ [CMake: PUBLIC vs PRIVATE vs INTERFACE](https://leimao.github.io/blog/CMake-Public-Private-Interface/) —— 三个关键字的图解
+ [compile_commands.json 格式说明（LLVM）](https://clang.llvm.org/docs/JSONCompilationDatabase.html) —— 验证编译参数的标准格式
+ [GNU Make 手册](https://www.gnu.org/software/make/manual/) —— 第三节手写 Makefile 部分
+ [Debugging CMake 常见问题（CMake Wiki）](https://gitlab.kitware.com/cmake/community/-/wikis/FAQ) —— 报错速查

