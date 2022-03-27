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

