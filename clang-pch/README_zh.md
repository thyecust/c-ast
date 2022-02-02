# Clang PCH

首先请阅读 PCHInternals，了解 Clang PCH 文件是什么。

然后在本文件夹内运行 `make`，查看 `hello.bcanalyzer_dump` 文件，这代表了 `hello.c` 产生的 AST。

其中的 AST_BLOCK 包含了这些信息

```plain
Types, Declarations
    DECLTYPES_BLOCK
Metadata
    TYPE_OFFSET
    DECL_OFFSET
    FILE_SORTED_DECLS
Source Manager
    SOURCE_MANAGER_BLOCK
    SOURCE_LOCATION_OFFSETS
    SOURCE_LOCATION_PRELOADS
    SOURCE_MANAGER_LINE_TABLE
    COMMENTS_BLOCK
Preprocessor
    PREPROCESSOR_BLOCK
    MACRO_OFFSET
    HEADER_SEARCH_BLOCK
Identifier Table
    IDENTIFER_TABLE
    IDENTIFER_OFFSET
```
