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

1. 熟悉inline hook
2. 掌握jmp和pop的hook

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

# 0x06 lab 06

## 1. goal

1. 通过结合之前的自己实现系统调用
2. 从三环直接调用不通过零环

## 2. step

先画一个图，说明一下我们需要干的事

我们得先明确应该有3个东西是需要我们构造的

1. **系统调用入口，用来处理参数，通过服务表进入真正的函数**
2. **我们自己构造的函数**
3. **系统服务表，每个表项对应一个系统调用**

![image-20220122210124849](kernel%20lab/image-20220122210124849.png)

先写第一个程序，让我们构造好内核空间

同时伪造好服务表，系统调用入口，我们伪造函数。

**kernelcode.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>

DWORD Target[3] = { 0x8003f180, 0x8003f220, 0x8003f250 };
int i;
int flag;
char* p;
void ReadMem();
void AllocMem();
void SystemCallEntry();
DWORD* ServiceTable = (DWORD*)0x8003f2a0;
//0x00401040
void __declspec(naked) idtEntry()
{
	p = (char*)Target[0];
	for (i = 0; i < 0x80; i++) {
		*p = ((char *)SystemCallEntry)[i];
		p++;
	}
	p = (char*)Target[1];
	for (i = 0; i < 0x30; i++) {
		*p = ((char*)ReadMem)[i];
		p++;
	}
	p = (char*)Target[2];
	for (i = 0; i < 0x30; i++) {
		*p = ((char*)AllocMem)[i];
		p++;
	}
	ServiceTable[0] = Target[1];
	ServiceTable[1] = Target[2];
	//eq 8003f500 8003ee00`0008f180
	__asm {
		mov eax, 0x0008f180
		mov ds:[0x8003f508], eax
		mov eax, 0x8003ee00
		mov ds : [0x8003f508+4] , eax
		iretd
	}


}
//0x8003f330
/*
eip
cs
eflags
esp
ss
*/
void __declspec(naked) SystemCallEntry()
{
	__asm {
		push 0x30
		pop fs
		sti

		mov ebx, ss:[esp + 0xc]
		mov ecx, ds:[ebx + 0x4]
		push 0x8003f2a0
		pop ebx
		mov ebx, ds:[ebx + eax*4]
		call ebx

		cli
		push 0x3b
		pop fs
		iretd
	}
}

void __declspec(naked) ReadMem()
{
	__asm {
		mov eax, ds:[ecx]
		ret
	}
}
//0x8053474A
//ExAllocatePool(NonPaged, 0x400)
void __declspec(naked) AllocMem()
{
	__asm {
		push ecx
		push 0
		mov eax, 0x8053474A
		call eax
		ret
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
	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

接下来用我们的三环调用程序

**ntdll.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


DWORD ReadMem(DWORD addr);
DWORD AllocMem(DWORD size);
DWORD __declspec(naked) ReadMem(DWORD addr) {
	__asm {
		mov eax, 0
		int 0x21
		ret
	}
}
DWORD __declspec(naked) AllocMem(DWORD size) {
	__asm {
		mov eax,1 
		int 0x21
		ret
	}
}
void main()
{
	printf("[+] The Addr's content is: 0x%p\n", ReadMem(0x8003f500));
	printf("[+] The Heap allocated is at: 0x%p\n", AllocMem(1024));
	system("pause");
}
```

![image-20220122220327598](kernel%20lab/image-20220122220327598.png)

好家伙自己写了快半个小时，下一个。

# 0x07 lab 07

## 1. goal

1. 熟悉PAE分页和非PAE分页
2. 对虚拟地址和物理地址熟练转换

## 2. step

### 非PAE分页

段保护 主要体现在 段选择子上；但是数据段、代码段、栈段等采用的都是4KB平坦模式，段的特征并没有那样展现。所以具体的**保护机制 采用的是页保护。**

非PAE可以进行数据执行

查看一下物理内存

其实开机启动的时候是可以看见的

```
kd> g
Enter DriverEntry(82E1E910,800939E8)
Required extension size: max: 10748124 Min: 63344
Physical Address: 1000     Length: 9e000 
Physical Address: 100000     Length: eff000 
Physical Address: 1000000     Length: 1eee0000 
Physical Address: 1ff00000     Length: 100000 
Total Physical Memory: 536334336 (1ff7d000)
Modified-> Physical Memory Pages: 130941 (1ff7d)
kd> !dd 1fffff70 
#1fffff70 fffffffe 00271eb8 6cb3e800 10c20000
#1fffff80 cccccc00 086acccc 4e9a5868 6c5ae8f8
#1fffff90 758b0000 39c93308 0575104e 74144e39
#1fffffa0 0c4d3949 468b4474 10453b04 4d893c72
#1fffffb0 147d80fc 6a0c7501 76ff5001 1c15ff10
#1fffffc0 fff84e93 75ff1075 1076ff0c ffff0de8
#1fffffd0 fc45c7ff fffffffe 13ebc033 c340c033
#1fffffe0 c7e8658b fffefc45 1eb8ffff e8000027
```

我们现在要看物理虚拟地址转换，其中最重要的就是cr3寄存器。

我们先来读一下

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


DWORD g_cr3;
//0x00401040
void __declspec(naked) idtEntry()
{
	__asm {
		mov eax, cr3
		mov g_cr3, eax
		iretd
	}
	
	

}
//0x8003f330

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
	printf("[+] The cr3 is: 0x%p\n", g_cr3);

	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

```
Failed to get VadRoot
PROCESS 82c98020  SessionId: 0  Cid: 03d0    Peb: 7ffda000  ParentCid: 05f4
    DirBase: 1735e000  ObjectTable: e2705320  HandleCount:  15.
    Image: kernel lab1.exe
```

![image-20220122225338007](kernel%20lab/image-20220122225338007.png)

启动dirbase就是cr3寄存器中内容，就是物理地址。

那么如何从虚拟地址推出物理地址呢？

```
kd> .formats 0x403018
Evaluate expression:
  Hex:     00403018
  Decimal: 4206616
  Octal:   00020030030
  Binary:  00000000 01000000 00110000 00011000
  Chars:   .@0.
  Time:    Thu Feb 19 00:30:16 1970
  Float:   low 5.89472e-039 high 0
  Double:  2.07834e-317

101012分页
00000000 01	<- 1 	pdi (Page Directory Index)
000000 0011	<- 3	pti
0000 00011000	<-18	offset

kd> !dd 1735e000 
#1735e000 172b5067 17342067 00000000 00000000

067是属性
17342000是基址

kd> !dd 17342000+3*4
#1734200c 173b4067 172b4025 00000000 00000000
#1734201c 00000000 00000000 00000000 00000000
#1734202c 00000000 00000000 00000000 00000000
#1734203c 00000000 1018f025 1722d025 fffff440
#1734204c fffff440 1292e025 1290f025 129a0025
#1734205c 17321025 12b12025 173b3025 101a3025
#1734206c 17304025 17305025 17346025 17307025
#1734207c 172f8025 17339025 1645a025 fffff440

067是属性
173b4000是基址

kd> !dd 173b4000+18
#173b4018 12345678 00000000 00000000 00000000
#173b4028 00000000 00000000 00000000 00000000
#173b4038 00000000 00000000 00000000 00000000
#173b4048 00000000 00000000 00000000 00000000
#173b4058 00000000 00000000 00000000 00000000
#173b4068 00000000 00000000 00000000 00000000
#173b4078 00000000 00000000 00000000 00000000
#173b4088 00000000 00000000 00000000 00000000
```

如下图

![image-20220126162910960](kernel%20lab/image-20220126162910960.png)

windbg有现成命令

```
kd> !vtop 1735e 403018
X86VtoP: Virt 0000000000403018, pagedir 000000001735e000
X86VtoP: PDE 000000001735e004 - 17342067
X86VtoP: PTE 000000001734200c - 173b4067
X86VtoP: Mapped phys 00000000173b4018
Virtual address 403018 translates to physical address 173b4018.
```

如果想直接读虚拟内存，可以挂载进程

```
kd> .process 82c98020
ReadVirtual: 82c98038 not properly sign extended
Implicit process is now 82c98020
WARNING: .cache forcedecodeuser is not enabled
kd> dd 0x403018
00403018  12345678 00000000 00000000 00000000
00403028  00000000 00000000 00000000 00000000
00403038  00000000 00000000 00000000 00000000
```

进程空间如图，每一个进程空间有1024个PTE表项，1个PDE表项

![image-20220123000006318](kernel%20lab/image-20220123000006318.png)

那么我们能否通过进程空间找到我们的PTE表项，从而修改页面属性，当然是可以

```
kd> !pte 403018
                 VA 00403018
PDE at C0300004         PTE at C000100C
contains 00000000
contains 0000000000000000
not valid
kd> ?? ((0x00403018 >> 12)<<2) + 0xc0000000
unsigned int 0xc000100c
kd> dq C000100C
ReadVirtual: c000100c not properly sign extended
c000100c  172b4025`173b4067 00000000`00000000
```

我这的环境可能有点问题

具体修改属性可以通过

```
.hh !pte
```

查看

我们也得到了公式

```
virtual address of pte = ((virtuall address >> 12) <<2) + 0xc0000000
```

![image-20220123002317864](kernel%20lab/image-20220123002317864.png)

上面看到我们可以修改属性

----

### PAE分页

PAE分页会开启数据执行保护。

具体实现方法看下面

![image-20220126221831316](kernel%20lab/image-20220126221831316.png)

先看cr3寄存器



```
kd> .formats 403018
Evaluate expression:
  Hex:     00403018
  Decimal: 4206616
  Octal:   00020030030
  Binary:  00000000 01000000 00110000 00011000
  Chars:   .@0.
  Time:    Thu Feb 19 00:30:16 1970
  Float:   low 5.89472e-039 high 0
  Double:  2.07834e-317
```

**这里采用29912分页**

>为什么采用29912分页？
>
>12：物理页的大小为4KB，4096字节。想要索引到每一个字节需要12个二进制位。因为2的12次方等于4096。所以是12
>9：由于PTE增加到36位，原先能保存1024个PTE成员的PTT表，现在只能保存512个PTE了。想要索引到512个成员需要9个二进制位。2的9次方等于512。
>9：PDE跟随PTE由原来的4个字节变成了8个字节。数据项也由1024变成了512。
>2：PDPTE的成员有4个，正好用两个二进制位索引。

```
00				0
000000 010		2
00000 0011		3
0000 00011000		18
//物理地址
kd> !dq 096101a0
# 96101a0 00000000`14011001 00000000`14022001
# 96101b0 00000000`13fc3001 00000000`14090001
# 96101c0 00000000`11111001 00000000`11122001
# 96101d0 00000000`11123001 00000000`111b0001
# 96101e0 00000000`11f63001 00000000`11f44001
# 96101f0 00000000`11f25001 00000000`11f72001
# 9610200 00000000`1202c001 00000000`1201d001
# 9610210 00000000`1202e001 00000000`1207b001
//PDE
kd> !dq 14011000+2*0x8
#14011010 00000000`13f94067 00000000`14585067
#14011020 00000000`00000000 00000000`00000000
#14011030 00000000`00000000 00000000`00000000
#14011040 00000000`00000000 00000000`00000000
#14011050 00000000`00000000 00000000`00000000
#14011060 00000000`00000000 00000000`00000000
#14011070 00000000`00000000 00000000`00000000
#14011080 00000000`00000000 00000000`00000000
//PTE
kd> !dq 13f94000+3*0x8
#13f94018 80000000`146ad067 80000000`14206025
#13f94028 00000000`00000000 00000000`00000000
#13f94038 00000000`00000000 00000000`00000000
#13f94048 00000000`00000000 00000000`00000000
#13f94058 00000000`00000000 00000000`00000000
#13f94068 00000000`00000000 00000000`00000000
#13f94078 00000000`00000000 80000000`0edeb025
#13f94088 00000000`0ee08025 ffffffff`00000440
//addr
kd> !dq 146ad000+0x18
#146ad018 00000000`12345678 00000000`00000000
#146ad028 00000000`00000000 00000000`00000000
#146ad038 00000000`00000000 00000000`00000000
#146ad048 00000000`00000000 00000000`00000000
#146ad058 00000000`00000000 00000000`00000000
#146ad068 00000000`00000000 00000000`00000000
#146ad078 00000000`00000000 00000000`00000000
#146ad088 00000000`00000000 00000000`00000000
```

具体解释可以看图

![image-20220126222704125](kernel%20lab/image-20220126222704125.png)

这里是如何实现PAE，也就是数据执行保护的呢？

通过看下面这个

```
kd> !dq 13f94000+3*0x8
#13f94018 80000000`146ad067 80000000`14206025
#13f94028 00000000`00000000 00000000`00000000
#13f94038 00000000`00000000 00000000`00000000
#13f94048 00000000`00000000 00000000`00000000
#13f94058 00000000`00000000 00000000`00000000
#13f94068 00000000`00000000 00000000`00000000
#13f94078 00000000`00000000 80000000`0edeb025
#13f94088 00000000`0ee08025 ffffffff`00000440
```

**PTE在每一页的最高位都置为8，即可开启数据执行保护**

下面我们来验证一下

```
//切换进程
kd> .process /i 0x829818c0
You need to continue execution (press 'g' <enter>) for the context
to be switched. When the debugger breaks in again, you will be in
the new process context.
kd> r cr3
cr3=00736000
kd> g
Break instruction exception - code 80000003 (first chance)
nt!RtlpBreakWithStatusInstruction:
80528d2c cc              int     3
kd> r cr3
cr3=096101a0
kd> db 0x403018
00403018  78 56 34 12 00 00 00 00-00 00 00 00 00 00 00 00  xV4.............
00403028  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00403038  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00403048  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00403058  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00403068  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00403078  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00403088  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
kd> !db 146ad000+0x18
#146ad018 78 56 34 12 00 00 00 00-00 00 00 00 00 00 00 00 xV4.............
#146ad028 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
#146ad038 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
#146ad048 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
#146ad058 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
#146ad068 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
#146ad078 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
#146ad088 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ...............
```

我们也可以用vtop来看一下

```
kd> !vtop 096101a0 0x403018
X86VtoP: Virt 0000000000403018, pagedir 00000000096101a0
X86VtoP: PAE PDPE 00000000096101a0 - 0000000014011001
X86VtoP: PAE PDE 0000000014011010 - 0000000013f94067
X86VtoP: PAE PTE 0000000013f94018 - 80000000146ad067
X86VtoP: PAE Mapped phys 00000000146ad018
Virtual address 403018 translates to physical address 146ad018.
kd> !dq 00000000146ad018
#146ad018 00000000`12345678 00000000`00000000
#146ad028 00000000`00000000 00000000`00000000
#146ad038 00000000`00000000 00000000`00000000
#146ad048 00000000`00000000 00000000`00000000
```

可以看到就是我们查的地址

下面具体来看进程空间

![image-20220126223547297](kernel%20lab/image-20220126223547297.png)

为什么只有PTE和PDE却没有PDPE？

因为太小了，不需要。

一个PTE管理4k页面，一个PDE管理4K*512页面，一个PDPTE管理1G空间

```
//PTE
kd> ?? 0xc0000000 + ((0x403018 >> 12) << 3)
unsigned int 0xc0002018
kd> dq 0xc0002018
ReadVirtual: c0002018 not properly sign extended
c0002018  80000000`146ad067 80000000`14206025
c0002028  00000000`00000000 00000000`00000000
c0002038  00000000`00000000 00000000`00000000
c0002048  00000000`00000000 00000000`00000000
kd> !dq 13f94018
#13f94018 80000000`146ad067 80000000`14206025
#13f94028 00000000`00000000 00000000`00000000
#13f94038 00000000`00000000 00000000`00000000
#13f94048 00000000`00000000 00000000`00000000
//PDE
kd> ?? 0xc0600000 + ((0x403018 >> 21) << 3)
unsigned int 0xc0600010
kd> dq 0xc0600010
ReadVirtual: c0600010 not properly sign extended
c0600010  00000000`13f94067 00000000`14585067
c0600020  00000000`00000000 00000000`00000000
c0600030  00000000`00000000 00000000`00000000
kd> !dq 14011010
#14011010 00000000`13f94067 00000000`14585067
#14011020 00000000`00000000 00000000`00000000
#14011030 00000000`00000000 00000000`00000000
#14011040 00000000`00000000 00000000`00000000
```

看权限我们是已经改了的，但看不懂为什么上面显示还是不可执行

```
kd> dq C0002018
ReadVirtual: c0002018 not properly sign extended
c0002018  00000000`146ad067 80000000`14206025
c0002028  00000000`00000000 00000000`00000000
c0002038  00000000`00000000 00000000`00000000
c0002048  00000000`00000000 00000000`00000000
c0002058  00000000`00000000 00000000`00000000
c0002068  00000000`00000000 00000000`00000000
c0002078  00000000`00000000 80000000`0edeb025
c0002088  00000000`0ee08025 ffffffff`00000440
kd> !pte 0x403018
                    VA 00403018
PDE at C0600010            PTE at C0002018
contains 0000000013F94067  contains 80000000146AD067	 ???明明上面内存都修改了
pfn 13f94     ---DA--UWEV  pfn 146ad     ---DA--UW-V
kd> !pte 0x403018
                    VA 00403018
PDE at C0600010            PTE at C0002018
contains 0000000013F94067  contains 00000000146AD067
pfn 13f94     ---DA--UWEV  pfn 146ad     ---DA--UWEV
```

后来发现是windbg是会读取物理内存的PTE内容从而确定是否可以执行。

记一下页表转换公式

```
#define PTE(x) (DWORD *)(0xc0000000 + ((x >> 12) << 3))
#define PDE(x) (DWORD *)(0xc0600000 + ((x >> 21) << 3))
```



# 0x08 lab 08

## 1. goal

1. 修改页表进行零地址读写


## 2. step

修改零地址的页表和我们变量地址的页表一样

**（lab8.1.exe）**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

DWORD g_var = 0x12345678;
DWORD g_out;
void __declspec(naked) idtEntry()
{

	PTE(0)[0] = PTE(0x403018)[0];
	PTE(0)[1] = PTE(0x403018)[1];
	__asm {
		mov eax,cr3
		mov cr3,eax
	}
	g_out = *((DWORD*)0x18);
	*(DWORD*)0x18 = 0xBBBBBBBB;
	__asm {
		iretd
	}
	
}
//0x8003f330

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
	g_var = 0x12345678;
	go();
	
	printf("[+] The var is:%p\n", g_var);
	printf("[+] The out is:%p\n", g_out);
	printf("[+] The var is:%p\n", *(DWORD*)(0x403018));
	system("pause");
}
```

>注意：release版会对变量有一些优化，记得去看反汇编
>
>

![image-20220127011415543](kernel%20lab/image-20220127011415543.png)

原本是12345678，映射完之后修改0地址的内容。

# 0x09 lab 09

## 1. goal

1. 两种跨进程访问修改内存


## 2. step

我们要实现一个跨进程内存读写。

具体操作如下

1. 打开Notepad，找到文字缓冲区，通过修改cr3寄存器，使我们当前的进程转换到notepad
2. 对现地址进行修改，其实修改的就是noteoad内存

我们要明确，写到哪和如何写（写到哪，怎么写）？

1. 找Notepad文字缓冲区
2. 写到公共地址空间

>为什么不能直接在当前进程执行？
>
>当修改cr3之后，进程空间已经发生切换，如果我们进行原有程序寻址，其实进行的是notepad寻找，导致寻址空间发生错误。

![image-20220127083442166](kernel%20lab/image-20220127083442166.png)

文字缓冲区是AA6D0

```
kd> !process 0 0 notepad.exe
Failed to get VadRoot
PROCESS 82cd3338  SessionId: 0  Cid: 0114    Peb: 7ffdf000  ParentCid: 0604
    DirBase: 096103a0  ObjectTable: e18a03d8  HandleCount:  46.
    Image: notepad.exe
```

cr3是096103a0

发现直接在该进程地址读写会出现错误

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

void __declspec(naked) idtEntry()
{

	__asm {
		mov eax, cr3
		push eax

		mov eax, 0x096103a0
		mov cr3, eax
		push 0x1234
		pop ecx
		mov ds:[0xAA6D0], ecx
		
		pop eax
		mov cr3, eax

		iretd
	}

}
//0x8003f330

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

我们可以像构造系统调用表一样，写在gdtr的位置，然后在本地直接调用中断即可。

代码（**lab9.1.exe**）

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

void __declspec(naked) idtEntry()
{

	__asm {
		mov eax, cr3
		push eax

		mov eax, 0x096103a0
		mov cr3, eax
		push 0x1234
		pop ecx
		mov ds:[0xAA6D0], ecx
		
		pop eax
		mov cr3, eax

		iretd
	}

}
//0x8003f330

void go()
{
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go2();
	
	system("pause");
}
```

**kernelcode2.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>

DWORD Target[3] = { 0x8003f180, 0x8003f220, 0x8003f250 };
int i;
int flag;
char* p;
void ReadMem();
void AllocMem();
void SystemCallEntry();
DWORD* ServiceTable = (DWORD*)0x8003f2a0;
//0x00401040
void __declspec(naked) idtEntry()
{
	p = (char*)Target[0];
	for (i = 0; i < 0x80; i++) {
		*p = ((char *)SystemCallEntry)[i];
		p++;
	}
	p = (char*)Target[1];
	for (i = 0; i < 0x30; i++) {
		*p = ((char*)ReadMem)[i];
		p++;
	}
	p = (char*)Target[2];
	for (i = 0; i < 0x30; i++) {
		*p = ((char*)AllocMem)[i];
		p++;
	}
	ServiceTable[0] = Target[1];
	ServiceTable[1] = Target[2];
	//eq 8003f500 8003ee00`0008f180
	__asm {
		mov eax, 0x0008f180
		mov ds:[0x8003f508], eax
		mov eax, 0x8003ee00
		mov ds : [0x8003f508+4] , eax
		iretd
	}


}
//0x8003f330
/*
eip
cs
eflags
esp
ss
*/
void __declspec(naked) SystemCallEntry()
{
	__asm {
		//push 0x30
		//pop fs
		//sti

		//mov ebx, ss:[esp + 0xc]
		//mov ecx, ds:[ebx + 0x4]
		//push 0x8003f2a0
		//pop ebx
		//mov ebx, ds:[ebx + eax*4]
		//call ebx

		//cli
		//push 0x3b
		//pop fs
		//iretd
		mov eax, cr3
		push eax

		mov eax, 0x096103a0
		mov cr3, eax
		push 0x1234
		pop ecx
		mov ds : [0xAA6D0] , ecx

		pop eax
		mov cr3, eax

		iretd
	}
}

void __declspec(naked) ReadMem()
{
	__asm {
		mov eax, ds:[ecx]
		ret
	}
}
//0x8053474A
//ExAllocatePool(NonPaged, 0x400)
void __declspec(naked) AllocMem()
{
	__asm {
		push ecx
		push 0
		mov eax, 0x8053474A
		call eax
		ret
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
	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```



可以发现地址内容已经修改

![image-20220127084806077](kernel%20lab/image-20220127084806077.png)

----

平行进程

通过一个进程修改另一个进程，上面我们用到的是写固定内存的数据，这段代码则是跨进程取指令。

具体原理

**每个进程取指令执行都会通过cr3寄存器进行读取基址，如果我们此时修改cr3寄存器为其他进程的基址，然后在指令执行到相应位置填写我们伪造的指令，即可执行我们想要的指令。**

只需要注意构造相应的指令地址即可

**lab9.2.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))
DWORD g_cr3, g_num;
void __declspec(naked) idtEntry()
{

	__asm {
		mov eax, cr3
		mov g_cr3, eax
		iretd
		
		mov eax, 0x12345678
		
		//0040104E
         //这是下一个进程将要执行的位置，执行完之后将会切换cr3寄存器
		nop
		nop
		nop
		mov eax, 1
		mov g_num, eax

		mov ecx, 0xaaaaaaaa
		mov eax, ds : [0x8003f3f0]
		mov cr3, eax



	}

}
//0x8003f330

void go()
{
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	while (1) {
		printf("[+] The cr3 is %p\n", g_cr3);
		printf("[+] The var is %p\n", g_num);
		Sleep(1000);
	}
	system("pause");
}
```

**lab9.3.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))
DWORD g_num;
void __declspec(naked) idtEntry()
{

	__asm {
		mov eax, cr3
		mov ds:[0x8003f3f0], eax

		mov eax, 0x9610400
		mov cr3, eax
		//这里由于已经切换cr3，那么下面指令不会执行，只有下一个注释下才会执行
		nop
		nop
		nop
		mov eax, 0xBBBBBBBB
		mov eax, 0xBBBBBBBB
		mov eax, 0xBBBBBBBB
		mov eax, 0xBBBBBBBB
		nop
		//00401066
		mov g_num, ecx


		iretd
	}

}
//0x8003f330

void go()
{
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	go();
	printf("[+] The var is :%p\n", g_num);
	system("pause");
}
```

访问并且修改成功。

![image-20220127140604196](kernel%20lab/image-20220127140604196.png)

# 0x10 lab10

## 1. goal

1. 理解延迟内存分配



## 2. step

讲一下windows内存分配原理，

windows内存和我们知道的都是延迟内存分配，只有访问时才会真正分配，具体就是页面异常然后在进行页面映射。

我们可以看一个实验

我们分配一个变量是页对齐的，然后在0环里访问它

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>

DWORD out = 0;
#pragma section("data_seg", read, write)
__declspec(allocate("data_seg")) DWORD var = 1;

DWORD g_num;
void __declspec(naked) idtEntry()
{

	__asm {
		mov eax, var
		mov out, eax

		iretd
	}

}
//0x8003f330

void go()
{
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	system("pause");
	go();
	printf("[+] The var is :%p\n", out);
	system("pause");
}
```

然后断下来看pte可以看到页面异常

```
kd> dq 0x404000
00404000  ????????`???????? ????????`????????
00404010  ????????`???????? ????????`????????
00404020  ????????`???????? ????????`????????
00404030  ????????`???????? ????????`????????
00404040  ????????`???????? ????????`????????
00404050  ????????`???????? ????????`????????
00404060  ????????`???????? ????????`????????
00404070  ????????`???????? ????????`????????
kd> !pte 0x404000
                    VA 00404000
PDE at C0600010            PTE at C0002020
contains 0000000017997067  contains 0000000000000000
pfn 17997     ---DA--UWEV  not valid
```

我们继续用CE去读一下

发现此时值已经可以读了，页面也可以访问到了

```
kd> dq 0x404000
00404000  00000000`00000001 00000000`00000000
00404010  00000000`00000000 00000000`00000000
00404020  00000000`00000000 00000000`00000000
00404030  00000000`00000000 00000000`00000000
00404040  00000000`00000000 00000000`00000000
00404050  00000000`00000000 00000000`00000000
kd> !pte 0x404000
                    VA 00404000
PDE at C0600010            PTE at C0002020
contains 0000000017997067  contains 80000000019DC225
pfn 17997     ---DA--UWEV  pfn 19dc      C---A--UR-V
```

![image-20220127144512890](kernel%20lab/image-20220127144512890.png)

还有一个现象，我们也可以来看一下

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>

DWORD out = 0;
#pragma section("data_seg", read, write)
__declspec(allocate("data_seg")) DWORD var = 1;

DWORD g_num;
void __declspec(naked) idtEntry()
{

	__asm {
		mov eax, var
		mov out, eax

		iretd
	}

}
//0x8003f330

void go()
{
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	//system("pause");
	//go();
	void* ptr = malloc(1024 * 1024 * 1024);
	printf("[+] The var is :%p\n", ptr);
	system("pause");
}
```

当我们用malloc分配内存，然后去访问时，内存使用会4k的增加，其实就是挂载了页面。

![image-20220127145106516](kernel%20lab/image-20220127145106516.png)

# 0x11 lab11

## 1. goal

1. 知道TLB的缓存，刷新，和G位刷新
2. 明白TLB和流水线之间的关系



## 2. step

TLB快表的刷新

1. TLB是否有缓存有-->3，没有-->2
2. 访问cr3---->访问内存PDE和PTE----->PTE和PDE是否合法，不是4，是---->在TLB中缓存物理与虚拟地址映射关系
3. 尝试访问物理地址，是否发生异常，是-->4，没有结束
4. 产生0xe异常

我们先来验证一下TLB存在

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

DWORD out = 0;
#pragma section("data_seg", read, write)
__declspec(allocate("data_seg")) DWORD page1[1024];
__declspec(allocate("data_seg")) DWORD page2[1024];
DWORD old_pte[2];
DWORD g_num;
void __declspec(naked) idtEntry()
{

	old_pte[0] = PTE(0x405000)[0];
	old_pte[1] = PTE(0x405000)[1];
	PTE(0x405000)[0] = PTE(0x404000)[0];
	PTE(0x405000)[1] = PTE(0x404000)[1];
	out = page2[0];
	PTE(0x405000)[0] = old_pte[0];
	PTE(0x405000)[1] = old_pte[1];

	__asm {
		iretd
	}

}
//0x8003f330

void go()
{
	page1[0] = 1;
	page2[0] = 2;
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	//system("pause");
	go();

	//void* ptr = malloc(1024 * 1024 * 1024);
	printf("[+] The var is :%p\n", out);
	system("pause");
}
```

>本来我们赋值out就是2，我们打印出来2好像没问题？
>
>其实我们是已经修改了page1的PTE表项，那么应该打印出来1才对，但为什么又打印出来2呢？
>
>其实就是TLB存在的原因，让虚拟地址和物理地址映射，我们可以直接访问到0x405000的内容。
>
>这里重新赋值page2表项是为了防止蓝屏

![image-20220127155347329](kernel%20lab/image-20220127155347329.png)

----

接下来是TLB刷新

**lab10.1.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

DWORD out = 0;
#pragma section("data_seg", read, write)
__declspec(allocate("data_seg")) DWORD page1[1024];
__declspec(allocate("data_seg")) DWORD page2[1024];
DWORD old_pte[2];
DWORD g_num;
void __declspec(naked) idtEntry()
{

	old_pte[0] = PTE(0x405000)[0];
	old_pte[1] = PTE(0x405000)[1];
	PTE(0x405000)[0] = PTE(0x404000)[0];
	PTE(0x405000)[1] = PTE(0x404000)[1];
	__asm {
		mov eax, cr3
		mov cr3, eax
	}
	out = page2[0];
	PTE(0x405000)[0] = old_pte[0];
	PTE(0x405000)[1] = old_pte[1];

	__asm {
		iretd
	}

}
//0x8003f330

void go()
{
	page1[0] = 1;
	page2[0] = 2;
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	//system("pause");
	go();

	//void* ptr = malloc(1024 * 1024 * 1024);
	printf("[+] The var is :%p\n", out);
	system("pause");
}
```

>```
>	__asm {
>		mov eax, cr3
>		mov cr3, eax
>	}
>```
>
>是让TLB无效，但不使g属性内容无效

![image-20220127155859013](kernel%20lab/image-20220127155859013.png)

可以看到值变为1和我们预想的一样

---

设置g位（通常内核页面都是有g位的）使TLB刷新无效

具体看注释吧

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

DWORD out = 0;
#pragma section("data_seg", read, write)
__declspec(allocate("data_seg")) DWORD page1[1024];
__declspec(allocate("data_seg")) DWORD page2[1024];
DWORD old_pte[2];
DWORD g_num;
void __declspec(naked) idtEntry()
{
	//存储原有PTE
	old_pte[0] = PTE(0x405000)[0];		
	old_pte[1] = PTE(0x405000)[1];
	//将0x405000PTE改写0x404000
	PTE(0x405000)[0] = PTE(0x404000)[0]|0x100;
	PTE(0x405000)[1] = PTE(0x404000)[1];
	//使TLB无效
	__asm {
		mov eax, cr3
		mov cr3, eax
	}
	//TLB有效
	__asm mov eax, ds: [0x405000] ;
	//将0x405000PTE恢复
	PTE(0x405000)[0] = old_pte[0];
	PTE(0x405000)[1] = old_pte[1];
	//使TLB无效
	__asm {
		mov eax, cr3
		mov cr3, eax
	}
	//此时应该显示是1
	//若PTE不设置G位那么应该是2
	out = page2[0];

	__asm {
		iretd
	}

}
//0x8003f330

void go()
{
	page1[0] = 1;
	page2[0] = 2;
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	//system("pause");
	go();

	//void* ptr = malloc(1024 * 1024 * 1024);
	printf("[+] The var is :%p\n", out);
	system("pause");
}
```

![image-20220127161650987](kernel%20lab/image-20220127161650987.png)

那么有什么方法能够使g位无效并且刷新TLB表呢？

有一个汇编指令

```
__asm invlpg ds:[0x405000]
```

结果就是2

**lab10.2.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

DWORD out = 0;
#pragma section("data_seg", read, write)
__declspec(allocate("data_seg")) DWORD page1[1024];
__declspec(allocate("data_seg")) DWORD page2[1024];
DWORD old_pte[2];
DWORD g_num;
void __declspec(naked) idtEntry()
{
	//存储原有PTE
	old_pte[0] = PTE(0x405000)[0];		
	old_pte[1] = PTE(0x405000)[1];
	//将0x405000PTE改写0x404000
	PTE(0x405000)[0] = PTE(0x404000)[0]|0x100;
	PTE(0x405000)[1] = PTE(0x404000)[1];
	//使TLB无效
	__asm {
		mov eax, cr3
		mov cr3, eax
	}
	//TLB有效
	__asm mov eax, ds: [0x405000] ;
	//将0x405000PTE恢复
	PTE(0x405000)[0] = old_pte[0];
	PTE(0x405000)[1] = old_pte[1];
	//使TLB无效
	//__asm {
	//	mov eax, cr3
	//	mov cr3, eax
	//}
	__asm invlpg ds:[0x405000]

	out = page2[0];

	__asm {
		iretd
	}

}
//0x8003f330

void go()
{
	page1[0] = 1;
	page2[0] = 2;
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	//system("pause");
	go();

	//void* ptr = malloc(1024 * 1024 * 1024);
	printf("[+] The var is :%p\n", out);
	system("pause");
}
```

![image-20220127161950331](kernel%20lab/image-20220127161950331.png)

自我理解：

TLB主要是为了加快内存读取速度，减少四级页表带来的影响。

但同时对我们攻击者来说就需要考虑TLB的影响，

1. 要考虑内存是否在TLB中，如果在，我们是否可以刷新TLB表进而修改页面权限
2. 如果不在，就要考虑是否会进入TLB表，以及是否有G位，我们是不是可以利用上面那条指令使其在TLB无效

----

接下来看一个神奇的现象

总共有三个

先看第一个

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

DWORD out = 0;
#pragma section("data_seg", read, write)
__declspec(allocate("data_seg")) DWORD page1[1024];
__declspec(allocate("data_seg")) DWORD page2[1024];
DWORD old_pte[2];
DWORD g_num;
void __declspec(naked) idtEntry()
{
	//存储原有PTE
	old_pte[0] = PTE(0x405000)[0];		
	old_pte[1] = PTE(0x405000)[1];
	//使TLB无效
	__asm {
		mov eax, cr3
		mov cr3, eax
	}
	//TLB有效
	__asm mov eax, ds: [0x405000] ;
	//将0x405000PTE改写0x404000
	PTE(0x405000)[0] = PTE(0x404000)[0];
	PTE(0x405000)[1] = PTE(0x404000)[1];

	__asm {
		mov eax, ds:[0x405000]
		mov out, eax
	}

	//将0x405000PTE恢复
	PTE(0x405000)[0] = old_pte[0];
	PTE(0x405000)[1] = old_pte[1];

	__asm {
		mov eax, cr3
		mov cr3, eax
		iretd
	}

}
//0x8003f330

void go()
{
	page1[0] = 0xc3;
	page2[0] = 0xc390;
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	//system("pause");
	
	for (int i = 0; i < 500000; i++) {

		go();
		if (out != 0xc390) {
			printf("[+] The var is %p\n", out);
		}
	}
			
	//void* ptr = malloc(1024 * 1024 * 1024);
	system("pause");
}
```

本来我们放的就是0x405000的东西，但为什么打印出来0xc3呢？

首先可以确定0x405000表项PTE绝对被修改了，那为什么会导致出现0xc3这种情况呢？

其实就是在修改之后，下一条指令执行期间，TLB重新刷新导致TLB表中PTE重新被修改

![image-20220127170151920](kernel%20lab/image-20220127170151920.png)

看下一个

我直接没看懂我调的什么东西

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

DWORD out = 0;
#pragma section("data_seg", read, write)
__declspec(allocate("data_seg")) DWORD page1[1024];
__declspec(allocate("data_seg")) DWORD page2[1024];
DWORD old_pte[2];
DWORD g_num;
void __declspec(naked) idtEntry()
{
	//存储原有PTE
	old_pte[0] = PTE(0x405000)[0];		
	old_pte[1] = PTE(0x405000)[1];
	//使TLB无效
	__asm {
		mov eax, cr3
		mov cr3, eax
	}
	//将0x405000PTE改写0x404000
	PTE(0x405000)[0] = PTE(0x404000)[0];
	PTE(0x405000)[1] = PTE(0x404000)[1];
	__asm int 3;
	__asm {
		mov eax, ds:[0x405000]
		mov out, eax
	}

	//将0x405000PTE恢复
	PTE(0x405000)[0] = old_pte[0];
	PTE(0x405000)[1] = old_pte[1];

	__asm {
		mov eax, cr3
		mov cr3, eax
		iretd
	}

}
//0x8003f330

void go()
{
	page1[0] = 0xc3;
	page2[0] = 0xc390;
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	//system("pause");
	

		go();
		if (out != 0xc3) {
			printf("[+] The var is %p\n", out);
		}
			
	//void* ptr = malloc(1024 * 1024 * 1024);
	system("pause");
}
```

>按理来说，我们没访问过0x405000，说明TLB中没有0x405000，当改变其PTE表项时，访问的也是0x404000，windbg显示的也是c3，
>
>但打印出来确实c390，
>
>现在还无法解释这个问题，需要后续深入学习
>
>先说原理
>
>1. **我们没有访问过0x405000这块内存，所以就不应该被加载在TLB表项里**
>2. **下一次访问0x405000时，由于我们已经修改了PTE表项，应该读取的就是0x404000的内容。**
>
>但莫名奇妙出现了这种东西

```
kd> dq 00403378
00403378  1bf62067`000000c3 00000000`80000000
00403388  00000000`00000024 00000000`00000000
```

![image-20220127172435355](kernel%20lab/image-20220127172435355.png)

最后一个现象，

当我们把数据改成代码，具体操作就是执行一下，编译器会自动将数据附上可执行属性

```
	((void(*)())(DWORD)page1)();
	((void(*)())(DWORD)page2)();
```

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

DWORD out = 0;
#pragma section("data_seg", read, write)
__declspec(allocate("data_seg")) DWORD page1[1024];
__declspec(allocate("data_seg")) DWORD page2[1024];
DWORD old_pte[2];
DWORD g_num;
void __declspec(naked) idtEntry()
{
	//存储原有PTE
	old_pte[0] = PTE(0x405000)[0];		
	old_pte[1] = PTE(0x405000)[1];
	//使TLB无效
	__asm {
		mov eax, cr3
		mov cr3, eax
	}
	//将0x405000PTE改写0x404000
	PTE(0x405000)[0] = PTE(0x404000)[0];
	PTE(0x405000)[1] = PTE(0x404000)[1];
	__asm {
		mov eax, ds:[0x405000]
		mov out, eax
	}

	//将0x405000PTE恢复
	PTE(0x405000)[0] = old_pte[0];
	PTE(0x405000)[1] = old_pte[1];

	__asm {
		mov eax, cr3
		mov cr3, eax
		iretd
	}

}
//0x8003f330

void go()
{
	page1[0] = 0xc3;
	page2[0] = 0xc390;
	((void(*)())(DWORD)page1)();
	((void(*)())(DWORD)page2)();
	__asm int 0x20;
}

void go2()
{
	__asm int 0x21;
}
void main()
{
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	//system("pause");
	
	for (int i = 0; i < 10000; i++) {
		go();
		if (out != 0xc3) {
			printf("[+] The var is %p\n", out);
		}
	}
			
	//void* ptr = malloc(1024 * 1024 * 1024);
	system("pause");
}
```

>淦，很神奇，这个又恢复了
>
>按理说第三个实验是在第二个实验的基础上做的，应该与第二个实验是相反的才对，
>
>不清楚什么情况，先放在这里，回来解决
>
>这里大佬给出的解释是：
>
>**流水线，就是预取指令，我们在给0x405000的PTE修改表项时，下一条指令已经预取了，**
>
>**由于前面的0x405000的指令已经执行过了，所以会存进去预取范围内**
>
>**上一个实验由于0x405000是数据段没有机会进预取指令的范围，所以不会被存进去**

但我这里想的是，可能不管什么属性，只要有可能进入预取指令周期内，那么就会读取。就像我实验二一样，还是有疑问，待解决....

![image-20220127174016264](kernel%20lab/image-20220127174016264.png)



# 0x12 lab12

## 1. goal

1. hook页面异常中断
2. 对页面异常能够分析

## 2. step

这个实验我们需要干一件事，就是对页面异常也就是0xe进行hook，然后打印其堆栈信息。

具体操作就是对该函数地址挂钩，然后跳转到gdt表项我们构造指令的位置，具体要实现哪些指令？

我们想要的是堆栈信息，那么我们就应该保存，这里应该保存在内核地址，也是gdt的位置，然后另起一个进程进行读取就可以。

写代码，这里可以复用以前的代码

首先明确我们要hook的函数

```
.text:80541694                                _KiTrap0E       proc near
.text:80541694
.text:80541694                                arg_8           = dword ptr  0Ch
.text:80541694
.text:80541694                                ; FUNCTION CHUNK AT .text:805415B1 SIZE 00000036 BYTES
.text:80541694
.text:80541694 000 66 C7 44 24 02 00 00                       mov     word ptr [esp+2], 0
.text:8054169B 000 55                                         push    ebp
.text:8054169C 004 53                                         push    ebx
.text:8054169D 008 56                                         push    esi
.text:8054169E 00C 57                                         push    edi
.text:8054169F 010 0F A0                                      push    fs
.text:805416A1 014 BB 30 00 00 00                             mov     ebx, 30h ; '0'
.text:805416A6 014 66 8E E3                                   mov     fs, bx
.text:805416A9                                                assume fs:nothing
.text:805416A9 014 64 8B 1D 00 00 00 00                       mov     ebx, large fs:0
.text:805416B0 014 53                                         push    ebx
.text:805416B1 018 83 EC 04                                   sub     esp, 4          ; Integer Subtraction
.text:805416B4 01C 50                                         push    eax
.text:805416B5 020 51                                         push    ecx
.text:805416B6 024 52                                         push    edx
.text:805416B7 028 1E                                         push    ds
.text:805416B8 02C 06                                         push    es
.text:805416B9 030 0F A8                                      push    gs
.text:805416BB 034 66 B8 23 00                                mov     ax, 23h ; '#'
.text:805416BF 034 83 EC 30                                   sub     esp, 30h        ; Integer Subtraction
.text:805416C2 064 66 8E D8                                   mov     ds, ax
.text:805416C5                                                assume ds:nothing
.text:805416C5 064 66 8E C0                                   mov     es, ax
.text:805416C8                                                assume es:nothing
```

我们要执行一段指令

```
push 0x8003f120
ret

硬编码
0x03f12068
0xc380
```

只需要把这个写到函数对应位置就可以

直接看代码吧。

**kernelcode3.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


#define K_ESP 0x8003f3f4
#define K_ESP_4 0x8003f3f0
#define K_TARGET_CR3 0x8003f3ec
#define K_CR2 0x8003f3e8

//0x00401040
ULONG32 i;
char* p;
void JumpTarget();
void __declspec(naked) idtEntry()
{
	p = (char*)0x8003f120;
	for (i = 0; i < 0x80; i++) {
		*p = ((char *)JumpTarget)[i];
		p++;
	}

	//eq 8003f500 8003ee00`0008f180
	__asm {
		mov eax, 0xffffffff
		mov ds:[K_TARGET_CR3], eax
		//取消写保护
		mov eax, cr0
		and eax, not 10000h
		mov cr0, eax

		mov eax, 0x03f12068
		mov ds:[0x80541694], eax
		mov ax, 0xc380
		mov ds:[0x80541694+0x4], ax

		xor eax, eax
		mov ds:[K_ESP], eax
		mov ds : [K_CR2] , eax
		mov ds : [K_ESP_4] , eax

		mov eax, cr0
		or eax, 10000h
		mov cr0, eax
		iretd
	}


}
//0x8003f330
/*
errno
eip
cs
eflags
esp
ss
*/
void __declspec(naked) JumpTarget() {
	__asm {
		push eax
		mov eax, cr3
		cmp eax, ds:[K_TARGET_CR3]
		jnz END

		mov eax, ds:[esp+4]
		mov ds:[K_ESP], eax
		mov eax, ds : [esp + 8]
		mov ds : [K_ESP_4] , eax
		mov eax, cr2
		mov ds : [K_CR2] , eax

		END:
		pop eax
		mov     word ptr[esp + 2], 0
		push 0x8054169B
		ret
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
	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

![image-20220127233753453](kernel%20lab/image-20220127233753453.png)

```
kd> u 0x80541694
80541694 6820f10380      push    8003F120h
80541699 c3              ret
8054169a 005553          add     byte ptr [ebp+53h],dl
8054169d 56              push    esi
8054169e 57              push    edi
8054169f 0fa0            push    fs
805416a1 bb30000000      mov     ebx,30h
805416a6 668ee3          mov     fs,bx
kd> u 0x8003F120 l20
8003f120 50              push    eax
8003f121 0f20d8          mov     eax,cr3
8003f124 3e3b05ecf30380  cmp     eax,dword ptr ds:[8003F3ECh]
8003f12b 751f            jne     8003f14c
8003f12d 3e8b442404      mov     eax,dword ptr ds:[esp+4]
8003f132 3ea3f4f30380    mov     dword ptr ds:[8003F3F4h],eax
8003f138 3e8b442408      mov     eax,dword ptr ds:[esp+8]
8003f13d 3ea3f0f30380    mov     dword ptr ds:[8003F3F0h],eax
8003f143 0f20d0          mov     eax,cr2
8003f146 3ea3e8f30380    mov     dword ptr ds:[8003F3E8h],eax
8003f14c 58              pop     eax
8003f14d 66c74424020000  mov     word ptr [esp+2],0
8003f154 689b165480      push    offset nt!KiTrap0E+0x7 (8054169b)
8003f159 c3              ret
```

hook成功

接下来写我们另一个程序

它主要用来读取页面异常的栈帧信息

这里面的栈帧信息

```
/*
errno
eip
cs
eflags
esp
ss
*/
```

我们要读取什么呢？

1. errno可以让我们知道是什么样的页面异常

2. eip可以让我们知道现在堆栈执行在哪里

3. cr2是将页面异常的虚拟地址打印出来

   ```
   mov ds:[0], eax
   此时0页面异常cr2就是0
   如果是指令执行异常，那么eip和cr2是一样的
   ```

   

   **lab12.1.exe**

   ```c
   #include<stdio.h>
   #include<stdlib.h>
   #include<Windows.h>
   
   
   #define K_ESP 0x8003f3f4
   #define K_ESP_4 0x8003f3f0
   #define K_TARGET_CR3 0x8003f3ec
   #define K_CR2 0x8003f3e8
   
   DWORD g_esp;
   DWORD g_esp_4;
   DWORD g_cr2;
   void __declspec(naked) idtEntry()
   {
   	__asm {
   		mov eax, cr3
   		mov ds:[K_TARGET_CR3], eax
   
   		mov eax, ds:[K_ESP]
   		mov g_esp, eax
   		mov eax, ds : [K_ESP_4]
   		mov g_esp_4, eax
   		mov eax, ds : [K_CR2]
   		mov g_cr2, eax
   
   		xor eax, eax
   		mov ds:[K_ESP_4], eax
   		iretd
   	}
   
   }
   //0x8003f330
   #pragma code_seg(".my_code") __declspec(allocate(".my_code")) void go();
   #pragma code_seg(".my_code") __declspec(allocate(".my_code")) void main();
   void go()
   {
   	__asm int 0x20;
   }
   
   void go2()
   {
   	__asm int 0x21;
   }
   void main()
   {
   	if ((DWORD32)idtEntry != 0x00401040) {
   		printf("wrong, the addr is %p\n", idtEntry);
   		exit(-1);
   	}
   	while (1) {
   		go();
   		if (g_esp_4){
   			printf("[+] The eip:%p\t", g_esp_4);
   			printf("The errno:%p\t", g_esp);
   			printf("The cr2:%p\n", g_cr2);
   		}
   			
   
   	}
   	//system("pause");
   			
   	system("pause");
   }
   ```

![image-20220128000203027](kernel%20lab/image-20220128000203027.png)

异常已经打印出来了，我们可以看一下这些异常

异常码

![image-20220128000527050](kernel%20lab/image-20220128000527050.png)

先看第一个

0x402000就是我们设置的main函数的页边界，很好理解，当我们执行main函数时，是缺页的，就会触发异常

**异常码表示用户态取指令异常**

看第二个

**异常码表示用户态取指令异常**，地址看起来像是大地址。

是写控制台，为什么会出现，我们调用这个函数是printf，其实是因为不常用所以不挂在物理页面上。

```
kd> u 0x7c81bffa
kernel32!WriteConsoleInternal:
7c81bffa 68ac000000      push    0ACh
7c81bfff 68d0c0817c      push    offset kernel32!`string'+0x40 (7c81c0d0)
7c81c004 e8cd64feff      call    kernel32!_SEH_prolog (7c8024d6)
7c81c009 a1cc56887c      mov     eax,dword ptr [kernel32!__security_cookie (7c8856cc)]
7c81c00e 8945e4          mov     dword ptr [ebp-1Ch],eax
7c81c011 8b750c          mov     esi,dword ptr [ebp+0Ch]
7c81c014 8b5d14          mov     ebx,dword ptr [ebp+14h]
7c81c017 64a118000000    mov     eax,dword ptr fs:[00000018h]
```

---

用CE看一下实验现象

当我们只要用户CE附加进程是会出现下面的页异常

![image-20220128001422837](kernel%20lab/image-20220128001422837.png)

**看起来都是内核地址，很明显是内核读异常**。

第一个我们看一下eip,发现是这个函数

```
 MiDoPoolCopy(PRKPROCESS PROCESS, int, PRKPROCESS, volatile void *Address, SIZE_T Length, char, int)
```

为什么会出现这个？

就是复制内存的函数，也就是复制431b28这个地址内存时出现缺页异常

第二个是这个函数

```
MmProbeAndLockPages(PMDL MemoryDescriptorList, KPROCESSOR_MODE AccessMode, LOCK_OPERATION Operation)
```

页面探测函数，应该是NtReadProcessMemory里面出现的

**当我们用CE读取不可读内存时，也会发生缺页异常，看起来CE是通过内核函数来读取内存的**

![image-20220128002348642](kernel%20lab/image-20220128002348642.png)

![image-20220128002306001](kernel%20lab/image-20220128002306001.png)

**用CE写0x402000内存时**

![image-20220128002537690](kernel%20lab/image-20220128002537690.png)

会产生用户态写异常

函数是这个

```
ProbeForWrite(volatile void *Address, SIZE_T Length, ULONG Alignment)
```

探测是否可写

---

当我们用内核CE又会发生不同现象

**附加进程**

![image-20220128002936971](kernel%20lab/image-20220128002936971.png)

内核读异常，我的理解：CE在我们读取地址时，需要复制一整块内存，但很明显0地址时不存在的，所以异常

```
memcpy(void *, const void *Src, size_t MaxCount)
```

**查看内存**

![image-20220128003233757](kernel%20lab/image-20220128003233757.png)

又是内核读异常，但这次cr2成了0x324，这是为什么？

CE高权限用内核读内存，CE应该是挂了零地址页面，修改了它的PTE表项，从而可读，然后复制内存到零地址页面进行读取。

为什么要这样做？可能是为了方便？

# 0x13 lab13

## 1. goal

1. 理解shadowwalker

## 2. step

基于上述实验我们要实现shadowwalker

具体原理是什么呢？

我们由上面可以看到，如果第一次读取0x402000指令时，会造成一次执行指令异常，那么我们就可以接管这个异常，然后把这个页面的PTE表项直接置0，这样我们用调试器看的时候就会发现此处的内存是空的，至于程序如何执行指令，可以直接从ITLB中直接读取。

如果是读写异常，我们可以把他挂载到一个假页面上，从而达到修改内存。

**kernel lab13.1.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


#define K_ESP 0x8003f3f4
#define K_ESP_4 0x8003f3f0
#define K_TARGET_CR3 0x8003f3ec
#define K_CR2 0x8003f3e8
#define K_REAL_PTE0 0x8003f3e4
#define K_REAL_PTE1 0x8003f3e0
#define K_FAKE_PTE0 0x8003f3dc
#define K_FAKE_PTE1 0x8003f3d8
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

#pragma section(".mydata", read,write)
__declspec(allocate(".mydata")) DWORD FAKE_PAGE[1024];


DWORD g_esp;
DWORD g_esp_4;
DWORD g_cr2;
void __declspec(naked) idtEntry()
{


	*(DWORD*)K_REAL_PTE0 = PTE(0x402000)[0];
	*(DWORD*)K_REAL_PTE1 = PTE(0x402000)[1];
	*(DWORD*)K_FAKE_PTE0 = PTE(0x405000)[0];
	*(DWORD*)K_FAKE_PTE0 = PTE(0x405000)[1];
	PTE(0x402000)[0] = PTE(0x402000)[1] = 0;
	__asm {
		mov eax, cr3
		mov ds:[K_TARGET_CR3], eax
		iretd
	}

}
//0x8003f330
#pragma code_seg(".my_code") __declspec(allocate(".my_code")) void go();
#pragma code_seg(".my_code") __declspec(allocate(".my_code")) void main();
void go()
{
	__asm int 0x20;
}

void main()
{

	__asm {
		jmp L
		ret
	L :
	}
	
	if ((DWORD32)idtEntry != 0x00401040) {
		printf("wrong, the addr is %p\n", idtEntry);
		exit(-1);
	}
	FAKE_PAGE[0] = 0;
	go();
	int i = 0;
	while (1) {	
		printf("[+] count:%d\n", i++);
		Sleep(1000);
	}
	//system("pause");
			
	system("pause");
}
```

**kernelcode4.exe**

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>


#define K_ESP 0x8003f3f4
#define K_ESP_4 0x8003f3f0
#define K_TARGET_CR3 0x8003f3ec
#define K_CR2 0x8003f3e8
#define K_REAL_PTE0 0x8003f3e4
#define K_REAL_PTE1 0x8003f3e0
#define K_FAKE_PTE0 0x8003f3dc
#define K_FAKE_PTE1 0x8003f3d8
#define PTE(x) ((DWORD *)(0xc0000000 + ((x >> 12) << 3)))
#define PDE(x) ((DWORD *)(0xc0600000 + ((x >> 21) << 3)))

//0x00401040
ULONG32 i;
char* p;
void JumpTarget();
void __declspec(naked) idtEntry()
{
	p = (char*)0x8003f120;
	for (i = 0; i < 0x100; i++) {
		*p = ((char *)JumpTarget)[i];
		p++;
	}

	//eq 8003f500 8003ee00`0008f180
	__asm {
		mov eax, 0xffffffff
		mov ds:[K_TARGET_CR3], eax
		//取消写保护
		mov eax, cr0
		and eax, not 10000h
		mov cr0, eax

		mov eax, 0x03f12068
		mov ds:[0x80541694], eax
		mov ax, 0xc380
		mov ds:[0x80541694+0x4], ax

		xor eax, eax
		mov ds:[K_ESP], eax
		mov ds : [K_CR2] , eax
		mov ds : [K_ESP_4] , eax

		mov eax, cr0
		or eax, 10000h
		mov cr0, eax
		iretd
	}


}
//0x8003f330
/*
eip
cs
eflags
esp
ss
*/
void __declspec(naked) JumpTarget() {
	__asm {
		//判断对应进程
		pushad
		mov eax, cr3
		cmp eax, ds: [K_TARGET_CR3]
		jnz PASS
		//判读是否是0x402000
		mov eax, cr2
		shr eax, 0xc
		cmp eax, 0x402
		jnz PASS

		//判断是否是执行
		mov eax, ss : [esp + 0x20]
		test eax, 0x10
		jnz EXECU
		jmp READ_WRITE

		EXECU :
	}
	//恢复真实页面
	PTE(0x402000)[0] = *(DWORD*)K_REAL_PTE0;
	PTE(0x402000)[1] = *(DWORD*)K_REAL_PTE1;
	//读取一下指令，放在TLB中
	__asm {
		mov eax, 0x00402004
		call eax
	}
	//把页面置为假页面
	PTE(0x402000)[0] = PTE(0x402000)[1] = 0;
	__asm {
		popad
		add esp, 4
		iretd

		READ_WRITE :
	}
	//将页面挂为假页面
	PTE(0x402000)[0] = *(DWORD*)K_FAKE_PTE0;
	PTE(0x402000)[1] = *(DWORD*)K_FAKE_PTE1;
	//读取到数据TLB中
	__asm {
		mov eax, ds:[0x402000]
	}
	//置为0
	PTE(0x402000)[0] = PTE(0x402000)[1] = 0;
	__asm{
		popad
		add esp, 4
		iretd
	PASS:
		popad
		mov     word ptr[esp + 2], 0
		push 0x8054169B
		ret
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
	//printf("[+] The ptr is:%p\n", g_pool);
	system("pause");
}
```

最终效果

```
kd> dq 0x402000
00402000  ????????`???????? ????????`????????
00402010  ????????`???????? ????????`????????
00402020  ????????`???????? ????????`????????
00402030  ????????`???????? ????????`????????
00402040  ????????`???????? ????????`????????
00402050  ????????`???????? ????????`????????
00402060  ????????`???????? ????????`????????
00402070  ????????`???????? ????????`????????
```

![image-20220129124216119](kernel%20lab/image-20220129124216119.png)

读写可能有点问题，暂未解决....





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

## inline hook

```
pop addr
ret

mov eax,addr
call eax
```

## syscall

1. 注意三环和零环哪里切换，要注意ret和iretd
2. 注意内平栈和外平栈
3. 知道如何用esp来定位参数

## TLB与流水线

第二个实验原理与现象不一致，需要进一步学习

预取指令到底是根据什么来划分的？还是说每一条指令的下一条指令一定会被预取？



## 自定义段

```
#pragma section(".mydata", read,write)
__declspec(allocate(".mydata")) DWORD FAKE_PAGE[1024]
#pragma code_seg(".my_code") __declspec(allocate(".my_code")) void go();
#pragma code_seg(".my_code") __declspec(allocate(".my_code")) void main();
```

