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
