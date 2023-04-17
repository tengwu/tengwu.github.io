+++
title = "使用flex, bison, llvm实现编译器"
date = 2023-04-02T14:12:00+08:00
lastmod = 2023-04-17T18:12:45+08:00
tags = ["编译", "LLVM"]
categories = ["编译", "LLVM"]
draft = false
toc = true
+++

## 2023年实现编译器竟然如此简单 {#2023年实现编译器竟然如此简单}

这周跟着[LLVM官方教程](https://llvm.org/docs/tutorial/)学习了一下 `LLVM` 的基础知识,实现了一个可以跑起来的编译器,当然其实就是把人家提供的代码稍微改一改,不理解的地方单步跟着调一下.

抱着学习 `LLVM` 的目的,周末突然产生了用 `flex`, `bison`, `LLVM` 实现一个编译器的想法,网上搜索了一下,十几年前就有人这样做了,索性就直接跟着别人的博客学习一下.还是先把别人的代码跑起来,读代码,不懂的地方单步跟一下,如果有什么想法的话,再把别人的代码魔改一下.


## 参考 {#参考}

1.  <https://gnuu.org/2009/09/18/writing-your-own-toy-compiler/>
2.  [动手写个玩具编译器](https://jeremyxu2010.github.io/2020/10/%E5%8A%A8%E6%89%8B%E5%86%99%E4%B8%AA%E7%8E%A9%E5%85%B7%E7%BC%96%E8%AF%91%E5%99%A8/#heading-1)
    `链接2` 参考了 `链接1`, `链接1` 是2009年写的一篇博客,基于 `LLVM 2.6`, `链接2` 是2020年写的一篇博客,基于更高版本的 `LLVM` (经过测试, `LLVM 9.0.1_4` 可以编译通过).
    现在是2023年4月2日, `LLVM` 稳定版本是16.0.0,也许我可以试着把 `链接2` 的代码改一改,适配 `LLVM 16.0.0`.


## 代码适配 `LLVM 16.0.0` {#代码适配-llvm-16-dot-0-dot-0}


### 项目构建 {#项目构建}


#### 将已有的代码编译通过 {#将已有的代码编译通过}

`LLVM` 每一个版本的接口都有很大的不同,每个版本使用的C++标准甚至都有差别,为了将已有代码编译通过,需要使用合适的C++标准和 `LLVM` 版本.

经测试,原始代码使用的C++标准是14, `LLVM` 版本为 `LLVM 9.0.1_4`.在 `macOS 13.2.1` 上,可以通过 `brew install llvm@9` 来安装这个版本的 `LLVM`,在安装结束后, `brew` 会输出安装位置(在这个目录下有 `bin`, `lib`, `include` 等常见目录),在本文中,我们将这个位置称为 `LLVM_DIR`.例如,在我的系统上, `LLVM_DIR` 为 `/usr/local/Cellar/llvm@9/9.0.1_4`.

原始代码中的 `Makefile` 文件使用 `llvm-config` 来设置头文件目录和库搜索目录:

```makefile
# ...
LLVMCONFIG = llvm-config
# ...
CPPFLAGS = `$(LLVMCONFIG) --cppflags` -std=c++14
LDFLAGS = `$(LLVMCONFIG) --ldflags` -lz -lncurses
# ...
```

关于 `llvm-config` 的更多资料,参考[官方文档](https://llvm.org/docs/CommandGuide/llvm-config.html).总而言之,这要求我们将 `LLVM 9.0.1_4` 的 `bin` 目录放到 `PATH` 路径最开头,以便 `make` 能找到合适的 `llvm-config`.

设置好上述的一切后,在我的系统上, `make compile` 会输出以下内容:

```scss
bison -d -o parser.cpp parser.y
clang++ -c `llvm-config --cppflags` -std=c++14 -o parser.o parser.cpp
clang++ -c `llvm-config --cppflags` -std=c++14 -o codegen.o codegen.cpp
clang++ -c `llvm-config --cppflags` -std=c++14 -o main.o main.cpp
flex -o tokens.cpp tokens.l parser.hpp
clang++ -c `llvm-config --cppflags` -std=c++14 -o tokens.o tokens.cpp
clang++ -c `llvm-config --cppflags` -std=c++14 -o corefn.o corefn.cpp
clang++ -c `llvm-config --cppflags` -std=c++14 -o native.o native.cpp
clang++ -o compiler parser.o codegen.o main.o tokens.o corefn.o native.o  `llvm-config --libs` `llvm-config --ldflags` -lz -lncurses
./compiler
Generating code...
Generating code for 18NExternDeclaration
Generating code for 20NFunctionDeclaration
Creating variable declaration int a
Generating code for 20NVariableDeclaration
Creating variable declaration int x
Creating assignment for x
Creating binary operation 276
Creating identifier reference: a
Creating integer: 5
Generating code for 16NReturnStatement
Generating return code for 15NBinaryOperator
Creating binary operation 274
Creating identifier reference: x
Creating integer: 3
Creating block
Creating function: do_math
Generating code for 20NExpressionStatement
Generating code for 11NMethodCall
Creating integer: 13
Creating method call: do_math
Creating method call: echo
Generating code for 20NExpressionStatement
Generating code for 11NMethodCall
Creating integer: 12
Creating method call: do_math
Creating method call: echo
Generating code for 20NExpressionStatement
Generating code for 11NMethodCall
Creating integer: 10
Creating method call: printi
Creating block
; ModuleID = 'main'
source_filename = "main"

@.str = private constant [4 x i8] c"%d\0A\00"

declare i32 @printf(i8*, ...)

define internal void @echo(i64 %toPrint) {
entry:
  %0 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i64 %toPrint)
  ret void
}

define i32 @main() {
entry:
  %0 = call i64 @do_math(i64 13)
  call void @echo(i64 %0)
  %1 = call i64 @do_math(i64 12)
  call void @echo(i64 %1)
  call void @printi(i64 10)
  ret i32 0
}

declare void @printi(i64)

define internal i64 @do_math(i64 %a1) {
entry:
  %a = alloca i64
  store i64 %a1, i64* %a
  %x = alloca i64
  %0 = load i64, i64* %a
  %1 = mul i64 %0, 5
  store i64 %1, i64* %x
  %2 = load i64, i64* %x
  %3 = add i64 %2, 3
  ret i64 %3
}
Code is generated.
sleep 1
llc test/example.bc -o test/example.S
sleep 1
clang++ native.cpp test/example.S -o test/example.native
```

说明编译成功, `example.native` 已经是一个可以在我的系统上运行的 `ELF` 文件:

```sh
$ file test/example.native
test/example.native: Mach-O 64-bit executable x86_64
```

运行它,输出的结果为:

```sh
$ ./test/example.native
68
63
10
```


#### 使用cmake构建项目 {#使用cmake构建项目}

关于 `cmake` 的基本概念,请参考官方文档.

`cmake` 相比于手写 `Makefile` 有以下优点:

1.  跨平台支持: `cmake` 可以生成不同平台下的 `Makefile` 或IDE项目,可以方便地在不同的操作系统上编译,构建和安装.
2.  自动依赖管理: `cmake` 能够自动地管理项目中的依赖,包括库文件和头文件等,减少了手动编写 `Makefile` 的繁琐过程.
3.  更简洁的语法:相比于 `Makefile` , `cmake` 的语法更为简洁,易于理解和维护.
4.  多配置支持: `cmake` 支持多配置构建,可以在一个项目中同时支持Debug和Release等多种构建模式.
5.  IDE集成支持: `cmake` 可以生成各种主流IDE所需的项目文件,如Visual Studio,Xcode,Clion等,方便开发者在自己喜欢的IDE中开发项目.

总的来说, `cmake` 可以帮助开发者更轻松地管理和构建项目,提高开发效率,减少出错概率,具有良好的可移植性和可扩展性.

使用 `cmake` 的方式是为项目编写一个 `CMakeLists.txt` 文件.

以下是一个可以编译本项目的 `CMakeLists.txt` 文件内容:

```cmake
cmake_minimum_required(VERSION 3.10)
project(toy-compiler)

set(CMAKE_CXX_STANDARD 14)

file(GLOB CPPS ${CMAKE_SOURCE_DIR}/*.cpp)

# Find Flex and Bison packages
find_package(FLEX REQUIRED)
find_package(BISON REQUIRED)

# Define the Flex and Bison input and output files
flex_target(Scanner tokens.l ${CMAKE_SOURCE_DIR}/tokens.cpp)
bison_target(Parser parser.y ${CMAKE_SOURCE_DIR}/parser.cpp)
add_flex_bison_dependency(Scanner Parser)

# Use customized LLVM
find_package(Clang REQUIRED CONFIG HINTS ${LLVM_DIR} NO_DEFAULT_PATH)

include_directories(${LLVM_INCLUDE_DIRS} ${CLANG_INCLUDE_DIRS} SYSTEM)
link_directories(${LLVM_LIBRARY_DIRS})

# Add the executable
add_executable(compiler
        ${CPPS}
        ${BISON_Parser_OUTPUTS}
        ${FLEX_Scanner_OUTPUTS}
        )

target_link_libraries(compiler
        LLVMX86AsmParser
        LLVMX86CodeGen
        LLVMAsmParser
        LLVMBitWriter
        LLVMRuntimeDyld
        LLVMExecutionEngine
        LLVMMCJIT
        z
        ncurses
        )
```

`cmake` 提供了一些宏来方便 `flex` 和 `bison` 的使用,这些宏包括: `flex_target`, `bison_target`, `add_flex_bison_dependency` 等,具体情况请参考官方文档.

因为在编写编译器的过程中用到了 `LLVM` 的许多功能,自然要把 `LLVM` 的库找到,并将他们链接到最后的二进制文件中,具体代码如下:

```scss
# Use customized LLVM
find_package(Clang REQUIRED CONFIG HINTS ${LLVM_DIR} NO_DEFAULT_PATH)

include_directories(${LLVM_INCLUDE_DIRS} ${CLANG_INCLUDE_DIRS} SYSTEM)
link_directories(${LLVM_LIBRARY_DIRS})
```

`find_package` 用来从 `${LLVM_DIR}` 这个路径中寻找 `Clang` 相关的库路径,头文件路径等, `include_directories` 用来将找到的库路径,头文件路径添加到搜索路径中,使得在编译和链接时能找到相应的文件.

`target_link_libraries` 用于指定生成的二进制程序需要链接上哪些外部库.

给项目中添加了上述 `CMakeLists.txt` 文件后,项目可以正常编译运行.


#### find_package {#find-package}

`find_package()` 是 `cmake` 中的一个命令,用于在系统上查找已安装的软件包,并设置相关变量.这个命令主要用于在
 `cmake` 构建系统中引入第三方库。

当使用 `find_package()` 命令查找软件包时, `cmake` 会在系统路径下查找该软件包，并设置相关变量,例如该软件包
的头文件路径,库文件路径,链接库等信息.一旦成功找到软件包,就可以将其与项目链接起来,使您的项目能够使用该软件
包提供的功能.

通常，使用 `find_package()` 命令需要执行以下步骤：

1.  在 `CMakeLists.txt` 文件中加入 `find_package()` 命令，例如： `find_package(PackageName REQUIRED)`,这里

的 `PackageName` 是要查找的软件包名称,如果该软件包不存在或未安装, `cmake` 将会在输出中报告错误.如果软件包
存在, `cmake` 会设置相关变量,例如包含路径,库文件路径等

1.  在 `CMakeLists.txt` 文件中使用 `include_directories()` 命令或 `target_include_directories()` 命令将包&gt;含路径添加到项目中
2.  使用 `add_library()` 或 `add_executable()` 命令将源文件与该软件包链接起来,例如:
    ```nil
    add_executable(MyApp main.cpp)
    target_link_libraries(MyApp PackageName)
    ```
    这里的 `PackageName` 是要使用的软件包名称

默认情况下, `find_package` 会从以下几个地方查找包：

1.  `CMAKE_PREFIX_PATH` :这是一个用分号分隔的路径列表, `cmake` 会在这些路径中查找安装的包
2.  环境变量:如果在 `CMAKE_PREFIX_PATH` 中找不到包, `cmake` 会检查环境变量，例如在 `linux` 上, `cmake` 会&gt;检查 `PKG_CONFIG_PATH` 变量
3.  `cmake` 内置模块: `cmake` 附带了许多内置模块,用于查找常见的包,比如我们上面使用的 `flex` 和 `bison`
4.  模块路径: `cmake` 允许开发人员编写自己的模块并在 `cmake` 脚本中使用它们.如果包不在 `CMAKE_PREFIX_PATH` 或内置模块中,则 `cmake` 会查找用户定义的模块路径

`find_package` 会设置一些变量,以便用户在 `CMakeLists.txt` 文件中使用.这些变量包括:

1.  `<PackageName>_FOUND`:用于指示是否找到了包
2.  `<PackageName>_INCLUDE_DIRS`:用于包含头文件的路径
3.  `<PackageName>_LIBRARIES`:用于链接库的路径
4.  `<PackageName>_DEFINITIONS`:用于包含预定义宏的定义
5.  `<PackageName>_VERSION`:包的版本号

这些变量可以在 `CMakeLists.txt` 文件中使用,以便正确地包含头文件,链接库等.


#### One More Thing {#one-more-thing}

<!--list-separator-->

-  为什么不需要find_package(LLVM...)?

    在使用 `cmake` 构建项目时,我发现了一件奇怪的事情: `CMakeLists.txt` 文件中,只对 `Clang` 进行了 `find_package`,并没有对 `LLVM` 进行 `find_package`,在最后链接的时候,竟然能链接 `LLVM` 的库.

    我猜测是在 `find_package(Clang...)` 时, `Clang` 去执行了 `find_package(LLVM...)`,查看 `ClangConfig.cmake` 文件后,果然是这样.在 `${LLVM_DIR}/lib/cmake/clang/ClangConfig.cmake` 文件中,有一行 `find_package(LLVM REQUIRED CONFIG HINTS "${CLANG_INSTALL_PREFIX}/lib/cmake/llvm")`.


### 适配 `LLVM 16.0.0` {#适配-llvm-16-dot-0-dot-0}


#### 主要是CPP语法+LLVM接口 {#主要是cpp语法-plus-llvm接口}

原始项目中,使用的是C++ 14的语法,但是 `LLVM 16.0.0` 使用了更新的C++标准,经过测试,C++ 20是可以编译通过的,所以我选择了C++20.

原始版本使用的是 `LLVM 9.0.1_4` 的接口,迁移到 `LLVM 16.0.0` 后,只需要将编译无法通过的接口适配一下既可.


#### 借鉴[Kaleidoscope](https://llvm.org/docs/tutorial/index.html)生成 `.o` 文件 {#借鉴-kaleidoscope-生成-dot-o-文件}

原始项目产生的编译器会将源文件编译为 `BitCode` 文件,接着要么使用 `llc` 和 `clang` 将 `BitCode` 编译为可运行的二进制程序,要么使用 `lli` 解释执行 `BitCode` 文件.借鉴[Kaleidoscope第八章](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl08.html),我们其实可以直接生成 `.o` 文件,后续再想办法将 `.o` 文件链接为可执行的二进制程序.

这一部分的代码直接参考了 `Kaleidoscope`,改动不是特别大.


### 使用LLVM的一些Pass对代码进行优化 {#使用llvm的一些pass对代码进行优化}

这一部分参考了[Kaleidoscope第四章](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl04.html)的 `4.3. LLVM Optimization Passes` 一节,只需要在代码生成之后,使用 `Pass` 对代码进行优化,非常方便,同样代码改动不大.


### 后续计划 {#后续计划}


#### 完善语法 {#完善语法}

1.  这个小项目目前还不支持分支等很多语法,需要陆续添加
2.  需要支持类型系统
3.  借鉴[Kaleidoscope第四章](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl04.html) `4.4. Adding a JIT Compiler` 一节的最后一部分,使用现有的C库扩充函数
4.  加入一些函数式编程的语法
5.  优化内存管理


#### 完善编译器功能 {#完善编译器功能}

1.  ~~编译器目前只支持生成 `.o` 文件,之后需要支持生成可执行的二进制文件,能跨平台更好了~~
    事实上,这个功能是由链接器提供的,不过一般的编译器前端(本文也只是通过 `flex` 和 `bison` 实现了一个编译器前端)会调用 `LLVM lld` 的接口,来将生成的目标文件链接为可执行文件.

    例如,只需要将如下代码加在生成目标文件的代码之后,就可以将目标文件和标准库一起链接为一个二进制程序:
    ```cpp
    const char *must_args[] = {
      "-demangle",
      "-lto_library",
      "/Applications/Xcode.app/Contents/Developer/Toolchains/"
      "XcodeDefault.xctoolchain/usr/lib/libLTO.dylib",
      "-no_deduplicate",
      "-dynamic",
      "-arch",
      "x86_64",
      "-platform_version",
      "macos",
      "13.0.0",
      "13.1",
      "-syslibroot",
      "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/"
      "Developer/SDKs/MacOSX.sdk",
      "-o",
      "a.out",
      "-L/usr/local/lib",
      "-lSystem",
      "/Applications/Xcode.app/Contents/Developer/Toolchains/"
      "XcodeDefault.xctoolchain/usr/lib/clang/14.0.0/lib/darwin/"
      "libclang_rt.osx.a"
    };
    std::vector<const char *> args;
    args.push_back("placeholder");
    for (auto arg: must_args) {
      args.push_back(arg);
     }
    args.push_back("path/to/file1");
    args.push_back("path/to/file2");
    // ...
    return lld::macho::link(args, llvm::outs(), llvm::errs(), false, false);
    ```
    **如何获取参数列表**:
    调用 `lld::ARCH::link` 最重要的步骤就是获取参数列表 `args`,我是通过实现了一个链接器桩来获取到这些参数,当然这些参数也不确保对所有的代码都适用.链接器桩代码如下:
    ```cpp
    #include <stdio.h>

    int main(int argc, char **argv) {
      for (int i = 0; i < argc; i++) {
        printf("%s\n", argv[i]);
      }

      return 0;
    }
    ```
    这段代码只是把所有的参数打印出来,将这段代码编译为 `myld`,接着在使用 `clang` 或者 `gcc` 编译代码时,只要添加 `-fuse-ld=/path/to/myld` 参数,就能将编译器传递给链接器的所有参数打印出来.

    以上的参数列表只针对 `macOS` 平台,在不同平台上编译不同文件时, `clang` 或者 `gcc` 生成的参数列表可能都是不一样的,具体是如何获取这些参数的,就得看 `clang` 的实现了.

    关于跨平台的写法,可以参考 `LLVM源码` 中的 `lld/tools/lld/lld.cpp` 文件,这个工具的作用就是调用 `LLVM` 的链接器接口实现 `lld` 这个链接器.

2.  ~~使用 `LLVM` 的 `JIT` 加入解释器的功能~~ 因为还有更重要的事情,这条搁浅了
