---
layout: post
title: container_of
author: kaifeng
date: 2008-07-21
categories: kernel
description: Linux kernel 常见宏 container_of。
---

Linux常用 container_of 宏从成员变量提取整个结构体的首地针，这个宏的定义如下：
```
    #define container_of(ptr, type, member) ({    \
            const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
            (type *)( (char *)__mptr - offsetof(type,member) );})
```

typeof 是 gcc 的一个扩展，用于确定一个变量的类型，有点像 C++ 的 RTTI，常用于表达式内的语句，在定义宏时，如果需要临时变量，可以这样做：
```
     #define max(a,b) \
       ({ typeof (a) _a = (a); \
           typeof (b) _b = (b); \
         _a > _b ? _a : _b; })
```

它可以保证我们定义的变量 _a/_b 与宏传入的变量 a/b 类型匹配，而不会产生告警。

因此 container_of 的第一行就是定义一个与 member 类型匹配的变量 `__mptr`，赋值后的`__mptr` 为宏参数中的待转换指针，因为只是类型转换，不涉及数据读写，((type*)0) 是没有任何副作用的。

第二行用 `__mptr` 减去 member 变量在 type 中的偏移，这样便可实际访问到 ptr 相同偏移，也即 type 的实际首地址了。offsetof 有两种定义：
```
    #ifdef __compiler_offsetof
    #define offsetof(TYPE,MEMBER) __compiler_offsetof(TYPE,MEMBER)
    #else
    #define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
    #endif
```

`__compiler_offsetof` 即 `__builtin_offsetof`，`__builtin_offsetof` 是 gcc 內建的支持(见 gcc 源码 c-parser.c)，

从后一种实现方式可以明显看到这一技巧，假定一个从0地址开始的结构体，取其成员 member 的地址正是结构体内的偏移。
