---
title: GDB/CGDB入门与拆弹游戏
date: 2020-03-29 16:22:19
updated: 2020-03-29 16:22:19
categories:
- 系统与底层
tags:
- gdb
- cgdb
- stm32
- disassembly
- binary
- defusing binary bomb
- TechTalk
---

这是我的[嵌入式笔记](https://joouis.com/2019/watching-notes-revolution-os/)第七篇同时也是完结篇，原文写于2015年。本文除了介绍GDB/CGBD的基础使用方法以及嵌入式开发应用外，还会分享一个解密类经典游戏 `Defusing a binary bomb with GDB` ，非常有趣！

<!-- more -->



## 使用GDB

### GDB

- `b [function name] *[address]`：設置斷點
- `d [breakpoint num]`：刪除斷點
- `c`：繼續執行，到下一個斷點處（或運行結束）
- `n`：單步跟蹤程序，當遇到函數調用時，也不會進入函數內（針對src）
- `s`：單步調試，如果有函數調用，則進入函數（針對src）
- `si`：同s（針對asm）
- `ni`：同n（針對asm）
- `p /x $r0`：以16進制格式打印寄存器r0的值
- `x/s [address]`：以字串方式打印出地址所指向的內容
- `layout asm/src`：會顯示出當前正在執行的asm/src代碼段（這樣就不用看objdump -D啦）
- `layout split`：同時顯示當前執行的原始碼和組語
- `layout regs`：會以10進制和16進制方式顯示寄存器的值，包含r0-r12、sp、lr、pc、cpsr（這樣就可以不用p了）
- `winheight name +/- count`：調整 TUI 窗口大小，例如`winheight src -5`代表TUI窗口減少5行代碼
- `info stack/variables/files/...`
- REF:
  - http://linuxtools-rst.readthedocs.org/zh_CN/latest/tool/gdb.html
  - [TUI-specific Commands](https://sourceware.org/gdb/onlinedocs/gdb/TUI-Commands.html)

### CGDB

- 安裝：`sudo apt-get install cgdb`
- 官方手冊：[CGDB Manual 0.6.6](http://cgdb.sourceforge.net/docs/cgdb-no-split.html)（**请查看手册最新版本**：[CGDB Manual 0.7.1](https://cgdb.github.io/docs/cgdb.html)）
- 相比GDB無需用layout調出TUI，CGDB會自動顯示出來，source window和GDB command window切換方式和VI相同，使用i和Esc。
- 新增及移除break point方法，直接在source window裏移動到想要新增/移除的位置，使用space鍵就可以了。
- **使用cross compiler方法**：`cgdb -d arm-none-eabi-gdb -x "gdb.script"`
- Source window類似VI界面，常見的`:`設定、`/`查詢等都可以使用，具體參考[CGDB configuration commands](http://cgdb.sourceforge.net/docs/cgdb.html/Configuring-CGDB.html#Configuring-CGDB)
- 使用`o`可以選擇要顯示的檔案源碼，超潮der
- **注意**：
  - 當前代碼行的行數會是粗體，不過不太明顯，用space敲敲看就知道在哪一行了
  - 相比GDB，記得在CGDB裏不要敲`layout`指令！會產生一堆亂碼死掉@@

### 可視化除錯工具

- [DDD](https://www.gnu.org/software/ddd/)
- [XXGDB](http://manpages.ubuntu.com/manpages/hardy/man1/xxgdb.1.html)



## Using STM32 discovery kits with open source tools

- 原PDF链接已失效，原文请自行Google
- Written by STLINK development team

### Installing GNU toolchain

- 请自行 Google

### Installing STLINK

- Dependencies
  - `libusb-1.0`
  - `pkg-config`
  - `autotools`
- Install stlink
  ```shell
  git clone https://github.com/texane/stlink stlink.git
  cd stlink.git
  ./autogen.sh
  ./configure
  make
  make install
  ```

It includes:

- A communication library (stlink.git/libstlink.a).
- A GDB server (stlink.git/st-util).
- A flash manipulation tool (stlink.git/st-flash).

### Using the GDB server

- A GDB server must be started to interact with the STM32. Depending on the discovery kit you are using, you must run one of the 2 commands:
  - `st-util --stlinkv1`: STM32VL discoverykit (onboard ST−link)
  - `st-util`: STM32L or STM32F4 discoverykit (onboard ST−link /V2)
- Then, GDB can be used to interact with the kit:
  - `arm-none-eabi-gdb XXX.elf`
- From GDB, connect to the server using:
  - `(gdb) target extended localhost: 4242`
  - `(gdb) load`
  - `(gdb) continue`

### Building and flashing a program

- If you want to simply flash binary files to arbitrary sections of memory, or read arbitary addresses of memory out to a binary file, use the st-flash tool, as shown below:
  - `st-flash read v1 out.bin 0x8000000 4096`: stlink-v1 command to read 4096 from flash into out.bin
  - `st-flash read out.bin 0x8000000 4096`: stlink-v2 command
  - `st-flash write v1 in.bin 0x8000000`: stlink-v1 command to write the file in.bin into flash
  - `st-flash write in.bin 0x8000000`: stlink-v1 command

### Notes

- Disassembling THUMB code in GDB
   By default, the disassemble command in GDB operates in ARM mode. The programs running on CORTEX-M3 are compiled in THUMB mode. To correctly disassemble them under GDB, uses an odd address. For instance, if you want to disassemble the code at 0x20000000, use `(gdb) disassemble 0x20000001`
- 每次都輸入一堆必備的指令很麻煩，可以寫入一個`gdb.script`檔案裏，例如：
  ```shell
  target remote:4242
  file stm32f429-example.elf
  layout src
  ```
  然後使用 `arm-none-eabi-gdb -x gdb.script` 進入後，單步執行即可。
  - 記得先打開 `st-util`；
  - 個人習慣把 `arm-none-eabi-gdb -x gdb.script` 寫進 `Makefile` 裏。



## Defusing a binary bomb

### [Lab43: Runtime Behavior](http://wiki.csie.ncku.edu.tw/embedded/Lab43)

預期目標
- 複習ARM處理器和指令集，並深入stack和執行時期的行為
- 熟悉objdump、gdb等工具，並利用上述工具分析執行檔

### Phase 0, Phase 1: Comparison

- 使用`b phase_0`設好斷點後，一個`c`來到`phase_0`函數，在源程式端輸入`help`，則可見：
  ![](https://cdn.joouis.com/lab-gdb-intro-1.png)
- 可以看到程式是用`r0`和`r1`作比較，`r0`是我們輸入的值，`r1`则是我們要猜的值，這裏已經告訴我們是`help`了
- 用 `si` 執行到 0x87ee ，可以看到`r0`和`r1`的值：
  ![](https://cdn.joouis.com/lab-gdb-intro-2.png)
- `x/s $r0` `x/s $r1` 直接可以印出`r0` `r1`的地址和指向的值：
  ![](https://cdn.joouis.com/lab-gdb-intro-3.png)
- 同理，`continue`到`phase_1`，印出`r1`
  ![](https://cdn.joouis.com/lab-gdb-intro-4.png)

> **Psalm 23:4**
> Yea, though I walk through the valley of the shadow of death, I will fear no evil; For You are with me; Your rod and Your staff, they comfort me.

### Phase 2: For loop

- 先隨意輸入，看下寄存器`r1`的值`0x8bc8: "%d %d %d %d"`，看起來是要輸入4個整數，重頭再來一次
- 發現用`ni`跳過function會讀不到指令，還是用`si`一步一步來吧
  ![](https://cdn.joouis.com/lab-gdb-intro-5.png)
- 走到0x885e，如果輸入的數的個數不是4個，就會explode bomb，若是則跳到0x8864
- 接下來的代碼中：
  ![](https://cdn.joouis.com/lab-gdb-intro-6.png)
  - 從0x886a到0x887a是一個迴圈，完整的`si`走了一邊，只是運行完後不能讀到`r2` `r3`的值所指向的地址裏的值
  - 類似的代碼爲:
    ```shell
    a=2;i=0;
    do {
      a+=1;
      i++;
    } while (i<=9);
    ```
  - 意外但肯定不是巧合地發現，之前輸入的四個數分別存在`r7+12` `r7+16` `r7+20` `r7+24`裏（其實是用`x/s $r7+n`試出來的）
  - 所以輸入的第二個數（上述代碼中的`a`）在迴圈後會增加10
  - 而函數最後對比的是`r7+16`和`r7+20`，需要他們相等，故輸入時第三個數減第二個數要等於10
  - 測試果然過關~
    ![](https://cdn.joouis.com/lab-gdb-intro-7.png)

### Phase 3: If condition

- 第一步，設置斷點在`phase_3`後隨意輸入進入斷點，印出`r1`的值，爲`"%d %d %d"`，後面也有判斷輸入個數的代碼，和階段2同理：
  ![](https://cdn.joouis.com/lab-gdb-intro-8.png)
- 接着把斷點設在0x88be，代碼如下：
  ![](https://cdn.joouis.com/lab-gdb-intro-9.png)
- `r3`會先取出輸入的第一個數，和40做對比，這裏我輸入的1，因此會繼續往下走
- 接着把給輸入的第二個數、第三個數的值賦給`r2` `r3`，然後`r3=r3+r2` ，再把`r3`的值存回`[r7,#16]`
- 然後把`r2`的值設爲`[r7,#16]`，再把`r3`的值還原回初始的第三個數的值，對比他們相不相等
  - 即要使得`r3=r2+r3`，所以在第一個數不爲40的情況下，第二個數和第三個數都只能是0
  - 那如果第一個數等於40，則`r2` `r3`相等即可
- 試驗一下
  - 當第一個數不爲40時，正確
    ![](https://cdn.joouis.com/lab-gdb-intro-10.png)
  - 當第一個數爲40時，正確
    ![](https://cdn.joouis.com/lab-gdb-intro-11.png)


### Phase 4: Iteration

- 與階段2、3同理，得知輸入爲一個整數
  ![](https://cdn.joouis.com/lab-gdb-intro-12.png)
- 接下來`phase_4`的函數很短，把輸入的數代入fun4後，return的值要等於1024
  ![](https://cdn.joouis.com/lab-gdb-intro-13.png)
- `fun4`的代碼如下：
  ![](https://cdn.joouis.com/lab-gdb-intro-14.png)
- 當`r3`不爲0時，會一直減1然後調用自身函數；當`r3`終於減到0後，會在0x88f2賦值爲1，然後跳到8904，回傳上一層的`fun`函數。然後`r3=r0=1`，開始左移
- 所以相當於輸入的數字n，可以滿足2的n次方爲1024，即是正確答案，真相如下：
  ![](https://cdn.joouis.com/lab-gdb-intro-15.png)

