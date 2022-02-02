all: ast bcanalyzer_dump

ast:
	clang -emit-ast hello.c -o hello.ast

bcanalyzer_dump: ast
	llvm-bcanalyzer --dump hello.ast > hello.bcanalyzer_dump

clean:
	rm -f hello.ast
	rm -f hello.bcanalyzer_dump
