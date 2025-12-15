---
title: CSAPP 实验 2 ：Bomb Lab
categories:
  - 学习笔记
  - 计组
tags:
  - 学习笔记
  - 汇编
  - 计组
abbrlink: 50d220bf
date: 2025-11-25 20:30:58
---

# CSAPP 实验 2 ：Bomb Lab

## 引言

Bomb Lab 实验文件目录如下

```shell
bomb
├── bomb
├── bomb.c
└── README
```

- `bomb`: 二进制可执行程序
- `bomb`: 程序的主函数 main 的源文件
- `README`: `This is an x86-64 bomb for self-study students. `

阅读 `bomb.c` 文件

```c
#include <stdio.h>
#include <stdlib.h>
#include "support.h"
#include "phases.h"

/*
 * Note to self: Remember to erase this file so my victims will have no
 * idea what is going on, and so they will all blow up in a
 * spectaculary fiendish explosion. -- Dr. Evil
 */

FILE *infile;

int main(int argc, char *argv[])
{
    char *input;

    /* Note to self: remember to port this bomb to Windows and put a
     * fantastic GUI on it. */

    /* When run with no arguments, the bomb reads its input lines
     * from standard input. */
    if (argc == 1) {
	infile = stdin;
    }

    /* When run with one argument <file>, the bomb reads from <file>
     * until EOF, and then switches to standard input. Thus, as you
     * defuse each phase, you can add its defusing string to <file> and
     * avoid having to retype it. */
    else if (argc == 2) {
	if (!(infile = fopen(argv[1], "r"))) {
	    printf("%s: Error: Couldn't open %s\n", argv[0], argv[1]);
	    exit(8);
	}
    }

    /* You can't call the bomb with more than 1 command line argument. */
    else {
	printf("Usage: %s [<input_file>]\n", argv[0]);
	exit(8);
    }

    /* Do all sorts of secret stuff that makes the bomb harder to defuse. */
    initialize_bomb();

    printf("Welcome to my fiendish little bomb. You have 6 phases with\n");
    printf("which to blow yourself up. Have a nice day!\n");

    /* Hmm...  Six phases must be more secure than one phase! */
    input = read_line();             /* Get input                   */
    phase_1(input);                  /* Run the phase               */
    phase_defused();                 /* Drat!  They figured it out!
				      * Let me know how they did it. */
    printf("Phase 1 defused. How about the next one?\n");

    /* The second phase is harder.  No one will ever figure out
     * how to defuse this... */
    input = read_line();
    phase_2(input);
    phase_defused();
    printf("That's number 2.  Keep going!\n");

    /* I guess this is too easy so far.  Some more complex code will
     * confuse people. */
    input = read_line();
    phase_3(input);
    phase_defused();
    printf("Halfway there!\n");

    /* Oh yeah?  Well, how good is your math?  Try on this saucy problem! */
    input = read_line();
    phase_4(input);
    phase_defused();
    printf("So you got that one.  Try this one.\n");

    /* Round and 'round in memory we go, where we stop, the bomb blows! */
    input = read_line();
    phase_5(input);
    phase_defused();
    printf("Good work!  On to the next...\n");

    /* This phase will never be used, since no one will get past the
     * earlier ones.  But just in case, make this one extra hard. */
    input = read_line();
    phase_6(input);
    phase_defused();

    /* Wow, they got it!  But isn't something... missing?  Perhaps
     * something they overlooked?  Mua ha ha ha ha! */

    return 0;
}
```

可以发现，在 main 函数中，总共有 6 个相似的 phase，每个 phase 读入一行输入，并调用 phase 函数和`phase_defused()`函数判断是否成功拆弹。

我们首先运行一下`bomb`程序，输入`hello`

![](https://image.echocyan.top/20251119214454807.webp)

很显然炸弹爆炸了，我们的输入不符合要求。

我们首先对`bomb`反汇编，输入命令`objdump -d bomb > bomb.asm`，这行命令将`bomb`这个文件反汇编，并将结果保存到`bomb.asm`文件中。

阅读`bomb.asm`文件，可以找到 6 个 phase

```assembly
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
  400ee9:	e8 4a 04 00 00       	call   401338 <strings_not_equal>
  400eee:	85 c0                	test   %eax,%eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	call   40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	ret
0000000000400efc <phase_2>:
......
0000000000400f43 <phase_3>:
......
000000000040100c <phase_4>:
......
0000000000401062 <phase_5>:
......
00000000004010f4 <phase_6>:
......
```

**将文件作为输入**

测试某个 phase 的答案时，运行`bomb`程序后都要手动重新输入前面几个 phase 的答案，这样未免太繁琐了，好在`bomb`文件中已经提示我们可以使用文件作为输入。

我们在目录下创建一个`txt`文件，名字随意，我这里叫`answer.txt`，输入`gdb bomb`命令启动调试器，输入`run answer.txt`命令将`answer.txt`文件作为输入。

### 基础知识

#### GDB 调试器常用命令

这一部分不用死记硬背，用到的时候查就可以了

##### 1. 启动与运行

| 命令       | 简写 | 作用               | 示例                |
| ---------- | ---- | ------------------ | ------------------- |
| gdb <file> |      | 启动调试器         | gdb bomb            |
| run        | r    | 开始运行程序       | r 或 r solution.txt |
| kill       |      | 停止正在运行的程序 | kill                |
| quit       | q    | 退出 GDB           | q                   |

##### 2. 断点控制

| 命令             | 简写 | 作用                                   | 示例                      |
| ---------------- | ---- | -------------------------------------- | ------------------------- |
| break <location> | b    | 在函数名或地址处设断点                 | b phase_1 或 b \*0x400ee0 |
| info break       | i b  | 查看当前所有断点                       | i b                       |
| delete <num>     | d    | 删除指定编号的断点                     | d 1                       |
| stepi            | si   | **单步执行**（汇编级），遇到函数会进入 | si                        |
| nexti            | ni   | **单步跳过**（汇编级），遇到函数不进入 | ni                        |
| continue         | c    | 继续执行直到下一个断点                 | c                         |

##### 3. 查看代码与寄存器

| 命令           | 简写  | 作用               | 示例                   |
| -------------- | ----- | ------------------ | ---------------------- |
| disassemble    | disas | 反汇编当前函数     | disas 或 disas phase_1 |
| info registers | i r   | 查看所有寄存器的值 | i r                    |
| print $<reg>   | p     | 打印特定寄存器的值 | p $rax                 |

##### 4. 核心命令：查看内存 (x)

**格式：** `x /<数量><格式><单位> <地址>`

- **格式 (Format):**
  - `s`: 字符串 (String) —— 查看密码常用
  - `d`: 十进制 (Decimal) —— 查看整数常用
  - `x`: 十六进制 (Hex)
  - `i`: 指令 (Instruction)
- **单位 (Unit):**
  - `b`: 1 字节 (Byte)
  - `w`: 4 字节 (Word) —— `int` 通常是这个
  - `g`: 8 字节 (Giant) —— 指针通常是这个

#### x86-64 寄存器

![](https://image.echocyan.top/20251124193858839.webp)

## phase_1

我们首先逐行分析下汇编代码

```assembly
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
```

- 将栈顶指针减去 8，这不重要，略过

```assembly
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
```

- 将十六进制数`0x402400`存放到`%esi`寄存器中

- `%esi`寄存器是`%rsi`寄存器的低 32 位，相当于给函数第二个参数赋值

- 那么函数的第一个参数是什么呢？

  - 在调用这个函数时，我们是这样执行的 ` phase_1(input)`
  - 所以，第一个参数`%rdi`已经存放了我们的之前的输入

```assembly
  400ee9:	e8 4a 04 00 00       	call   401338 <strings_not_equal>
```

- 调用函数`strings_not_equal`
- 看名字就知道，这个函数用来比较两个字符串是否相等

```assembly
  400eee:	85 c0                	test   %eax,%eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	call   40143a <explode_bomb>
```

- `test`: 检查返回值`%eax`是否为 0
- `je`(Jump if Equal): 如果等于 0，就跳转到`400ef7`，如果不等，那么跳过这行代码，执行下面的`explode_bomb`
  - 到这里我们可以推测`strings_not_equal`函数的判断条件
    - 如果两个字符串相等，返回 0，那么我们不会触发炸弹爆炸
    - 如果两个字符串不相等，返回 1，炸弹爆炸

我们可以尝试将这段汇编代码整理成伪代码的形式

```c
void phase_1(rdi)
{
	esi = 0x402400;
	result = stirng_not_equal(rdi,esi);
	if(result != 0)
	{
		explode_bomb();
	}
	return;
}
```

通过以上分析，我们可以发现，关键是内存地址`0x402400`

输入命令`x/s 0x402400`，其中，`x`= eXamine (检查内存)，`/s`= String (按字符串格式显示)。

![](https://image.echocyan.top/20251119214410120.webp)

我们可以发现，答案就是`Border relations with Canada have never been better.`

## phade_2

我们首先将这段汇编代码初步处理成伪代码

```c
void phase_2(rdi)
{
    rsp -= 0x28;
    rsi = rsp;

    read_six_numbers();

    if(*rsp == 1)
    	goto 400f30;
    else
        explode_bomb();

    400f17:
    	eax = *(rbx - 0x4);
    	eax += eax;
    	if(eax == *rbx)
            goto 400f25;
    	else
            explode_bomb();
    400f25:
    	rbx += 0x4;
    	if(rbp != rbx)
            goto 400f17;
    	else
            goto 400f3c;

    400f30:
    	rbx = rsp + 0x4;
    	rbp = rsp + 0x18;
    	goto 400f17;

    return;
}
```

尝试阅读并理解这段代码，可以发现，其中蕴含着一个循环。我们作进一步整理。

```c
void phase_2(rdi)
{
    rsp -= 0x28; // 栈顶指针减0x28，在栈上开辟空间
    rsi = rsp;

    read_six_numbers(rdi,rsi);

    if(*rsp != 1) explode_bomb();

    // 初始化，400f30
    rbx = rsp + 0x4;
    rbp = rsp + 0x18;

    while(rbp != rbx) //判断
    {
        // 进入循环体,400f17
        eax = *(rbx - 0x4);
    	eax += eax;
    	if(eax != *rbx) explode_bomb();

        rbx += 0x4; // 更新步进，400f25
    }

    return;
}
```

整理后，此时就比较好理解了

![](https://image.echocyan.top/20251120001419248.webp)

1. `read_six_numbers(rdi,rsi);`函数读取我们的输入，并将其写到之前开辟栈空间上。
2. `rsp -= 0x28;`在栈上开辟了一块空间。
   1. `rbp`为高位指针，初始指向`rsp + 0x18`
   2. `rbx`为低位指针，初始指向`rsp + 0x4`
3. `rsp`指向栈底，`*rsp`为 1
4. `rbx`每次增加`0x4`，直到`rbp == rbx`，总计循环 5 次
5. `eax`等于`*(rbx-0x4)`，由低位到高位每隔`0x4`逐步遍历，初始为 1 ，从 2 开始检查，每次倍增，一共循环五次

所以栈上的数字应该是`1 2 4 8 16 32`。

## phase_3

我们一步步来分析 phase_3 的汇编代码

```assembly
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp				;开辟栈空间
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx			;第四个参数，不妨称其为y
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx			;第三个参数，不妨称其为x
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	call   400bf0 <__isoc99_sscanf@plt>	;调用sscanf函数
  400f60:	83 f8 01             	cmp    $0x1,%eax				;比较sscanf返回值与1
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>	;如果返回值大于1，跳转
  400f65:	e8 d0 04 00 00       	call   40143a <explode_bomb>	;如果返回值小于等于1，炸弹爆炸
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)			;比较x与7
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>	;如果大于则爆炸
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax			;把x放入%eax
  400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8)		;间接跳转
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax				;%eax等于207
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	call   40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax			;比较y和%eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>	;如果相等，过关
  400fc4:	e8 71 04 00 00       	call   40143a <explode_bomb>	;否则爆炸
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	ret
```

我们首先注意到

```assembly
  400f5b:	e8 90 fc ff ff       	call   400bf0 <__isoc99_sscanf@plt>
```

这里调用了`sscnaf`函数，`sscanf`的 C 语言函数签名为`int sscanf(const char *str, const char *format, ...)`，返回值表示成功匹配并赋值的参数个数。

又注意到

```assembly
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
```

`%esi`寄存器存储第二个参数，即`sscanf`函数的格式化字符串，在终端中输入`x/s 0x4025cf`，发现终端输出

```shell
(gdb) x/s 0x4025cf
0x4025cf:       "%d %d"
```

这表明`sscanf`函数的第二个参数格式化字符串为两个整数，说明函数的第三和第四个参数，即要写入的变量是两个整数。

我们又知道，通常第三和第四个参数用`%rdx`和`%rcx`寄存器存储，注意到这两行

```assembly
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
```

- `lea`（Load Effective Address) 是取地址

- 第三个参数`%rdx`对应第一个`%d`，表明这里存储的地址是栈地址`%rsp + 12`
- 第四个参数`%rcx`对应第二个`%d`，表明这里存储的地址是栈地址`%rsp + 8`
- 这两个地址是`sscanf`函数将要写入的地方，我们不妨称其为`x`和`y`

我们接着来看

```assembly
  400f60:	83 f8 01             	cmp    $0x1,%eax
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>
  400f65:	e8 d0 04 00 00       	call   40143a <explode_bomb>
```

这里对`sscanf`函数的返回值做比较，如果返回值大于 1，跳转到`400f6a`，否则炸弹爆炸。

我们继续向下看

```assembly
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)			;比较x与7
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>	;如果大于则爆炸
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax			;把x放入%eax
```

这说明`x`必须是 **0 到 7** 之间的一个数字(ja 用于无符号数的比较)，

**跳转表**

```assembly
  400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8)		;间接跳转
```

- 这个指令会不是跳转到一个固定的地址，而是先计算出一个地址再跳转
- `Address = *(0x402470 + %rax + 8)`
- `0x402470`是一个数组起始地址
- `%rax`是一个索引
- `8`是因为在 64 位系统中，内存地址占 8 个字节

注意到我们之前已经将把`x`放入`%eax`中了，所以这里会根据我们输入的`x`的值，来决定跳转到哪里

我们打开`gdb`调试器，输入`x/8xg 0x402470`

- `8`表示显示数量，显示 8 个内存单元
- `x`表示显示格式，显示十六进制
- `g`表达数据大小，(Giant words)表示 8 个字节

可以看到终端输出

```shell
0x402470:       0x0000000000400f7c      0x0000000000400fb9
0x402480:       0x0000000000400f83      0x0000000000400f8a
0x402490:       0x0000000000400f91      0x0000000000400f98
0x4024a0:       0x0000000000400f9f      0x0000000000400fa6
```

`gdb`调试器为了节省空间，每行展示了两个内存值，完整显示是这样的

```shell
0x402470:       0x0000000000400f7c
0x402478:       0x0000000000400fb9
0x402480:       0x0000000000400f83
0x402488:       0x0000000000400f8a
0x402490:       0x0000000000400f91
0x402498:       0x0000000000400f98
0x4024a0:       0x0000000000400f9f
0x4024a8:       0x0000000000400fa6
```

这样就很清晰了，这说明我们输入的`x`索引要跳转到的对应的内存地址

我们接着往下看

```assembly
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
```

这里的代码结构高度相似，都是先给`%eax`赋一个值，再跳转到`400fbe`

结合前一步分析，我们可以得出以下映射表

| 输入 x (%rax) | 内存地址 | 跳转目标     | 对应逻辑                |
| ------------- | -------- | ------------ | ----------------------- |
| **0**         | 0x402470 | **0x400f7c** | mov $0xcf, %eax -> 207  |
| **1**         | 0x402478 | **0x400fb9** | mov $0x137, %eax -> 311 |
| **2**         | 0x402480 | **0x400f83** | mov $0x2c3, %eax -> 707 |
| **3**         | 0x402488 | **0x400f8a** | mov $0x100, %eax -> 256 |
| **4**         | 0x402490 | **0x400f91** | mov $0x185, %eax -> 389 |
| **5**         | 0x402498 | **0x400f98** | mov $0xce, %eax -> 206  |
| **6**         | 0x4024a0 | **0x400f9f** | mov $0x2aa, %eax -> 682 |
| **7**         | 0x4024a8 | **0x400fa6** | mov $0x147, %eax -> 327 |

最后比较输入`y`和`%eax`，如果相等则过关，否则炸弹爆炸

```shell
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	call   40143a <explode_bomb>
```

综合以上分析，答案呼之欲出，我们要输入两个数字`x`和`y`，`x`为 0 到 7 之间的一个整数，`y`为与之对应的一个整数

随意填入以下一组数据即可

- 0 207
- 1 311
- 2 707
- 3 256
- 4 389
- 5 206
- 6 682
- 7 327

## phase_4

```assembly
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx				;第二个数y的地址
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx				;第一个数x的地址
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi				;"%d %d"
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	call   400bf0 <__isoc99_sscanf@plt>	;调用函数sscanf
  401029:	83 f8 02             	cmp    $0x2,%eax					;比较返回值%eax与2
  40102c:	75 07                	jne    401035 <phase_4+0x29>		;如果不相等则跳转
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)				;比较14和x
  401033:	76 05                	jbe    40103a <phase_4+0x2e>		;jump if below or equal
  401035:	e8 00 04 00 00       	call   40143a <explode_bomb>		;如果大于则爆炸
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx					;%edx = 14
  40103f:	be 00 00 00 00       	mov    $0x0,%esi					;%esi = 0
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi				;%edi = x
  401048:	e8 81 ff ff ff       	call   400fce <func4>				;调用函数func4
  40104d:	85 c0                	test   %eax,%eax					;检查%eax是否为0
  40104f:	75 07                	jne    401058 <phase_4+0x4c>		;如果非0，炸弹爆炸
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)				;比较y和0
  401056:	74 05                	je     40105d <phase_4+0x51>		;如果相等，过关
  401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>		;否则，炸弹爆炸
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	ret
```

这里前面和`phase_3`类似，不再赘述

同样的，我们要输入两个整数`x`和`y`，`x`必须在 0 到 14 之间

注意，这里调用了一个函数`func4`，传入的参数分别为`x`，`0`，`14`，且`func4`返回值必须为 0

```assembly
  40103a:	mov    $0xe,%edx            ; 参数 3 (%edx) = 14
  40103f:	mov    $0x0,%esi            ; 参数 2 (%esi) = 0
  401044:	mov    0x8(%rsp),%edi       ; 参数 1 (%edi) = x
  401048:	call   400fce <func4>       ; 调用 func4(x, 0, 14)
  40104d:	85 c0                	test   %eax,%eax					;检查%eax是否为0
  40104f:	75 07                	jne    401058 <phase_4+0x4c>		;如果非0，炸弹爆炸
```

我们接着往下看

```assembly
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)				;比较y和0
  401056:	74 05                	je     40105d <phase_4+0x51>		;如果相等，过关
  401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>		;否则，炸弹爆炸
```

这里我们可以推断第二个输入`y`必须为`0`

那么接下来的问题就是`x`了，让我们来看看函数`func4`的代码

- `%edi` = 第 1 个参数 `x`
- `%esi` = 第 2 个参数 `0`
- `%edx` = 第 3 个参数 `14`

```assembly
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax			;%eax = %edx
  400fd4:	29 f0                	sub    %esi,%eax			;%eax -= %esi
  400fd6:	89 c1                	mov    %eax,%ecx			;%ecx = %eax
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx			;逻辑右移，提出符号位
  400fdb:	01 c8                	add    %ecx,%eax			;%eax += %ecx
  400fdd:	d1 f8                	sar    $1,%eax				;%eax /= 2，算术右移
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx	;ecx = eax + esi
  400fe2:	39 f9                	cmp    %edi,%ecx
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx
  400fe9:	e8 e0 ff ff ff       	call   400fce <func4>
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax
  400ff7:	39 f9                	cmp    %edi,%ecx
  400ff9:	7d 0c                	jge    401007 <func4+0x39>
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
  400ffe:	e8 cb ff ff ff       	call   400fce <func4>
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	ret
```

我们将以上汇编代码整理成伪代码的形式

```c
int func4(int x, int a, int b)
{
    eax = b - a;
    ecx = eax;

    eax /= 2;
    ecx = eax + a * 1;  // ecx = ( b - a ) / 2 + a

    if(ecx <= x)
    {
        if(ecx >= x)
        {
            // 也就是 ecx == x
            return 0;
        }
        else
        {
            // ecx < x，递归调用func4
            result = func4(x, ecx + 1, b);
            return 1 + 2 * result;
        }
    }
    else
    {
        // ecx > x，递归调用func4
        result = func4(x, a, ecx - 1);
        return 2 * result;
    }
}

```

到这里就很清楚了，这是一个二分查找的递归函数，ecx 就是 mid，如果

- 如果 mid == x，返回 **0**。
- 如果 mid > x，返回 **2 \* 子结果**。
- 如果 mid < x，返回 **2 \* 子结果 + 1**。

我们前面提到过，`func4`函数返回值必须为**0**，那么返回值如何为 0 呢？就是在二分的过程中，`x`必须一直在左半边( < mid )，或者`x`= mid，因为一旦`x`位于右半边，结果+1，就不可能为 0 了。

所以我们现在要找到一个`x`使得在二分`0`到`14`的情况下，要么 `x < mid`，要么`x == mid`。

下面我们来手动算一下

- left = 0, right = 14
  - mid = 0 + ( 14 - 0 ) / 2 = 7
  - 若 x = 7，mid == x，返回 0
  - 若 x < 7 ，递归调用 func4(x, 0, 6)
- left = 0, right = 6
  - mid = 0 + ( 6 - 0 ) / 2 = 3
  - 若 x = 3, mid == x，返回 0
  - 若 x < 3 ，递归调用 func4(x, 0, 2)
- left = 0, right = 2
  - mid = 0 + ( 2 - 0 ) / 2 = 1
  - 若 x = 1, mid == x，返回 0
  - 若 x < 1( x == 0 ) ，递归调用 func4(x, 0, 0)
- left = 0, right = 0
  - mid = 0 + ( 0 - 0 ) / 2 = 0
  - 若 x = 0，mid == x，返回 0

综上，我们最终的答案(x，y)是

- 7 0
- 3 0
- 1 0
- 0 0

## phase_5

```assembly
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rsp
  401067:	48 89 fb             	mov    %rdi,%rbx				;%rbx=%rdi
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax			;这个不用管
  401071:	00 00
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)			;(%rsp+24)=rax
  401078:	31 c0                	xor    %eax,%eax				;%eax=0
  40107a:	e8 9c 02 00 00       	call   40131b <string_length>	;调用函数string_length
  40107f:	83 f8 06             	cmp    $0x6,%eax				;检查字符串长度
  401082:	74 4e                	je     4010d2 <phase_5+0x70>	;如果长度为6，跳转
  401084:	e8 b1 03 00 00       	call   40143a <explode_bomb>	;否则，炸弹爆炸
  401089:	eb 47                	jmp    4010d2 <phase_5+0x70>
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx		;%ecx=%rbx+%rax*1
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
  401096:	83 e2 0f             	and    $0xf,%edx
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
  4010bd:	e8 76 02 00 00       	call   401338 <strings_not_equal>
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	call   40143a <explode_bomb>
  4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
  4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax				;将%eax初始化为 0
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
  4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
  4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
  4010e5:	00 00
  4010e7:	74 05                	je     4010ee <phase_5+0x8c>
  4010e9:	e8 42 fa ff ff       	call   400b30 <__stack_chk_fail@plt>
  4010ee:	48 83 c4 20          	add    $0x20,%rsp
  4010f2:	5b                   	pop    %rbx
  4010f3:	c3                   	ret
```

我们从`40107a`这一步开始分析

```assembly
  40107a:	e8 9c 02 00 00       	call   40131b <string_length>	;调用函数string_length
  40107f:	83 f8 06             	cmp    $0x6,%eax				;检查字符串长度
  401082:	74 4e                	je     4010d2 <phase_5+0x70>	;如果长度为6，跳转
  401084:	e8 b1 03 00 00       	call   40143a <explode_bomb>	;否则，炸弹爆炸
```

这里表明我们的输入必须为 6 个字符的字符串

继续来看跳转到的`4010d2`

```assembly
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
```

这里将`%eax`初始化为 0，然后跳转回去

我们回头接着看

```assembly
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx		;%ecx = input[i]，取出字符
  40108f:	88 0c 24             	mov    %cl,(%rsp)				;把字符存到栈上
  401092:	48 8b 14 24          	mov    (%rsp),%rdx				;又把字符读回到 %rdx
  401096:	83 e2 0f             	and    $0xf,%edx				;只保留低4位
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx		;查表
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)	;把查到的字符存入这个位置
  4010a4:	48 83 c0 01          	add    $0x1,%rax				;i++
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax				;i<6?
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>	;继续循环
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)			;添加字符串结束符 \0
```

- `movzbl (%rbx,%rax,1),%ecx`，%ecx = %rbx + %rax \* 1，%rbx 中存的是我们的输入字符串，%rax 在这里作为一个索引，初始为 0，这里的意思是取出`input[i]`，i 是索引
- `and    $0xf,%edx`，`0xf`是二进制数`00001111`，这一步只保留了字符的低四位，意味着结果是一个 0 到 15 之间的数字
- `movzbl 0x4024b0(%rdx),%edx`，%edx = 0x4024b0 + %rbx，这一步相当于取 0x4024b0 这个数组上的第%rbx 个索引的值，我们这里输入命令`x/s 0x4024b0`，可以发现输出为`maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?`

我们用 c 语言来描述就是

```c
char *table = (char *)0x4024b0; // 这是一个长度为16的字符数组
char output[7];

for (int i = 0; i < 6; i++) {
    int index = input[i] & 0xF; // 取低4位
    output[i] = table[index];   // 查表替换
}
output[6] = '\0';
```

我们继续向下看

```assembly
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi				;参数2，一个内存地址
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi				;参数1，我们之前生成的字符串output
  4010bd:	e8 76 02 00 00       	call   401338 <strings_not_equal>	;比较我output和这个内存地址中的值
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>		;如果相等，过关
  4010c6:	e8 6f 03 00 00       	call   40143a <explode_bomb>		;否则，爆炸
```

我们输入`x/s 0x40245e`，可以看到终端的输出是`flyers`

综合上述分析，我们推断，`*output = "flyers"`，我们接下来来推断`input`

首先建立 `Table`表的字符和下标的映射关系

| 下标 (Hex) | 0   | 1   | 2   | 3   | 4   | 5   | 6   | 7   | 8   | 9   | 10(a) | 11(b) | 12(c) | 13(d) | 14(e) | 15(f) |
| ---------- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | ----- | ----- | ----- | ----- | ----- | ----- |
| **字符**   | m   | a   | d   | u   | i   | e   | r   | s   | n   | f   | o     | t     | v     | b     | y     | l     |

1. **f** -> 在 Table 中对应下标 **9**
2. **l** -> 在 Table 中对应下标 **15 (0xF)**
3. **y** -> 在 Table 中对应下标 **14 (0xE)**
4. **e** -> 在 Table 中对应下标 **5**
5. **r** -> 在 Table 中对应下标 **6**
6. **s** -> 在 Table 中对应下标 **7**

所以我们输入的 6 个字符，它们的低 4 位依次是：9、15、14、5、6、7

我们知道 ASCLL 码都是 8 比特，为了方便，我们转换成十六进制数就是：0x9、0xf、0xe、0x5、0x6、0x7

对于高 4 位，我们不妨填入 0x6，这里不唯一

那么答案就是，`ionefg`

## phase_6

```assembly
00000000004010f4 <phase_6>:
  4010f4:	41 56                	push   %r14
  4010f6:	41 55                	push   %r13
  4010f8:	41 54                	push   %r12
  4010fa:	55                   	push   %rbp
  4010fb:	53                   	push   %rbx
  4010fc:	48 83 ec 50          	sub    $0x50,%rsp
  401100:	49 89 e5             	mov    %rsp,%r13					;%r13指向栈顶指针%rsp
  401103:	48 89 e6             	mov    %rsp,%rsi
  401106:	e8 51 03 00 00       	call   40145c <read_six_numbers>	;读取6个数字
  40110b:	49 89 e6             	mov    %rsp,%r14
  40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
  401114:	4c 89 ed             	mov    %r13,%rbp
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax
  40111b:	83 e8 01             	sub    $0x1,%eax
  40111e:	83 f8 05             	cmp    $0x5,%eax
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	call   40143a <explode_bomb>
  401128:	41 83 c4 01          	add    $0x1,%r12d
  40112c:	41 83 fc 06          	cmp    $0x6,%r12d
  401130:	74 21                	je     401153 <phase_6+0x5f>
  401132:	44 89 e3             	mov    %r12d,%ebx
  401135:	48 63 c3             	movslq %ebx,%rax
  401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
  40113b:	39 45 00             	cmp    %eax,0x0(%rbp)
  40113e:	75 05                	jne    401145 <phase_6+0x51>
  401140:	e8 f5 02 00 00       	call   40143a <explode_bomb>
  401145:	83 c3 01             	add    $0x1,%ebx
  401148:	83 fb 05             	cmp    $0x5,%ebx
  40114b:	7e e8                	jle    401135 <phase_6+0x41>
  40114d:	49 83 c5 04          	add    $0x4,%r13
  401151:	eb c1                	jmp    401114 <phase_6+0x20>
  401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
  401158:	4c 89 f0             	mov    %r14,%rax
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
  401160:	89 ca                	mov    %ecx,%edx
  401162:	2b 10                	sub    (%rax),%edx
  401164:	89 10                	mov    %edx,(%rax)
  401166:	48 83 c0 04          	add    $0x4,%rax
  40116a:	48 39 f0             	cmp    %rsi,%rax
  40116d:	75 f1                	jne    401160 <phase_6+0x6c>
  40116f:	be 00 00 00 00       	mov    $0x0,%esi
  401174:	eb 21                	jmp    401197 <phase_6+0xa3>
  401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx
  40117a:	83 c0 01             	add    $0x1,%eax
  40117d:	39 c8                	cmp    %ecx,%eax
  40117f:	75 f5                	jne    401176 <phase_6+0x82>
  401181:	eb 05                	jmp    401188 <phase_6+0x94>
  401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)
  40118d:	48 83 c6 04          	add    $0x4,%rsi
  401191:	48 83 fe 18          	cmp    $0x18,%rsi
  401195:	74 14                	je     4011ab <phase_6+0xb7>
  401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx
  40119a:	83 f9 01             	cmp    $0x1,%ecx
  40119d:	7e e4                	jle    401183 <phase_6+0x8f>
  40119f:	b8 01 00 00 00       	mov    $0x1,%eax
  4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
  4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
  4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
  4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
  4011ba:	48 89 d9             	mov    %rbx,%rcx
  4011bd:	48 8b 10             	mov    (%rax),%rdx
  4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)
  4011c4:	48 83 c0 08          	add    $0x8,%rax
  4011c8:	48 39 f0             	cmp    %rsi,%rax
  4011cb:	74 05                	je     4011d2 <phase_6+0xde>
  4011cd:	48 89 d1             	mov    %rdx,%rcx
  4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
  4011d9:	00
  4011da:	bd 05 00 00 00       	mov    $0x5,%ebp
  4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax
  4011e3:	8b 00                	mov    (%rax),%eax
  4011e5:	39 03                	cmp    %eax,(%rbx)
  4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
  4011e9:	e8 4c 02 00 00       	call   40143a <explode_bomb>
  4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx
  4011f2:	83 ed 01             	sub    $0x1,%ebp
  4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
  4011f7:	48 83 c4 50          	add    $0x50,%rsp
  4011fb:	5b                   	pop    %rbx
  4011fc:	5d                   	pop    %rbp
  4011fd:	41 5c                	pop    %r12
  4011ff:	41 5d                	pop    %r13
  401201:	41 5e                	pop    %r14
  401203:	c3                   	ret
```

phase_6 的代码非常长，我们一步步来分析

### 第一步：读取并检查输入

```assembly
401106:	call   40145c <read_six_numbers> ; 读取 6 个数字
```

- `read_six_numbers`函数我们之前在 phase_2 见过了，这个函数读取我们输入的 6 个整数并分配到栈上，此时，栈顶指针`%rsp`指向数组开头`input[0]`的地址

接下来的 `40110b` 到 `401151` 这段代码是一个**双重循环**：

```assembly
40111b:	sub    $0x1,%eax            ; input[i] - 1
40111e:	cmp    $0x5,%eax            ; 必须 <= 5
401121:	jbe    ...                  ; 否则炸
```

- **条件 1**：每个数字必须在 `1` 到 `6` 之间。

```assembly
40113b:	cmp    %eax,0x0(%rbp)       ; 比较 input[i] 和 input[j]
40113e:	jne    ...                  ; 如果相等，炸
```

- **条件 2**：这 6 个数字必须 **互不相同**。
- **结论**：输入的 6 个数是 **1 到 6 的全排列** (比如 `3 2 1 6 5 4`)。

---

### 第二步：数值反转

这段代码很关键：

```assembly
401153:	lea    0x18(%rsp),%rsi      ; 数组结尾
...
401160:	mov    %ecx,%edx            ; %ecx = 7
401162:	sub    (%rax),%edx          ; %edx = 7 - input[i]
401164:	mov    %edx,(%rax)          ; input[i] = 7 - input[i]
```

- **作用**：把每个输入的数字 `x` 变成 `7 - x`。
  - 如果你输入 `1`，它变成 `6`。
  - 如果你输入 `6`，它变成 `1`。

---

### 第三步：链表重组

这就是本关的核心。

```assembly
401183:	mov    $0x6032d0,%edx       ; 【关键】链表头地址 Node 1
```

这段代码（`401176` 到 `4011ab`）的作用是：

- 内存里有 6 个节点（Node），每个节点存了一个值和下一个节点的指针。
- 程序根据你输入的数字（变换后的 `7-x`），把这 6 个节点按照你指定的顺序，**重新排队**。
- 比如你的输入对应的是 `3 1 ...`，那么现在的链表顺序就变成了 `Node 3 -> Node 1 -> ...`。

**我们先不去深究这里的指针操作，直接看最后的检查条件。**

---

### 第四步：链表排序检查

程序最后会遍历这个重新排好序的链表：

```assembly
4011df:	mov    0x8(%rbx),%rax       ; %rax = 当前节点的下一个节点 (Next Node)
4011e3:	mov    (%rax),%eax          ; 取出 下一个节点的值 (Value)
4011e5:	cmp    %eax,(%rbx)          ; 比较 当前节点的值 vs 下一个节点的值
4011e7:	jge    4011ee               ; 如果 Current >= Next，继续
4011e9:	call   <explode_bomb>       ; 否则炸！
```

- **条件**：`Current_Value >= Next_Value`
- **结论**：重组后的链表，节点里的值必须是 **从大到小** 排序的！

---

### 破解行动

我们不需要手动去推指针，我们只需要知道：

1.  这 6 个节点里存的值分别是多少？
2.  哪个节点的值最大，哪个最小？
3.  按从大到小的顺序，它们原本的索引（编号）是多少？

#### 1. 探查链表数据

在 GDB 中，我们看到链表头是 `0x6032d0`。
链表节点的结构通常是：

- `Offset 0`: **Value (4 字节)**
- `Offset 8`: **Next Pointer (8 字节)**

我们来查看这 6 个节点：

**Node 1 (0x6032d0):**

```bash
x/d 0x6032d0        -> 查看值
x/gx 0x6032d0+8     -> 查看下一个节点的地址
```

- **Node 1**: Value = **332** (0x14c)
- **Node 2**: Value = **168** (0xa8)
- **Node 3**: Value = **924** (0x39c)
- **Node 4**: Value = **691** (0x2b3)
- **Node 5**: Value = **477** (0x1dd)
- **Node 6**: Value = **443** (0x1bb)

#### 2. 排序

我们把刚才找到的值按 **从大到小** 排序：

1.  **Node 3**: 924
2.  **Node 4**: 691
3.  **Node 5**: 477
4.  **Node 6**: 443
5.  **Node 1**: 332
6.  **Node 2**: 168

对应的节点编号顺序是：**3, 4, 5, 6, 1, 2**

#### 3. 反转陷阱

还记得第二步里的 `7 - x` 吗？
程序是按 `7 - input` 来重排链表的。
所以：
`7 - input[0] = 3` => `input[0] = 4`
`7 - input[1] = 4` => `input[1] = 3`
`7 - input[2] = 5` => `input[2] = 2`
`7 - input[3] = 6` => `input[3] = 1`
`7 - input[4] = 1` => `input[4] = 6`
`7 - input[5] = 2` => `input[5] = 5`

**最终答案：** **4 3 2 1 6 5**
