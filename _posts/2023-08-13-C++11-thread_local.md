---
layout: post
title: thread_local
categories: C++ C++11 C++17
description: 
keywords: 存储类说明符  thread_local
---

## 存储类说明符

thread_local 是 C++ 11 新引入的一种存储类说明符，它会影响变量的存储周期。

存储类说明符是一个名字的声明语法的声明说明符序列的一部分。它与名字的作用域一同控制名字的两个独立性质：它的“存储期”和它的“链接”。

截止c++17,共有6种存储类说明符
1. auto：自动存储期（C++11加入）；
1. register：自动存储期，提示编译器将此变量置于寄存器中(C++17后弃用)；
1. static：静态或线程存储期，同时提示是内部链接；
1. extern：静态或线程存储期，同时提示是外部链接；
1. thread_local：线程存储期（C++11加入）；
1. mutable：不影响存储期或链接。

声明中只能出现一个存储类说明符，但 thread_local 可以与 static 或 extern 结合 (C++11 起)。
### 解释

1. auto 说明符只能搭配在块作用域或函数形参列表中声明的对象。它指示自动存储期，不带存储类说明符的变量均为auto存储类。此关键词的含义在 C++11 有变更。
(C++11 前)
1. register 说明符只能搭配在块作用域或函数形参列表中声明的对象。它指示自动存储期，即这种声明的默认情况。另外，此关键词的存在可以用来提示优化器将此变量的值存储到 CPU 寄存器。此关键词已经被弃用。
(C++17 前)
1. static 说明符只能搭配（函数形参列表外的）对象声明、（块作用域外的）函数声明及匿名联合体声明。当用于声明类成员时，它会声明一个静态成员。当用于声明对象时，它指定静态存储期（除非与 thread_local 协同出现）。在命名空间作用域内声明时，它指定内部链接。
1. extern 声明符只能搭配变量声明和函数声明（除了类成员或函数形参）。它指定外部链接，而且技术上不影响存储期，但它不能用来定义自动存储期的对象，故所有 extern 对象都具有静态或线程存储期。另外，使用 extern 且没有初始化器的声明不是定义。
1. thread_local 关键词只能搭配在命名空间作用域声明的对象、在块作用域声明的对象及静态数据成员。它指示对象具有线程存储期。如果对块作用域变量只应用了 thread_local 这一个存储类说明符，那么同时也意味着应用了 static。它能与 static 或 extern 结合，以分别指定内部或外部链接（但静态数据成员始终拥有外部链接）。
(C++11 起)


类的成员函数内定义了 thread_local 变量，则对于同一个线程内的该类的多个对象都会共享一个变量实例，并且只会在第一次执行这个成员函数时初始化这个变量实例，这一点是跟类的静态成员变量类似的。


## 存储期
C++程序中的所有对象共有四类存储期：

1. 自动（automatic）存储期。这类对象的存储在外围代码块开始时分配，并在结束时解分配。未声明为 thread_local、 (C++11 起)static 或 extern 的所有局部对象均拥有此存储期。
1. 静态（static）存储期。这类对象的存储在程序开始时分配，并在程序结束时解分配。这类对象只存在一个实例。所有在命名空间（包含全局命名空间）作用域声明的对象，加上声明带有 static 或 extern 的对象均拥有此存储期。有关拥有此存储期的对象的初始化的细节，见非局部变量与静态局部变量。
1. 线程（thread）存储期。这类对象的存储在线程开始时分配，并在线程结束时解分配。每个线程拥有它自身的对象实例。只有声明为 thread_local 的对象拥有此存储期。thread_local 能与 static 或 extern 一同出现，它们用于调整链接。关于具有此存储期的对象的初始化的细节，见非局部变量和静态局部变量。
(C++11 起)
1. 动态（dynamic）存储期。这类对象的存储是通过使用动态内存分配函数来按请求进行分配和解分配的。关于具有此存储期的对象的初始化的细节，见 new 表达式。

子对象和引用成员的存储期与它们的完整对象一致。

## 参考资料
1. [CPP标准对thread_local的解释](https://zh.cppreference.com/w/cpp/language/storage_duration)
1. [stackoverflow](https://stackoverflow.com/questions/21237905/how-do-i-generate-thread-safe-uniform-random-numbers)
1. [一位老哥的尝试](http://cifangyiquan.net/programming/thread_local/)