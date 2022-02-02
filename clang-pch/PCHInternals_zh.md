# PCH 内部实现

```plain
本文是对 https://clang.llvm.org/docs/PCHInternals.html 的翻译
```

本文件描述了 Clang 的预编译头（Precompiled Headers，PCH）文件和模块的设计和实现。如果你对用户接触的内容感兴趣，请参阅[用户手册](https://clang.llvm.org/docs/UsersManual.html#usersmanual-precompiled-headers)。

## 通过 Clang 使用 PCH

Clang 编译器前端，`clang -cc1`，支持两个用以生成和使用 PCH 文件的命令行选项。

如果要用 `clang -cc1` 生成 PCH 文件，使用 `-emit-pch` 选项

```bash
clang -cc1 test.h -emit-pch -o test.h.pch
```

使用 `clang` 来生成 PCH 文件的时候这个选项是透明的。生成的 PCH 文件包含完成了解析（parsing）和语义分析的，编译器内部表示的序列化形式。这个 PCH 文件可以通过 `-include-pch` 选项作为 prefix 头文件使用

```bash
clang -cc1 -include-pch test.h.pch test.c -o test.s
```

## 设计哲学

PCH 的意义在于提升整个项目的总体编译时间，所以 PCH 的设计完全是性能驱动的。PCH 的用例相对简单：当项目中有一些常用的头文件几乎被每一个源文件引用的时候，我们把这一组头文件*预编译*成一个单一的预编译头（PCH）文件。然后，当编译项目中的源代码文件的时候，我们首先加载这个 PCH 文件（作为一个 prefix 头文件），相当于引入了这一组头文件。

一个 PCH 实现在这种情况下才可以提升性能

* 加载 PCH 显著地快于重新解析 PCH 文件中包含的一组头文件。因此，PCH 的设计尝试尽可能地最小化读 PCH 文件的成本。理想情况下，这个成本不随 PCH 文件的大小而变化。
* 最初生成 PCH 文件的成本不要太大以至于抵消了对每个源代码文件单独带来的性能优化，因为我们只在这组头文件第一次出现的时候才需要解析。这对于多核系统来说尤为重要，因为只有当所有的编译都要求 PCH 文件保持最新时，PCH 文件生成才会序列化构建。

Clang 中实现的模块，使用和 PCH 一样的机制，来序列化 AST 文件（每个模块对应一个 AST 文件）和使用这些 AST。从实现的角度看，模块是 PCH 的推广，解除了对 PCH 的一些限制。特别是只能由一个 PCH 而且必须包含在翻译单元的开头。模块的 AST 文件格式拓展在[模块](https://clang.llvm.org/docs/PCHInternals.html#pchinternals-modules)的部分单独讨论。

Clang 的 AST 文件设计为一个紧密的硬盘表示，这样最小化了创建和首次加载的成本。AST 文件本身包含一个 Clang AST 的序列化表示和一些支持性数据结构，使用和 [LLVM 二进制流格式（LLVM's bitstream format）](https://llvm.org/docs/BitCodeFormat.html)相同的压缩二进制数据流保存。

Clang 的 AST 文件是从硬盘上惰性加载的。当 AST 文件首次加载时，Clang 只读取 AST 文件中的小部分数据，明确每个重要的数据结构存储在哪里。这时候读取的数据量是和 AST 文件的大小无关的，所以很大的 AST 文件并不会使用很长的 AST 加载时间。AST 文件中的实际头文件数据——宏，函数，变量，类型，等等——只有在它们在用户的代码中被引用的时候才会加载，这时只有该实体（以及该实体所依赖的实体）从 AST 文件中反序列化出来。使用这种方法，一个翻译单元使用 AST 文件的成本和它实际使用 AST 文件的代码量成正比，而不是和 AST 文件本身的大小成正比。

当使用 `-print-stats` 选项的时候，Clang 可以描述到底 AST 文件中的多少从硬盘上加载出来。对于一个使用了 Apple 的 `Cocoa.h` 头文件（已经进行预编译）的简单 HelloWorld 程序，这个选项描述出有多么少的预编译头是实际需要的

```plain
*** AST File Statistics:
  895/39981 source location entries read (2.238563%)
  19/15315 types read (0.124061%)
  20/82685 declarations read (0.024188%)
  154/58070 identifiers read (0.265197%)
  0/7260 selectors read (0.000000%)
  0/30842 statements read (0.000000%)
  4/8400 macros read (0.047619%)
  1/4995 lexical declcontexts read (0.020020%)
  0/4413 visible declcontexts read (0.000000%)
  0/7230 method pool entries read (0.000000%)
  0 method pool misses
```

对于这个简单的程序，只有很少的源代码位置、类型、声明、标识符和宏实际上从预编译头中反序列化出来了。这些统计对于判断通过实现惰性加载，AST 文件实现到底能不能提升性能来说很有帮助。

PCH 可以是链式的。当你创建一个 PCH 文件，这个文件里引入一个已经存在的 PCH 的时候，Clang 可以在新的 PCH 里引用原有的文件并且只向新文件里写入新数据。例如，你可以创建一个 PCH，包含了整个项目里非常常用的头文件，然后对每个源代码文件生成一个这个文件自己需要的 PCH，这样重新编译这个文件就会非常快，也不需要重复来自所有常用头文件的数据。链式 PCH 背后的机制在[后面]进行讨论。

## AST 文件格式

```plain
译者注：要理解这一节的部分内容，需要了解常见操作系统的目标文件（Object File）格式。可以参考《程序员的自我修养》（俞甲子, 石凡, 潘爱民, 电子工业出版社, 2009）。
```

一个 clang 产生的 AST 文件就是一个带有特殊节的目标文件，这个节保存序列化的 AST，对 COFF 文件是  `clangast` 节，对 ELF 或者 Mach-O 文件是 `__clangast` 节。目标文件中的其他节（由编译目标确定）保存 AST 中定义的数据类型的调试信息。使用 libclang 的工具也可能产生没有其他调试信息节，只有序列化 AST 的裸 AST 文件。

`clangast` 节由几个不同的块组成，每个部分保存一部分 Clang 内部表示的序列化表示。每个块要么对应一个块要么对应一个 [LLVM 二进制流格式（LLVM's bitstream format）](https://llvm.org/docs/BitCodeFormat.html) 的记录。这些逻辑块的内容如下图所示。

```ascii
┌──────────────────────────┐
│    Precompiled Header    │
│   ┌──────────────────┐   │
│   │ Metadata         │   │
│   └──────────────────┘   │
│   ┌──────────────────┐   │
│   │ Source Manager   │   │
│   └──────────────────┘   │
│   ┌──────────────────┐   │
│   │ Preprocesoor     │   │
│   └──────────────────┘   │
│   ┌──────────────────┐   │
│   │ Types            │   │
│   └──────────────────┘   │
│   ┌──────────────────┐   │
│   │ Declarations     │   │
│   └──────────────────┘   │
│   ┌──────────────────┐   │
│   │ Identifier Table │   │
│   └──────────────────┘   │
│   ┌──────────────────┐   │
│   │ Method Pool      │   │
│   └──────────────────┘   │
└──────────────────────────┘
```

`llvm-objdump` 程序提供了一个 `-raw-clang-ast` 选项，用来从一个目标文件容器中提取 AST 节的二进制内容。

[`llvm-bcanalyzer`](https://llvm.org/docs/CommandGuide/llvm-bcanalyzer.html) 可以用来检验 AST 节的二进制流的真实结构。这个信息可以用来帮助理解 AST 节的结构，也可以用来隔离可进一步优化的 AST 表示的区域，比如通过引入缩写。

### 元数据块（Metadata Block）

元数据块包含几个记录，提供用以构建这个 AST 文件的信息。这个元数据主要用来验证 AST 文件的使用情景。例如，一个编译目标为 32 位 x86 平台的 PCH 文件不能用于目标是 64 位平台的编译。元数据块包含以下信息

**语言选项**。描述用于编译 AST 文件的语言方言，包括主要选项（例如 Obj-C 支持）和次要选项（例如是否支持 "//" 注释）。这个记录的内容对应 [`LangOpts`]。

**目标架构**。描述生成的 AST 文件的编译目标架构、平台和 ABI 三元组，例如 `i386-apple-darwin9`。

**AST 版本**。AST 文件格式的主要版本号和次要版本号。次要版本号变化不影响向后兼容，而主要版本号更新的编译器不能读取更旧的 PCH（反之亦反）。

**原始文件名**。用来生成 AST 文件的头文件的完整路径。

**预定义缓冲**。虽然没有明确地作为元数据的一部分，但是预定义缓冲区是用于验证 AST 文件的。预定义缓冲区本身包含了编译器生成的，根据当前目标、平台和命令行选项，用来初始化与预处理器状态的代码。例如当我们在编译没有 Microsoft 拓展的 C 程序的时候，预定义缓冲会包含 `#define __STDC__ 1`。预定义缓冲区本身存储在源代码管理块（译者注：见下一节）中，但其内容与其他元数据一起被验证。

一个链式 PCH 文件（即引用了其他 PCH 文件的 PCH 文件）或者一个模块（一个模块内可能引入其他模块）带有额外的元数据，保存当前 AST 文件所依赖的所有 AST 文件的列表。当加载当前 AST 文件的时候这些 AST 文件都会被加载。

对于链式 PCH 文件，语言选项、目标架构和预定义缓冲数据可能被这个链的末尾文件替代，因为它们总是匹配的。

### 源代码管理块（Source Manager Block）

源代码管理块保存 Clang 的 [`SourceManager`] 的序列化表示，它处理从源代码位置（表示在 Clang AST 中），到真实的源代码文件的行列位置或宏实例的映射。源代码管理块中的文件表示还包含在构建 AST 文件时（传递性地）引入的所有头文件的信息。

源代码管理块的大部分内容是关于各种文件、缓冲区和宏实例的信息，一个源代码位置可以引用这些信息。每个文件都被一个数字“文件 ID”引用，这是一个存储在源文件位置的唯一整数（从 1 开始）。Clang 将每个文件 ID 的信息序列化，还保存一个索引，把文件 ID 映射到 AST 文件中存储该文件 ID 信息的位置。与文件 ID 有关的数据只有在编译器需要的时候才会被加载，例如在发出一个包含头文件中宏实例的诊断信息时。

源代码管理块还包含构建 AST 文件时引入的所有头文件的信息。这包含头文件的控制宏的信息（例如当预处理器确定头文件的内容依赖于 `LLVM_CLANG_SOURCEMANAGER_H` 这样的宏时）。

### 预处理器块（Preprocessor Block）

预处理块保存预处理器（Preprocessor）的序列化表示。特别地，里边保存了用来构建 AST 文件所用的头文件（header）末尾定义的所有宏，以及构成每个宏的标记序列。只有当程序中第一次出现宏的名字时，才会在 AST 文件中读取宏定义。对标识符表进行查找时，才会触发这种对宏定义的惰性加载。

### 类型块（Types Block）

每一个在翻译单元中被引用的类型，在类型块中都有一个序列化表示。每一个 Clang 类型节点（`PointerType`, `FunctionProtoType` 等等）在 AST 文件中都有一个对应的记录类型。当从 AST 文件中反序列化以得到类型时，我们用这些记录类型中的信息和 AST 上下文，重新构建正确的类型节点。

每个类型都有唯一的类型 ID，这是一个整数，可以用以区分不同的类型。类型 ID 0 用来表示 Null 类型，小于 `NUM_PREDEF_TYPE_IDS` 的类型 ID 表示预定义类型（`void`, `float` ，而其他“用户定义”的类型 ID 从 `NUM_PREDEF_TYPE_IDS` 开始递增。AST 文件有一个从用户定义类型块，到这个块所依赖的序列化表示的类型块位置的，关联映射。这个机制使我们可以实现惰性序列化。当一个类型在 AST 文件中被引用的时候，这个引用使用类型 ID 左移 3 位的值。最低的 3 位用来表示 `const`, `volatile`, `restrict` 定语，和 Clang 的 [`QualType`](https://clang.llvm.org/docs/InternalsManual.html#qualtype) 一样。

### 声明块（）

### 语句和表达式（）

### 标识符表块（）

### 方法池块（）

## AST Reader 的集成点

## 链式 PCH

## 模块

## 附录

翻译名词表

* translation unit，翻译单元，见 C99
* record type，记录类型
* AST context，AST 上下文，是一个 Clang 类
* qualifier，定语，见 C99
* preprocessor，预处理器，见 C99
* header，头文件
* token sequence，标记序列，见 C99
* section，节，这是目标文件（object file）的概念
* source location，源代码位置，这是 clang 类

翻译进度

```plain
0% 通过 Clang 使用 PCH
0% 设计哲学
50% AST 文件格式
100%  元数据块（Metadata Block）
100%  源代码管理块（）
100%  预处理器块（）
100%  类型块（Types Block）
0%  声明块（）
0%  语句和表达式（）
0%  标识符表块（）
0%  方法池块（）
0% AST Reader 的集成点
0% 链式 PCH
0% 模块
```
