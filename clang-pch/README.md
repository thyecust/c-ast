# Clang PCH

You can read `PCHInternals` to learn what is the Clang PCH file.

You can run `make` in this folder and then open the `hello.bcanalyzer_dump`, which represents the AST generated from `hello.c`.

Its AST_BLOCK contains these info

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
