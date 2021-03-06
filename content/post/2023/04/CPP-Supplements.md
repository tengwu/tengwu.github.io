+++
title = "C++拾遗"
date = 2023-04-03T20:21:00+08:00
lastmod = 2023-04-24T16:52:01+08:00
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

根据[cppreference](https://en.cppreference.com/w/cpp/11),C++11合并了一些开源库的特性:

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


### nullptr {#nullptr}

参考[这篇博客](https://www.cnblogs.com/porter/p/3611718.html).

简单来说,如果C++中存在这样的两个重载函数:

```c++
void bar(sometype1 a, sometype2 *b);
void bar(sometype1 a, int i);
```

`bar(a, NULL)` 在编译时会报错 `error: call to 'bar' is ambiguous`,但是如果使用 `bar(a, nullptr)`,在调用时会匹配到 `void bar(sometype1 a, sometype2 *b);` 这个函数.


### long long {#long-long}

很难相信, `long long` 类型是在C++11引入的.


### type alias {#type-alias}

类型别名,替代 `typedef`,相比 `typedef` 的优点是支持模板.


### range-for {#range-for}

```c++
for(auto ele : eles) {
  // do something
}
```


### unique_ptr &amp; shared_ptr {#unique-ptr-and-shared-ptr}

`unique_ptr` 是一种独占式智能指针,它代表了一块动态分配的内存对象,并且保证只有一个 `unique_ptr` 指向这块内存, `unique_ptr` 的特点是它的所有权是独占的,也就是说当一个 `unique_ptr` 被销毁时,它所指向的对象也会被销毁.因此, `unique_ptr` 不支持复制或赋值操作,只能通过移动构造函数或移动赋值函数来转移所有权.这种特性使得 `unique_ptr` 非常适合用来管理单个资源.

`shared_ptr` 是一种共享式智能指针,它允许多个指针同时指向同一块内存. `shared_ptr` 的特点是它使用引用计数来追踪有多少个指针指向同一块内存.每当一个新的 `shared_ptr` 指向一块内存时,内部的引用计数就会增加1,而当一个 `shared_ptr` 被销毁时,引用计数就会减少1.当引用计数降为0时, `shared_ptr` 会自动销毁所指向的对象. `shared_ptr` 支持复制和赋值操作,每个 `shared_ptr` 都有一个指向所共享对象的引用计数器,用来保证内存的安全性.

`unique_ptr` 和 `shared_ptr` 都可以用来管理动态分配的内存资源,避免了手动释放内存的麻烦.其中, `unique_ptr` 适用于需要独占资源的场合,而 `shared_ptr` 适用于需要共享资源的场合.在实际开发中,可以根据具体的情况来选择使用哪种智能指针类型.

注意, `make_shared` C++11就有, `make_unique` C++14才有.


#### shared_ptr和unique_ptr的值可以赋给普通指针吗? {#shared-ptr和unique-ptr的值可以赋给普通指针吗}

可以,这两个智能指针只是C++11 **方便** 程序员来管理指针的语法,如果非得把智能指针的值赋给普通指针当然是可以的,但是如果智能指针主动把所指向对象给删除了,此时普通指针还指向的是对象所属的那块内存,这时普通指针就是悬空指针了.


### <span class="org-todo todo TODO">TODO</span> function {#function}


### <span class="org-todo todo TODO">TODO</span> constexpr {#constexpr}


## C++14 {#c-plus-plus-14}


### <span class="org-todo todo TODO">TODO</span> variadic templates {#variadic-templates}

可变参数模板.

真复杂啊,模板编程需要再抽空对应 `LLVM` 的一些例子学习一下.

基本用法参考[这篇博客](https://zhuanlan.zhihu.com/p/149405532).


### lambda expressions {#lambda-expressions}

```c++
#include <iostream>
#include <vector>

int main() {
  std::vector<int> vec {1, 2, 3, 4, 5};
  int x = 10;

  // 值捕获
  auto lambda1 = [=]() {
    for (auto i : vec) {
      std::cout << i << " ";
    }
    std::cout << x << std::endl;
  };
  lambda1(); // 输出：1 2 3 4 5 10

  // 引用捕获
  auto lambda2 = [&]() {
    for (auto& i : vec) {
      i *= 2;
    }
    x = 20;
  };
  lambda2();
  for (auto i : vec) {
    std::cout << i << " ";
  }
  std::cout << x << std::endl; // 输出：2 4 6 8 10 20

  return 0;
}
```

可以通过 `captures` 的方式让 `lambda表达式` 看到外部的值,有值捕获和引用捕获两种方式.在使用值捕获时,如果对捕获的值进行了修改,编译时会报语法错误.


### make_unique {#make-unique}

创建 `unique_ptr`.


## C++17 {#c-plus-plus-17}


## C++20 {#c-plus-plus-20}


## C++23 {#c-plus-plus-23}

原来C++23还没定下来.
