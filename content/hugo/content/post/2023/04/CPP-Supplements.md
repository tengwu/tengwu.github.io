+++
title = "C++拾遗"
date = 2023-04-03T20:21:00+08:00
lastmod = 2023-04-03T21:23:46+08:00
tags = ["C++"]
categories = ["C++"]
draft = false
toc = true
+++

## 动机 {#动机}

最近在尝试使用 `LLVM` 写一个编译器,但是发现工作期间学习的C++语法都差不多忘光光了,所以就想抽空把C++语法复习一下,顺便学习一下最新出的C++20和23.

本篇文章会陆续更新,基本原则是,在写代码的过程中,看到什么语法,再学习,更新它.


## C++11 {#c-plus-plus-11}

C++11是C++的第二个主要版本,第一个主要版本是C++98.从C++11开始,每三年会出一版新的C++标准.

根据[cppreferences](https://en.cppreference.com/w/cpp/11),C++11合并了一些开源库的特性:

> -   From TR1: all of TR1 except Special Functions.
> -   From Boost: The thread library, exception_ptr, error_code and error_condition, iterator improvements (begin, end, next, prev)
> -   From C: C-style Unicode conversion functions

更具体的特性列表请查看 `cppreference`, 这篇博客只列出了在之后可能会经常用到的特性.


### auto和decltype {#auto和decltype}

`auto` 让编译器根据初始化表达式的类型来推断所定义变量的类型,它推断的变量类型必须在编译时确定,不能用于动态类型或运行时类型推断.

`decltype` 用于获取表达式的类型,不会执行该表达式.它可以帮助程序员编写更灵活的代码,避免手动指定类型,同事保持类型安全.

我目前很少看到 `decltype` 的使用,具体怎么用,有什么需要注意的,看到再说吧.


### rvalue references &amp; move {#rvalue-references-and-move}

右值引用和 `move` 语义.

参考[这篇博客](https://zhuanlan.zhihu.com/p/335994370).

总而言之, `vector` 使用 `push_back` 可以避免深拷贝,提升性能.至于为什么,请详读上面这篇博客.

另外, `vector` 等容器或者一些C++函数实现了 `move` 语义,即在使用 `move` 之后,原始变量中的一些值被移走,这么说比较抽象,可以看一段示例代码:

```c
#include <iostream>
#include <string>
#include <vector>
using namespace std;

int main() {
        int a = 5;
        vector<int> v;
        v.push_back(move(a));
        cout << a << endl;

        string str = "hello";
        vector<string> vp;
        vp.push_back(move(str));
        cout << str << endl;


        return 0;
}
```

这段代码使用C++11编译运行后,只会输出 `5`,因为5的字面值是不存在深拷贝的问题的,但是 `str` 内的字符串值已经被 `vector` 实现的 `move` 语义函数移走了,所以打印出来的字符串值为空.

但是要清楚, `move` 本质上是一个类型转换函数,将左值转换为右值,要实现上面的功能( `move` 后 `str` 变量的字符串值就没有了)还需要函数或者容器的辅助.同时,编译器会默认在用户自定义的 `class` 和 `struct` 中生成 `move` 语义函数(除非用户主动定义了拷贝构造等函数,具体 `STFW` ).
