---
layout: post
title: new/delete
categories: C++
description: 
keywords: 
---

# 动态内存分配

采用new和delete可以在堆区分配内存。

## new
```cpp
new TYPE;
```
TYPE可以是内置类型，也可以是已知的class类型。

可以在分配内存时进行初始化：
```cpp
new TYPE(init_value);
```

例如：
```cpp
int *pi;
pi = new int;
```

```cpp
int *pi;
pi = new int(10);
```

## new分配数组
```cpp
int *pia = new int[10];
```

## delete释放
```cpp
delete pi;
```
## delete释放数组
释放数组必须要在delete和数组指针间，加上空的[]。

```cpp
delete [] pia;
```
## delete无需判0
```cpp
if(pi != 0)
    delete pi;
```
是多此一举的。


## 对数组错误的delete会发生什么
正确的构造和析构是这样成对出现的:
```cpp
Student *r = new Student[10];
delete [] r;
```
这样,会先为10个Student类型的对象分配空间,之后逐个调用每个对象的构造函数.
在delete时,则会逐个调用每个对象的析构函数,之后再释放整个空间.

如果我们采用了错误的delete方法,只```delete r```,会产生什么样的结果呢?

这会导致,只会调用首个对象的析构函数,随后删除当初申请的整个空间.

## 结构体也可以提供构造函数

```cpp
#include<iostream>
using namespace std;

struct stru
{
    int a;
    stru():a(){}        //无参构造
    stru(int i) :a(i){}//有参构造
};


int main(int argc, char const *argv[])
{
    struct stru s[5] = {stru(1), stru(2), stru(3)}; //前三个用有参,两个用无参
    return 0;
}


```