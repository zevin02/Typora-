对valgrind的memcheck工具文档的翻译与理解

# 认识的新单词

overrunning ：超过

underrunning： 不足

derive：衍生，派生

versus：对

overlap：重叠,覆盖

negative： 负

presumably（presume）： 可能

fishy：可疑的

occational： 偶然的

diagnose：诊断

crash：崩溃

issue： 发出，提出

reckon：评估

likewise： 同样的

subscript：下标

note that：请注意

# 概述

memcheck是一个内存错误检错工具，它可以检测c/c++代码中的如下问题

* 访问一些你不应该去访问的内存，如访问一块内存，但是他已经被释放掉了，堆溢出或不足，超出了栈顶[因为栈是向下增长）
* 使用未定义的值，如一些值未被初始化，或从其他派生出来的值
* 错误的释放堆上的空间，如将一块堆空间释放两次，或malloc/free与new/delete匹配错误
* 对于memcpy相关函数对于目标原指针和目标地指针发生了重叠[^1]

[^1]:**拷贝的目的地址在源地址范围内。所谓内存重叠就是拷贝的目的地址和源地址有重叠**

* 传入一个可疑值（可能是一个负值）到分配内存大小的参数的函数里面

* 内存泄漏

  向这样的问题通常很难通过其他方法被找到，通常很久都无法被检测到，但偶然就会导致难以发现的崩溃

  memcheck还提供了Execution Trees 内存轮廓，使用命令行选项 <u>--xtree-memory</u>和管理命令<u>xtmemory</u>

# 从memcheck解析错误信息

memcheck可以发出一系列的错误信息，这些快速的显示了错误信息代表了什么，

## illegal read/illegal write errors（不合理的读写错误）

如：

~~~
Invalid read of size 4
   at 0x40F6BBCC: (within /usr/lib/libpng.so.2.1.0.9)
   by 0x40F6B804: (within /usr/lib/libpng.so.2.1.0.9)
   by 0x40B07FF4: read_png_image(QImageIO *) (kernel/qpngio.cpp:326)
   by 0x40AC751B: QImageIO::read() (kernel/qimage.cpp:3621)
 Address 0xBFFFF0E0 is not stack'd, malloc'd or free'd
~~~

不合法的读入4个字节大小，因为他并不是栈上的，堆上的，或已经被free掉了，

这个错误信息发生在，Memcheck认为你在你不应该去读写的地方去读写，

在上面信息中，程序在地址为0xBFFFF0E0处读入了4个字节，通过系统提供的库so.2.1.0.9在某处，在某处被同样的库调用，从qpngio.cpp中的326行被调用

Memcheck尝试去建立违法地址的相关联的地方，因为那通常很有效，所以如果他指向内存块中的一个已经释放的空间，你就会被提示，提示也会提示内存在哪里被释放。同样的，如果他被证明在堆底部，一个普遍的结果在数组中，你也会被提示这个信息，以及这个被分配的地方。如果你使用 

<u>--read-var-info</u> 这选项，Memcheck会运行的更慢一点，但是会提供违法地址的更加详细的信息

在这个例子中，Memcheck无法识别这个地址，事实上，这个地址在栈上，但是处于某种原因，这个并不是一个有效的栈地址 ---他在栈指针的底部，并不被允许，在这种特定情况下，他可能是由gcc导致的这种不合理的，一个出名的bug在比较早期的gcc版本里面

请注意：Memcheck只会告诉你你的程序要访问一个违法的地址，但是他不会阻止访问的发生，所以，如果你的程序要正常的，正常来说就会导致段错误，你的程序将会遭受同样的命运，但是你memcheck会立即告诉你这信息，在这个特定的例子中，在栈上读一个垃圾的数据，并不致命，程序仍然能够运行。





















