---
title: 小练习：通过反汇编一段C代码观察栈行为
date: 2019-12-11 16:34:21
updated: 2019-12-11 16:34:21
categories:
- 系统与底层
tags:
- embedded system course
- stack
- assembly
- disassemble
- c
- gcc
- TechTalk
---

这是我的[嵌入式笔记](https://joouis.com/2019/watching-notes-revolution-os/)第二篇，原文写于2015年。

深入了解计算机的底层行为是软件工程师的内功之一，有助于理解程序代码是如何被执行的。本文的练习小而完整，不需要太多背景知识，因此摘选出来与大家分享。

<!-- more -->



## 目的

- 在學習Linux kernel分析的基礎課程時有分析到C代碼執行時stack的行爲，那麼這裏做一個最簡單的實驗，通過把C代碼反彙編成ASM代碼，來看堆棧的操作情況。
- 備註：
  - 直接用的**64-bit**環境，`gcc -S -o main.s main.c`（因爲網路大多都是32-bit的分析，照做沒新意）
  - 也可以用GDB來看，覺得有點殺雞用牛刀XDD



## 源代碼和編譯出來的代碼

- 源代碼：

![](https://cdn.joouis.com/observe-stack-behavior-by-disassembling-a-simple-c-code-1.png)

- 反編譯出來的代碼（64-bit）：

![](https://cdn.joouis.com/observe-stack-behavior-by-disassembling-a-simple-c-code-2.png)



## 分析

- 首先，可以看到一件事是：編譯出來的組合語言是32-bit和64-bit混合的，應該是針對寄存器的bit不同指令做的相應的使用。

- 指令從第14行開始也就是main函數第一條指令開始執行，首先是把`%rbp`寄存器的值壓入棧內，保存當前工作的棧基指。

- 接着把`rsp`的地址賦給`rbp`，含義是初始化一個新的stack。

- 第16行是預設的空間，GCC下`-O1`指令似乎可以優化。

- 第17、18行將`a(1,2)`中的兩個參數分別存到`esi`和`edi`寄存器中，這個地方需要說明的是X86-32中會使用push將實參存入stack內，如下圖（原`main.c`反編譯成 32-bit ASM 代碼）：
  - ![](https://cdn.joouis.com/observe-stack-behavior-by-disassembling-a-simple-c-code-3.png)
  - 而在 x86-64 中，如果參數在6個以內，GCC 就可以利用寄存器來存儲參數值；超過6個的參數，還是通過上述操作實現，這裏給我們了至少兩個啓發：
    - 儘量使用6個以下的參數列表。
    - 傳大的數值時儘量使用指針或引用，因爲寄存器只有64位並且只能存整形數值。

- 然後就呼叫`a`函數了，call其實也分爲兩個動作：
  - 第一個是把當前`rip`的值保存。
  - 把`a`函數的指令初始地址給`rip`。

- `a`函數也是同理，先保存當前的`rbp`然後初始化新的stack。

- 第4、5行，就將保存在`esi`和`edi`內的值按順序放入`-24(%ebp)`和`-20(%ebp)`內。這可以說明，**在X86-64中，實參入棧和賦值給形參都是在子函數代碼段完成的**。

- 第6到8行就是從stack中取出參數到通用寄存器完成加法運算。

- 第9、10行是算出的值會先從`eax`存到`-4(%rbp)`，即給tmp變量賦值。

- 然後再將`tmp`的值給`eax`寄存器，函數的返回值默認使用`eax`寄存器返回給上一級函數。

- 第11行將`rbp`出棧，然後第12行ret就是把之前保存的`rip`出棧。

- 回到第20行指令，將`eax`的值給`-4(%rbp)`，也就是給變量i。

- 爲了`return 0`把0給`eax`。

- 因爲是main函數，所以最後是leave（leave: Releases the local stack storage created by the previous ENTER instruction.）。


## 流程圖

![](https://cdn.joouis.com/observe-stack-behavior-by-disassembling-a-simple-c-code-4.png)



## Ref

- http://www.searchtb.com/2013/03/x86-64_register_and_function_frame.html
- http://en.wikipedia.org/wiki/X86_instruction_listings