all: hello.ast hello.bcanalyzer_dump hello.xxd

hello.ast:
	clang -emit-ast hello.c -o hello.ast

hello.bcanalyzer_dump: hello.ast
	llvm-bcanalyzer --dump hello.ast > hello.bcanalyzer_dump

hello.xxd: hello.ast
	xxd hello.ast > hello.xxd

.PHONY: clean
clean:
	rm -f hello.ast
	rm -f hello.bcanalyzer_dump
	rm -f hello.xxd
