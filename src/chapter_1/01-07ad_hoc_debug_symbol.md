# 特殊的调试符号

前面的章节已经讨论了调试符号和调试器是怎么在一个调试会话里使用这个信息。作为一个增强，如果需要，我们可以往调试器添加更多的调试符号。在即使我们知道一个变量的具体类型，仍然不能打印这个变量的时候是非常有帮助的。调试器不能理解变量的问题是没有它的调试符号。这对系统库、三方库、遗留的符号只有部分或者全部去掉的二进制或者一些情况不编译带调试符号情况来说，是常见的。一种变通这个困难的方式是编译一个新的带有想要的调试符号的库文件。当调试器把新库的符号加载后，我们就可以具有调试这些二进制的更好准备。让我们看看一个第三方库的数据结构的例子。

为了打印第三方库管理的一列自由内存块，下面的数据结构体被声明在一个头文件`sh_type.h`.

```
typedef struct _FreeBlock
{
    PageSize sizeAndTags;
    struct _FreeBlock *next;
    struct _FreeBlock *prev;
} FreeBlock;
```

编译文件到带有所有调试符号的目标文件
`gcc -g -c -fPIC -o sh_symbol.o sh_symbol.c`

接着把这个文件加入到一个调试会话，会给出我们这个数据结构`FreeBlock`的类型符号。gdb命令`add-symbol-file`会从上面显示的输入文件读入额外的调试符号，显示如下。地址参数`0x3f68700000`在这里不重要。输入文件通常是共享库，但也可以是目标文件。你可以用这种方式加入更多你想要的符号。

```
gdb) add-symbol-file /home/myan/bin/sh_symbols.o 0x3f68700000
(gdb) print *(FreeBlock*)0x290c098
$1 = {
  sizeAndTags = 490,
  next = 0x290d560,
  prev = 0x290ffe8
}

```

这个方法让用户在使用调试器解释数据的时候具有更多的灵活性。但是它仅仅可以提供额外的类型信息，它不可替换其他的调试符号如行号或者变量位置，这些是原始二进制生成的编译时期确定。

