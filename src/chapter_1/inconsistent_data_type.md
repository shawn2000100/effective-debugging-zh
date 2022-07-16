
# 不一致的数据类型


我们的服务器程序在测试中随机crash。在调试一段时间以后，怀疑是内存越界错误导致的。这个问题被缩小到一个特定的数据对象。当一个程序更新对象其中一个数据成员的时候，它损坏了紧随其后的数据对象（被我们将在第二章讨论的内存调试工具发现）。

但是，代码看起来是无辜的因为它正在访问它自己的数据成员。非常难以理解，这怎么可能损坏另外一个数据对象。进一步的调查发现这个被怀疑的对象在一个模块创建，然后传入另外一个更新它的数据成员的模块。鼓捣一下以后，发明两个模块的数据对象大小不一致。调试器在第一个模块显示一个大小，在第二个模块打印一个更大的大小。这让人非常吃惊，因为对象是在一个头文件声明，这个头文件被两个项目共享。通过更进一步在他们每个模块的作用域打印出和对比数据的布局和它们对象成员偏移（对象的类型调试符号），对象被编译器布局成不同的大小非常清楚：一个所有的数据成员合适地对齐，另外一个并没有，而是把所有的数据成员打包在一起。这也被底层的内存管理器分配的内存块的大小证实（第二章具有更多细节怎么获取这样的信息）。但是另外一个模块认为对象是通常的未打包布局。当对象被传入这个模块，它覆盖了内存且损坏了附近的对象。图1-3用更简化的形式描述这个bug。一个结构体`T`的对象被模块A创建为打包的格式。它又被传入模块B，模块B认为它是未打包的格式。模块B的灰色数据成员`data3`覆盖了分配的内存块。

![图1-3 因为数据类型不一致导致的内存覆写](../images/fig-1-3-Memory-Corruption-Due-To-Inconsistent-Data-Type.png)

You will probably ask how it could happen. It turned out the object is declared correctly in the header file. The bug comes from another header file which uses the following pragma:

你可能会问题它是怎么能够发生。结果表明对象在头文件声明。bug来源于另外一个头文件使用下面的编译指令：

```
    #pragma pack(4)
    ...
    #pragma pack()
```
The developer intends to pack the structure declared between these two pragma statements on 4 bytes boundary. The syntax is well understood by Microsoft’s Visual Studio C++ compiler. However, the problem occurs when the same code is compiled by the Visual Age C++ compiler on AIX.  This compiler has similar but slightly different pragma syntax to end the packing scope.
那个开发者打算打包在两个编译指令中间的结构体为4字节边界。这个指令很好地被微软Visual Studio编译器理解。但是，当同样的代码被AIX里的Visual Age C++编译的时候，问题发生了。这个编译器有详细但是有点区别的编译指令语法来结束打包作用域。

```
    #pragma pack(4)
    ...
    #pragma pack(nopack)
```

这个语法差别的结构是，Visual Age C++编译器捡起了开始的打包编译指令（第一行）但是忽略了结束的打包编译指令（最后一行）。在程序员意图结束数据打包的那一行之后，它继续打包数据结构体。在模块A,我们的受害数据对象生命在引入包含上面的编译指令的头文件的后面。在模块B，这个有问题的头文件没有被引入所以这个对象没有被打包。这就是不一致性是如何发生的。数据类型的调试符号准确地反映一个编译器如何查看一个数据类型。生成的机器指令将以这样操作数据对象。比如，在创建的时候，它请求了一个结构体的大小；对象的数据成员通过相对开始内存块的偏移来访问。