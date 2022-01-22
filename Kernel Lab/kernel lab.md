主要来熟悉一下windows内核，全部都是在xp系统下运行的，非常入门。

写一些自己不熟悉的地方，有些东西可能实验了，觉得不太重要就没用写。

配环境自己配吧，配个双机调试环境就行

教程B站[_周壑的个人空间_哔哩哔哩_bilibili](https://space.bilibili.com/37877654/video?tid=0&page=3&keyword=&order=pubdate)

可以用VirtualKd其他也行

调试机：Windows10 10.0.19042

被调试机：Windows XP sp3

工具：Windbg，VS2019，PcHunter

有些问题都写在后面了。

# 0x01 lab 01

**为什么要做这个实验？**

我们需要在0环下操作，并且执行一些程序，但我们一般程序都是在3环下执行的，根本碰不到0环。那么这里就需要找一些0环权限的操作，修改中断向量表就是一个很好的办法。

我们把中断向量表修改成我们自己的函数地址，从而执行我们自己想要的操作，这个实验的目的也就达成了。

## 1. goal

1. 熟悉中断描述符表，中断异常等。
2. 能够用汇编读取或修改0环的内存。

## 2. step

可以先查看一下中断向量表

中断向量表(idtr interrupt descriptor table register)

```
kd> r idtr
idtr=8003f400
kd> dq idtr l40
8003f400  80538e00`0008f3bc 80538e00`0008f534
8003f410  00008500`0058113e 8053ee00`0008f904
8003f420  8053ee00`0008fa84 80538e00`0008fbe0
8003f430  80538e00`0008fd54 80548e00`000803bc
8003f440  00008500`00501198 80548e00`000807e0
8003f450  80548e00`00080900 80548e00`00080a40
8003f460  80548e00`00080c9c 80548e00`00080f80
8003f470  80548e00`00081694 80548e00`0008190c
8003f480  80548e00`00081a2c 80548e00`00081b64
8003f490  80548500`00a0190c 80548e00`00081ccc
8003f4a0  80548e00`0008190c 80548e00`0008190c
8003f4b0  80548e00`0008190c 80548e00`0008190c
8003f4c0  80548e00`0008190c 80548e00`0008190c
8003f4d0  80548e00`0008190c 80548e00`0008190c
8003f4e0  80548e00`0008190c 80548e00`0008190c
8003f4f0  80548e00`0008190c 806d8e00`00083fd0
8003f500  00000000`00080000 00000000`00080000
8003f510  00000000`00080000 00000000`00080000
8003f520  00000000`00080000 00000000`00080000
8003f530  00000000`00080000 00000000`00080000
8003f540  00000000`00080000 00000000`00080000
8003f550  8053ee00`0008ebfe 8053ee00`0008ed00
8003f560  8053ee00`0008eea0 8053ee00`0008f7e0
8003f570  8053ee00`0008e691 80548e00`0008190c
8003f580  80538e00`0008dd50 80538e00`0008dd5a
8003f590  80538e00`0008dd64 80538e00`0008dd6e
8003f5a0  80538e00`0008dd78 80538e00`0008dd82
8003f5b0  80538e00`0008dd8c 806d8e00`00083728
8003f5c0  80538e00`0008dda0 80538e00`0008ddaa
8003f5d0  80538e00`0008ddb4 80538e00`0008ddbe
8003f5e0  80538e00`0008ddc8 806d8e00`00084b70
8003f5f0  80538e00`0008dddc 80538e00`0008dde6
```

开xp来看一下PcHunter中的内容

![image-20220119152743368](kernel%20lab/image-20220119152743368.png)

根据PcHunter，我们可以知道中断函数地址和中断向量表中的内容对应位置。

其实这是保护模式的相关内容，具体可以了解一下保护模式，这里就只看我们相关的

我们需要关注一个东西DPL(descriptor Privilege Level)，这是有关中断门的内容

具体可以看intel白皮书，

![image-20220119172759753](kernel%20lab/image-20220119172759753.png)

可以看到中断门中的内容，OFFSET就是函数地址，DPL是占两位，

它主要是表示该处是在什么权限下运行的，就是可以接收哪里的异常。

向我们上面所见到的断点是e（3环）

![image-20220119173907015](kernel%20lab/image-20220119173907015.png)

而我们的除0就是8（0环）

![image-20220119174055696](kernel%20lab/image-20220119174055696.png)

简单来说，断点（int 3）我们是可以直接在应用程序中直接运行的，像常见的int3断点

而除0（int 0）我们是无法在程序中运行的，他会直接返回访问错误，我们是无法用int 0去访问的。

我们为了触发一个中断，需要修改idtr，那么应该修改成什么值呢？

很简单模仿上面的，需要修改DPL和函数地址就可以

```
kd> eq 8003f500 0040ee00`00081040
```

具体代码**（lab.exe）**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
DWORD temp;
//0x00401040
void __declspec(naked) idtEntry()
{

	__asm {
		mov eax, dword ptr ds:[0x8003f5f0]
		mov temp, eax
		iretd
	}


}
//eq 8003f500 0040ee00`00081040
void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	printf("0x%p\n", temp);
	system("pause");
}
```

>- 注意汇编代码段寻址的完整编写

![image-20220119180238015](kernel%20lab/image-20220119180238015.png)

可以看到和我们上面的windbg打印内容是一样的。

# 0x02 lab 02

## 1. goal

1. 理解多核对程序的影响

## 2. step

进入内核之后，IF是直接被关中断的

eflags的IF标志位为0

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
DWORD temp;
//0x00401040
void __declspec(naked) idtEntry()
{

	__asm {
		pushfd
		pop eax
		mov temp,eax
		iretd
	}


}

void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	printf("0x%p\n", temp);
	system("pause");
}
```

![image-20220119231106547](kernel%20lab/image-20220119231106547.png)

多核CPU的所有东西都是两份的，如IDT表等

windbg可以切换不同CPU

```
kd> ~1
```

由于不同CPU会接管不同程序，所以在IDT Hook时候要多核都要考虑

正常情况下，一些内核地址是不可写入的，我们需要关闭写保护，

关闭写保护

```
mov eax, cr0
and eax, not 10000h
mov cr0, eax
```

查看页属性

```
kd> !pte 8003f400
                    VA 8003f400
PDE at C0602000            PTE at C04001F8
contains 0000000000749163  contains 000000000003F163
pfn 749       -G-DA--KWEV  pfn 3f        -G-DA--KWEV

```

>可写一定可读，可读不一定可写

单核代码

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
DWORD temp;
//0x00401040
void __declspec(naked) idtEntry()
{

	__asm {
		mov eax, cr0
		and eax, not 10000h
		mov cr0, eax
		mov eax, 0xffffffff
		mov ds:[0x80542520,eax]
Label:
		jmp Label
		iretd
	}


}

void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	printf("0x%p\n", temp);
	system("pause");
}
```



# 0x03 lab 03

## 1. goal

1. 理解中断时所处环境
2. 明白CPU的哪些资源可以调度

## 2. step

先打印一下各个寄存器的值 **(lab3.1.exe)**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
DWORD temp;
DWORD dEax[2], dEbx[2], dEcx[2], dEdx[2];
DWORD dEsp[2], dEbp[2], dEsi[2], dEdi[2];
WORD wCs[2], wDs[2], wSs[2], wEs[2], wFs[2], wGs[2];

//0x00401040
void __declspec(naked) idtEntry()
{

	__asm {
		mov [dEax+4], eax
		mov [dEbx + 4], ebx
		mov [dEcx + 4], ecx
		mov [dEdx + 4], edx
		mov [dEsp + 4], esp
		mov [dEbp + 4], ebp
		mov [dEsi + 4], esi
		mov [dEdi + 4], edi
		mov [wCs+2], cs
		mov [wDs+2], ds
		mov [wSs+2], ss
		mov [wEs+2], es
		mov [wFs+2], fs
		mov [wGs+2], gs
		iretd
	}
}

void go()
{
	__asm {
		mov [dEax], eax
		mov [dEbx], ebx
		mov [dEcx], ecx
		mov [dEdx], edx
		mov [dEsp], esp
		mov [dEbp], ebp
		mov [dEsi], esi
		mov [dEdi], edi
		mov [wCs], cs
		mov [wDs], ds
		mov [wSs], ss
		mov [wEs], es
		mov [wFs], fs
		mov [wGs], gs
	
	}
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	printf("[+]Pre eax:0x%p, ebx:0x%p, ecx:0x%p, edx:0x%p, esi:0x%p, edi:0x%p, esp:0x%p, ebp:0x%p\n", dEax[0], dEbx[0], dEcx[0], dEdx[0], dEsi[0], dEdi[0], dEsp[0], dEbp[0]);
	printf("[+]Pre cs:0x%p, ds:0x%p, ss:0x%p, es:0x%p, fs:0x%p, gs:0x%p\n", wCs[0], wDs[0], wSs[0], wEs[0], wFs[0], wGs[0]);
	printf("[+]Now eax:0x%p, ebx:0x%p, ecx:0x%p, edx:0x%p, esi:0x%p, edi:0x%p, esp:0x%p, ebp:0x%p\n", dEax[1], dEbx[1], dEcx[1], dEdx[1], dEsi[1], dEdi[1], dEsp[1], dEbp[1]);
	printf("[+]Now cs:0x%p, ds:0x%p, ss:0x%p, es:0x%p, fs:0x%p, gs:0x%p\n", wCs[1], wDs[1], wSs[1], wEs[1], wFs[1], wGs[1]);
	//printf("0x%p\n", temp);
	system("pause");
}
```

![image-20220120140931218](kernel%20lab/image-20220120140931218.png)

修改的有cs，ss，esp

接下来就是保护模式的内容

先看gdtr

```
kd> r gdtr
gdtr=8003f000
kd> r gdtl
gdtl=000003ff
kd> dq gdtr l80
8003f000  00000000`00000000 00cf9b00`0000ffff
8003f010  00cf9300`0000ffff 00cffb00`0000ffff
8003f020  00cff300`0000ffff 80008b04`200020ab
8003f030  ffc093df`f0000001 0040f300`00000fff
8003f040  0000f200`0400ffff 00000000`00000000
8003f050  80008954`b1000068 80008954`b1680068
8003f060  00009302`2f40ffff 0000920b`80003fff
8003f070  ff0092ff`700003ff 80009a40`0000ffff
8003f080  80009240`0000ffff 00009200`00000000
8003f090  00000000`00000000 00000000`00000000
8003f0a0  820089f8`51f80068 00000000`00000000
8003f0b0  00000000`00000000 00000000`00000000
8003f0c0  00000000`00000000 00000000`00000000
8003f0d0  00000000`00000000 00000000`00000000
8003f0e0  f8009f51`f000ffff 00009200`0000ffff
8003f0f0  8000984f`b6b003b7 00009200`0000ffff
8003f100  f7409351`a400ffff f7409351`a400ffff
8003f110  f7409351`a400ffff 00000000`8003f120
8003f120  00000000`8003f128 00000000`8003f130
8003f130  00000000`8003f138 00000000`8003f140
8003f140  00000000`8003f148 00000000`8003f150
8003f150  00000000`8003f158 00000000`8003f160
8003f160  00000000`8003f168 00000000`8003f170
8003f170  00000000`8003f178 00000000`8003f180
8003f180  00000000`8003f188 00000000`8003f190
8003f190  00000000`8003f198 00000000`8003f1a0
8003f1a0  00000000`8003f1a8 00000000`8003f1b0
8003f1b0  00000000`8003f1b8 00000000`8003f1c0
8003f1c0  00000000`8003f1c8 00000000`8003f1d0
8003f1d0  00000000`8003f1d8 00000000`8003f1e0
8003f1e0  00000000`8003f1e8 00000000`8003f1f0
8003f1f0  00000000`8003f1f8 00000000`8003f200
8003f200  00000000`8003f208 00000000`8003f210
8003f210  00000000`8003f218 00000000`8003f220
8003f220  00000000`8003f228 00000000`8003f230
8003f230  00000000`8003f238 00000000`8003f240
8003f240  00000000`8003f248 00000000`8003f250
8003f250  00000000`8003f258 00000000`8003f260
8003f260  00000000`8003f268 00000000`8003f270
8003f270  00000000`8003f278 00000000`8003f280
8003f280  00000000`8003f288 00000000`8003f290
8003f290  00000000`8003f298 00000000`8003f2a0
8003f2a0  00000000`8003f2a8 00000000`8003f2b0
8003f2b0  00000000`8003f2b8 00000000`8003f2c0
8003f2c0  00000000`8003f2c8 00000000`8003f2d0
8003f2d0  00000000`8003f2d8 00000000`8003f2e0
8003f2e0  00000000`8003f2e8 00000000`8003f2f0
8003f2f0  00000000`8003f2f8 00000000`8003f300
8003f300  00000000`8003f308 00000000`8003f310
8003f310  00000000`8003f318 00000000`8003f320
8003f320  00000000`8003f328 00000000`8003f330
8003f330  00000000`8003f338 00000000`8003f340
8003f340  00000000`8003f348 00000000`8003f350
8003f350  00000000`8003f358 00000000`8003f360
8003f360  00000000`8003f368 00000000`8003f370
8003f370  00000000`8003f378 00000000`8003f380
8003f380  00000000`8003f388 00000000`8003f390
8003f390  00000000`8003f398 00000000`8003f3a0
8003f3a0  00000000`8003f3a8 00000000`8003f3b0
8003f3b0  00000000`8003f3b8 00000000`8003f3c0
8003f3c0  00000000`8003f3c8 00000000`8003f3d0
8003f3d0  00000000`8003f3d8 00000000`8003f3e0
8003f3e0  00000000`8003f3e8 00000000`8003f3f0
8003f3f0  00000000`8003f3f8 00000000`00000000

```

描述符

![image-20220120142356469](kernel%20lab/image-20220120142356469.png)

![image-20220120142941047](kernel%20lab/image-20220120142941047.png)

>内存区域：基址，界限，用途
>
>门类：权限变化，跳转目标

根据上面我们看一个描述符

```
ffc093df`f0000001

base 		   -> ffdff000
segment limit   -> 0001
descprtion	   -> c093 --> 1100 | 0000 | 1001 | 0011 
DPL: 00
P: 1
S: 1
```

上PcHunter看一下

![image-20220120144539266](kernel%20lab/image-20220120144539266.png)

可以看到验证一样

说回cs，cs里面是段选择子

```
prev cs -> 1B 
0x1B = 0x18 + 0x3
0x18 = 0x3 << 3
00cffb00`0000ffff
DPL: 3

now cs -> 08
0x08 = 0x1 << 3
00cf9b00`0000ffff
DPL: 0
```

那么代码段是从ring3到ring 0，那么cs是从哪里来的呢

我们知道，我们是通过中断门进入内核的，那我们就来看一下我们的中断门描述符

```
8003f500  0040ee00`00081040 00000000`00080000
```

根据上面的图，我们可以知道

```
segement selector -> 0008
cs				 -> 08
```

接下来我们去看一下ss和esp，这里需要讲一下TSS段和TSS描述符

先看TSS描述符

![image-20220120165734116](kernel%20lab/image-20220120165734116.png)

```
kd> dq gdtr
8003f000  00000000`00000000 00cf9b00`0000ffff
8003f010  00cf9300`0000ffff 00cffb00`0000ffff
8003f020  00cff300`0000ffff 80008b04`200020ab

base	-> 80842000

kd> dd 0x80042000
80042000  0c458b24 8054aef0 8b080010 758b0855
80042010  eac14008 ffe68110 030000ff 00736000
80042020  e1750855 08458b5e 0310e8c1 c25d0845
80042030  ff8b000c 8bec8b55 c9330845 f7104d39
80042040  8b1f76d0 b60f0c55 d0331114 00ffe281
80042050  e8c10000 95043308 00433990 104d3b41
80042060  d0f70000 20ac0000 18000004 00000018
80042070  00000000 00000000 00000000 00000000
```

![image-20220120165605665](kernel%20lab/image-20220120165605665.png)

通过上面图，我们可以发现esp0

```
esp0 8054aef0
```

和我们上面看到的esp好像是不一样的

为什么呢？

- 是我们TSS段选错了吗
- 还是因为调试器影响产生了错误

CPU中有一个寄存器存储任务寄存器

```
kd> r tr
tr=00000028
```

我们可以读一下

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
DWORD temp;
DWORD dEax[2], dEbx[2], dEcx[2], dEdx[2];
DWORD dEsp[2], dEbp[2], dEsi[2], dEdi[2];
WORD wCs[2], wDs[2], wSs[2], wEs[2], wFs[2], wGs[2];
DWORD g_8003f038;
DWORD g_8003f03c;
WORD g_tr;
//0x00401040
void __declspec(naked) idtEntry()
{

	__asm {
		mov [dEax+4], eax
		mov [dEbx + 4], ebx
		mov [dEcx + 4], ecx
		mov [dEdx + 4], edx
		mov [dEsp + 4], esp
		mov [dEbp + 4], ebp
		mov [dEsi + 4], esi
		mov [dEdi + 4], edi
		mov [wCs+2], cs
		mov [wDs+2], ds
		mov [wSs+2], ss
		mov [wEs+2], es
		mov [wFs+2], fs
		mov [wGs+2], gs
		mov eax, ds:[0x8003f038]
		mov g_8003f038, eax
		mov eax, ds : [0x8003f03c]
		mov g_8003f03c, eax
		str ax
		mov g_tr, ax
		iretd
	}
}

void go()
{
	__asm {
		mov [dEax], eax
		mov [dEbx], ebx
		mov [dEcx], ecx
		mov [dEdx], edx
		mov [dEsp], esp
		mov [dEbp], ebp
		mov [dEsi], esi
		mov [dEdi], edi
		mov [wCs], cs
		mov [wDs], ds
		mov [wSs], ss
		mov [wEs], es
		mov [wFs], fs
		mov [wGs], gs
	
	}
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	printf("[+]Pre eax:0x%p, ebx:0x%p, ecx:0x%p, edx:0x%p, esi:0x%p, edi:0x%p, esp:0x%p, ebp:0x%p\n", dEax[0], dEbx[0], dEcx[0], dEdx[0], dEsi[0], dEdi[0], dEsp[0], dEbp[0]);
	printf("[+]Pre cs:0x%p, ds:0x%p, ss:0x%p, es:0x%p, fs:0x%p, gs:0x%p\n", wCs[0], wDs[0], wSs[0], wEs[0], wFs[0], wGs[0]);
	printf("[+]Now eax:0x%p, ebx:0x%p, ecx:0x%p, edx:0x%p, esi:0x%p, edi:0x%p, esp:0x%p, ebp:0x%p\n", dEax[1], dEbx[1], dEcx[1], dEdx[1], dEsi[1], dEdi[1], dEsp[1], dEbp[1]);
	printf("[+]Now cs:0x%p, ds:0x%p, ss:0x%p, es:0x%p, fs:0x%p, gs:0x%p\n", wCs[1], wDs[1], wSs[1], wEs[1], wFs[1], wGs[1]);
	//printf("0x%p\n", temp);
	printf("DWORD g_8003f038:%p\n", g_8003f038);
	printf("DWORD g_8003f03c:%p\n", g_8003f03c);
	printf("g_tr:%p\n", g_tr);
	system("pause");
}
```

![image-20220120171454629](kernel%20lab/image-20220120171454629.png)

读出来发现就是28

**说明我们选择的任务描述符是没错的，那么是不是我们的调试器影响了这个内存？**

我们可以拿汇编去读一下这个内容(**lab3.2.exe**)

读出来之后发现确实是调试器影响了内存，但还是有一些差距

![image-20220120181935422](kernel%20lab/image-20220120181935422.png)

```
[+]Pre eax:0x00401040, ebx:0x7FFDE000, ecx:0xB9E38100, edx:0x00414A74, esi:0x00156A18, edi:0x00155C70, esp:0x0012FF7C, ebp:0x0012FFC0
[+]Pre cs:0x0000001B, ds:0x00000023, ss:0x00000023, es:0x00000023, fs:0x0000003B, gs:0x00000000
[+]Now eax:0x00401040, ebx:0x7FFDE000, ecx:0xB9E38100, edx:0x00414A74, esi:0x00156A18, edi:0x00155C70, esp:0xB1E76DCC, ebp:0x0012FFC0
[+]Now cs:0x00000008, ds:0x00000023, ss:0x00000010, es:0x00000023, fs:0x0000003B, gs:0x00000000
DWORD g_8003f038:D0000FFF
DWORD g_8003f03c:7F40F3FD
DWORD g_80042004:B1E76DE0
DWORD g_80042008:8B080010
g_tr:00000028

0xB1E76DE0-0xB1E76DCC = 0x14
```

>中断门不会用到其他东西，只用到ss和esp

```
kd> dps esp
f8946dcc  00401151	<- eip
f8946dd0  0000001b	<- ring3 cs
f8946dd4  00000246	<- ring3 eflags
f8946dd8  0012ff7c	<- ring3 esp
f8946ddc  00000023	<- ring3 ss
```

从上面我们可以有自己的概括

1. **ring3 -> ring 0 通过中断int20，找中断描述符从而寻找中断函数，此时我们已经由0环进入了3环，并且把eip cs eflags esp ss压栈，这是为了0环回3环时恢复现场，通过iretd**
2. **具体cs，ss，esp是如何变化的呢？cs就是更换代码段，它主要变化是通过中断门的选择子进行切换**
3. **ss和esp就是通过任务段描述符进行切换**

![image-20220120180045644](kernel%20lab/image-20220120180045644.png)

由上面我们已经知道在内核中是关中断的，那么如何开中断进行调用呢？

我们可以看内核文件中ms是怎么开中断的

![image-20220121131502818](kernel%20lab/image-20220121131502818.png)

从上面图可以看到fs是0x30

那么也就是

```
kd> dq gdtr
8003f000  00000000`00000000 00cf9b00`0000ffff
8003f010  00cf9300`0000ffff 00cffb00`0000ffff
8003f020  00cff300`0000ffff 80008b04`200020ab
8003f030  ffc093df`f0000001 0040f300`00000fff

fs	-> ffdff000
```

这个指向的是KPCR

```
kd> dt _KPCR ffdff000
nt!_KPCR
   +0x000 NtTib            : _NT_TIB
   +0x01c SelfPcr          : 0xffdff000 _KPCR
   +0x020 Prcb             : 0xffdff120 _KPRCB
   +0x024 Irql             : 0 ''
   +0x028 IRR              : 0
   +0x02c IrrActive        : 0
   +0x030 IDR              : 0xffffffff
   +0x034 KdVersionBlock   : 0x80546cb8 Void
   +0x038 IDT              : 0x8003f400 _KIDTENTRY	<- IDT
   +0x03c GDT              : 0x8003f000 _KGDTENTRY	<- GDT
   +0x040 TSS              : 0x80042000 _KTSS		<- TSS
   +0x044 MajorVersion     : 1
   +0x046 MinorVersion     : 1
   +0x048 SetMember        : 1
   +0x04c StallScaleFactor : 0xc7a
   +0x050 DebugActive      : 0 ''
   +0x051 Number           : 0 ''
   +0x052 Spare0           : 0 ''
   +0x053 SecondLevelCacheAssociativity : 0x8 ''
   +0x054 VdmAlert         : 0
   +0x058 KernelReserved   : [14] 0
   +0x090 SecondLevelCacheSize : 0x80000
   +0x094 HalReserved      : [16] 0
   +0x0d4 InterruptMode    : 0
   +0x0d8 Spare1           : 0 ''
   +0x0dc KernelReserved2  : [17] 0
   +0x120 PrcbData         : _KPRCB
```

从而我们可以知道是因为我们没有把fs设为30

写代码**(lab3.3.exe)**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
DWORD temp;
DWORD dEax[2], dEbx[2], dEcx[2], dEdx[2];
DWORD dEsp[2], dEbp[2], dEsi[2], dEdi[2];
WORD wCs[2], wDs[2], wSs[2], wEs[2], wFs[2], wGs[2];
DWORD g_8003f038;
DWORD g_8003f03c;
WORD g_tr;
DWORD g_80042004;
DWORD g_80042008;
//0x00401040
void __declspec(naked) idtEntry()
{

	__asm {
		push 0x30
		pop fs
		sti
	L:
		jmp L
		iretd
	}
}

void go()
{

	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	system("pause");
}
```

也可以直接硬编码

```
	__asm {
		push 0x30
		__emit 0x6a
		__emit 0x30
		sti
		iretd
	}
```

可以发现电脑关不掉程序，但相比关中断，我们发现可以运行其他程序

![image-20220121133052849](kernel%20lab/image-20220121133052849.png)

关闭办法就是把jmp指令修改

```
kd> u 401045
00401045 ebfe            jmp     00401045
00401047 cf              iretd
00401048 cc              int     3
00401049 cc              int     3
0040104a cc              int     3
0040104b cc              int     3
0040104c cc              int     3
0040104d cc              int     3
kd> ew 00401045 9090
kd> u 401045
00401045 90              nop
00401046 90              nop
00401047 cf              iretd
00401048 cc              int     3
00401049 cc              int     3
0040104a cc              int     3
0040104b cc              int     3
0040104c cc              int     3
```

可以发现程序已经关闭了

----

接下来就是题外话

```
PROCESS 8294f020  SessionId: 0  Cid: 0234    Peb: 7ffde000  ParentCid: 05d8
    DirBase: 092201a0  ObjectTable: e2990ae0  HandleCount:   7.
    Image: kernel lab1.exe
kd> dt _eprocess  8294f020 
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x06c ProcessLock      : _EX_PUSH_LOCK
   +0x070 CreateTime       : _LARGE_INTEGER 0x01d80de5`27b86f20
   +0x078 ExitTime         : _LARGE_INTEGER 0x0
   +0x080 RundownProtect   : _EX_RUNDOWN_REF
   +0x084 UniqueProcessId  : 0x00000234 Void
   +0x088 ActiveProcessLinks : _LIST_ENTRY [ 0x8055b358 - 0x82b870a8 ]
   +0x090 QuotaUsage       : [3] 0x898
   +0x09c QuotaPeak        : [3] 0x898
   +0x0a8 CommitCharge     : 0xda
   +0x0ac PeakVirtualSize  : 0x750000
   +0x0b0 VirtualSize      : 0x750000
   +0x0b4 SessionProcessLinks : _LIST_ENTRY [ 0xf89c5014 - 0x82b870d4 ]
   +0x0bc DebugPort        : (null) 
   +0x0c0 ExceptionPort    : 0xe158df68 Void
   +0x0c4 ObjectTable      : 0xe2990ae0 _HANDLE_TABLE
   +0x0c8 Token            : _EX_FAST_REF
   +0x0cc WorkingSetLock   : _FAST_MUTEX
   +0x0ec WorkingSetPage   : 0x1c888
   +0x0f0 AddressCreationLock : _FAST_MUTEX
   +0x110 HyperSpaceLock   : 0
   +0x114 ForkInProgress   : (null) 
   +0x118 HardwareTrigger  : 0
   +0x11c VadRoot          : 0x8295bc10 Void
   +0x120 VadHint          : 0x829582b8 Void
   +0x124 CloneRoot        : (null) 
   +0x128 NumberOfPrivatePages : 0xd3
   +0x12c NumberOfLockedPages : 0
   +0x130 Win32Process     : 0xe12aa008 Void
   +0x134 Job              : (null) 
   +0x138 SectionObject    : 0xe29c44a0 Void
   +0x13c SectionBaseAddress : 0x00400000 Void
   +0x140 QuotaBlock       : 0x82ab6a10 _EPROCESS_QUOTA_BLOCK
   +0x144 WorkingSetWatch  : (null) 
   +0x148 Win32WindowStation : 0x00000024 Void
   +0x14c InheritedFromUniqueProcessId : 0x000005d8 Void
   +0x150 LdtInformation   : (null) 
   +0x154 VadFreeHint      : (null) 
   +0x158 VdmObjects       : (null) 
   +0x15c DeviceMap        : 0xe1c421b0 Void
   +0x160 PhysicalVadList  : _LIST_ENTRY [ 0x8294f180 - 0x8294f180 ]
   +0x168 PageDirectoryPte : _HARDWARE_PTE
   +0x168 Filler           : 0
   +0x170 Session          : 0xf89c5000 Void
   +0x174 ImageFileName    : [16]  "kernel lab1.exe"
   +0x184 JobLinks         : _LIST_ENTRY [ 0x0 - 0x0 ]
   +0x18c LockedPagesList  : (null) 
   +0x190 ThreadListHead   : _LIST_ENTRY [ 0x829dd24c - 0x829dd24c ]
   +0x198 SecurityPort     : (null) 
   +0x19c PaeTop           : 0xf8b771a0 Void
   +0x1a0 ActiveThreads    : 1
   +0x1a4 GrantedAccess    : 0x1f0fff
   +0x1a8 DefaultHardErrorProcessing : 1
   +0x1ac LastThreadExitStatus : 0n0
   +0x1b0 Peb              : 0x7ffde000 _PEB
   +0x1b4 PrefetchTrace    : _EX_FAST_REF
   +0x1b8 ReadOperationCount : _LARGE_INTEGER 0x0
   +0x1c0 WriteOperationCount : _LARGE_INTEGER 0x0
   +0x1c8 OtherOperationCount : _LARGE_INTEGER 0xea
   +0x1d0 ReadTransferCount : _LARGE_INTEGER 0x0
   +0x1d8 WriteTransferCount : _LARGE_INTEGER 0x0
   +0x1e0 OtherTransferCount : _LARGE_INTEGER 0x0
   +0x1e8 CommitChargeLimit : 0
   +0x1ec CommitChargePeak : 0x106
   +0x1f0 AweInfo          : (null) 
   +0x1f4 SeAuditProcessCreationInfo : _SE_AUDIT_PROCESS_CREATION_INFO
   +0x1f8 Vm               : _MMSUPPORT
   +0x238 LastFaultCount   : 0
   +0x23c ModifiedPageCount : 0
   +0x240 NumberOfVads     : 0x37
   +0x244 JobStatus        : 0
   +0x248 Flags            : 0x50800
   +0x248 CreateReported   : 0y0
   +0x248 NoDebugInherit   : 0y0
   +0x248 ProcessExiting   : 0y0
   +0x248 ProcessDelete    : 0y0
   +0x248 Wow64SplitPages  : 0y0
   +0x248 VmDeleted        : 0y0
   +0x248 OutswapEnabled   : 0y0
   +0x248 Outswapped       : 0y0
   +0x248 ForkFailed       : 0y0
   +0x248 HasPhysicalVad   : 0y0
   +0x248 AddressSpaceInitialized : 0y10
   +0x248 SetTimerResolution : 0y0
   +0x248 BreakOnTermination : 0y0
   +0x248 SessionCreationUnderway : 0y0
   +0x248 WriteWatch       : 0y0
   +0x248 ProcessInSession : 0y1
   +0x248 OverrideAddressSpace : 0y0
   +0x248 HasAddressSpace  : 0y1
   +0x248 LaunchPrefetched : 0y0
   +0x248 InjectInpageErrors : 0y0
   +0x248 VmTopDown        : 0y0
   +0x248 Unused3          : 0y0
   +0x248 Unused4          : 0y0
   +0x248 VdmAllowed       : 0y0
   +0x248 Unused           : 0y00000 (0)
   +0x248 Unused1          : 0y0
   +0x248 Unused2          : 0y0
   +0x24c ExitStatus       : 0n259
   +0x250 NextPageColor    : 0xa0cb
   +0x252 SubSystemMinorVersion : 0x1 ''
   +0x253 SubSystemMajorVersion : 0x5 ''
   +0x252 SubSystemVersion : 0x501
   +0x254 PriorityClass    : 0x2 ''
   +0x255 WorkingSetAcquiredUnsafe : 0 ''
   +0x258 Cookie           : 0x26f09902
kd> !process 8294f020
Failed to get VadRoot
PROCESS 8294f020  SessionId: 0  Cid: 0234    Peb: 7ffde000  ParentCid: 05d8
    DirBase: 092201a0  ObjectTable: e2990ae0  HandleCount:   7.
    Image: kernel lab1.exe
    VadRoot 00000000 Vads 0 Clone 0 Private 211. Modified 0. Locked 0.
    DeviceMap e1c421b0
    Token                             e29d5030
    ElapsedTime                       00:00:00.046
    UserTime                          00:00:00.046
    KernelTime                        00:00:00.000
    QuotaPoolUsage[PagedPool]         0
    QuotaPoolUsage[NonPagedPool]      0
    Working Set Sizes (now,min,max)  (477, 50, 345) (1908KB, 200KB, 1380KB)
    PeakWorkingSetSize                477
    VirtualSize                       7 Mb
    PeakVirtualSize                   7 Mb
    PageFaultCount                    469
    MemoryPriority                    FOREGROUND
    BasePriority                      8
    CommitCharge                      218

        THREAD 829dd020  Cid 0234.04a4  Teb: 7ffdd000 Win32Thread: 00000000 RUNNING on processor 0
        Not impersonating
        DeviceMap                 e1c421b0
        Owning Process            00000000       Image:         <Invalid process>
        Attached Process          8294f020       Image:         kernel lab1.exe
        Wait Start TickCount      740660         Ticks: 0
        Context Switch Count      19             IdealProcessor: 0             
        UserTime                  00:00:00.031
        KernelTime                00:00:00.000
        Win32 Start Address 0x004014f5
        Stack Init b238c000 Current b238b598 Base b238c000 Limit b2389000 Call 00000000
        Priority 8 BasePriority 8 PriorityDecrement 0 IoPriority 0 PagePriority 0
        ChildEBP RetAddr      
WARNING: Frame IP not in any known module. Following frames may be wrong.
        b238be28 7c930222     0x4010ce
        b238be2c 7c93019b     0x7c930222
        b238be30 7c9301db     0x7c93019b
        b238bef4 7c809acc     0x7c9301db
        b238bef8 0007fa1c     0x7c809acc
        b238befc 7c839b48     0x7fa1c
        b238bf00 7c809ae0     0x7c839b48
        b238bf04 0007dbe6     0x7c809ae0
        b238bf08 00000000     0x7dbe6
```

> 不要过度依赖调试器，看一个例子
>
> 下面这个是windbg最新版的
>
> ```
> kd> dq gdtr
> 8003f000  00000000`00000000 00cf9b00`0000ffff
> 8003f010  00cf9300`0000ffff 00cffb00`0000ffff
> 8003f020  00cff300`0000ffff 80008b04`200020ab
> 8003f030  ffc093df`f0000001 7f40f3fd`e0000fff
> 8003f040  0000f200`0400ffff 00000000`00000000
> 8003f050  80008954`b1000068 80008954`b1680068
> 8003f060  00009302`2f40ffff 0000920b`80003fff
> 8003f070  ff0092ff`700003ff 80009a40`0000ffff
> ```
>
> 看上面的
>
> ```
> kd> dq gdtr l80
> 8003f000  00000000`00000000 00cf9b00`0000ffff
> 8003f010  00cf9300`0000ffff 00cffb00`0000ffff
> 8003f020  00cff300`0000ffff 80008b04`200020ab
> 8003f030  ffc093df`f0000001 0040f300`00000fff
> 8003f040  0000f200`0400ffff 00000000`00000000
> ```
>
> 我们可以发现gdtr[7]这个位置的值是不一样的。
>
> ```
> windbg preview
> 7f40f3fd`e0000fff
> base	->	7ffde000
> windbg
> 0040f300`00000fff
> base	->	00000000
> ```
>
> 那么到底哪个是正确的，我们可以汇编读一下
>
> ```c
> #include<stdio.h>
> #include<stdlib.h>
> #include<Windows.h>
> DWORD temp;
> DWORD dEax[2], dEbx[2], dEcx[2], dEdx[2];
> DWORD dEsp[2], dEbp[2], dEsi[2], dEdi[2];
> WORD wCs[2], wDs[2], wSs[2], wEs[2], wFs[2], wGs[2];
> DWORD g_8003f038;
> DWORD g_8003f03c;
> //0x00401040
> void __declspec(naked) idtEntry()
> {
> 
> 	__asm {
> 		mov [dEax+4], eax
> 		mov [dEbx + 4], ebx
> 		mov [dEcx + 4], ecx
> 		mov [dEdx + 4], edx
> 		mov [dEsp + 4], esp
> 		mov [dEbp + 4], ebp
> 		mov [dEsi + 4], esi
> 		mov [dEdi + 4], edi
> 		mov [wCs+2], cs
> 		mov [wDs+2], ds
> 		mov [wSs+2], ss
> 		mov [wEs+2], es
> 		mov [wFs+2], fs
> 		mov [wGs+2], gs
> 		mov eax, ds:[0x8003f038]
> 		mov g_8003f038, eax
> 		mov eax, ds : [0x8003f03c]
> 		mov g_8003f03c, eax
> 		iretd
> 	}
> }
> 
> void go()
> {
> 	__asm {
> 		mov [dEax], eax
> 		mov [dEbx], ebx
> 		mov [dEcx], ecx
> 		mov [dEdx], edx
> 		mov [dEsp], esp
> 		mov [dEbp], ebp
> 		mov [dEsi], esi
> 		mov [dEdi], edi
> 		mov [wCs], cs
> 		mov [wDs], ds
> 		mov [wSs], ss
> 		mov [wEs], es
> 		mov [wFs], fs
> 		mov [wGs], gs
> 	
> 	}
> 	__asm int 0x20;
> 
> }
> 
> void main()
> {
> 	if ((DWORD32)idtEntry != 0x00401040) {
> 		printf("wrong, the addr is %p\n", idtEntry);
> 		exit(-1);
> 	}
> 	go();
> 	printf("[+]Pre eax:0x%p, ebx:0x%p, ecx:0x%p, edx:0x%p, esi:0x%p, edi:0x%p, esp:0x%p, ebp:0x%p\n", dEax[0], dEbx[0], dEcx[0], dEdx[0], dEsi[0], dEdi[0], dEsp[0], dEbp[0]);
> 	printf("[+]Pre cs:0x%p, ds:0x%p, ss:0x%p, es:0x%p, fs:0x%p, gs:0x%p\n", wCs[0], wDs[0], wSs[0], wEs[0], wFs[0], wGs[0]);
> 	printf("[+]Now eax:0x%p, ebx:0x%p, ecx:0x%p, edx:0x%p, esi:0x%p, edi:0x%p, esp:0x%p, ebp:0x%p\n", dEax[1], dEbx[1], dEcx[1], dEdx[1], dEsi[1], dEdi[1], dEsp[1], dEbp[1]);
> 	printf("[+]Now cs:0x%p, ds:0x%p, ss:0x%p, es:0x%p, fs:0x%p, gs:0x%p\n", wCs[1], wDs[1], wSs[1], wEs[1], wFs[1], wGs[1]);
> 	//printf("0x%p\n", temp);
> 	printf("DWORD g_8003f038:%p\n", g_8003f038);
> 	printf("DWORD g_8003f03c:%p\n", g_8003f03c);
> 	system("pause");
> }
> ```
>
> ![image-20220120162238586](kernel%20lab/image-20220120162238586.png)
>
> 可以看到新版的是一样的。
>
> 好像是软件调试里面讲到的一个内容，调试器会对内核产生影响。

# 0x04 lab 04

## 1. goal

1. 构建驱动环境，调用驱动函数

## 2. step

构建API调用环境

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>

//0x00401040
void __declspec(naked) idtEntry()
{

	__asm {
		push 0x30
		pop fs
		sti
            
            
		push 0x3b
		pop fs
		iretd
	}
}

void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	system("pause");
}
```

接下来我们调用一个函数

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
typedef DWORD (__stdcall *EX_ALLOCATE ) (DWORD PoolType, DWORD NumberOfBytes);
EX_ALLOCATE ExAllocatePool = (EX_ALLOCATE)0x8053474A;
DWORD g_pool;
//0x00401040
void __declspec(naked) idtEntry()
{

	__asm {
		push 0x30
		pop fs
		sti
	}
	g_pool = ExAllocatePool(0, 4096);
	__asm{
		push 0x3b
		pop fs
		iretd
	}
}

void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

![image-20220121135824923](kernel%20lab/image-20220121135824923.png)

接下来可以看一下如何在这里面弄驱动

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
typedef DWORD (__stdcall *EX_ALLOCATE ) (DWORD PoolType, DWORD NumberOfBytes);
EX_ALLOCATE ExAllocatePool = (EX_ALLOCATE)0x8053474A;
typedef DWORD (*DBG_PRINT)(PCSTR Format, ...);
DBG_PRINT DbgPrint = (DBG_PRINT)0x80528FB2;
DWORD g_pool;
char str[] = "Hello m1keya!";
//0x00401040
void __declspec(naked) idtEntry()
{

	__asm {
		push 0x30
		pop fs
		sti
		int 0x3
	}
	g_pool = ExAllocatePool(0, 4096);
	DbgPrint(str);
	__asm{
        cli
		push 0x3b
		pop fs
		iretd
	}
}

void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

```
kd> g
Hello m1keya!
```

>```
>cli
>```
>
>关中断，因为在修改fs时如果发生时钟中断进行线程调度就会出现问题导致蓝屏

# 0x05 lab 05



## 1. goal



## 2. step

这节要做的就是inline hook，具体该怎么做呢。

我们要明白三点

- hook谁
- hook到哪里
- 用哪种方式hook

第一，我们hook这个函数

![image-20220121221947213](kernel%20lab/image-20220121221947213.png)

windbg看一下我们的基地址是否正确

```
kd> u 0x8053E750
8053e750 b923000000      mov     ecx,23h
8053e755 6a30            push    30h
8053e757 0fa1            pop     fs
8053e759 8ed9            mov     ds,cx
8053e75b 8ec1            mov     es,cx
```

可以看到正确，那么下一步我们要hook到哪里

需要满足一个条件就是可写可执行

```
!pte
```

这里选了gdtr里面的内容

```
8003f120  00000000`8003f128 00000000`8003f130
8003f130  00000000`8003f138 00000000`8003f140
8003f140  00000000`8003f148 00000000`8003f150
8003f150  00000000`8003f158 00000000`8003f160
8003f160  00000000`8003f168 00000000`8003f170
8003f170  00000000`8003f178 00000000`8003f180
8003f180  00000000`8003f188 00000000`8003f190
8003f190  00000000`8003f198 00000000`8003f1a0
8003f1a0  00000000`8003f1a8 00000000`8003f1b0
8003f1b0  00000000`8003f1b8 00000000`8003f1c0
8003f1c0  00000000`8003f1c8 00000000`8003f1d0
8003f1d0  00000000`8003f1d8 00000000`8003f1e0
8003f1e0  00000000`8003f1e8 00000000`8003f1f0
8003f1f0  00000000`8003f1f8 00000000`8003f200
```

我们可以修改8003f120这的内容，改成我们想要的内容

但有两点我们需要记住

- hook的前不能发生改变，也就是我们占指令的地方需要我们在8003f120同样执行
- hook后要跳转回我们原有的位置

那么我们该如何写汇编

```
//prev
mov ecx, 0x23
//content
...
//after
//8053e755-8003f125-5 = 4FF62B
jmp 0x4FF62B
```

这里直接硬编码复制过去（**lab5.1.exe**）

```
b923000000  
e92bf64f00
```

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


char code[64] = { 0xb9, 0x23, 0x00, 0x00, 0x00, 0xe9, 0x2b, 0xf6, 0x4f, 0x00 };
char* p = (char* )0x8003f120;
int i;
//0x00401040
void __declspec(naked) idtEntry()
{
	for (i = 0; i < 10; i++) {
		*p = code[i];
		p++;
	}
	__asm {
		iretd
	}
}

void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

看一下结果

```
kd> u 0x8003f120
8003f120 b923000000      mov     ecx,23h
8003f125 e92bf64f00      jmp     nt!KiFastCallEntry+0x5 (8053e755)
```

但这样出现一个问题，我们发现计算相对偏移较为复杂，所以我们另外用一个函数进行汇编

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


//char code[64] = { 0xb9, 0x23, 0x00, 0x00, 0x00, 0xe9, 0x2b, 0xf6, 0x4f, 0x00 };
char* p = (char* )0x8003f120;
int i;
void JmpEntry();
//0x00401040
void __declspec(naked) idtEntry()
{
	for (i = 0; i < 64; i++) {
		*p = ((char *)JmpEntry)[i];
		p++;
	}
	__asm {
		iretd
	}
}

void __declspec(naked) JmpEntry()
{
	__asm {
		mov     ecx, 0x23
		push    0x30
		pop     fs
		mov     ds, cx
        mov     es, cx

		mov ecx, 0x8053E75D
		jmp ecx

	}
}
void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

看效果

```
kd> u 8003f120
8003f120 b923000000      mov     ecx,23h
8003f125 6a30            push    30h
8003f127 0fa1            pop     fs
8003f129 668ed9          mov     ds,cx
8003f12c 668ec1          mov     es,cx
8003f12f b95de75380      mov     ecx,offset nt!KiFastCallEntry+0xd (8053e75d)
8003f134 ffe1            jmp     ecx
```

最后一步就是修改需要hook函数那里了

首先查看权限

```
kd> !pte 8053e75d
                    VA 8053e75d
PDE at C0602010            PTE at C04029F0
contains 000000000074B163  contains 000000000053E121
pfn 74b       -G-DA--KWEV  pfn 53e       -G--A--KREV
```

>可读不可写，需要关闭cr0寄存器

**（lab5.2.exe）**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


//char code[64] = { 0xb9, 0x23, 0x00, 0x00, 0x00, 0xe9, 0x2b, 0xf6, 0x4f, 0x00 };
char* p = (char* )0x8003f120;
int i;
void JmpEntry();
//0x00401040
void __declspec(naked) idtEntry()
{
	for (i = 0; i < 64; i++) {
		*p = ((char *)JmpEntry)[i];
		p++;
	}
	__asm {
		mov eax, cr0
		and eax, not 10000h
		mov cr0, eax

		mov al, 0xe9
		mov ds:[0x8053E750], al
		mov eax, 0xFFB009CB
		mov ds:[0x8053E751] , eax

		mov eax, cr0
		or eax, 10000h
		mov cr0, eax
		iretd
	}
}

void __declspec(naked) JmpEntry()
{
	__asm {
		mov     ecx, 0x23
		push    0x30
		pop     fs
		mov     ds, cx
        mov     es, cx

		mov ecx, 0x8053E75D
		jmp ecx

	}
}
void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

看效果已经可以了

```
kd> u 8053e750
8053e750 e9cb09b0ff      jmp     8003f120
8053e755 6a30            push    30h
8053e757 0fa1            pop     fs
8053e759 8ed9            mov     ds,cx
8053e75b 8ec1            mov     es,cx
8053e75d 8b0d40f0dfff    mov     ecx,dword ptr ds:[0FFDFF040h]
8053e763 8b6104          mov     esp,dword ptr [ecx+4]
8053e766 6a23            push    23h
kd> u 8003f120
8003f120 b923000000      mov     ecx,23h
8003f125 6a30            push    30h
8003f127 0fa1            pop     fs
8003f129 668ed9          mov     ds,cx
8003f12c 668ec1          mov     es,cx
8003f12f b95de75380      mov     ecx,offset nt!KiFastCallEntry+0xd (8053e75d)
8003f134 ffe1            jmp     ecx
```

![image-20220121231859282](kernel%20lab/image-20220121231859282.png)

最后利用hook实现统计系统调用的次数(**lab5.3.exe**)

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


//char code[64] = { 0xb9, 0x23, 0x00, 0x00, 0x00, 0xe9, 0x2b, 0xf6, 0x4f, 0x00 };
char* p = (char* )0x8003f120;
int i;
DWORD g_temp;
void JmpEntry();
//0x00401040
void __declspec(naked) idtEntry()
{

	/**for (i = 0; i < 64; i++) {
		*p = ((char *)JmpEntry)[i];
		p++;
	}


	__asm {
		mov eax, cr0
		and eax, not 10000h
		mov cr0, eax

		mov al, 0xe9
		mov ds:[0x8053E750], al
		mov eax, 0xFFB009CB
		mov ds:[0x8053E751] , eax

		mov eax, cr0
		or eax, 10000h
		mov cr0, eax
		iretd

	}*/
	
	__asm {
		test eax, eax
		jnz L
		mov ds : [0x8003f330], eax
		mov g_temp, eax
		jmp End
	L:
		mov eax, ds : [0x8003f330]
		mov g_temp, eax
	End:
		iretd
			
	}
}
//0x8003f330
void __declspec(naked) JmpEntry()
{
	__asm {

		pushad
		pushfd

		mov eax, ds:[0x8003f330]
		inc eax
		mov ds:[0x8003f330], eax
		

		popfd
		popad

		mov     ecx, 0x23
		push    0x30
		pop     fs
		mov     ds, cx
        mov     es, cx

		mov ecx, 0x8053E75D
		jmp ecx

	}
}
void reset() {

	__asm xor eax, eax;
	__asm int 0x20;
}
void go()
{
	__asm mov eax, 1;
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}

	reset();
	while (1) {
		go();
		Sleep(1000);
		printf("[+]The syscall's count in one second is:%d\n", g_temp);
	}


	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

![image-20220122001418158](kernel%20lab/image-20220122001418158.png)



用另外一种hook

```
push addr
ret 
```

- hook哪个函数 Debug
- hook到哪里，还是我们gdt的地方

![image-20220122160959047](kernel%20lab/image-20220122160959047.png)

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


//char code[64] = { 0xb9, 0x23, 0x00, 0x00, 0x00, 0xe9, 0x2b, 0xf6, 0x4f, 0x00 };
char* p = (char* )0x8003f120;
int i;
DWORD g_temp;
void JmpEntry();
//0x00401040
void __declspec(naked) idtEntry()
{

	for (i = 0; i < 64; i++) {
		*p = ((char *)JmpEntry)[i];
		p++;
	}

	__asm {
	iretd
	}
	/**__asm {
		mov eax, cr0
		and eax, not 10000h
		mov cr0, eax

		mov al, 0xe9
		mov ds:[0x8053E750], al
		mov eax, 0xFFB009CB
		mov ds:[0x8053E751] , eax

		mov eax, cr0
		or eax, 10000h
		mov cr0, eax
		iretd

	}
	
	__asm {
		test eax, eax
		jnz L
		mov ds : [0x8003f330], eax
		mov g_temp, eax
		jmp End
	L:
		mov eax, ds : [0x8003f330]
		mov g_temp, eax
	End:
		iretd
			
	}*/
}
//0x8003f330
void __declspec(naked) JmpEntry()
{
	__asm {

		push eax
		mov eax, ss:[esp]
		mov ds:[0x8003f330], eax
		pop eax

		push    0
		mov    word ptr[esp + 2], 0
		push 0x8053F53D
		ret


	}
}
void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}

	go();


	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

windbg

```
kd> u 0x8003f120
8003f120 50              push    eax
8003f121 368b0424        mov     eax,dword ptr ss:[esp]
8003f125 3ea330f30380    mov     dword ptr ds:[8003F330h],eax
8003f12b 58              pop     eax
8003f12c 6a00            push    0
8003f12e 66c74424020000  mov     word ptr [esp+2],0
8003f135 683df55380      push    offset nt!KiTrap01+0x9 (8053f53d)
8003f13a c3              ret
```

之后写一下那个被hook函数就可以

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


//char code[64] = { 0xb9, 0x23, 0x00, 0x00, 0x00, 0xe9, 0x2b, 0xf6, 0x4f, 0x00 };
char* p = (char* )0x8003f120;
int i;
DWORD g_temp;
void JmpEntry();
//0x00401040
void __declspec(naked) idtEntry()
{

	for (i = 0; i < 64; i++) {
		*p = ((char *)JmpEntry)[i];
		p++;
	}

	//68 20 f1 03 80   push 0x8003f120
	//c3				ret
	__asm {
		mov eax, cr0
		and eax, not 10000h
		mov cr0, eax
		mov eax, 0x03f12068
		mov ds:[0x8053F534], eax
		mov ax, 0xc380
		mov ds:[0x8053F534+4] , ax

		mov eax, cr0
		or eax, 10000h
		mov cr0, eax
		iretd

	}
	

}
//0x8003f330
void __declspec(naked) JmpEntry()
{
	__asm {

		push eax
		mov eax, ss:[esp+4]
		mov ds:[0x8003f330], eax
		pop eax

		push    0
		mov    word ptr[esp + 2], 0
		push 0x8053F53D
		ret

	}
}
void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}

	go();


	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

可以看到已经hook好了

```
kd> u 0x8053F534
8053f534 6820f10380      push    8003F120h
8053f539 c3              ret
8053f53a 0200            add     al,byte ptr [eax]
8053f53c 005553          add     byte ptr [ebp+53h],dl
8053f53f 56              push    esi
8053f540 57              push    edi
8053f541 0fa0            push    fs
kd> u 8003F120
8003f120 50              push    eax
8003f121 368b0424        mov     eax,dword ptr ss:[esp]
8003f125 3ea330f30380    mov     dword ptr ds:[8003F330h],eax
8003f12b 58              pop     eax
8003f12c 6a00            push    0
8003f12e 66c74424020000  mov     word ptr [esp+2],0
8003f135 683df55380      push    offset nt!KiTrap01+0x9 (8053f53d)
8003f13a c3              ret
```

![image-20220122171411626](kernel%20lab/image-20220122171411626.png)

hook成功,回三环打印一下就可以

**lab5.4.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


//char code[64] = { 0xb9, 0x23, 0x00, 0x00, 0x00, 0xe9, 0x2b, 0xf6, 0x4f, 0x00 };
char* p = (char* )0x8003f120;
int i;
int flag;
DWORD g_temp,g_enabled;
void JmpEntry();
//0x00401040
void __declspec(naked) idtEntry()
{

	for (i = 0; i < 64; i++) {
		*p = ((char *)JmpEntry)[i];
		p++;
	}

	//68 20 f1 03 80   push 0x8003f120
	//c3				ret
	__asm {
		mov eax, cr0
		and eax, not 10000h
		mov cr0, eax
		mov eax, 0x03f12068
		mov ds : [0x8053F534] , eax
		mov ax, 0xc380
		mov ds : [0x8053F534 + 4] , ax

		mov eax, cr0
		or eax, 10000h
		mov cr0, eax
	}
	__asm{
		mov eax, ds: [0x8003f330]
		mov g_temp, eax
		mov eax, ds : [0x8003f330+4]
		mov g_enabled, eax
		xor eax, eax
		mov ds : [0x8003f330+4], eax
		iretd

	}
	

}
//0x8003f330
void __declspec(naked) JmpEntry()
{
	__asm {

		push eax
		mov eax, ss:[esp+4]
		mov ds:[0x8003f330], eax
		mov eax, 1 
		mov ds :[0x8003f330+4] , eax
		pop eax

		push    0
		mov    word ptr[esp + 2], 0
		push 0x8053F53D
		ret

	}
}
void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
/**	while (1) {
		if (g_enabled) {
			go();
			printf("[+]The instruction is in:0x%p", g_temp);
		}
	
	}*/


	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

**lab5.5.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


//char code[64] = { 0xb9, 0x23, 0x00, 0x00, 0x00, 0xe9, 0x2b, 0xf6, 0x4f, 0x00 };
char* p = (char* )0x8003f120;
int i;
int flag;
DWORD g_temp,g_enabled;
void JmpEntry();
//0x00401040
void __declspec(naked) idtEntry()
{

	/**for (i = 0; i < 64; i++) {
		*p = ((char *)JmpEntry)[i];
		p++;
	}

	//68 20 f1 03 80   push 0x8003f120
	//c3				ret
	__asm {
		mov eax, cr0
		and eax, not 10000h
		mov cr0, eax
		mov eax, 0x03f12068
		mov ds : [0x8053F534] , eax
		mov ax, 0xc380
		mov ds : [0x8053F534 + 4] , ax

		mov eax, cr0
		or eax, 10000h
		mov cr0, eax
	}*/
	__asm{
		mov eax, ds: [0x8003f330]
		mov g_temp, eax
		mov eax, ds : [0x8003f330+4]
		mov g_enabled, eax
		xor eax, eax
		mov ds : [0x8003f330+4], eax
		iretd

	}
	

}
//0x8003f330
void __declspec(naked) JmpEntry()
{
	__asm {

		push eax
		mov eax, ss:[esp+4]
		mov ds:[0x8003f330], eax
		mov eax, 1 
		mov ds :[0x8003f330+4] , eax
		pop eax

		push    0
		mov    word ptr[esp + 2], 0
		push 0x8053F53D
		ret

	}
}
void go()
{
	__asm int 0x20;

}

void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	while (1) {
		go();
		if (g_enabled) {
			printf("[+]The instruction is at: 0x%p\n", g_temp);
		}
	
	}


	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

![image-20220122173617215](kernel%20lab/image-20220122173617215.png)

完成

![image-20220122173758244](kernel%20lab/image-20220122173758244.png)













# some question

## **vs2019生成程序无法在xp中运行**

![image-20220119163356187](kernel%20lab/image-20220119163356187.png)

属性修改内核版本即可

![image-20220119163510452](kernel%20lab/image-20220119163510452.png)

## 一些指令

**汇编**

```
//开中断
sti
//关中断
cli
//加载gdt寄存器
LGDT
//读取tr寄存器
str ax
//压入eflags寄存器
pushfd
//压入通用寄存器
pushad
//test
test eax, eax
jnz L
eax = 0 不跳转
eax = 1 跳转
```

**硬编码**

```
e8 call
e9 jmp
6a push 8bit
68 push 32bit
c3 ret
```

