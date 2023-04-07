+++
title = "分支与循环"
date = 2023-04-06T20:45:00+08:00
lastmod = 2023-04-07T22:02:19+08:00
tags = ["编译", "LLVM"]
categories = ["编译", "LLVM"]
draft = false
toc = true
+++

## Clean code {#clean-code}

`toy-compiler` 编译器目前的类图是这样的:
 ![](/images/arch_v0.png)


## 分支 {#分支}

为了实现分支语句,我们创建一个新的 `NBranchStatement` 类,这个类继承自 `NStatement` 类.每一条分支语句( `NBranchStatement` )对象应该包含若干个 `if` 语句块( `NIFBlock` )和至多一个 `else` 语句块( `NBlock` ).而每个 `NIFBlock` 中应包含一个条件表达式( `NExpression` )和一个基本语句块( `NBlock` ).

到这里,可以发现上述项目结构至少存在一个问题: `NIFBlock` 和 `NBlock` 不应该是继承关系而应该是组合关系.

所以将代码中涉及到 `NIFBlock` 的类进行改进.修改后的类图如下: ~~稍后放送...~~.


## 循环 {#循环}
