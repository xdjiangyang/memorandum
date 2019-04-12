# 前言
## 概述
Hyperscan是一种正则表达式引擎，旨在提供高性能，同时匹配多个表达式的能力以及扫描操作的灵活性。

模式被提供给编译接口，该接口生成不可变模式数据库。然后，扫描接口可用于扫描给定模式的目标数据缓冲区，从该数据缓冲区返回任何匹配结果。Hyperscan还提供流模式，其中检测跨越流中的若干块的匹配。

本文档旨在促进Hyperscan库与现有或新应用程序的代码级集成。

简介是Hyperscan库的简短概述，更详细介绍了后续章节中提供的Hyperscan API：编译模式和 扫描模式。

性能注意事项提供了可能影响Hyperscan集成性能的各种因素的详细信息。

API参考：常量和 API参考：文件提供了Hyperscan应用程序编程接口（API）的详细摘要。

## 受众
本指南面向有兴趣将Hyperscan集成到应用程序中的开发人员。有关构建Hyperscan库的信息，请参阅“快速入门指南”。

## 约定
固定宽度字体中的文本是指代码元素，例如类型名称;功能或方法名称
彩色固定宽度字体中的文本是指正则表达式或正则表达式的一部分

---------------

# 介绍
Hyperscan是一个软件正则表达式匹配引擎，设计时考虑到了高性能和灵活性。它被实现为一个公开简单的C API的库。

Hyperscan API本身由两个主要组件组成：

## 编译
这些函数采用一组正则表达式以及标识符和选项标志，并将它们编译为可由Hyperscan扫描API使用的不可变数据库。此编译过程执行大量分析和优化工作，以便构建将有效匹配给定表达式的数据库。

如果由于任何原因（例如使用不受支持的表达式构造或资源限制溢出）无法将模式构建到数据库中，则模式编译器将返回错误。

编译的数据库可以被序列化和重新定位，以便它们可以存储到磁盘或在主机之间移动。它们还可以针对特定平台功能（例如，使用英特尔®高级矢量扩展指令2（英特尔®AVX2）指令）。

有关详细信息，请参阅编译模式。

## 扫描
创建Hyperscan数据库后，它可用于扫描内存中的数据。Hyperscan提供多种扫描模式，具体取决于要扫描的数据是否可用作单个连续块，是否同时分布在内存中的多个块中，或者是否将其作为一系列块进行扫描。流。

匹配通过用户提供的回调函数传递给应用程序，该函数对于每个匹配同步调用。

对于给定的数据库，Hyperscan提供了几种保证：

- 除了两个固定大小的分配外，运行时不会发生内存分配，对于性能关键型应用程序，这两个分配都应该提前完成：
  - 划痕空间：扫描时用于内部数据的临时存储器。暂存空间中的结构不会在单次扫描调用结束后持续存在。
  - 流状态：仅在流模式下，需要一些状态空间来存储在每个流的扫描调用之间持续存在的数据。这允许Hyperscan跟踪跨越多个数据块的匹配。

- 给定数据库所需的临时空间和流状态（以流模式）的大小是固定的，并在数据库编译时确定。这意味着应用程序的内存需求是提前知道的，如果出于性能原因需要，可以预先分配这些结构。

- 可以针对任何输入扫描Hyperscan编译器成功编译的任何模式。运行时没有内部资源限制或其他限制可能导致扫描调用返回错误。

有关更多详细信息，请参阅扫描模式。

## 工具
一些用于测试和基准测试Hyperscan的实用程序包含在库中。有关更多信息，请参见工具

## 示例代码
examples/Hyperscan发行版的子目录中提供了一些演示Hyperscan API使用的简单示例代码。

-----------------

# 入门
## 快速开始
1. 克隆Hyperscan

        cd  < where - you - want - hyperscan - source > 
        git  clone  git ：// github 。com / intel / hyperscan

2. 配置Hyperscan

	确保您具有正确的依赖项，然后：

        cd  < where - you - want - to - build - hyperscan > 
        mkdir  < build - dir > 
        cd  < build - dir > 
        cmake  [ - G  < generator > ]  [ options ]  < hyperscan - source - path >
	已知的工作发电机：
	- Unix Makefiles - make兼容的makefile（默认在Linux / FreeBSD / Mac OS X上）
	- Ninja- 忍者构建文件。
	- Visual Studio 15 2017 - Visual Studio项目

	可能有效的生成器包括：
	- Xcode - OS X Xcode项目。

3. 构建Hyperscan

	取决于使用的发电机：
	- cmake --build . - 将构建一切
	- make -j&lt;jobs&gt; - 并行使用makefile
	- ninja - 使用Ninja构建
	- MsBuild.exe - 使用Visual Studio MsBuild

4. 检查Hyperscan

	运行Hyperscan单元测试：

	- bin / unit - hyperscan

## 要求
### 硬件
Hyperscan将在64位（英特尔®64架构）和32位（IA-32架构）模式的x86处理器上运行。
Hyperscan是一个高性能的软件库，它利用了最新的英特尔架构优势。至少需要支持Supplemental Streaming SIMD Extensions 3（SSSE3），这应该适用于任何现代x86处理器。

此外，Hyperscan可以利用：

- 英特尔流式SIMD扩展4.2（SSE4.2）
- POPCNT指令
- 位操作指令（BMI，BMI2）
- 英特尔高级矢量扩展2（英特尔AVX2）

如果存在的话

这些可以在库编译时确定，请参阅目标体系结构。

### 软件
作为一个软件库，Hyperscan不会强加任何特定的运行时软件要求，但是为了构建Hyperscan库，我们需要一个现代的C和C ++编译器 - 特别是，Hyperscan需要C99和C ++ 11编译器支持。支持的编译器是：

- GCC，v4.8.1或更高版本
- Clang，v3.4或更高版本（使用libstdc ++或libc ++）
- 英特尔C ++编译器v15或更高版本
- Visual C ++ 2017构建工具

已知Hyperscan可以使用的操作系统示例包括：

Linux：
- Ubuntu 14.04 LTS或更新版本
- RedHat / CentOS 7或更新版本

FreeBSD的：
- 10.0或更新

Windows：
- 8或更新

Mac OS X：
- 10.8或更新，使用XCode / Clang

Hyperscan 可以在其他平台上编译和运行，但不能保证。我们目前使用英特尔C ++编译器或Visual Studio 2017对Windows进行实验性支持。

此外，编译Hyperscan库需要以下软件：

|依赖	|版本	|笔记|
|----|----|----|
|CMake	|> = 2.8.11	||
|Ragel	|6.9||
|Python	|2.7||
|Boost	|> = 1.57|需要头文件|
|PCAP	|> = 0.8|可选：仅用于示例代码|

大多数这些依赖项可以由构建系统上的包管理器提供（例如Debian / Ubuntu / RedHat包，FreeBSD端口等）。但是，请确保存在正确的版本。至于Windows，为了拥有Ragel，你可以使用Cygwin从源代码构建它。

### Boost头文件
编译Hyperscan取决于最新版本的Boost C ++头库。如果Boost库以常规路径安装在构建机器上，CMake将找到它们。如果未安装Boost库，则可以在CMake配置步骤期间使用BOOST_ROOT变量（如下所述）指定Boost源树的位置。

另一种方法是将boost子目录的副本（或符号链接）放入<hyperscan-source-path>/include/boost。

例如：对于Boost-1.59.0版本：

	ln  - s  boost_1_59_0 / boost  < hyperscan - source - path > / include / boost
由于Hyperscan使用Boost的仅头部分，因此不必编译Boost库。

### CMake配置
调用CMake时，它使用给定的选项生成构建文件。选项以表格形式传递给CMake 。CMake的常见选项包括：-D&lt;variable name&gt;=&lt;value&gt;

|变量	|描述 |
|---|---|
|CMAKE_C_COMPILER	|C编译器使用。默认为/ usr / bin / cc。|
|CMAKE_CXX_COMPILER	|使用C ++编译器。默认为/ usr / bin / c ++。|
|CMAKE_INSTALL_PREFIX	|安装install目标目录|
|CMAKE_BUILD_TYPE	|定义要生成的构建类型。有效选项包括Debug，Release，RelWithDebInfo和MinSizeRel。默认值为RelWithDebInfo。|
|BUILD_SHARED_LIBS	|将Hyperscan构建为共享库而不是默认静态库。|
|BUILD_STATIC_AND_SHARED	|构建静态和共享Hyperscan库。默认关闭。|
|BOOST_ROOT|	Boost源码树的位置。|
|DEBUG_OUTPUT	|启用非常详细的调试输出。默认关闭。|
|FAT_RUNTIME	|构建胖运行时。在Linux上默认为true，在其他地方不可用。|

例如，要生成Debug构建：

    cd  < build - dir > 
    cmake  - DCMAKE_BUILD_TYPE = Debug  < hyperscan - source - path >

### 构建类型
CMake根据构建类型确定构建的许多功能。Hyperscan默认为RelWithDebInfo，即“随调试信息发布”。这是一个性能优化的构建，没有运行时断言但启用了调试符号。

其他类型的构建是：
- Release：如上所述，但没有调试符号
- MinSizeRel：剥离版本
- Debug：在开发Hyperscan时使用。包括运行时断言（对运行时性能有很大影响），还将启用一些其他构建功能，如构建内部单元测试。

### 目标架构
除非使用胖运行时，否则默认情况下将编译Hyperscan以定位用于编译的机器的处理器的指令集。这是通过使用来完成的-march=native。这样的结果意味着如果在支持的指令子集中它们不同，则在一台机器上构建的库可能无法在不同的机器上工作。

要覆盖使用-march=native的编译器，设置相应的标志CFLAGS，并CXXFLAGS调用CMake的，或者之前的环境变量CMAKE_C_FLAGS，并CMAKE_CXX_FLAGS在CMake的命令行。例如，要将指令子集设置为SSE4.2使用GCC 4.8：

    cmake  - DCMAKE_C_FLAGS = “ - march = corei7” \
       - DCMAKE_CXX_FLAGS = “  - march = corei7” <hyperscan - source - path>
有关更多信息，请参阅指令集专业化。

### 胖运行时
Hyperscan v4.4中引入的一项功能是Hyperscan库能够为主机处理器分配最合适的运行时代码。此功能称为“胖运行时”，因为单个Hyperscan库包含不同指令集的运行时代码的多个副本。

    注意
    胖运行时功能仅适用于Linux。Hyperscan的发布版本默认在支持的情况下启用胖运行时。

使用胖运行时构建库时，Hyperscan运行时代码将针对这些不同的指令集进行多次编译，并且这些编译对象将组合到一个库中。对此库构建用户应用程序的方式没有任何变化。

执行应用程序时，会为运行它的计算机选择正确的运行时版本。这是通过CPUID检查是否存在指令集来完成的，然后解析间接函数，以便使用每个API函数的正确版本。对函数调用性能没有影响，因为在加载二进制文件时，ELF加载程序会执行一次检查和解析。

如果在没有的情况下在x86系统上使用Hyperscan库SSSE3，则运行时API函数将解析为返回HS_ARCH_ERROR 而不是可能执行非法指令的函数。hs_valid_platform()应用程序编写者可以使用API函数 来确定Hyperscan是否支持当前平台。

在此版本中，构建的运行时变体以及所需的CPU功能如下：

|种类	|需要的CPU功能标志	|gcc arch flag|
|----|----|----|
|Core 2	|SSSE3	|-march=core2|
|Core i7	|SSE4_2 和 POPCNT	|-march=corei7|
|AVX 2	|AVX2	|-march=core-avx2|
|AVX 512	|AVX512BW	|-march=skylake-avx512|
    注意
    Hyperscan v4.5增加了对AVX-512指令的支持 - 特别AVX-512BW是在英特尔“Skylake”Xeon处理器上引入的 指令集 - 但是在胖运行时版本中默认情况下不启用AVX-512运行时变体，因为并非所有工具链都支持AVX -512条指令集。要构建AVX-512运行时，BUILD_AVX512必须在配置期间手动启用CMake变量 。例如：

    cmake  - DBUILD_AVX512 = 在 <...>上

由于胖运行时需要编译器，libc和binutils支持，此时它仅对Linux版本启用，其中编译器支持 间接函数“ifunc”函数属性。

此属性应该适用于所有受支持的GCC版本以及最新版本的Clang和ICC。目前，非Linux系统上没有此功能的操作系统支持。

-----------------

# 编译模式
## 构建数据库
Hyperscan编译器API接受正则表达式并将它们转换为已编译的模式数据库，然后可用于扫描数据。

API提供了三个将正则表达式编译到数据库中的函数：

1. hs_compile()：将单个表达式编译到模式数据库中。
2. hs_compile_multi()：将一组表达式编译到一个模式数据库中。扫描时将同时扫描所有提供的模式，并在匹配时返回用户提供的标识符。
3. hs_compile_ext_multi()：如上所述编译表达式数组，但允许为每个表达式指定扩展参数。
编译允许Hyperscan库分析给定的模式，并预先确定如何以优化的方式扫描这些模式，这在运行时计算成本太高。

在编译表达式时，需要决定是否要在流，块或向量模式中使用生成的编译模式：

- 流模式：要扫描的目标数据是连续流，并非所有流都可以一次获得; 按顺序扫描数据块，匹配可以跨越流中的多个块。在流模式下，每个流需要一块内存来在扫描调用之间存储其状态。
- 块模式：目标数据是一个离散的连续块，可以在一次调用中扫描，不需要保留状态。
- 向量模式：目标数据由一次可用的非连续块列表组成。对于块模式，不需要保留状态。
要编译要在流模式下使用的模式，必须将mode参数 hs_compile()设置为HS_MODE_STREAM; 同样，块模式需要使用HS_MODE_BLOCK和向量模式需要使用HS_MODE_VECTORED。为一种模式（流，块或向量）编译的模式数据库只能在该模式下使用。用于生成编译模式数据库的Hyperscan版本必须与用于扫描的Hyperscan版本匹配。

Hyperscan支持在特定CPU平台上定位数据库; 有关详细信息，请参阅指令集专业化

## 模式支持
Hyperscan支持PCRE库（“libpcre”）使用的模式语法，如< http://www.pcre.org/ >所述。但是，并不支持libpcre中可用的所有构造。使用不受支持的结构将导致编译错误。

用于验证Hyperscan对此语法的解释的PCRE版本为8.41或更高版本。

### 支持的构造
Hyperscan支持以下正则表达式结构：

- 文字字符和字符串，所有libpcre引用和字符转义。

- 字符类如.（点），[abc]和 [^abc]，以及预定义的字符类\s， \d，\w，\v，和\h及其否定同行（\S，\D，\W， \V，和\H）。

- POSIX命名的字符类[[:xxx:]]和否定命名的字符类[[:^xxx:]]。

- Unicode字符属性，诸如\p{L}，\P{Sc}， \p{Greek}。

- 量词：

	- 当应用于任意支持的子表达式时?，支持*和+支持量词。
	- 有限重复限定符（如{n},, {m,n}） {n,}受限制。
		- 对于任意重复的子模式：n和m应该是小的或无限的，例如(a|b}{4}，(ab?c?d){4,10}或 (ab(cd)*){6,}。
		- 对于单字符宽度子模式（如[^\a]or .或）x，几乎所有重复计数都受支持，除非重复次数非常大（最大边界大于32767）。对于大的有界重复，流状态可能非常大，例如 a.{2000}b。注意：如果在模式的开头或结尾处，这样的子模式可能会相当便宜，特别是如果该模式的 HS_FLAG_SINGLEMATCH标志是打开的话。
	- 延迟修饰符（?附加到另一个量词，例如 \w+?）被支持但被忽略（因为Hyperscan报告所有匹配）。
- 括号，包括命名和未命名的捕获和非捕获表单。但是，忽略捕获。

- 与|符号交替，如foo|bar。

- 锚^，$和\A，\Z和 \z。

- 选项修饰符：

	这些允许为子模式打开（使用(?&lt;option&gt;)）和关闭（使用(?-&lt;option&gt;)）行为。支持的选项是：

	- i：不区分大小写的匹配，按照 HS_FLAG_CASELESS。
	- m：多行匹配，按照HS_FLAG_MULTILINE。
	- s：解释.为“任何角色”，按照 HS_FLAG_DOTALL。
	- x：扩展语法，它将忽略模式中的大多数空格以与libpcre的PCRE_EXTENDED选项兼容。

    例如，表达式foo(?i)bar(?-i)baz将仅为匹配的bar部分打开不区分大小写的匹配。

- \b和\B零宽度断言（分别为字边界和“不字边界”，）。

- 语法注释。(?# comment)

- (*UTF8)和(*UCP)在模式的开始控制动词，用于启用UTF-8和UCP模式。

        注意
        具有任意表达式的重复计数（例如([a-z]|bc*d|xy?z){1000,5000}）的有界重复量词将在模式编译时导致“模式太大”错误。

        注意
        此时，并非所有模式都可以使用HS_FLAG_SOM_LEFTMOST标志成功编译 ，这使得每个模式支持 匹配开始。支持此标志的模式是可以使用Hyperscan成功编译的模式子集; 值得注意的是，许多有限的重复形式可以使用Hyperscan编译而不启用匹配开始标志，无法在启用标志的情况下进行编译。

### 不支持的构造
Hyperscan不支持以下正则表达式结构：

- 反向引用和捕获子表达式。
- 任意零宽度断言。
- 子例程引用和递归模式。
- 条件模式。
- 回溯控制动词。
- \C“单字节”指令（它打破UTF-8序列）。
- 新线\R匹配。
- \K匹配重置指令的开始。
- 标注和嵌入式代码。
- 原子分组和占有量词。

## 语义
虽然Hyperscan遵循libpcre语法，但它提供了不同的语义。libpcre语义的主要偏差是由流媒体和多个同时模式匹配的要求所驱动的。

libpcre语义的主要偏离是：

1. **多模式匹配**：Hyperscan允许同时报告多个模式的匹配。这不等同于|在libpcre中分离模式，libpcre从左到右评估交替。
2. **缺乏排序**：Hyperscan产生的多个匹配不保证有序，尽管它们总是落在当前扫描的范围内。
3. **仅限结束偏移**：Hyperscan的默认行为仅用于报告匹配的结束偏移。可以在模式编译时使用per-expression标志启用报告起始偏移量。有关详细信息，请参见匹配开始。
4. **“所有匹配”报道**：扫描/foo.*bar/反对 fooxyzbarbar将Hyperscan返回两个匹配结果-在对应于两端的点fooxyzbar和fooxyzbarbar。相比之下，libpcre语义默认只报告一个匹配fooxyzbarbar （贪婪语义），或者，如果打开非贪婪语义，则匹配一个匹配 fooxyzbar。这意味着在Hyperscan中，贪婪和非贪婪语义之间的切换是无操作的。

支持libpcre量词语义，同时在发生流式匹配时准确报告它们是不可能的。例如，考虑上面的模式/foo.*bar/，在流模式下，对照以下流（按顺序扫描三个块）：

|块1	|块2	|第3块|
|---|---|---|
|fooxyzbar	|baz	|qbar|

由于.*模式中的重复是libpcre中的贪婪重复，因此它必须尽可能匹配而不会导致模式的其余部分失败。但是，在流模式下，这将需要知道正在扫描的当前块之外的流中的数据。

在这个例子中，第一个块中偏移9处的匹配只是正确匹配（在libpcre语义下），如果bar在后续块中没有- 如在块3中 - 这将构成模式的更好匹配。

### 开始匹配
在标准操作中，Hyperscan仅在调用匹配回调时提供匹配的结束偏移量。如果HS_FLAG_SOM_LEFTMOST为特定模式指定了标志，则返回相同的匹配集，但每个匹配也将提供与其结束偏移相对应的最左边可能的起始偏移量。

使用SOM标志需要进行一些权衡和限制：

- 减少模式支持：对于许多模式，跟踪SOM很复杂，并且可能导致Hyperscan无法编译具有“模式太大”错误的模式，即使在正常操作中支持该模式也是如此。
- 增加的流状态：在扫描时，需要状态空间来跟踪潜在的SOM偏移，并且必须以流模式将其存储在持久流状态中。因此，SOM通常会增加匹配模式所需的流状态。
- 性能开销：同样，跟踪SOM通常会产生性能成本。
- 不兼容的功能：其他一些Hyperscan模式标志（例如 HS_FLAG_SINGLEMATCH和HS_FLAG_PREFILTER）不能与SOM结合使用。同时指定它们 HS_FLAG_SOM_LEFTMOST将导致编译错误。

在流模式下，SOM提供的精度可以通过SOM水平标志来控制。这些指示Hyperscan在结束偏移的特定距离内提供准确的SOM信息，HS_OFFSET_PAST_HORIZON否则返回特殊的起始偏移。指定小型或中型SOM层次通常会减少给定数据库所需的流状态。

    注意
    在流模式中，为匹配返回的起始偏移可以指在扫描当前块之前的流中的点。Hyperscan不提供访问早期块的工具; 如果调用应用程序需要检查历史数据，那么它必须自己存储它。

### 扩展参数
在某些情况下，需要对使用正则表达式语法轻松指定的模式匹配行为进行更多控制。对于这些场景，Hyperscan提供的hs_compile_ext_multi()功能允许在每个模式的基础上设置一组“扩展参数”。

使用hs_expr_ext_t结构指定扩展参数，该结构提供以下字段：

- flags：标志管理结构中的哪些其他字段。
- min_offset：数据流中此表达式应成功匹配的最小结束偏移量。
- max_offset：数据流中此表达式应成功匹配的最大结束偏移量。
- min_length：成功匹配此表达式所需的最小匹配长度（从开始到结束）。
- edit_distance：在给定的Levenshtein距离内匹配此表达式。
- hamming_distance：在给定的汉明距离内匹配此表达式。

这些参数允许模式生成的匹配集在编译时受到约束（而不是依赖于应用程序在运行时处理不需要的匹配），或允许大致匹配模式（在给定的编辑距离内）以产生更多匹配。

例如，该图案/foo.*bar/当给定一个min_offset的10和max_offset15时对扫描不会产生匹配 foobar或foo0123456789bar但会产生对数据流的匹配foo0123bar或foo0123456bar。

类似地，该图案/foobar/当给予edit_distance的2时对扫描会产生匹配foobar，f00bar，fooba， fobr，fo_baz，foooobar，和其他任何的位于2编辑距离内（由Levenshtein距离所定义的）。

当相同的图案/foobar/被赋予hamming_distance的图2，当对扫描会产生匹配foobar，boofar， f00bar，和其他任何至多两个字符从原始图案取代。有关更多详细信息，请参阅“ 近似匹配” 部分。

### 预过滤模式
Hyperscan提供了每模式标志，HS_FLAG_PREFILTER可用于实现模式的预过滤器，而不是Hyperscan通常不支持的模式。

此标志指示Hyperscan编译此模式的“近似”版本以用于预过滤应用程序，即使Hyperscan在正常操作中不支持该模式也是如此。

使用此标志时返回的匹配集保证是非预先过滤表达式指定的匹配的超集。

如果模式包含Hyperscan不支持的模式构造（例如零宽度断言，反向引用或条件引用），则这些构造将在内部被更广泛的构造替换，这些构造可能更频繁地匹配。

例如，模式包含反向引用。在预过滤模式中，可以通过将其后引用替换为其引用形成来近似该模式 。/(\w+) again \1/\1/\w+ again \w+/

此外，在预过滤模式中，Hyperscan可以简化在编译时或者出于性能原因（在上面的匹配保证的情况下）返回“模式太大”错误的模式。

通常预期应用程序随后将确认预过滤器与另一个可以为模式提供精确匹配的正则表达式匹配器匹配。

    注意
    HS_FLAG_SOM_LEFTMOST目前不支持将此标志与Start of Match模式结合使用（使用标志），这将导致模式编译错误。

## 指令集专业化
Hyperscan能够利用x86处理器上的几个现代指令集功能来提高扫描性能。

在构建库时选择其中一些功能; 例如，Hyperscan将POPCNT在可用的处理器上使用本机指令，并且库已针对主机架构进行了优化。

    注意
    默认情况下，Hyperscan运行时使用-march=native 编译器标志构建，并且（如果可能）将使用主机的C编译器已知的所有指令。

但是，要使用某些指令集功能，Hyperscan必须构建一个专门的数据库来支持它们。这意味着必须在模式编译时指定目标平台。

Hyperscan编译器API函数都接受一个可选 hs_platform_info_t参数，该参数描述了要构建的数据库的目标平台。如果此参数为NULL，则数据库将以当前主机平台为目标。

该hs_platform_info_t结构有两个领域：

1. tune：这允许应用程序指定有关目标平台的信息，该信息可用于指导编译的优化过程。使用此字段不会限制生成的数据库可以运行的处理器，但可能会影响生成的数据库的性能。
2. cpu_features：这允许应用程序指定可在目标平台上使用的CPU功能的掩码。例如， HS_CPU_FEATURES_AVX2可以指定英特尔®高级矢量扩展指令集2（英特尔®AVX2）指令集支持。如果指定了特定CPU功能的标志，则数据库将不在没有该功能的CPU上使用。

hs_platform_info_t可以使用该hs_populate_platform()函数构建针对当前主机的结构。

有关CPU调整和功能标志的完整列表，请参阅API参考：常量。

## 近似匹配
Hyperscan提供实验近似匹配模式，该模式将匹配给定编辑距离内的模式。精确匹配行为定义如下：

1. **编辑距离**定义为Levenshtein距离。也就是说，考虑了三种可能的编辑类型：插入，删除和替换。可以在维基百科上找到更正式的描述 。
2. **汉明距离**是两条相等长度的弦不同的位置数。也就是说，它是将一个字符串转换为另一个字符串所需的替换次数。使用汉明距离进行近似匹配时，没有插入或删除。可以在维基百科上找到更正式的描述 。
3. **近似匹配**将匹配给定编辑或汉明距离内的所有语料库。也就是说，给定一个模式，近似匹配将匹配任何可以编辑的内容，以达到与原始模式完全匹配的语料库。
4. **匹配的语义**是完全一样的描述语义。

以下是近似匹配的几个示例：

-  使用常规Hyperscan匹配行为时，模式/foo/可以匹配foo。随着编辑距离2内近似匹配，当对扫描模式会产生匹配foo，foooo，f00， f，和其他任何处于原始模式（匹配语料库的编辑距离2以内foo在这种情况下）。
- 模式/foo(bar)+/与编辑距离1将匹配foobarbar， foobarb0r，fooarbar，foobarba，f0obarbar，fobarbar和其他任何处于原始模式匹配语料库的编辑距离1之内（foobarbar在这种情况下）。
- 模式/foob?ar/与编辑距离2匹配fooar， foo，fabar，oar和其他任何处于原始模式匹配语料库的编辑距离2内（fooar在这种情况下）。

目前，存在与近似匹配支持相关的权衡和限制。简而言之，他们在这里：

- 减少模式支持：
	- 对于许多模式，近似匹配很复杂，并且可能导致Hyperscan无法编译具有“模式太大”错误的模式，即使在正常操作中支持该模式也是如此。
	- 另外，一些模式不能近似匹配，因为它们减少到所谓的“空洞”模式（匹配所有模式的模式）。例如，/foo/具有编辑距离3的模式（如果实现）将减少为匹配零长度缓冲区。这样的模式将导致“模式无法近似匹配”编译错误。汉明距离内的近似匹配不会删除符号，因此不会减少为真空模式。
	- 最后，由于定义匹配行为的固有复杂性，近似匹配实现了正则表达式语法的简化子集。近似匹配不支持UTF-8（和其他多字节字符编码）和字边界（即\b，\B 和其他等效构造）。包含不受支持的结构的模式将导致“模式无法近似匹配”编译错误。
	- 当与SOM结合使用近似匹配时，SOM的所有限制也适用。有关详细信息，请参阅匹配开始。
- 增加流状态/字节代码大小要求：由于近似匹配的字节代码本质上比精确匹配更大且更复杂，因此相应的要求也增加。
- 性能开销：类似地，通常存在与近似匹配相关联的性能成本，这是由于增加的匹配复杂性以及由于它将产生更多匹配的事实。

默认情况下，始终禁用近似匹配，并且可以使用扩展参数中描述的扩展参数在每个模式的基础上启用。

## 逻辑组合
对于用户需要依赖于模式组匹配是否存在的行为的情况，Hyperscan支持给定模式集中模式的逻辑组合，具有三个运算符： NOT，AND和OR。

这种组合的逻辑值基于给定偏移处的每个表达式的匹配状态。任何表达式的匹配状态都有一个布尔值：如果表达式尚未匹配则为false;如果表达式已匹配，则为true。特别是，如果在此偏移处引用的表达式为false，则NOT给定偏移处的运算值为true。

例如，表示表达式101在此偏移处尚未匹配。NOT 101

逻辑组合在编译时作为表达式传递给Hyperscan。此组合表达式将在其中一个子表达式匹配且整个表达式的逻辑值为true的每个偏移处引发匹配。

为了说明，这是一个示例组合表达式：

	（（301  OR  302 ） AND  303 ） AND  （304  OR  NOT  305 ）
如果表达式301在偏移10处匹配，则301的逻辑值为真， 而其他模式的值为假。因此，整个组合的价值是 错误的。

然后表达式303在偏移20处匹配。现在301和303的 值为真，而其他模式的值仍为假。在这种情况下，组合的值为true，因此组合表达式在偏移量20处引发匹配。

最后，表达式305在偏移30处具有匹配。现在301,303和305的值为真，而其他模式的值仍为假。在这种情况下，组合的值为false，不会引发匹配。

**使用逻辑组合**

在逻辑组合语法中，表达式被写为中缀表示法，它由操作数，运算符和括号组成。操作数是表达式ID，运算符是!（NOT），&（AND）或|（OR）。例如，上一节中描述的组合将写为：

（（301 | 302）＆303）＆（304 |！305）
在逻辑组合表达式中：

- 运营商的优先级是!> &> |。例如：
	- A&B|C被视为(A&B)|C，
	- A|B&C被视为A|(B&C)，
	- A&!B被视为A&(!B)。
- 允许使用额外的括号。例如：
	- (A)&!(B)是一样的A&!B，
	- (A&B)|C是一样的A&B|C。
- 空格被忽略。

要使用逻辑组合表达式，必须将其与标志一起传递给Hyperscan编译函数（hs_compile_multi()， hs_compile_ext_multi()）之一，该HS_FLAG_COMBINATION标志将模式标识为逻辑组合表达式。逻辑组合表达式中引用的模式必须在与组合表达式相同的模式集中一起编译。

当表达式HS_FLAG_COMBINATION设置了标志时，它将忽略除HS_FLAG_SINGLEMATCH标志和 标志之外的所有其他HS_FLAG_QUIET标志。

Hyperscan将在编译时拒绝逻辑组合表达式，在没有模式匹配时评估为true ; 例如：

    ！101
    ！101 | 102
    ！101！102
    ！（101 102）
在逻辑组合中被称为操作数的模式（例如，上面示例中的301到305）也可以使用该 HS_FLAG_QUIET标志来静默报告那些模式的各个匹配。如果没有此标志，将报告所有匹配（针对单个模式及其逻辑组合）。

当表达式同时设置了HS_FLAG_COMBINATION标志和 HS_FLAG_QUIET标志时，将不会报告此逻辑组合的匹配项。

---------------

# 扫描模式
Hyperscan提供三种不同的扫描模式，每种模式都有自己的扫描功能hs_scan。此外，流模式还具有许多用于管理流状态的其他API函数。

## 处理匹配
当找到匹配项时，所有这些函数都将调用用户提供的回调函数。此功能具有以下签名：

    typedef int ( * match_event_handler)（ unsigned int  id，unsigned long long  from，unsigned long long  to，unsigned int  flags，void  * context ）
的ID参数将被设置为标识符在编译时提供的匹配表达式，并且对参数将被设置到端部的偏移的匹配。如果为模式请求了SOM（请参阅匹配开始），则 from参数将设置为匹配的最左侧可能的起始偏移量。

匹配回调函数具有通过返回非零值来停止扫描的能力。

有关match_event_handler更多信息，请参阅

## 流模式
Hyperscan流运行时API的核心包括打开，扫描和关闭Hyperscan数据流的功能：

- **hs_open_stream()**：分配并初始化一个新的流以进行扫描。
- **hs_scan_stream()**：扫描给定流中的数据块，在检测到匹配时引发匹配。
- **hs_close_stream()**：完成对给定流的扫描（引发在流末尾发生的任何匹配）并释放流状态。调用之后hs_close_stream()，流句柄无效，不应再次用于任何目的。

扫描时在数据中检测到的任何匹配都将通过函数指针回调返回到调用应用程序。

匹配回调函数能够通过返回非零值来暂停当前数据流的扫描。在流模式下，结果是流然后处于不能扫描更多数据的状态，并且hs_scan_stream()对该流的任何后续调用将立即返回HS_SCAN_TERMINATED。呼叫者仍然必须调用hs_close_stream()以完成该流的清理过程。

Hyperscan库中存在流，因此可以跨多个目标数据块维护模式匹配状态 - 无需维护此状态，就无法检测跨越这些数据块的模式。然而，这确实需要以每个流需要大量存储（此存储的大小在编译时固定）为代价，并且在某些情况下管理状态会略微降低性能。

虽然Hyperscan总是支持多个匹配的严格排序，但是在当前流写入之前不会在偏移处传递流匹配，除了零宽度断言，其中诸如\b和之类的构造 $可以导致匹配最终字符。流写入被延迟，直到下一个流写或流关闭操作。

### 流管理
除hs_open_stream()，hs_scan_stream()以及 hs_close_stream()，所述Hyperscan API提供了大量的用于流的管理等功能：

- **hs_reset_stream()**：将流重置为其初始状态; 这相当于调用hs_close_stream()但不会释放用于流状态的内存。
- **hs_copy_stream()**：构造流的（新分配的）副本。
- **hs_reset_and_copy_stream()**：将流的副本构造到另一个流中，首先重置目标流。此调用可以避免完成分配hs_copy_stream()。

### 流压缩
流对象被分配为固定大小的存储器区域，其大小已经确定以确保在扫描操作期间不需要存储器分配。当系统处于内存压力下时，减少预计不会很快使用的流所消耗的内存可能会很有用。为此，Hyperscan API提供了将流转换为压缩表示的调用。压缩表示与完整流对象不同，因为它不为给定当前流状态的不需要的组件保留空间。此功能的Hyperscan API函数包括：

- **hs_compress_stream()**：使用流的压缩表示填充提供的缓冲区，并返回压缩表示所消耗的字节数。如果缓冲区不足以容纳压缩表示，HS_INSUFFICIENT_SPACE则返回所需大小。此调用不会以任何方式修改原始流：它仍可以写入hs_scan_stream()，用作各种重置调用的一部分以重新初始化其状态，或者 hs_close_stream()可以调用以释放其资源。
- **hs_expand_stream()**：基于包含压缩表示的缓冲区创建新流。
- **hs_reset_and_expand_stream()**：基于包含在现有流之上的压缩表示的缓冲区构造流，首先重置现有流。此调用可以避免完成分配 hs_expand_stream()。
注意：出于性能原因，不建议在每次扫描调用之间使用流压缩，因为在压缩表示和标准流之间进行转换需要时间。

## 块模式
块模式运行时API由单个函数组成：hs_scan()。使用编译的模式，此函数使用函数指针回调来识别目标数据中的匹配项以与应用程序通信。

这个单一hs_scan()函数基本上等同于调用 hs_open_stream()，进行单个调用hs_scan_stream()，然后hs_close_stream()，除了块模式操作不会产生所有与流相关的开销。

## 向量模式
向量模式运行时API（如块模式API）由单个函数组成：hs_scan_vector()。此函数接受一组数据指针和长度，便于按顺序扫描一组在内存中不连续的数据块。

从调用者的角度来看，此模式将产生相同的匹配，就好像数据块集合（a）按顺序扫描一系列流模式扫描，或（b）按顺序复制到单个内存块中然后扫描在块模式下。

## 划痕空间
扫描数据时，Hyperscan需要少量临时内存来存储即时内部数据。遗憾的是，这个数量太大而不适合堆栈，特别是对于嵌入式应用程序，并且动态分配内存太昂贵，因此必须为扫描功能提供预先分配的“划痕”空间。

该函数hs_alloc_scratch()分配足够大的临时空间区域以支持给定的数据库。如果应用程序使用多个数据库，则只需要一个临时区域：在这种情况下，调用 hs_alloc_scratch()每个数据库（使用相同的scratch指针）将确保临时空间足够大，以支持对任何给定数据库进行扫描。

虽然Hyperscan库是可重入的，但刮擦空间的使用却不是。例如，如果设计认为有必要运行递归或嵌套扫描（例如，来自匹配回调函数），则该上下文需要额外的临时空间。

在没有递归扫描的情况下，每个线程只需要一个这样的空间，并且可以（并且实际上应该）在数据扫描开始之前分配。

在一个表达式由单个“主”线程编译并且数据将由多个“工作”线程扫描的情况下，便利功能hs_clone_scratch()允许为每个线程创建现有临时空间的多个副本（而不是强制）调用者hs_alloc_scratch()多次传递所有已编译的数据库）。

例如：

    hs_error_t  err;
    hs_scratch_t  * scratch_prototype  =  NULL;
    err  =  hs_alloc_scratch （db ， ＆scratch_prototype ）;
    if  （err ！=  HS_SUCCESS ） {
        printf （“hs_alloc_scratch failed！” ）;
        exit（1 ）;
    }

    hs_scratch_t  * scratch_thread1  =  NULL ;
    hs_scratch_t  * scratch_thread2  =  NULL ;

    err  =  hs_clone_scratch （scratch_prototype ， ＆scratch_thread1 ）;
    if  （err ！=  HS_SUCCESS ） {
        printf （“hs_clone_scratch failed！” ）;
        退出（1 ）;
    }
    err  =  hs_clone_scratch （scratch_prototype ， ＆scratch_thread2 ）;
    if  （err ！=  HS_SUCCESS ） {
        printf （“hs_clone_scratch failed！” ）;
        exit（1 ）;
    }

    hs_free_scratch （scratch_prototype ）;

    / *现在两个线程都可以扫描数据库db，
       每个都有自己的临时空间。* /

## 自定义分配器
默认情况下，在运行时（暂存空间，流状态，等等）使用Hyperscan结构被分配与默认的系统分配器，通常是 malloc()和free()。

Hyperscan API提供了一种更改此行为的工具，以支持使用自定义内存分配器的应用程序。

这些功能是：

- **hs_set_database_allocator()**，它设置用于编译模式数据库的分配和自由函数。
- **hs_set_scratch_allocator()**，设置用于临时空间的分配和自由函数。
- **hs_set_stream_allocator()**，设置用于流模式的流状态的分配和自由函数。
- **hs_set_misc_allocator()**，设置用于杂项数据的分配和自由函数，例如编译错误结构和信息字符串。

该hs_set_allocator()函数可用于将所有自定义分配器设置为相同的分配/释放对。

-------------------------

# 序列化
对于某些应用程序，在使用之前立即编译Hyperscan模式数据库不是一个合适的设计。有些用户可能希望：

- 在不同的主机上编译模式数据库;
- 将已编译的数据库保留到存储中，并仅在模式更改时重新编译模式数据库;
- 控制已编译数据库所在的内存区域。

Hyperscan模式数据库在内存中并不完全平坦：它们包含指针并具有特定的对齐要求。因此，它们不能直接复制（或以其他方式重新定位）。为了启用这些用例，Hyperscan提供了序列化和反序列化已编译模式数据库的功能。

API提供以下功能：

1. **hs_serialize_database()**：将模式数据库序列化为平坦的可重定位字节缓冲区。
2.** hs_deserialize_database()**：从输出中重建新分配的模式数据库hs_serialize_database()。
3. **hs_deserialize_database_at()**：从输出中重建给定内存位置的模式数据库 hs_serialize_database()。
4. **hs_serialized_database_size()**：给定序列化模式数据库，在反序列化时返回数据库所需的内存块大小。
5. **hs_serialized_database_info()**：给定序列化模式数据库，返回包含有关数据库信息的字符串。这个电话类似于hs_database_info()。

    注意
    Hyperscan在反序列化时执行版本和平台兼容性检查。该hs_deserialize_database()和 hs_deserialize_database_at()功能将只允许用（a）和同一版本Hyperscan的当前主机平台支持（B）平台特性编写数据库反序列化。有关平台专业化的更多信息，请参阅 指令集专业化。

## 运行时库
Hyperscan主库（libhs）包含库的编译器和运行时部分。这意味着为了支持用C ++编写的Hyperscan编译器，它需要C ++链接并且依赖于C ++标准库。

许多嵌入式应用程序仅需要Hyperscan库的扫描（“运行时”）部分。在这些情况下，模式编译通常在另一个主机上进行，序列化模式数据库将传递给应用程序以供使用。

为了在不需要C ++依赖性的情况下支持这些应用程序libhs_runtime，还分发了名为的仅运行时版本的Hyperscan库。该库不依赖于C ++标准库，并提供用于编译数据库的所有Hyperscan函数。

--------------------

# 性能注意事项
Hyperscan支持所有三种扫描模式中的各种模式。它具有极高的性能水平，但某些模式可能会显着降低性能。

以下指南将帮助构建更好的模式和模式集：

## 正则表达式构造
    Tips
    不要手动优化正则表达式构造。

可以用多种方式编写大量正则表达式。例如，无壳匹配/abc/可以写成：

- /[Aa][Bb][Cc]/
- /(A|a)(B|b)(C|c)/
- /(?i)abc(?-i)/
- /abc/i
Hyperscan能够处理所有这些结构。除非有特殊原因，否则不要将模式从一种形式重写为另一种形式。

作为另一个例子，匹配/foo(bar|baz)(frotz)?/可以等效地写为：

- /foobarfrotz|foobazfrotz|foobar|foobaz/
此更改不会提高性能或减少开销。

## 库使用
    Tips
    不要手动优化库使用。

Hyperscan库能够处理小写入，异常大小的模式集等。除非对库的某些使用存在特定的性能问题，否则最好以简单直接的方式使用Hyperscan。例如，除非流写入很小（例如，每次1-2个字节），否则将库的输入缓冲到较大的块中不太可能有很多好处。

与许多其他模式匹配产品不同，Hyperscan将以较少数量的模式运行得更快，并且以平滑的方式更快地运行大量模式（相反，通常以中等速度运行到某个固定限制，然后打破或运行一半一样快）。

Hyperscan还提供高吞吐量匹配，每个核心具有单个控制线程; 如果数据库在Hyperscan中以3.0 Gbps运行，则意味着在单个控制线程中将以1微秒扫描3000位数据块，而不是需要在22微秒内扫描22个3000位数据块。因此，通常不需要缓冲数据以提供具有可用并行性的Hyperscan。

## 基于块的匹配
    Tips
    在可能的情况下，首选基于块的匹配流匹配。

每当输入数据出现在离散记录中，或者已经需要某种转换（例如URI规范化）时，需要在处理之前累积所有数据，应该在块中而不是在流模式下扫描。

不必要地使用流模式会减少可在Hyperscan中应用的优化次数，并可能使某些模式运行得更慢。

如果存在“块”和“流”模式模式的混合，则应在单独的数据库中扫描这些模式，除非流模式远远超过块模式模式。

## 不必要的数据库
    Tips
    避免不必要的“联合”数据库。

如果有5种不同类型的网络流量T1到T5必须针对5种不同的签名集进行扫描，那么构建5个独立的数据库并扫描流量相对于合适的数据库将比合并所有5个签名集要高得多并在事后删除不适当的匹配。

即使在签名之间存在大量重叠的情况下也是如此。只有当签名的公共子集非常大（例如，90％的签名出现在所有5种流量类型中）时，才应考虑合并所有5个签名集的数据库，并且只有在特定模式没有性能问题时才会考虑出现在公共子集之外。

## 提前分配刮痕
    Tips
    在调用扫描函数之前，不要为模式数据库分配临时空间。相反，在编译或反序列化模式数据库之后执行此操作。

划痕分配不一定是廉价的操作。由于这是第一次（在编译或反序列化之后）使用模式数据库，因此Hyperscan在内部执行一些验证检查，hs_alloc_scratch()并且还必须分配内存。

因此，重要的是确保hs_alloc_scratch()在之前hs_scan()（例如）不在应用程序的扫描路径中调用。

相反，应在编译或反序列化模式数据库之后立即分配临时，然后保留该临时扫描操作。

## 每个扫描上下文分配一个临时空间
    Tips
    可以分配临时空间，以便它可以与许多数据库中的任何一个一起使用。每个并发扫描操作（例如线程）都需要自己的临时空间。

该hs_alloc_scratch()函数可以接受现有的临时空间并“增长”它以支持使用另一个模式数据库进行扫描。这意味着，不是为应用程序使用的每个数据库分配一个临时空间，而是可以hs_alloc_scratch()使用指向hs_scratch_t它的指针进行调用， 并且它的大小将适当地与任何给定数据库一起使用。例如：

    hs_database_t  * db1  =  buildDatabaseOne （）;
    hs_database_t  * db2  =  buildDatabaseTwo （）;
    hs_database_t  * db3  =  buildDatabaseThree （）;

    hs_error_t  err;
    hs_scratch_t  * scratch  =  NULL ;
    err  =  hs_alloc_scratch （db1 ， ＆scratch ）;
    if  （err ！=  HS_SUCCESS ） {
        printf （“hs_alloc_scratch failed！” ）;
        exit（1 ）;
    }
    err  =  hs_alloc_scratch （db2 ， ＆scratch ）;
    if  （err ！=  HS_SUCCESS ） {
        printf （“hs_alloc_scratch failed！” ）;
        exit（1 ）;
    }
    err  =  hs_alloc_scratch （db3 ， ＆scratch ）;
    if  （err ！=  HS_SUCCESS ） {
        printf （“hs_alloc_scratch failed！” ）;
        exit（1 ）;
    }

    / * scratch现在可用于扫描任何
       数据库db1，db2，db3。* /

## 锚定模式
    Tips
    如果模式要出现在数据的开头，请务必将其锚定。

锚定模式（/^.../）比其他模式更容易匹配，尤其是锚定到缓冲区开始的模式（或流模式下的流）。将模式锚定到缓冲区的末尾导致较少的性能增益，尤其是在流模式下。

有多种方法可以将模式锚定到特定偏移量：

- ^和\A构建体锚定图案到缓冲区的开始。例如，/^foo/可以只匹配在偏移3。
- $，\z并\Z构建体锚定图案到缓冲器的末尾。例如，/foo\z/只能在扫描的数据缓冲区结束时匹配foo。（应该注意的是， $并且\Z也会在缓冲区末尾的换行符之前匹配，因此/foo\z/会匹配任何一个或 。）abc fooabc foo\n
- min_offset和max_offset扩展的参数也可以用于约束其中的图案可以匹配。例如，/foo/a max_offset为10 的模式 仅匹配缓冲区中小于或等于10的偏移量。（此模式也可以写为 /^.{0,7}foo/，使用HS_FLAG_DOTALL标志编译）。

## 到处匹配
    Tips
    避免在任何地方都匹配的模式，并记住我们的语义是“匹配到处，仅匹配结束”。

由于他们返回的绝对匹配数量，匹配到处的模式将运行缓慢。

状图案/.*/在基于自动匹配器将之前和之后的每一个字符的位置相匹配，所以用100个字符的缓冲器将返回101个匹配。在这种情况下，像libpcre这样的贪婪模式匹配器将返回单个匹配，但我们的语义是返回所有匹配。对于我们的代码和库的客户端代码，这可能非常昂贵。

我们的语义的另一个结果（“无处不在”）是具有可选的开始或结束部分的模式 - 例如/x?abcd*/- 可能无法按预期执行。

首先，x?模式的一部分是不必要的，因为它不会影响匹配结果。

其次，上面的模式将匹配“更多”，/abc/但 /abc/总会检测到任何将匹配的输入数据 /x?abcd*/- 它只会产生更少的匹配。

例如，输入数据0123abcdddd将匹配/abc/一次，但 /abcd*/5次（在abc，abcd，abcdd，abcddd，和 abcdddd）。

## 流模式下的有界重复
    Tips
    有界重复在流模式下很昂贵。

有限的重复结构，例如/X.{1000,1001}abcd/在流模式下非常昂贵，是必要的。它要求我们对每个X字符采取操作（相对于搜索更长的字符串本身很昂贵）并且可能记录数百个偏移的历史记录X，以防出现X和abcd字符由流边界分隔。

应避免重复和不必要地使用有界重复，特别是在签名的其他部分非常具体的情况下。例如，匹配病毒有效载荷的病毒签名可能足够，而不包括预先包括例如2字符Windows可执行前缀和有界重复的前缀。

## 喜欢文字
    Tips
    在可能的情况下，更喜欢“需要”文字的模式，特别是更长的文字，并且在流模式下，更喜欢在模式中较早地“需要”文字的签名。

必须与文字匹配的模式将比不支持文字的模式运行得更快。例如：

- /\wab\d*\w\w\w/ 会跑得比
- /\w\w\d*\w\w/或者，就此而言
- /\w(abc)?\d*\w\w\w/ （这包含一个文字但不需要出现在输入中）。
即使隐式文字也比没有好：/[0-2][3-5].*\w\w/ 仍然有效地包含9个2字符文字。不需要对这种情况进行手工优化; 如果重写为：这种模式将不会运行得更快 /(03|04|05|13|14|15|23|24|25).*\w\w/。

在任何情况下，最好使用较短的文字而不是较短的文字。由100个14个字符的文字组成的数据库将比由100个4个字符的文字组成的数据库扫描速度快得多，并且返回的正数减少。

此外，在流模式下，在模式的早期包含较长文字的签名优先于不具有较长文字的签名。

例如：/b\w*foobar/不如图​​案好 /blah\w*foobar/。

在块模式下，这些模式之间的差异要小得多。

在流模式下，模式中任何地方的文字都更长。例如，上述两种模式都更强，并且/b\w*fo/即使在流模式下扫描也会更快 。

## “点全部”模式
小费
尽可能使用“全点”模式。

不使用HS_FLAG_DOTALL模式标志可能是昂贵的，因为隐含地，它意味着形式的模式/A.*B/变得 /A[^\n]*B/。

没有DOTALL标志的扫描任务可能更好地“一次排队”，新行序列标记每个块的开始和结束。

在大多数用例中都是如此（例外情况是DOTALL标志关闭但模式包含显式换行符或结构，例如 \s隐式匹配换行符）。

## 单匹配标志
    Tips
    如果可能，请考虑使用单匹配标志将匹配限制为每个模式一个匹配。

如果每个模式只需要一个匹配项，请使用提供的标志来指示此（HS_FLAG_SINGLEMATCH）。此标志可以允许应用许多优化，从而在流式传输时允许性能改进和状态空间减少。

但是，跟踪模式集中的每个模式是否匹配存在一些开销，并且一些具有不频繁匹配的应用程序在使用单匹配标志时可能会看到性能降低。

## 比赛开始标志
    Tips
    如果不需要，请勿请求匹配开始信息。

匹配开始（SOM）信息收集起来可能很昂贵，并且可能需要大量流状态以流模式存储。因此，只应使用HS_FLAG_SOM_LEFTMOST 需要它的模式的标志来请求SOM信息。

SOM信息通常不会比使用有界重复更便宜（在性能方面或在流状态开销中）。因此，/foo.*bar/L在回调之后检查匹配值的开始是相当昂贵和一般的 /foo.{300}bar/。

类似地，hs_expr_ext::min_length扩展参数可用于指定模式匹配长度的下限。在某些情况下使用此工具可能比使用SOM标志和在调用应用程序中确认匹配长度后更轻量级。

## 近似匹配
    Tips
    近似匹配是一项实验性功能。

由于匹配的特异性降低，通常存在与近似匹配相关的性能影响。根据模式和编辑距离，此影响可能会有很大差异。

-------------------

# 工具
本节介绍Hyperscan库附带的一组实用程序。

## 快速检查：hscheck 
该hscheck工具允许用户快速检查Hyperscan是否支持一组模式。如果Hyperscan的编译器拒绝某个模式，则会在标准输出上提供编译错误。

例如，在一个名为的文件中给出以下三种模式（最后一种包含语法错误）/tmp/test：

    1 ：/ foo 。* bar /
    2 ：/ abc | def | ghi /
    3 ：/ （（foo | bar ）/
...该hscheck工具将产生以下输出：

$ bin / hscheck -e / tmp / test

    好的：1：/foo.*bar/
    好的：2：/ abc | def | ghi /
    FAIL（编译）：3：/（（foo | bar）/：缺少组的右括号从索引0开始。
    摘要：3个中的1个失败。

## Benchmarker：hsbench 
该hsbench工具提供了一种简单的方法来测量Hyperscan针对特定模式集和要扫描的数据集的性能。

模式以下面描述的 模式格式提供，而语料库必须以语料库数据库的形式提供 ：这是一种简单的SQLite数据库格式，旨在允许轻松控制语料库如何分解为块和流。

    注意
    可以在Hyperscan源代码树中找到一组用于从各种输入类型构建语料库数据库的Python脚本，例如PCAP网络流量捕获或文本文件tools/hsbench/scripts。

### Running hsbench
给定一个文件，其中包含指定的模式-e和指定的语料库数据库-c，hsbench将执行单线程基准测试并生成如下输出：

    $ hsbench -e / tmp / patterns -c /tmp/corpus.db

    签名：/ tmp / patterns
    Hyperscan信息：版本：4.3.1功能：AVX2模式：STREAM
    表达计数：200
    字节码大小：342,540字节
    数据库CRC：0x6cd6b67c
    流状态大小：252字节
    划痕大小：18,406字节
    编译时间：0.153秒
    峰值使用量：78,073,856字节

    扫描时间：0.600秒
    语料库大小：72,138,183字节（8,891个流中63,946个块）
    扫描匹配：81（0.001匹配/千字节）
    整体阻滞率：2,132,004.45块/秒
    总吞吐量：19,241.10 Mbit / sec

默认情况下，语料库被扫描20次，报告的整体性能是根据执行所有20次扫描所花费的时间内扫描的总字节数计算的。可以使用-n参数更改重复次数，如果--per-scan指定了 参数，则将显示每次扫描的结果 。

要在多个核心上对Hyperscan进行基准测试，您可以提供带有-T参数的核心列表，这将指示hsbench每个核心启动一个基准线程并从完成所有核心所需的时间计算吞吐量。

    Tips
    对于多处理器系统上的单线程基准测试，我们建议使用一个实用程序，例如taskset将hsbench进程锁定到一个核心，并最大限度地减少操作系统调度程序引起的抖动。

## 正确性测试：hscollider
该hscollider工具或Pattern Collider提供了一种验证Hyperscan匹配行为的方法。它通过针对已知语料库编译和扫描模式（单独或成组）并将结果与​​另一个引擎（“基础事实”）进行比较来实现此目的。有两个基本事实来源可供比较：

- PCRE库（http://pcre.org/）。
- 在Hyperscan的编译时图形表示上运行NFA模拟。如果PCRE无法支持该模式，或者由于资源限制导致PCRE执行失败，则使用此方法。

Hyperscan的大部分测试基础架构都是基于hscollider该工具构建的，该工具旨在利用多个内核并在控制测试方面提供相当大的灵活性。这些选项在help（）中描述，包括：hscollider -h

- 以流，块或向量模式进行测试。
- 在内存中的不同路线上测试语料库。
- 以不同大小的组测试模式。
- 在测试之间操纵流状态或临时空间。
- 数据库的交叉编译和序列化/反序列化。
- 给定模式集合的语料库生成。

### 使用hscollider调试模式
一个常见的用例hscollider是确定Hyperscan是否匹配预期位置中的模式，以及这是否符合PCRE在相同情况下的行为。

这是一个例子。我们将模式放在Hyperscan模式格式的文件中：

    $ cat / tmp / pat
    1：/hatstand.*badgerbrush/
我们将语料库扫描到另一个文件中，在开头使用相同的数字标识符表示它应该匹配模式1：

    $ cat / tmp / corpus
    1：__ hatstand__hatstand__badgerbrush_badgerbrush
然后我们可以运行hscollider其详细程度up（-vv），以便在输出中显示单个匹配：

    $ bin / ue2collider -e / tmp / pat -c / tmp / corpus -Z 0 -T 1 -vv
    ue2collider：模式碰撞器Mark II

    线程数：1（1台扫描仪，1台发电机）
    表达式路径：/ tmp / pat
    签名文件：无
    操作模式：块模式
    UE2扫描对齐：0
    Corpora从文件中读取：/ tmp / corpus

    对1个表达式运行单模式/单编译测试。

    PCRE比赛@（2,45）
    PCRE比赛@（2,33）
    PCRE比赛@（12,45）
    PCRE比赛@（12,33）
    UE2匹配@（0,33）为1
    UE2匹配@（0,45）为1
    扫描电话返回0
    通过：id 1，对齐0，语料库0（匹配的pcre：2，ue2：2）
    线程0处理1个单位。

    摘要：
    模式：单/块
    =========
    表达式处理：1
    Corpora处理：1
    表达失败：0
      Corpora一代失败：0
      编译失败：pcre：0，ng：0，ue2：0
      匹配失败：pcre：0，ng：0，ue2：0
      匹配差异：0
      没有根据：0
    总匹配差异：0

    总耗时：0.00522815秒。
我们可以从这个输出中看到PCRE和Hyperscan都找到了以偏移量33和45结尾的匹配，因此hscollider认为这个测试用例已经过去了。

（在上面的示例命令行中，指示我们仅在语料库对齐0处测试，并指示我们仅使用一个线程。）-Z 0-T 1

    注意
    在默认操作中，PCRE只为扫描生成一个匹配项，这与Hyperscan的自动机语义不同。该hscollider工具使用libpcre的“callout”功能来匹配Hyperscan的语义。

### 运行更大的扫描测试
用于测试目的的一组模式与Hyperscan一起分发，这些模式可以通过hscollider树内构建进行测试。提供了两个CMake目标来轻松完成此操作：

|制定目标	|描述|
|--|--|
|make collide_quick_test	|以流模式测试所有模式。|
|make collide_quick_test_block	|以块模式测试所有模式。|

## 调试：hsdump
当在调试模式下构建（使用CMAKE_BUILD_TYPE设置为 CMake的指令Debug）时，Hyperscan支持在使用该hsdump工具进行模式编译期间转储有关其内部的信息。

这些信息主要用于熟悉库内部结构的Hyperscan开发人员，但可用于诊断模式问题并在错误报告中提供更多信息。

## 模式格式
所有Hyperscan工具都接受相同格式的模式，从每行一个模式的纯文本文件中读取。每一行看起来像这样：

- <integer id>:/<regex>/<flags>
例如：

    1 ：/ hatstand 。* teakettle / s
    2 ：/ （hatstand | teakettle ）/ iH
    3 ：/ ^。{ 10 ，20 } hatstand / 米

整数ID是Hyperscan找到匹配时将报告的值，并且必须是唯一的。

模式本身是PCRE语法中的正则表达式; 有关支持的功能的更多信息，请参阅 编译模式。

标志是映射到Hyperscan标志的单个字符，如下所示：

|字符	|API标志	|描述|
|i	|HS_FLAG_CASELESS	|不区分大小写的匹配|
|s	|HS_FLAG_DOTALL	|Dot（.）将匹配换行符|
|m	|HS_FLAG_MULTILINE	|多线锚定|
|H	|HS_FLAG_SINGLEMATCH	|最多报告一次匹配ID|
|V	|HS_FLAG_ALLOWEMPTY	|允许可以匹配空缓冲区的模式|
|8	|HS_FLAG_UTF8	|UTF-8模式|
|W	|HS_FLAG_UCP	|Unicode属性支持|
|P	|HS_FLAG_PREFILTER	|预过滤模式|
|L	|HS_FLAG_SOM_LEFTMOST	|最左边的比赛开始报道|
|C	|HS_FLAG_COMBINATION	|模式的逻辑组合|
|Q	|HS_FLAG_QUIET	|在匹配时安静|

除了上面的标志集之外，还可以为每个模式提供扩展参数。这些是key=value在大括号之间的标志之后提供的，以逗号分隔。例如：

	1 ：/ hatstand 。* 茶壶/ 小号{ min_offset = 50 ，max_offset = 100 }
所有Hyperscan工具都将接受带有参数的模式文件（或包含模式文件的目录）-e。如果没有给出限制模式集的其他参数，则使用这些文件中的所有模式。

要选择模式的子集，可以为-z 参数提供单个ID ，或者可以为-s 参数提供包含一组ID的文件。

--------------------

# API参考：常量
## 错误代码
**HS_SUCCESS**
发动机正常完成。

**HS_INVALID**
传递给此函数的参数无效。

仅当函数可以检测到无法依赖于检测（例如）指向已释放内存或其他无效数据的指针的无效参数时，才会返回此错误。

**HS_NOMEM**
内存分配失败。

**HS_SCAN_TERMINATE**D
引擎被回调终止。

此返回值表示目标缓冲区已被部分扫描，但回调函数请求在找到匹配项后停止扫描。

**HS_COMPILER_ERROR**
模式编译器失败，应检查hs_compile_error_t以获取更多详细信息。

**HS_DB_VERSION_ERROR**
给定的数据库是为不同版本的Hyperscan构建的。

**HS_DB_PLATFORM_ERROR**
给定的数据库是为不同的平台（即CPU类型）构建的。

**HS_DB_MODE_ERROR**
给定的数据库是为不同的操作模式构建的。当流式调用与块或向量数据库一起使用时，将返回此错误，反之亦然。

**HS_BAD_ALIGN**
传递给此函数的参数未正确对齐。

**HS_BAD_ALLOC**
内存分配器（malloc（）或使用hs_set_allocator（）设置的分配器）未正确返回适合此平台上最大可表示数据类型的内存。

**HS_SCRATCH_IN_USE**
划痕区域已经在使用中。

当Hyperscan能够检测到给定的临时区域已被另一个Hyperscan API调用使用时，将返回此错误。

Hyperscan API的每个并发调用者都需要一个单独的临时区域，分配有hs_alloc_scratch（）或hs_clone_scratch（）。

例如，当在使用相同临时区域的当前正在执行的hs_scan（）调用传递的回调内调用hs_scan（）时，可能会返回此错误。

注意：并非可以检测到临时区域的所有并发使用。此错误旨在作为尽力而为的调试工具，而非保证。

**HS_ARCH_ERROR**
不支持的CPU架构。

当Hyperscan能够检测到当前系统不支持所需的指令集时，将返回此错误。

Hyperscan至少需要Supplemental Streaming SIMD Extensions 3（SSSE3）。

**HS_INSUFFICIENT_SPACE**
提供的缓冲区太小了。

此错误表示缓冲区中没有足够的空间。应使用较大的提供缓冲区重复调用。

注意：在这种情况下，返回所需的空间量是正常的，与调用成功时返回的已用空间的方式相同。

**HS_UNKNOWN_ERROR**
意外的内部错误。

此错误表示存在意外的匹配行为。这可能与用户对流和暂存空间的无效使用或无效的内存操作有关。

## hs_expr_ext标志
**HS_EXT_FLAG_MIN_OFFSET**
指示使用hs_expr_ext :: min_offset字段的标志。

**HS_EXT_FLAG_MAX_OFFSET**
指示使用hs_expr_ext :: max_offset字段的标志。

HS_EXT_FLAG_MIN_LENGTH
指示使用hs_expr_ext :: min_length字段的标志。

HS_EXT_FLAG_EDIT_DISTANCE
指示使用hs_expr_ext :: edit_distance字段的标志。

HS_EXT_FLAG_HAMMING_DISTANCE
指示使用hs_expr_ext :: hamming_distance字段的标志。

## 模式标志
**HS_FLAG_CASELESS**
编译标志：设置不区分大小写的匹配。

此标志将表达式设置为默认情况下不区分大小写。该表达式仍然可以使用PCRE令牌（特别是(?i)和(?-i)）来打开和关闭不区分大小写的匹配。

**HS_FLAG_DOTALL**
编译标志：匹配a .不会排除换行符。

此标志设置.令牌的任何实例以匹配换行符和所有其他字符。PCRE规范声明该.标记默认情况下与换行符不匹配，因此如果没有此标记，则.标记将不会跨越行边界。

**HS_FLAG_MULTILINE**
编译标志：设置多行锚定。

此标志指示表达式使^和$标记匹配换行符以及流的开始和结束。如果未指定此标志，则^令牌将仅在流的开头匹配，并且$令牌将仅在PCRE规范的指导内的流的末尾匹配。

**HS_FLAG_SINGLEMATCH**
编译标志：设置仅匹配模式。

此标志将表达式的匹配ID设置为最多匹配一次。在流模式下，这意味着表达式将在流的生命周期内仅返回单个匹配，而不是按照标准Hyperscan语义报告每个匹配。在块模式或向量模式下，将仅返回每次调用hs_scan（）或hs_scan_vector（）的第一个匹配项。

如果数据库中的多个表达式共享相同的匹配ID，则它们必须全部指定HS_FLAG_SINGLEMATCH，或者它们都不指定HS_FLAG_SINGLEMATCH。如果共享匹配ID的一组表达式指定该标志，则每个流最多将生成一个与匹配ID匹配的匹配。

注意：目前不支持将此标志与HS_FLAG_SOM_LEFTMOST结合使用。

**HS_FLAG_ALLOWEMPTY**
编译标志：允许可以匹配空缓冲区的表达式。

此标志指示编译器允许，可以匹配对空的缓冲器，如表情.?，.*，(a|)。由于Hyperscan可以返回表达式的每个可能的匹配，因此这些表达式通常执行得非常慢; 默认行为是在尝试编译一个时返回错误。使用此标志将强制编译器允许这样的表达式。

**HS_FLAG_UTF8**
编译标志：为此表达式启用UTF-8模式。

此标志指示Hyperscan将模式视为UTF-8字符序列。使用此标志使用一个或多个模式编译的Hyperscan库扫描无效UTF-8序列的结果未定义。

**HS_FLAG_UCP**
编译标志：为此表达式启用Unicode属性支持。

此标志指示Hyperscan使用Unicode属性，而不是默认的ASCII解释，字符助记符像\w和\s还有POSIX字符类。它仅与HS_FLAG_UTF8一起使用才有意义。

**HS_FLAG_PREFILTER**
编译标志：为此表达式启用预过滤模式。

此标志指示Hyperscan编译此模式的“近似”版本以用于预过滤应用程序，即使Hyperscan在正常操作中不支持该模式也是如此。

使用此标志时返回的匹配集保证是非预先过滤表达式指定的匹配的超集。

如果模式包含Hyperscan不支持的模式构造（例如零宽度断言，反向引用或条件引用），则这些构造将在内部被更广泛的构造替换，这些构造可能更频繁地匹配。

此外，在预过滤模式中，Hyperscan可以简化在编译时或者出于性能原因（在上面的匹配保证的情况下）返回“模式太大”错误的模式。

通常预期应用程序随后将确认预过滤器与另一个可以为模式提供精确匹配的正则表达式匹配器匹配。

注意：目前不支持将此标志与HS_FLAG_SOM_LEFTMOST结合使用。

**HS_FLAG_SOM_LEFTMOST**
编译标志：启用最左侧的匹配报告开始。

此标志指示Hyperscan在报告此表达式的匹配时报告最左侧可能的匹配偏移开始。（默认情况下，不返回匹配开始。）

启用此行为可能会降低性能并增加流模式下的流状态要求。

**HS_FLAG_COMBINATION**
编译标志：逻辑组合。

此标志指示Hyperscan将此表达式解析为逻辑组合语法。逻辑约束由操作数，运算符和括号组成。操作数是表达式索引，运算符可以是'！'（NOT），'＆'（AND）或'|'（OR）。例如：（101＆102＆103）|（104＆！105）（（301 | 302）＆303）＆（304 | 305）

**HS_FLAG_QUIET**
编译标志：不做任何匹配报告。

此标志指示Hyperscan忽略此表达式的匹配报告。它被设计用于逻辑组合中的子表达式。

## CPU功能支持标志
**HS_CPU_FEATURES_AVX2**
CPU功能标志 - 英特尔（R）高级矢量扩展2（英特尔（R）AVX2）

设置此标志表示目标平台支持AVX2指令。

**HS_CPU_FEATURES_AVX512**
CPU功能标志 - 英特尔（R）高级矢量扩展512（英特尔（R）AVX512）

设置此标志表示目标平台支持AVX512指令，特别是AVX-512BW。使用AVX512意味着使用AVX2。

## CPU调整标志
**HS_TUNE_FAMILY_GENERIC**
调整参数 - 通用

这表明不应针对任何特定目标平台调整已编译的数据库。

**HS_TUNE_FAMILY_SNB**
调整参数 - 英特尔（R）微体系结构代码名称Sandy Bridge

这表明编译的数据库应该针对Sandy Bridge微体系结构进行调整。

**HS_TUNE_FAMILY_IVB**
调整参数 - 英特尔（R）微体系结构代码名称Ivy Bridge

这表明编译的数据库应该针对Ivy Bridge微体系结构进行调整。

**HS_TUNE_FAMILY_HSW**
调整参数 - 英特尔（R）微体系结构代码名称Haswell

这表明编译的数据库应该针对Haswell微体系结构进行调整。

**HS_TUNE_FAMILY_SLM**
调整参数 - 英特尔（R）微体系结构代码名称Silvermont

这表明编译的数据库应该针对Silvermont微体系结构进行调整。

**HS_TUNE_FAMILY_BDW**
调整参数 - 英特尔（R）微体系结构代码名称Broadwell

这表明编译的数据库应该针对Broadwell微体系结构进行调整。

**HS_TUNE_FAMILY_SKL**
调整参数 - 英特尔（R）微体系结构代码名称Skylake

这表明编译的数据库应该针对Skylake微体系结构进行调整。

**HS_TUNE_FAMILY_SKX**
调整参数 - 英特尔（R）微体系结构代码名称Skylake Server

这表明应该针对Skylake Server微体系结构调整已编译的数据库。

**HS_TUNE_FAMILY_GLM**
调整参数 - 英特尔（R）微体系结构代码名称Goldmont

这表明编译的数据库应该针对Goldmont微体系结构进行调整。

## 编译模式标志
**HS_MODE_BLOCK**
编译器模式标志：块扫描（非流式）数据库。

**HS_MODE_NOSTREAM**
编译器模式标志：HS_MODE_BLOCK的别名。

**HS_MODE_STREAM**
编译模式标志：流数据库。

**HS_MODE_VECTORED**
编译器模式标志：向量扫描数据库。

**HS_MODE_SOM_HORIZON_LARGE**
编译器模式标志：使用全精度跟踪流状态下匹配偏移的开始。

此模式将使用每个模式的最流状态，但始终会返回准确的匹配偏移开始，而不管过去发现的距离有多远。

必须选择其中一个SOM_HORIZON模式才能使用HS_FLAG_SOM_LEFTMOST表达式标志。

**HS_MODE_SOM_HORIZON_MEDIUM**
编译器模式标志：使用中等精度跟踪流状态下匹配偏移的开始。

此模式将使用比HS_MODE_SOM_HORIZON_LARGE更少的流状态，并将匹配准确度的开始限制为报告的匹配结束偏移的2 ^ 32字节内的偏移。

必须选择其中一个SOM_HORIZON模式才能使用HS_FLAG_SOM_LEFTMOST表达式标志。

**HS_MODE_SOM_HORIZON_SMALL**
编译器模式标志：使用有限精度来跟踪流状态中匹配偏移的开始。

此模式将使用比HS_MODE_SOM_HORIZON_LARGE更少的流状态，并将匹配精度的开始限制为报告的匹配结束偏移的2 ^ 16字节内的偏移。

必须选择其中一个SOM_HORIZON模式才能使用HS_FLAG_SOM_LEFTMOST表达式标志。

-------------------

# cAPI参考：文件
## 文件中：hs.h
完整的Hyperscan API定义。

Hyperscan是一种高速正则表达式引擎。

此标头包括Hyperscan编译器和运行时组件。有关文档，请参阅各个组件标题。

## 文件：hs_common.h
Hyperscan通用API定义。

Hyperscan是一种高速正则表达式引擎。

此标头包含Hyperscan编译器和运行时可用的函数。

定义

**HS_SUCCESS**
发动机正常完成。

**HS_INVALID**
传递给此函数的参数无效。

仅当函数可以检测到无法依赖于检测（例如）指向已释放内存或其他无效数据的指针的无效参数时，才会返回此错误。

**HS_NOMEM**
内存分配失败。

**HS_SCAN_TERMINATED**
引擎被回调终止。

此返回值表示目标缓冲区已被部分扫描，但回调函数请求在找到匹配项后停止扫描。

**HS_COMPILER_ERROR**
模式编译器失败，应检查hs_compile_error_t以获取更多详细信息。

**HS_DB_VERSION_ERROR**
给定的数据库是为不同版本的Hyperscan构建的。

**HS_DB_PLATFORM_ERROR**
给定的数据库是为不同的平台（即CPU类型）构建的。

**HS_DB_MODE_ERROR**
给定的数据库是为不同的操作模式构建的。当流式调用与块或向量数据库一起使用时，将返回此错误，反之亦然。

**HS_BAD_ALIGN**
传递给此函数的参数未正确对齐。

**HS_BAD_ALLOC**
内存分配器（malloc（）或使用hs_set_allocator（）设置的分配器）未正确返回适合此平台上最大可表示数据类型的内存。

**HS_SCRATCH_IN_USE**
划痕区域已经在使用中。

当Hyperscan能够检测到给定的临时区域已被另一个Hyperscan API调用使用时，将返回此错误。

Hyperscan API的每个并发调用者都需要一个单独的临时区域，分配有hs_alloc_scratch（）或hs_clone_scratch（）。

例如，当在使用相同临时区域的当前正在执行的hs_scan（）调用传递的回调内调用hs_scan（）时，可能会返回此错误。

注意：并非可以检测到临时区域的所有并发使用。此错误旨在作为尽力而为的调试工具，而非保证。

**HS_ARCH_ERROR**
不支持的CPU架构。

当Hyperscan能够检测到当前系统不支持所需的指令集时，将返回此错误。

Hyperscan至少需要Supplemental Streaming SIMD Extensions 3（SSSE3）。

HS_INSUFFICIENT_SPACE
提供的缓冲区太小了。

此错误表示缓冲区中没有足够的空间。应使用较大的提供缓冲区重复调用。

注意：在这种情况下，返回所需的空间量是正常的，与调用成功时返回的已用空间的方式相同。

**HS_UNKNOWN_ERROR**
意外的内部错误。

此错误表示存在意外的匹配行为。这可能与用户对流和暂存空间的无效使用或无效的内存操作有关。

### 类型定义

typedef的结构hs_database hs_database_t
Hyperscan模式数据库。

由Hyperscan编译器函数之一生成：

- hs_compile（）
- hs_compile_multi（）
- hs_compile_ext_multi（）
typedef INT **hs_error_t**
Hyperscan函数返回的错误类型。

typedef void \* ( \* hs_alloc_t)（ size_t  size ）
Hyperscan将根据需要在运行时分配更多内存的回调函数的类型，例如在hs_open_stream（）中分配流状态。

如果要在多线程或类似并发环境中使用Hyperscan，则分配功能需要是可重入的，或者对于并发使用同样安全。

**返回**
指向已分配内存区域的指针，或出错时为NULL。
**参数**
- size：要分配的字节数。

typedef void ( * hs_free_t)（ void  * ptr ）
Hyperscan将用于释放先前使用hs_alloc_t函数分配的内存区域的回调函数的类型。

**参数**
- ptr：要释放的内存区域。
**功能**

hs_error_t hs_free_database（hs_database_t *  db ）
释放已编译的模式数据库。

此函数将使用hs_set_database_allocator（）（或hs_set_allocator（））设置的空闲回调。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- db：已编译的模式数据库。也可以安全地提供NULL，在这种情况下该函数不执行任何操作。
hs_error_t hs_serialize_database（ const hs_database_t *  db，char **  bytes，size_t *  length ）
将模式数据库序列化为字节流。

由hs_set_misc_allocator（）（或hs_set_allocator（））设置的分配器回调将由此函数使用。

**返回**
HS_SUCCESS成功， HS_NOMEM如果无法分配字节数组，则在检测到错误时可能返回其他值。
**参数**
- db：已编译的模式数据库。
- bytes：成功时，将返回指向字节数组的指针。这些字节可以随后重新定位或写入磁盘。调用者负责释放此块。
- length：成功时，将在此处返回生成的字节数组中的字节数。

hs_error_t hs_deserialize_database（ const char *  bytes，const size_t  length，hs_database_t **  db ）
从先前由hs_serialize_database（）生成的字节流重建模式数据库。

此函数将使用带有hs_set_database_allocator（）（或hs_set_allocator（））的分配器集为数据库分配足够的空间; 要使用预先分配的内存区域，请使用hs_deserialize_database_at（）函数。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- bytes：由hs_serialize_database（）生成的字节数组，表示已编译的模式数据库。
- length：hs_serialize_database（）生成的字节数组的长度。该值应与hs_serialize_database（）返回的值相同。
- db：成功时，将在此处返回指向新分配的hs_database_t的指针。然后，此数据库可用于扫描，并最终由调用者使用hs_free_database（）释放。

hs_error_t hs_deserialize_database_at（ const char *  bytes，const size_t  length，hs_database_t *  db ）
从先前由hs_serialize_database（）在给定内存位置生成的字节流重建模式数据库。

此函数（与hs_deserialize_database（）不同）将重建的数据库写入db参数中给出的内存位置。可以使用hs_serialized_database_size（）函数确定此位置所需的空间量。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- bytes：由hs_serialize_database（）生成的字节数组，表示已编译的模式数据库。
- length：hs_serialize_database（）生成的字节数组的长度。该值应与hs_serialize_database（）返回的值相同。
- db：指向8字节对齐的内存块的指针，其大小足以容纳反序列化的数据库。成功后，重建的数据库将写入此位置。然后，该数据库可用于模式匹配。用户负责释放这种记忆; 在hs_free_database（）调用不应该使用。

hs_error_t hs_stream_size（ const hs_database_t *  database，size_t *  stream_size ）
提供由针对给定数据库打开的单个流分配的流状态的大小。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- database：指向已编译（流模式）模式数据库的指针。
- stream_size：成功时，针对给定数据库打开的单个流的大小（以字节为单位）将放在此参数中。

hs_error_t hs_database_size（ const hs_database_t *  database，size_t *  database_size ）
以字节为单位提供给定数据库的大小。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- database：指向已编译的模式数据库的指针。
- database_size：成功时，已编译数据库的大小（以字节为单位）放在此参数中。

hs_error_t hs_serialized_database_size（常量字符*  字节，常量为size_t  长度，为size_t *  deserialized_size ）
实用程序函数，用于报告数据库反序列化时所需的大小。

在使用hs_deserialize_database_at（）函数进行反序列化之前，这可用于分配共享内存区域或其他“特殊”分配。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- bytes：指向由hs_serialize_database（）生成的字节数组的指针，表示已编译的模式数据库。
- length：hs_serialize_database（）生成的字节数组的长度。该值应与hs_serialize_database（）返回的值相同。
- deserialized_size：成功时，将在此处返回由hs_deserialize_database_at（）生成的已编译数据库的大小。

hs_error_t hs_database_info（ const hs_database_t *  database，char **  info ）
提供有关数据库的信息的实用功能。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- database：指向已编译数据库的指针。
- info：成功时，包含所提供数据库的版本和平台信息的字符串将放置在参数中。使用hs_set_misc_allocator（）中提供的分配器（或者如果未设置分配器，则为 malloc（））分配字符串，并且应由调用者释放。

hs_error_t hs_serialized_database_info（ const char *  bytes，size_t  length，char **  info ）
实用程序功能提供有关序列化数据库的信息。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- bytes：指向序列化数据库的指针。
- length：序列化数据库的长度（以字节为单位）。
- info：成功时，将在参数中放置包含所提供的序列化数据库的版本和平台信息的字符串。使用hs_set_misc_allocator（）中提供的分配器（或者如果未设置分配器，则为 malloc（））分配字符串，并且应由调用者释放。

hs_error_t hs_set_allocator（hs_alloc_t  alloc_func，hs_free_t  free_func ）
设置Hyperscan使用的分配和释放函数，以便在运行时为流状态，暂存空间，数据库字节码以及Hyperscan API返回的各种其他数据结构分配内存。

该函数等效于使用提供的参数调用hs_set_stream_allocator（），hs_set_scratch_allocator（），hs_set_database_allocator（）和hs_set_misc_allocator（）。

此调用将覆盖已设置的任何先前分配器。

注意：无法更改用于在各种编译调用期间创建的临时对象的分配器（hs_compile（），hs_compile_multi（），hs_compile_ext_multi（））。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- alloc_func：一个分配内存的回调函数指针。此函数必须返回适合此平台上最大可表示数据类型的内存。
- free_func：一个回调函数指针，释放已分配的内存。

hs_error_t hs_set_database_allocator（hs_alloc_t  alloc_func，hs_free_t  free_func ）
设置Hyperscan使用的分配和释放函数，用于为编译调用（hs_compile（），hs_compile_multi（），hs_compile_ext_multi（））和数据库反序列化（hs_deserialize_database（））生成的数据库字节码分配内存。

如果未设置数据库分配函数，或者如果使用NULL代替两个参数，则内存分配将默认为标准方法（例如系统malloc（）和free（）调用）。

此调用将覆盖已设置的任何先前数据库分配器。

注意：也可以通过调用hs_set_allocator（）来设置数据库分配器。

注意：无法更改在各种编译调用（hs_compile（），hs_compile_multi（），hs_compile_ext_multi（））期间创建的临时对象的分配方式。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- alloc_func：一个分配内存的回调函数指针。此函数必须返回适合此平台上最大可表示数据类型的内存。
- free_func：一个回调函数指针，释放已分配的内存。

hs_error_t hs_set_misc_allocator（hs_alloc_t  alloc_func，hs_free_t  free_func ）
设置Hyperscan使用的分配和释放函数，以便为Hyperscan API返回的项目分配内存，例如hs_compile_error_t，hs_expr_info_t和序列化数据库。

如果没有设置misc分配函数，或者如果使用NULL代替两个参数，则内存分配将默认为标准方法（例如系统malloc（）和free（）调用）。

此调用将覆盖已设置的任何先前的misc分配器。

注意：也可以通过调用hs_set_allocator（）来设置misc分配器。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- alloc_func：一个分配内存的回调函数指针。此函数必须返回适合此平台上最大可表示数据类型的内存。
- free_func：一个回调函数指针，释放已分配的内存。

hs_error_t hs_set_scratch_allocator（hs_alloc_t  alloc_func，hs_free_t  free_func ）
设置Hyperscan使用的分配和释放函数，以便通过hs_alloc_scratch（）和hs_clone_scratch（）为暂存空间分配内存。

如果没有设置临时分配函数，或者如果使用NULL代替两个参数，则内存分配将默认为标准方法（例如系统malloc（）和free（）调用）。

此调用将覆盖已设置的任何先前暂存分配器。

注意：也可以通过调用hs_set_allocator（）来设置临时分配器。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- alloc_func：一个分配内存的回调函数指针。此函数必须返回适合此平台上最大可表示数据类型的内存。
- free_func：一个回调函数指针，释放已分配的内存。

hs_error_t hs_set_stream_allocator（hs_alloc_t  alloc_func，hs_free_t  free_func ）
设置Hyperscan使用的分配和释放函数，以便通过hs_open_stream（）为流状态分配内存。

如果没有设置流分配函数，或者如果使用NULL代替两个参数，则内存分配将默认为标准方法（例如系统malloc（）和free（）调用）。

此调用将覆盖已设置的任何先前的流分配器。

注意：也可以通过调用hs_set_allocator（）来设置流分配器。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- alloc_func：一个分配内存的回调函数指针。此函数必须返回适合此平台上最大可表示数据类型的内存。
- free_func：一个回调函数指针，释放已分配的内存。

const char * hs_version（ void ）
用于识别此发行版本的实用程序功能

**返回**
包含此发行版本的版本号和构建日期的字符串。它是静态分配的，因此不需要被调用者释放。

hs_error_t hs_valid_platform（ void ）
用于测试当前系统架构的实用程序功能。

Hyperscan需要Supplemental Streaming SIMD Extensions 3指令集。可以在任何x86平台上调用此函数，以确定系统是否提供所需的指令集。

如果为更具体的架构（例如AVX2指令集）构建Hyperscan，则此功能不会测试更高级的功能。

**返回**
HS_SUCCESS成功， HS_ARCH_ERROR，如果系统不支持Hyperscan。

## 文件：hs_compile.h
Hyperscan编译器API定义。

Hyperscan是一种高速正则表达式引擎。

此标头包含用于将正则表达式编译为可由Hyperscan运行时使用的Hyperscan数据库的函数。

定义

**HS_EXT_FLAG_MIN_OFFSET**
指示使用hs_expr_ext :: min_offset字段的标志。

**HS_EXT_FLAG_MAX_OFFSET**
指示使用hs_expr_ext :: max_offset字段的标志。

**HS_EXT_FLAG_MIN_LENGTH**
指示使用hs_expr_ext :: min_length字段的标志。

**HS_EXT_FLAG_EDIT_DISTANCE**
指示使用hs_expr_ext :: edit_distance字段的标志。

**HS_EXT_FLAG_HAMMING_DISTANCE**
指示使用hs_expr_ext :: hamming_distance字段的标志。

**HS_FLAG_CASELESS**
编译标志：设置不区分大小写的匹配。

此标志将表达式设置为默认情况下不区分大小写。该表达式仍然可以使用PCRE令牌（特别是(?i)和(?-i)）来打开和关闭不区分大小写的匹配。

**HS_FLAG_DOTALL**
编译标志：匹配a .不会排除换行符。

此标志设置.令牌的任何实例以匹配换行符和所有其他字符。PCRE规范声明该.标记默认情况下与换行符不匹配，因此如果没有此标记，则.标记将不会跨越行边界。

**HS_FLAG_MULTILINE**
编译标志：设置多行锚定。

此标志指示表达式使^和$标记匹配换行符以及流的开始和结束。如果未指定此标志，则^令牌将仅在流的开头匹配，并且$令牌将仅在PCRE规范的指导内的流的末尾匹配。

**HS_FLAG_SINGLEMATCH**
编译标志：设置仅匹配模式。

此标志将表达式的匹配ID设置为最多匹配一次。在流模式下，这意味着表达式将在流的生命周期内仅返回单个匹配，而不是按照标准Hyperscan语义报告每个匹配。在块模式或向量模式下，将仅返回每次调用hs_scan（）或hs_scan_vector（）的第一个匹配项。

如果数据库中的多个表达式共享相同的匹配ID，则它们必须全部指定HS_FLAG_SINGLEMATCH，或者它们都不指定HS_FLAG_SINGLEMATCH。如果共享匹配ID的一组表达式指定该标志，则每个流最多将生成一个与匹配ID匹配的匹配。

注意：目前不支持将此标志与HS_FLAG_SOM_LEFTMOST结合使用。

**HS_FLAG_ALLOWEMPTY**
编译标志：允许可以匹配空缓冲区的表达式。

此标志指示编译器允许，可以匹配对空的缓冲器，如表情.?，.*，(a|)。由于Hyperscan可以返回表达式的每个可能的匹配，因此这些表达式通常执行得非常慢; 默认行为是在尝试编译一个时返回错误。使用此标志将强制编译器允许这样的表达式。

**HS_FLAG_UTF8**
编译标志：为此表达式启用UTF-8模式。

此标志指示Hyperscan将模式视为UTF-8字符序列。使用此标志使用一个或多个模式编译的Hyperscan库扫描无效UTF-8序列的结果未定义。

**HS_FLAG_UCP**
编译标志：为此表达式启用Unicode属性支持。

此标志指示Hyperscan使用Unicode属性，而不是默认的ASCII解释，字符助记符像\w和\s还有POSIX字符类。它仅与HS_FLAG_UTF8一起使用才有意义。

**HS_FLAG_PREFILTER**
编译标志：为此表达式启用预过滤模式。

此标志指示Hyperscan编译此模式的“近似”版本以用于预过滤应用程序，即使Hyperscan在正常操作中不支持该模式也是如此。

使用此标志时返回的匹配集保证是非预先过滤表达式指定的匹配的超集。

如果模式包含Hyperscan不支持的模式构造（例如零宽度断言，反向引用或条件引用），则这些构造将在内部被更广泛的构造替换，这些构造可能更频繁地匹配。

此外，在预过滤模式中，Hyperscan可以简化在编译时或者出于性能原因（在上面的匹配保证的情况下）返回“模式太大”错误的模式。

通常预期应用程序随后将确认预过滤器与另一个可以为模式提供精确匹配的正则表达式匹配器匹配。

注意：目前不支持将此标志与HS_FLAG_SOM_LEFTMOST结合使用。

**HS_FLAG_SOM_LEFTMOST**
编译标志：启用最左侧的匹配报告开始。

此标志指示Hyperscan在报告此表达式的匹配时报告最左侧可能的匹配偏移开始。（默认情况下，不返回匹配开始。）

启用此行为可能会降低性能并增加流模式下的流状态要求。

**HS_FLAG_COMBINATION**
编译标志：逻辑组合。

此标志指示Hyperscan将此表达式解析为逻辑组合语法。逻辑约束由操作数，运算符和括号组成。操作数是表达式索引，运算符可以是'！'（NOT），'＆'（AND）或'|'（OR）。例如：（101＆102＆103）|（104＆！105）（（301 | 302）＆303）＆（304 | 305）

**HS_FLAG_QUIET**
编译标志：不做任何匹配报告。

此标志指示Hyperscan忽略此表达式的匹配报告。它被设计用于逻辑组合中的子表达式。

**HS_CPU_FEATURES_AVX2**
CPU功能标志 - 英特尔（R）高级矢量扩展2（英特尔（R）AVX2）

设置此标志表示目标平台支持AVX2指令。

**HS_CPU_FEATURES_AVX512**
CPU功能标志 - 英特尔（R）高级矢量扩展512（英特尔（R）AVX512）

设置此标志表示目标平台支持AVX512指令，特别是AVX-512BW。使用AVX512意味着使用AVX2。

**HS_TUNE_FAMILY_GENERIC**
调整参数 - 通用

这表明不应针对任何特定目标平台调整已编译的数据库。

**HS_TUNE_FAMILY_SNB**
调整参数 - 英特尔（R）微体系结构代码名称Sandy Bridge

这表明编译的数据库应该针对Sandy Bridge微体系结构进行调整。

**HS_TUNE_FAMILY_IVB**
调整参数 - 英特尔（R）微体系结构代码名称Ivy Bridge

这表明编译的数据库应该针对Ivy Bridge微体系结构进行调整。

**HS_TUNE_FAMILY_HSW**
调整参数 - 英特尔（R）微体系结构代码名称Haswell

这表明编译的数据库应该针对Haswell微体系结构进行调整。

**HS_TUNE_FAMILY_SLM**
调整参数 - 英特尔（R）微体系结构代码名称Silvermont

这表明编译的数据库应该针对Silvermont微体系结构进行调整。

**HS_TUNE_FAMILY_BDW**
调整参数 - 英特尔（R）微体系结构代码名称Broadwell

这表明编译的数据库应该针对Broadwell微体系结构进行调整。

**HS_TUNE_FAMILY_SKL**
调整参数 - 英特尔（R）微体系结构代码名称Skylake

这表明编译的数据库应该针对Skylake微体系结构进行调整。

**HS_TUNE_FAMILY_SKX**
调整参数 - 英特尔（R）微体系结构代码名称Skylake Server

这表明应该针对Skylake Server微体系结构调整已编译的数据库。

**HS_TUNE_FAMILY_GLM**
调整参数 - 英特尔（R）微体系结构代码名称Goldmont

这表明编译的数据库应该针对Goldmont微体系结构进行调整。

**HS_MODE_BLOCK**
编译器模式标志：块扫描（非流式）数据库。

**HS_MODE_NOSTREAM**
编译器模式标志：HS_MODE_BLOCK的别名。

**HS_MODE_STREAM**
编译模式标志：流数据库。

**HS_MODE_VECTORED**
编译器模式标志：向量扫描数据库。

**HS_MODE_SOM_HORIZON_LARGE**
编译器模式标志：使用全精度跟踪流状态下匹配偏移的开始。

此模式将使用每个模式的最流状态，但始终会返回准确的匹配偏移开始，而不管过去发现的距离有多远。

必须选择其中一个SOM_HORIZON模式才能使用HS_FLAG_SOM_LEFTMOST表达式标志。

**HS_MODE_SOM_HORIZON_MEDIUM**
编译器模式标志：使用中等精度跟踪流状态下匹配偏移的开始。

此模式将使用比HS_MODE_SOM_HORIZON_LARGE更少的流状态，并将匹配准确度的开始限制为报告的匹配结束偏移的2 ^ 32字节内的偏移。

必须选择其中一个SOM_HORIZON模式才能使用HS_FLAG_SOM_LEFTMOST表达式标志。

**HS_MODE_SOM_HORIZON_SMALL**
编译器模式标志：使用有限精度来跟踪流状态中匹配偏移的开始。

此模式将使用比HS_MODE_SOM_HORIZON_LARGE更少的流状态，并将匹配精度的开始限制为报告的匹配结束偏移的2 ^ 16字节内的偏移。

必须选择其中一个SOM_HORIZON模式才能使用HS_FLAG_SOM_LEFTMOST表达式标志。

### 类型定义

typedef的结构hs_compile_error hs_compile_error_t
包含失败时编译调用（hs_compile（），hs_compile_multi（）和hs_compile_ext_multi（））返回的错误详细信息的类型。调用者可以检查此类型返回的值以确定失败的原因。

编译过程中产生的常见错误包括：

无效的参数

编译调用中指定了无效参数。

无法识别的旗帜

在flags参数中传递了无法识别的值。

模式匹配空缓冲区

默认情况下，Hyperscan仅支持始终消耗至少一个输入字节的模式。/(abc)?/除非提供HS_FLAG_ALLOWEMPTY标志，否则不具有此属性（例如）的模式将产生此错误。请注意，扫描时，此类模式将为每个字节生成匹配项。

嵌入式锚点不受支持

Hyperscan仅支持在模式中使用锚元字符（例如^和$），它们只能在缓冲区的开头或结尾处匹配。包含嵌入式锚点的模式（例如）/abc^def/永远不会匹配，因为无法abc在数据流的开始之前进行操作。

有界重复太大了

该模式包含具有非常大的有限边界的重复构造。

不支持的组件类型

在模式中使用了不支持的PCRE构造。

无法生成字节码

此错误表示Hyperscan无法编译语法上有效的模式。最常见的原因是一个非常长且复杂的模式或包含大型重复子模式的模式。

无法分配内存

该库无法分配编译期间使用的临时存储。

分配器返回未对齐的内存

内存分配器（malloc（）或使用hs_set_allocator（）设置的分配器）未正确返回适合此平台上最大可表示数据类型的内存。

内部错误

发生意外错误：如果报告此错误，请联系Hyperscan团队并提供相关情况说明。

typedef的结构hs_platform_info hs_platform_info_t
包含目标平台信息的类型，可以选择提供给编译调用（hs_compile（），hs_compile_multi（），hs_compile_ext_multi（））。

甲hs_platform_info结构可以通过使用被填充为当前平台hs_populate_platform（）调用。

typedef的结构hs_expr_info hs_expr_info_t
包含与hs_expression_info（）或hs_expression_ext_info返回的表达式相关的信息的类型。

typedef的结构hs_expr_ext hs_expr_ext_t
包含与表达式相关的其他参数的结构，在构建时传递给hs_compile_ext_multi（）或hs_expression_ext_info。

这些参数允许模式生成的匹配集在编译时受到约束，而不是依赖于应用程序在运行时处理不需要的匹配。

### 函数

hs_error_t hs_compile（ const char *  表达式，unsigned int  标志，unsigned int  模式，const hs_platform_info_t *  platform，hs_database_t **  db，hs_compile_error_t **  错误）
基本的正则表达式编译器。

这是一个函数调用，使用该函数调用将表达式编译到Hyperscan数据库中，该数据库可以传递给运行时函数（例如hs_scan（），hs_open_stream（）等）。

**返回**
成功编译后返回 HS_SUCCESS ; HS_COMPILER_ERROR失败，错误参数中提供了详细信息。
**参数**
- expression：要解析的以NULL结尾的表达式。请注意，此字符串必须仅表示要匹配的模式，没有分隔符或标志; 应使用flags参数指定任何全局标志。例如，表达/abc?def/i应该通过提供被编译abc?def为expression，和HS_FLAG_CASELESS作为标志。
- flags：标志，用于修改表达式的行为。通过将它们“或”在一起可以使用多个标志。有效值为：
HS_FLAG_CASELESS - 匹配将不区分大小写。
HS_FLAG_DOTALL - 匹配a .不会排除换行符。
HS_FLAG_MULTILINE - ^和$锚点匹配数据中的任何换行符。
HS_FLAG_SINGLEMATCH - 每个流只为表达式生成一个匹配项。
HS_FLAG_ALLOWEMPTY - 允许可以匹配空字符串的表达式，例如.*。
HS_FLAG_UTF8 - 将此模式视为UTF-8字符序列。
HS_FLAG_UCP - 对字符类使用Unicode属性。
HS_FLAG_PREFILTER - 在预过滤模式下编译模式。
HS_FLAG_SOM_LEFTMOST - 在找到匹配项时报告最左侧的匹配偏移开始。
- mode：编译器模式标志，作为一个整体影响数据库。一个HS_MODE_STREAM或HS_MODE_BLOCK或HS_MODE_VECTORED必须提供，流，块或向量数据库的生成之间进行选择。此外，可以提供其他标志（以HS_MODE_开头）以启用特定功能。有关详细信息，请参阅编译模式标志。
- platform：如果不为NULL，则使用平台结构来确定数据库的目标平台。如果为NULL，则生成适合在当前主机平台上运行的数据库。
- db：成功时，将在此参数中返回指向生成的数据库的指针，或者在失败时返回NULL。调用者负责使用hs_free_database（）函数释放缓冲区。
- error：如果编译失败，将返回指向hs_compile_error_t的指针，提供错误条件的详细信息。调用者负责使用hs_free_compile_error（）函数释放缓冲区。

hs_error_t hs_compile_multi（ const char * const *  表达式，const unsigned int *  flags，const unsigned int *  ids，unsigned int  elements，unsigned int  mode，const hs_platform_info_t *  platform，hs_database_t **  db，hs_compile_error_t **  error ）
多个正则表达式编译器。

这是一个函数调用，使用该函数调用将一组表达式编译到数据库中，该数据库可以传递给运行时函数（例如hs_scan（），hs_open_stream（）等）。每个表达式都可以用唯一的整数标记，该整数被传递进入匹配回调以识别匹配的模式。

**返回**
成功编译后返回 HS_SUCCESS ; HS_COMPILER_ERROR失败，error参数中提供了详细信息。
**参数**
- expressions：要编译的以NULL结尾的表达式数组。请注意（对于hs_compile（）），这些字符串必须仅包含要匹配的模式，不包含分隔符或标志。例如，/abc?def/i应该通过提供数组中abc?def的第一个字符串expressions并将HS_FLAG_CASELESS作为数组中的第一个值来编译表达式flags。
- flags：一组标志，用于修改每个表达式的行为。通过将它们“或”在一起可以使用多个标志。指定NULL指针代替数组会将所有模式的flags值设置为零。有效值为：
HS_FLAG_CASELESS - 匹配将不区分大小写。
HS_FLAG_DOTALL - 匹配a .不会排除换行符。
HS_FLAG_MULTILINE - ^和$锚点匹配数据中的任何换行符。
HS_FLAG_SINGLEMATCH - 每个流具有此匹配ID的模式只会生成一个匹配项。
HS_FLAG_ALLOWEMPTY - 允许可以匹配空字符串的表达式，例如.*。
HS_FLAG_UTF8 - 将此模式视为UTF-8字符序列。
HS_FLAG_UCP - 对字符类使用Unicode属性。
HS_FLAG_PREFILTER - 在预过滤模式下编译模式。
HS_FLAG_SOM_LEFTMOST - 在找到匹配项时报告最左侧的匹配偏移开始。
- ids：整数数组，指定要与表达式数组中的相应模式关联的ID号。指定NULL指针代替数组会将所有模式的ID值设置为零。
elements：输入数组中的元素数。
- mode：编译器模式标志，作为一个整体影响数据库。一个HS_MODE_STREAM或HS_MODE_BLOCK或HS_MODE_VECTORED必须提供，流，块或向量数据库的生成之间进行选择。此外，可以提供其他标志（以HS_MODE_开头）以启用特定功能。有关详细信息，请参阅编译模式标志。
- platform：如果不为NULL，则使用平台结构来确定数据库的目标平台。如果为NULL，则生成适合在当前主机平台上运行的数据库。
- db：成功时，将在此参数中返回指向生成的数据库的指针，或者在失败时返回NULL。调用者负责使用hs_free_database（）函数释放缓冲区。
- error：如果编译失败，将返回指向hs_compile_error_t的指针，提供错误条件的详细信息。调用者负责使用hs_free_compile_error（）函数释放缓冲区。

hs_error_t hs_compile_ext_multi（ const char * const *  表达式，const unsigned int *  flags，const unsigned int *  ids，const hs_expr_ext_t * const *  ext，unsigned int  elements，unsigned int  mode，const hs_platform_info_t *  platform，hs_database_t **  db，hs_compile_error_t **  error ）
具有扩展参数支持的多个正则表达式编译器。

此函数调用以与hs_compile_multi（）相同的方式将一组表达式编译到数据库中，但允许通过每个表达式的hs_expr_ext_t结构指定其他参数。

**返回**
成功编译后返回 HS_SUCCESS ; HS_COMPILER_ERROR失败，error参数中提供了详细信息。
**参数**
expressions：要编译的以NULL结尾的表达式数组。请注意（对于hs_compile（）），这些字符串必须仅包含要匹配的模式，不包含分隔符或标志。例如，/abc?def/i应该通过提供数组中abc?def的第一个字符串expressions并将HS_FLAG_CASELESS作为数组中的第一个值来编译表达式flags。
- flags：一组标志，用于修改每个表达式的行为。通过将它们“或”在一起可以使用多个标志。指定NULL指针代替数组会将所有模式的flags值设置为零。有效值为：
HS_FLAG_CASELESS - 匹配将不区分大小写。
HS_FLAG_DOTALL - 匹配a .不会排除换行符。
HS_FLAG_MULTILINE - ^和$锚点匹配数据中的任何换行符。
HS_FLAG_SINGLEMATCH - 每个流具有此匹配ID的模式只会生成一个匹配项。
HS_FLAG_ALLOWEMPTY - 允许可以匹配空字符串的表达式，例如.*。
HS_FLAG_UTF8 - 将此模式视为UTF-8字符序列。
HS_FLAG_UCP - 对字符类使用Unicode属性。
HS_FLAG_PREFILTER - 在预过滤模式下编译模式。
HS_FLAG_SOM_LEFTMOST - 在找到匹配项时报告最左侧的匹配偏移开始。
- ids：整数数组，指定要与表达式数组中的相应模式关联的ID号。指定NULL指针代替数组会将所有模式的ID值设置为零。
- ext：填充hs_expr_ext_t结构的指针数组，用于定义每个模式的扩展行为。如果单个模式不需要扩展行为，则可以指定NULL，如果任何表达式不需要扩展行为，则可以指定整个数组。这些结构使用的内存必须由调用者分配和释放。
- elements：输入数组中的元素数。
- mode：编译器模式标志，作为一个整体影响数据库。之一的HS_MODE_STREAM，HS_MODE_BLOCK或HS_MODE_VECTORED必须提供，流，块或向量数据库的生成之间进行选择。此外，可以提供其他标志（以HS_MODE_开头）以启用特定功能。有关详细信息，请参阅编译模式标志。
- platform：如果不为NULL，则使用平台结构来确定数据库的目标平台。如果为NULL，则生成适合在当前主机平台上运行的数据库。
- db：成功时，将在此参数中返回指向生成的数据库的指针，或者在失败时返回NULL。调用者负责使用hs_free_database（）函数释放缓冲区。
- error：如果编译失败，将返回指向hs_compile_error_t的指针，提供错误条件的详细信息。调用者负责使用hs_free_compile_error（）函数释放缓冲区。

hs_error_t hs_free_compile_error（hs_compile_error_t *  错误）
释放由hs_compile（），hs_compile_multi（）或hs_compile_ext_multi（）生成的错误结构。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- error：要释放的hs_compile_error_t。也可以安全地提供NULL。

hs_error_t hs_expression_info（ const char *  表达式，unsigned int  标志，hs_expr_info_t **  info，hs_compile_error_t **  错误）
实用功能提供有关正则表达式的信息。hs_expr_info_t中提供的信息包括模式匹配的最小和最大宽度。

注意：使用此函数成功分析表达式并不意味着编译同一表达式（通过hs_compile（），hs_compile_multi（）或hs_compile_ext_multi（））将成功。对于Hyperscan无法编译的正则表达式，此函数可能返回HS_SUCCESS。

注意：此调用接受一些每模式标志（例如HS_FLAG_ALLOWEMPTY，HS_FLAG_SOM_LEFTMOST），但由于它们不影响hs_expr_info_t结构中返回的属性，因此它们不会影响此函数的结果。

**返回**
成功编译后返回 HS_SUCCESS ; HS_COMPILER_ERROR失败，错误参数中提供了详细信息。
**参数**
- expression：要解析的以NULL结尾的表达式。请注意，此字符串必须仅表示要匹配的模式，没有分隔符或标志; 应使用flags参数指定任何全局标志。例如，表达/abc?def/i应该通过提供被编译abc?def为expression，和HS_FLAG_CASELESS作为标志。
- flags：标志，用于修改表达式的行为。通过将它们“或”在一起可以使用多个标志。有效值为：
HS_FLAG_CASELESS - 匹配将不区分大小写。
HS_FLAG_DOTALL - 匹配a .不会排除换行符。
HS_FLAG_MULTILINE - ^和$锚点匹配数据中的任何换行符。
HS_FLAG_SINGLEMATCH - 每个流表达式只生成一个匹配项。
HS_FLAG_ALLOWEMPTY - 允许可以匹配空字符串的表达式，例如.*。
HS_FLAG_UTF8 - 将此模式视为UTF-8字符序列。
HS_FLAG_UCP - 对字符类使用Unicode属性。
HS_FLAG_PREFILTER - 在预过滤模式下编译模式。
HS_FLAG_SOM_LEFTMOST - 在找到匹配项时报告最左侧的匹配偏移开始。
- info：成功时，将在此参数中返回指向模式信息的指针，或者在失败时返回NULL。使用hs_set_allocator（）中提供的分配器（或者如果未设置分配器，则为 malloc（））分配此结构，并且应由调用者释放。
- error：如果调用失败，将返回指向hs_compile_error_t的指针，提供错误条件的详细信息。调用者负责使用hs_free_compile_error（）函数释放缓冲区。

hs_error_t hs_expression_ext_info（ const char *  表达式，unsigned int  标志，const hs_expr_ext_t *  ext，hs_expr_info_t **  info，hs_compile_error_t **  错误）
实用程序功能提供有关正则表达式的信息，具有扩展参数支持。hs_expr_info_t中提供的信息包括模式匹配的最小和最大宽度。

注意：使用此函数成功分析表达式并不意味着编译同一表达式（通过hs_compile（），hs_compile_multi（）或hs_compile_ext_multi（））将成功。对于Hyperscan无法编译的正则表达式，此函数可能返回HS_SUCCESS。

注意：此调用接受一些每模式标志（例如HS_FLAG_ALLOWEMPTY，HS_FLAG_SOM_LEFTMOST），但由于它们不影响hs_expr_info_t结构中返回的属性，因此它们不会影响此函数的结果。

**返回**
成功编译后返回 HS_SUCCESS ; HS_COMPILER_ERROR失败，错误参数中提供了详细信息。
**参数**
- expression：要解析的以NULL结尾的表达式。请注意，此字符串必须仅表示要匹配的模式，没有分隔符或标志; 应使用flags参数指定任何全局标志。例如，表达/abc?def/i应该通过提供被编译abc?def为expression，和HS_FLAG_CASELESS作为标志。
- flags：标志，用于修改表达式的行为。通过将它们“或”在一起可以使用多个标志。有效值为：
HS_FLAG_CASELESS - 匹配将不区分大小写。
HS_FLAG_DOTALL - 匹配a .不会排除换行符。
HS_FLAG_MULTILINE - ^和$锚点匹配数据中的任何换行符。
HS_FLAG_SINGLEMATCH - 每个流表达式只生成一个匹配项。
HS_FLAG_ALLOWEMPTY - 允许可以匹配空字符串的表达式，例如.*。
HS_FLAG_UTF8 - 将此模式视为UTF-8字符序列。
HS_FLAG_UCP - 对字符类使用Unicode属性。
HS_FLAG_PREFILTER - 在预过滤模式下编译模式。
HS_FLAG_SOM_LEFTMOST - 在找到匹配项时报告最左侧的匹配偏移开始。
- ext：指向已填充的hs_expr_ext_t结构的指针，该结构定义此模式的扩展行为。如果不需要扩展参数，则可以指定NULL。
- info：成功时，将在此参数中返回指向模式信息的指针，或者在失败时返回NULL。使用hs_set_allocator（）中提供的分配器（或者如果未设置分配器，则为 malloc（））分配此结构，并且应由调用者释放。
- error：如果调用失败，将返回指向hs_compile_error_t的指针，提供错误条件的详细信息。调用者负责使用hs_free_compile_error（）函数释放缓冲区。

hs_error_t hs_populate_platform（hs_platform_info_t *  platform ）
根据当前主机填充平台信息。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- platform：成功时，将根据当前主机填充指向的结构。

struct hs_compile_error
#include <hs_compile.h>
包含失败时编译调用（hs_compile（），hs_compile_multi（）和hs_compile_ext_multi（））返回的错误详细信息的类型。调用者可以检查此类型返回的值以确定失败的原因。

编译过程中产生的常见错误包括：

- 无效的参数

编译调用中指定了无效参数。

- 无法识别的旗帜

在flags参数中传递了无法识别的值。

- 模式匹配空缓冲区

默认情况下，Hyperscan仅支持始终消耗至少一个输入字节的模式。/(abc)?/除非提供HS_FLAG_ALLOWEMPTY标志，否则不具有此属性（例如）的模式将产生此错误。请注意，扫描时，此类模式将为每个字节生成匹配项。

- 嵌入式锚点不受支持

Hyperscan仅支持在模式中使用锚元字符（例如^和$），它们只能在缓冲区的开头或结尾处匹配。包含嵌入式锚点的模式（例如）/abc^def/永远不会匹配，因为无法abc在数据流的开始之前进行操作。

- 有界重复太大了

该模式包含具有非常大的有限边界的重复构造。

- 不支持的组件类型

在模式中使用了不支持的PCRE构造。

- 无法生成字节码

此错误表示Hyperscan无法编译语法上有效的模式。最常见的原因是一个非常长且复杂的模式或包含大型重复子模式的模式。

- 无法分配内存

该库无法分配编译期间使用的临时存储。

- 分配器返回未对齐的内存

内存分配器（malloc（）或使用hs_set_allocator（）设置的分配器）未正确返回适合此平台上最大可表示数据类型的内存。

- 内部错误

发生意外错误：如果报告此错误，请联系Hyperscan团队并提供相关情况说明。

**公开成员变量**

char * **message**
描述错误的人类可读错误消息。

INT **expression**
导致错误的表达式的从零开始的数字（如果可以确定）。如果错误不是特定于表达式，则此值将小于零。

struct hs_platform_info
#include <hs_compile.h>
包含目标平台信息的类型，可以选择提供给编译调用（hs_compile（），hs_compile_multi（），hs_compile_ext_multi（））。

甲hs_platform_info结构可以通过使用被填充为当前平台hs_populate_platform（）调用。

**公开成员变量**

unsigned int类型tune
有关目标平台的信息，可用于指导编译的优化过程。

使用此字段不会限制生成的数据库可以运行的处理器，但可能会影响生成的数据库的性能。

unsigned cpu_features
目标平台上提供的相关CPU功能

可以通过组合HS_CPU_FEATURE_ *标志（例如HS_CPU_FEATURES_AVX2）来产生该值。可以将多个CPU功能组合在一起以产生该值。

unsigned reserved1
保留供将来使用。

unsigned reserved2
保留供将来使用。

struct hs_expr_info
#include <hs_compile.h>
包含与hs_expression_info（）或hs_expression_ext_info返回的表达式相关的信息的类型。

**公开成员变量**

unsigned int类型min_width
模式匹配的最小长度（以字节为单位）。

注意：在某些情况下，当使用高级功能来抑制匹配（例如扩展参数或HS_FLAG_SINGLEMATCH标志）时，这可能代表匹配的真实最小长度的保守下限。

unsigned int类型max_width
模式匹配的最大长度（以字节为单位）。如果模式具有无限制的最大长度，则将其设置为unsigned int（UINT_MAX）的最大值。

注意：在某些情况下，当使用高级功能来抑制匹配（例如扩展参数或HS_FLAG_SINGLEMATCH标志）时，这可能表示匹配的真实最大长度的保守上限。

焦unordered_matches
此表达式是否可以生成未按顺序返回的匹配项，例如断言生成的匹配项。如果为false则为零，如果为true则为非零。

焦matches_at_eod
此表达式是否可以在数据末尾（EOD）生成匹配项。在流模式下，在hs_close_stream（）期间引发 EOD匹配，因为只有在调用hs_close_stream（）时才知道 EOD位置。如果为false则为零，如果为true则为非零。

注意：尾随\b字边界断言也可能导致EOD匹配，因为数据结束可以充当字边界。

焦matches_only_at_eod
此表达式是否只能在数据末尾（EOD）生成匹配项。在流模式下，此表达式的所有匹配都在hs_close_stream（）期间引发。如果为false则为零，如果为true则为非零。

struct hs_expr_ext
#include <hs_compile.h>
包含与表达式相关的其他参数的结构，在构建时传递给hs_compile_ext_multi（）或hs_expression_ext_info。

这些参数允许模式生成的匹配集在编译时受到约束，而不是依赖于应用程序在运行时处理不需要的匹配。

**公开成员变量**

unsigned flags
控制编译器将使用此结构的哪些部分的标志。请参阅hs_expr_ext_t标志。

unsigned min_offset
数据流中此表达式应成功匹配的最小结束偏移量。要使用此参数，请在hs_expr_ext :: flags字段中设置HS_EXT_FLAG_MIN_OFFSET标志。

unsigned max_offset
此表达式应成功匹配的数据流中的最大结束偏移量。要使用此参数，请在hs_expr_ext :: flags字段中设置HS_EXT_FLAG_MAX_OFFSET标志。

unsigned min_length
成功匹配此表达式所需的最小匹配长度（从开始到结束）。要使用此参数，请在hs_expr_ext :: flags字段中设置HS_EXT_FLAG_MIN_LENGTH标志。

无符号edit_distance
允许模式在此编辑距离内大致匹配。要使用此参数，请在hs_expr_ext :: flags字段中设置HS_EXT_FLAG_EDIT_DISTANCE标志。

无符号hamming_distance
允许模式在此汉明距离内大致匹配。要使用此参数，请在hs_expr_ext :: flags字段中设置HS_EXT_FLAG_HAMMING_DISTANCE标志。

## 文件：hs_runtime.h
Hyperscan运行时API定义。

Hyperscan是一种高速正则表达式引擎。

此标头包含使用已编译的Hyperscan数据库在运行时扫描数据的功能。

定义

**HS_OFFSET_PAST_HORIZON**
回调'来自'返回值，表示此匹配的开始太早，无法使用请求的SOM_HORIZON精度进行跟踪。

### 类型定义

typedef的结构hs_stream hs_stream_t
hs_open_stream（）返回的流标识符。

typedef的结构hs_scratch hs_scratch_t
Hyperscan临时空间。

typedef int ( * match_event_handler)（ unsigned int  id，unsigned long long  from，unsigned long long  to，unsigned int  flags，void  * context ）
匹配事件回调函数类型的定义。

调用hs_scan（），hs_scan_vector（）或hs_scan_stream（）函数（或其他可以产生匹配的流调用）的应用程序必须提供与定义类型匹配的回调函数。

只要在执行扫描期间匹配位于目标数据中，就会调用此回调函数。匹配的详细信息作为参数传递给回调函数，回调函数应返回一个值，指示是否应继续匹配目标数据。如果扫描调用不需要回调，则可以提供NULL以抑制匹配生成。

此回调函数不应尝试在同一个流上调用Hyperscan API函数，也不应尝试重用为导致触发它的API调用分配的临时空间。使用完全独立的参数再次调用Hyperscan库应该可以工作（例如，在新流中扫描不同的数据库并使用新的临时空间），但重用流状态和/或暂存空间等数据结构将产生未定义的行为。

**返回**
如果匹配应该停止，则为非零，否则为零。如果在流模式下执行扫描并返回非零值，则对该流的任何后续hs_scan_stream（）调用将立即返回HS_SCAN_TERMINATED。
**参数**
- id：匹配的表达式的ID号。如果表达式是使用hs_compile（）编译的单个表达式，则此值将为零。
- from：
	- 如果为当前模式启用了匹配开始标志，则该参数将设置为模式的匹配开始，假设匹配值的开始位于由SOM_HORIZON模式之一选择的当前“匹配范围的开始”内标志。
如果匹配开始值位于此范围之外（仅在SOM_HORIZON值不是HS_MODE_SOM_HORIZON_LARGE时可能），则该from值将设置为HS_OFFSET_PAST_HORIZON。
	- 如果未对给定模式启用匹配开始标志，则此参数将设置为零。
- to：与表达式匹配的最后一个字节后的偏移量。
- flags：这是供将来使用的，目前尚未使用。
- context：用户提供给hs_scan（），hs_scan_vector（）或hs_scan_stream（）函数的指针。

### 函数

hs_error_t hs_open_stream（ const hs_database_t *  db，unsigned int  flags，hs_stream_t **  stream ）
打开并初始化流。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- db：已编译的模式数据库。
- flags：标志修改流的行为。此参数仅供将来使用，目前尚未使用。
- stream：成功时，将返回指向生成的hs_stream_t的指针; 失败时为NULL。

hs_error_t hs_scan_stream（hs_stream_t *  id，const char *  data，unsigned int  length，unsigned int  flags，hs_scratch_t *  scratch，match_event_handler  onEvent，void *  ctxt ）
将要扫描的数据写入打开的流。

这是函数调用，其中实际模式匹配在数据写入流时发生。匹配将通过提供的match_event_handler回调返回。

**返回**
成功时返回HS_SUCCESS ; HS_SCAN_TERMINATED如果匹配回调表明扫描应该停止; 其他错误值。
**参数**
- id：将写入数据的流ID（由hs_open_stream（）返回）。
- data：指向要扫描的数据的指针。
- length：要扫描的字节数。
- flags：标志修改流的行为。此参数仅供将来使用，目前尚未使用。
- scratch：由hs_alloc_scratch（）分配的每线程暂存空间。
- onEvent：指向匹配事件回调函数的指针。如果给出NULL指针，则不返回任何匹配项。
- ctxt：用户定义的指针，当匹配发生时将传递给回调函数。

hs_error_t hs_close_stream（hs_stream_t *  id，hs_scratch_t *  scratch，match_event_handler  onEvent，void *  ctxt ）
关闭一条小溪。

此函数完成对给定流的匹配，并释放与流状态关联的内存。在此调用之后，指向的流id无效，无法再使用。要在完成后重用流状态，而不是关闭它，可以使用hs_reset_stream函数。

必须为使用hs_open_stream（）创建的任何流调用此函数，即使扫描已被匹配回调函数的非零返回终止。

注意：对于锚定到数据流末尾的表达式（例如，通过使用$元字符），此操作可能导致返回匹配（通过调用匹配事件回调）。如果不需要这些匹配，则可以提供NULL作为match_event_handler回调。

如果提供NULL作为match_event_handler回调，则允许提供NULL暂存。

**返回**
成功时返回HS_SUCCESS，失败时返回其他值。
**参数**
- id：hs_open_stream（）返回的流ID 。
- scratch：由hs_alloc_scratch（）分配的每线程暂存空间。仅当onEvent回调也为NULL时，才允许为NULL。
- onEvent：指向匹配事件回调函数的指针。如果给出NULL指针，则不返回任何匹配项。
- ctxt：用户定义的指针，当匹配发生时将传递给回调函数。

hs_error_t hs_reset_stream（hs_stream_t *  id，unsigned int  flags，hs_scratch_t *  scratch，match_event_handler  onEvent，void *  context ）
将流重置为初始状态。

从概念上讲，这相当于在给定流上执行hs_close_stream（），然后是hs_open_stream（）。这个新流替换了内存中的原始流，避免了释放旧流和分配新流的开销。

注意：对于锚定到原始数​​据流末尾的表达式（例如，通过使用$元字符），此操作可能导致返回匹配（通过调用匹配事件回调）。如果不需要这些匹配，则可以提供NULL作为match_event_handler回调。

注意：流也将绑定到同一个数据库。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- id：要替换的流（由hs_open_stream（）创建）。
- flags：标志修改流的行为。此参数仅供将来使用，目前尚未使用。
scratch：由hs_alloc_scratch（）分配的每线程暂存空间。仅当onEvent回调也为NULL时，才允许为NULL。
- onEvent：指向匹配事件回调函数的指针。如果给出NULL指针，则不返回任何匹配项。
- context：用户定义的指针，当匹配发生时将传递给回调函数。

hs_error_t hs_copy_stream（hs_stream_t **  to_id，const hs_stream_t *  from_id ）
复制给定的流。新流将具有与原始流相同的状态，包括当前流偏移。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- to_id：成功时，将返回指向新的复制hs_stream_t的指针; 失败时为NULL。
- from_id：要复制的流（由hs_open_stream（）创建）。

hs_error_t hs_reset_and_copy_stream（hs_stream_t *  to_id，const hs_stream_t *  from_id，hs_scratch_t *  scratch，match_event_handler  onEvent，void *  context ）
将给定的“from”流状态复制到“to”流。首先重置'to'流（如果提供了非NULL onEvent回调处理程序，则报告任何EOD匹配）。

注意：必须针对同一数据库打开'to'流和'from'流。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- to_id：成功时，将返回指向新的复制hs_stream_t的指针; 失败时为NULL。
- from_id：要复制的流（由hs_open_stream（）创建）。
scratch：由hs_alloc_scratch（）分配的每线程暂存空间。仅当onEvent回调也为NULL时，才允许为NULL。
- onEvent：指向匹配事件回调函数的指针。如果给出NULL指针，则不返回任何匹配项。
- context：用户定义的指针，当匹配发生时将传递给回调函数。

hs_error_t hs_compress_stream（ const hs_stream_t *  stream，char *  buf，size_t  buf_space，size_t *  used_space ）
在提供的缓冲区中创建提供的流的压缩表示。可以使用hs_expand_stream（）或hs_reset_and_expand_stream（）将此压缩表示转换回流状态。压缩表示的大小将被放入used_space。

如果缓冲区中没有足够的空间来保存压缩表示，则将返回HS_INSUFFICIENT_SPACE，used_space并将填充所需的空间量。

注意：此函数不会关闭提供的流，您可以继续使用流或使用hs_close_stream（）释放它。

**返回**
HS_SUCCESS成功， HS_INSUFFICIENT_SPACE，如果提供的缓冲区太小。
**参数**
- stream：要压缩的流（由hs_open_stream（）创建）。
- buf：Buffer将压缩表示写入。注意：如果调用仅用于确定所需的空间量，则允许在此处传递NULL并buf_space为0。
- buf_space：中的字节数buf。如果buf_space太小，则调用将因HS_INSUFFICIENT_SPACE而失败。
- used_space：指向将使用的空间量写入的位置的指针。使用的缓冲区空间始终小于或等于buf_space。如果调用因HS_INSUFFICIENT_SPACE而失败，则该指针将用于写出所需的缓冲区空间量。

hs_error_t hs_expand_stream（ const hs_database_t *  db，hs_stream_t **  stream，const char *  buf，size_t  buf_size ）
将hs_compress_stream（）创建的压缩表示解压缩到新流中。

注意：buf必须对应于创建一个完整的压缩表示hs_compress_stream（）这是对未流的db。如果不满足这些属性，则无法始终检测到此API的滥用行为并且未定义行为。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- db：打开压缩流的已编译模式数据库。
- stream：成功时，将返回指向展开的hs_stream_t的指针; 失败时为NULL。
- buf：流的压缩表示。这些压缩格式由hs_compress_stream（）创建。
- buf_size：压缩表示的大小（以字节为单位）。

hs_error_t hs_reset_and_expand_stream（hs_stream_t *  to_stream，const char *  buf，size_t  buf_size，hs_scratch_t *  scratch，match_event_handler  onEvent，void *  context ）
解压缩由'to'流顶部的hs_compress_stream（）创建的压缩表示。首先重置'to'流（如果提供了非NULL onEvent回调处理程序，则报告任何EOD匹配）。

注意：必须在与压缩流相同的数据库上打开'to'流。

注意：buf必须对应于创建一个完整的压缩表示hs_compress_stream（）这是对未流的db。如果不满足这些属性，则无法始终检测到此API的滥用行为并且未定义行为。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- to_stream：指向有效流状态的指针。将返回指向展开的hs_stream_t的指针; 失败时为NULL。
- buf：流的压缩表示。这些压缩格式由hs_compress_stream（）创建。
- buf_size：压缩表示的大小（以字节为单位）。
- scratch：由hs_alloc_scratch（）分配的每线程暂存空间。仅当onEvent回调也为NULL时，才允许为NULL。
- onEvent：指向匹配事件回调函数的指针。如果给出NULL指针，则不返回任何匹配项。
- context：用户定义的指针，当匹配发生时将传递给回调函数。

hs_error_t hs_scan（ const hs_database_t *  db，const char *  data，unsigned int  length，unsigned int  flags，hs_scratch_t *  scratch，match_event_handler  onEvent，void *  context ）
块（非流式）正则表达式扫描程序。

这是函数调用，其中实际模式匹配发生在块模式模式数据库中。

**返回**
成功时返回HS_SUCCESS ; HS_SCAN_TERMINATED如果匹配回调表明扫描应该停止; 其他错误值。
**参数**
- db：已编译的模式数据库。
- data：指向要扫描的数据的指针。
- length：要扫描的字节数。
- flags：标志修改此函数的行为。此参数仅供将来使用，目前尚未使用。
- scratch：由hs_alloc_scratch（）为此数据库分配的每线程临时空间。
- onEvent：指向匹配事件回调函数的指针。如果给出NULL指针，则不返回任何匹配项。
- context：用户定义的指针，将传递给回调函数。

hs_error_t hs_scan_vector（ const hs_database_t *  db，const char * const *  data，const unsigned int *  length，unsigned int  count，unsigned int  flags，hs_scratch_t *  scratch，match_event_handler  onEvent，void *  context ）
矢量正则表达式扫描仪。

这是函数调用，其中实际模式匹配发生在矢量化模式模式数据库中。

**返回**
成功时返回HS_SUCCESS ; HS_SCAN_TERMINATED如果匹配回调表明扫描应该停止; 其他错误值。
**参数**
- db：已编译的模式数据库。
- data：指向要扫描的数据块的指针数组。
- length：要扫描的每个数据块的长度数组（以字节为单位）。
- count：要扫描的数据块数。这应该data与length数组的大小相对应。
- flags：标志修改此函数的行为。此参数仅供将来使用，目前尚未使用。
- scratch：由hs_alloc_scratch（）为此数据库分配的每线程临时空间。
- onEvent：指向匹配事件回调函数的指针。如果给出NULL指针，则不返回任何匹配项。
- context：用户定义的指针，将传递给回调函数。

hs_error_t hs_alloc_scratch（ const hs_database_t *  db，hs_scratch_t **  scratch ）
分配一个“划痕”空间供Hyperscan使用。

这是运行时使用所必需的，每个线程或并发调用者需要一个临时空间。此函数将使用由hs_set_scratch_allocator（）或hs_set_allocator（）设置的任何分配器回调。

**返回**
HS_SUCCESS成功分配; 如果分配失败，则为HS_NOMEM。如果指定了无效参数，则可能会返回其他错误。
**参数**
- db：数据库，由hs_compile（）生成。
- scratch：在第一次分配时，应提供指向NULL的指针，以便可以分配新的临时值。如果先前已经分配了临时块，则应该传回指向它的指针，以查看它是否对该数据库块有效。如果需要新的暂存块，则将释放原始块并返回新原始块，否则将返回先前的暂存块。成功时，除了原始临时空间适合的任何数据库之外，临时块将适用于所提供的数据库。

hs_error_t hs_clone_scratch（ const hs_scratch_t *  src，hs_scratch_t **  dest ）
分配作为现有临时空间的克隆的临时空间。

当多个并发线程将使用同一组编译数据库时，这很有用，并且需要另一个临时空间。此函数将使用由hs_set_scratch_allocator（）或hs_set_allocator（）设置的任何分配器回调。

**返回**
HS_SUCCESS成功; 如果分配失败，则为HS_NOMEM。如果指定了无效参数，则可能会返回其他错误。
**参数**
- src：要克隆的现有hs_scratch_t。
- dest：这里将返回指向新临时空间的指针。

hs_error_t hs_scratch_size（ const hs_scratch_t *  scratch，size_t *  scratch_size ）
提供给定临时空间的大小。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- scratch：由hs_alloc_scratch（）或hs_clone_scratch（）分配的每线程临时空间。
- scratch_size：成功时，临时空间的大小（以字节为单位）放在此参数中。

hs_error_t hs_free_scratch（hs_scratch_t *  scratch ）
释放先前由hs_alloc_scratch（）或hs_clone_scratch（）分配的暂存块。

此函数将使用由hs_set_scratch_allocator（）或hs_set_allocator（）设置的可用回调。

**返回**
HS_SUCCESS成功，其他值失败。
**参数**
- scratch：要释放的临时块。也可以安全地提供NULL。
