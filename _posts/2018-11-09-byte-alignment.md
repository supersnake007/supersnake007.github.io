---
layout:     post
title:      "字节对齐"
subtitle:   " \"字节对齐\""
date:       2018-11-09 12:30:00
author:     "ZhuDong"
catalog: true
tags:
    - 技术
---

### 前言
之前一直对字节对齐的规则很迷惑，感觉对齐规则有时候和自己想的一样，有时候不一样。很莫名其妙（其实就是对对齐的规则没有完全了解)。于是就借着这次机会好好的总结一下。

---

### 正文
先看以下例子

```
#include<stdio.h>

int main() {
    printf("the char's size is %d\n", sizeof(char));
    printf("the int's size is %d\n", sizeof(int));
    printf("the long's size is %d\n", sizeof(long));
    printf("the long long's size is %d\n", sizeof(long long));
    printf("the double's size is %d\n", sizeof(double));
    printf("the float's size is %d\n", sizeof(float));
    printf("the \*'s size is %d\n", sizeof(void *));
    return 0;
}
```

以上例程输出如下

```
the char's size is 1
the int's size is 4
the long's size is 8
the long long's size is 8
the double's size is 8
the float's size is 4
the *'s size is 8
```
可以看出此机器是64位机器。long是按照8字节对齐的。我们稍稍修改上面的程序，增加一个Test结构体，看结构体里面的成员是按照什么规则对齐的。修改后的程序如下：

```
#include<stdio.h>

typedef struct _Test{
    long a;
    char b;
    int c;
} Test;

int main() {
    printf("the struct's size is %d\n",sizeof(Test));
    return 0;
}
```
以上例程输出如下：

```
the struct's size is 16
```

可以看到这个结构体总共占用16个字节。但是我们看到Test中只有long a、char b、 int c三个变量。他们的长度分别是8、1、4。8+1+4等于13，不等于16。这就是因为字节对齐造成的。那么16到底是怎么来的呢？

![bytes_assign.png](https://zhudong.site/img/bytes_assign.png)

在内存中补齐的3位到底是在前面还是在后面呢？ 答案是int和char之间。原因是struct中的字节对齐主要遵循以下两条规则。

- 变量的存放位置相对于结构体起始位置的偏移量一定要是该变量字节数的整数倍

- 结构体占用的总字节数一定要是结构体中最长字节数变量字节数的整数倍

以上就是结构体字节对齐的规则。

---

### 后记

为什么要自己对齐？

CPU获取数据的时候，都是从固定位置开始获取数据, 如果不进行字节对齐，可能获取一个char类型的数据需要获取多次。不仅如此，获取多次之后还要根据大小端机不同的规则将获取的数据还原成原始数据。如此一来增加了获取数据的难度和效率。因此，编译器会将结构体存储进行自己对齐，以提高CPU获取的数据的速度和效率。

