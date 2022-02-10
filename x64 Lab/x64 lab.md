# 0x01 Lab 01 IA-32e

## 1. goal

1. 理解IA-32e模式
2. 熟悉x86和x64的不同
3. 能够分析64位下32位的系统调用

## 2. step

IA-32e：内核64，用户64或32，强制平坦段

Legacy：内核32，用户32，非平坦段

---

**MSR寄存器：C0000080H**

确定是否是IA-32e

```
kd> rdmsr C0000080
msr[c0000080] = 00000000`00004d01
kd> .formats 00000000`00004d01
Evaluate expression:
  Hex:     00000000`00004d01
  Decimal: 19713
  Octal:   0000000000000000046401
  Binary:  00000000 00000000 00000000 00000000 00000000 00000000 01001101 00000001
  Chars:   ......M.
  Time:    Thu Jan  1 13:28:33 1970
  Float:   low 2.76238e-041 high 0
  Double:  9.73952e-320
```

---

**与x86的不同**

**gdt表**

gdt仍选择64，由于强制平坦，所以基址为0，界限为最大

描述符

![image-20220204160029785](x64%20lab/image-20220204160029785.png)

```
kd> dq gdtr
fffff801`87a77fb0  00000000`00000000 00000000`00000000
fffff801`87a77fc0  00209b00`00000000 00409300`00000000
fffff801`87a77fd0  00cffb00`0000ffff 00cff300`0000ffff
fffff801`87a77fe0  0020fb00`00000000 00000000`00000000
fffff801`87a77ff0  87008ba7`60000067 00000000`fffff801
fffff801`87a78000  0040f300`00003c00 00000000`00000000
fffff801`87a78010  00000000`00000000 00000000`00000000
fffff801`87a78020  00000000`00000000 00000000`00000
```

我们看几个描述符

```
00209b00`00000000
0环代码段64位
kd> dg 10
                                                    P Si Gr Pr Lo
Sel        Base              Limit          Type    l ze an es ng Flags
---- ----------------- ----------------- ---------- - -- -- -- -- --------
0010 00000000`00000000 00000000`00000000 Code RE Ac 0 Nb By P  Lo 0000029b

0环数据段
kd> dg 18
                                                    P Si Gr Pr Lo
Sel        Base              Limit          Type    l ze an es ng Flags
---- ----------------- ----------------- ---------- - -- -- -- -- --------
0018 00000000`00000000 00000000`00000000 Data RW Ac 0 Bg By P  Nl 00000493

3环代码段32位
kd> dg 20
                                                    P Si Gr Pr Lo
Sel        Base              Limit          Type    l ze an es ng Flags
---- ----------------- ----------------- ---------- - -- -- -- -- --------
0020 00000000`00000000 00000000`ffffffff Code RE Ac 3 Bg Pg P  Nl 00000cfb
3环数据段
kd> dg 28
                                                    P Si Gr Pr Lo
Sel        Base              Limit          Type    l ze an es ng Flags
---- ----------------- ----------------- ---------- - -- -- -- -- --------
0028 00000000`00000000 00000000`ffffffff Data RW Ac 3 Bg Pg P  Nl 00000cf3

3环代码段64位
kd> dg 30
                                                    P Si Gr Pr Lo
Sel        Base              Limit          Type    l ze an es ng Flags
---- ----------------- ----------------- ---------- - -- -- -- -- --------
0030 00000000`00000000 00000000`00000000 Code RE Ac 3 Nb By P  Lo 000002fb
```

1. **我们可以看到CS=0x23就是32位，CS=0x33就是64位、**
2. **32位和64位都用同一数据段**

---

**TSS段描述符**

TSS是128位，主要存放一些RSP指针

![image-20220204160820314](x64%20lab/image-20220204160820314.png)

```
fffff801`87a77ff0  87008ba7`60000067 00000000`fffff801
base -> fffff80187a76000
kd> dq fffff80187a76000
fffff801`87a76000  cb187c90`00000000 00000000`ffffb10d
fffff801`87a76010  00000000`00000000 00000000`00000000
fffff801`87a76020  87a9e000`00000000 87aac000`fffff801
fffff801`87a76030  87aa5000`fffff801 87ab3000`fffff801
fffff801`87a76040  00000000`fffff801 00000000`00000000
fffff801`87a76050  00000000`00000000 00000000`00000000
fffff801`87a76060  00680000`00000000 00000000`00000000
fffff801`87a76070  fffff801`7f876000 00000000`00000000
```

---

IDT：扩展128位

![image-20220204161620893](x64%20lab/image-20220204161620893.png)

```
kd> dq idtr
fffff801`87a75000  83a08e00`00101d00 00000000`fffff801
fffff801`87a75010  83a08e04`00102040 00000000`fffff801
fffff801`87a75020  83a08e03`00102540 00000000`fffff801
fffff801`87a75030  83a0ee00`00102a00 00000000`fffff801
fffff801`87a75040  83a0ee00`00102d40 00000000`fffff801
fffff801`87a75050  83a08e00`00103080 00000000`fffff801
fffff801`87a75060  83a08e00`001035c0 00000000`fffff801
fffff801`87a75070  83a08e00`00103ac0 00000000`fffff801
kd> !idt

Dumping IDT: fffff80187a75000

00:	fffff80183a01d00 nt!KiDivideErrorFault
01:	fffff80183a02040 nt!KiDebugTrapOrFault	Stack = 0xFFFFF80187AB3000
02:	fffff80183a02540 nt!KiNmiInterrupt	Stack = 0xFFFFF80187AA5000
03:	fffff80183a02a00 nt!KiBreakpointTrap
04:	fffff80183a02d40 nt!KiOverflowTrap
05:	fffff80183a03080 nt!KiBoundFault
06:	fffff80183a035c0 nt!KiInvalidOpcodeFault
07:	fffff80183a03ac0 nt!KiNpxNotAvailableFault
08:	fffff80183a03dc0 nt!KiDoubleFaultAbort	Stack = 0xFFFFF80187A9E000
09:	fffff80183a040c0 nt!KiNpxSegmentOverrunAbort
0a:	fffff80183a043c0 nt!KiInvalidTssFault
0b:	fffff80183a046c0 nt!KiSegmentNotPresentFault
0c:	fffff80183a04a80 nt!KiStackFault
0d:	fffff80183a04dc0 nt!KiGeneralProtectionFault
0e:	fffff80183a05100 nt!KiPageFault
10:	fffff80183a05740 nt!KiFloatingErrorFault
11:	fffff80183a05b00 nt!KiAlignmentFault
12:	fffff80183a05e40 nt!KiMcheckAbort	Stack = 0xFFFFF80187AAC000
kd> !idt 0

Dumping IDT: fffff80187a75000

00:	fffff80183a01d00 nt!KiDivideErrorFault


83a08e00`00101d00 00000000`fffff801
IST --> 00
Selector --> 0010
```

---

x86时，fs是指向KPCR的，但x64，gs是指向KPCR，MSR寄存器有几个重要的东西

```
MSR[0xc0000100] = fs_base
MSR[0xc0000101] = gs_base
MSR[0xc0000102] = gs_kernel_base



kd> rdmsr 0xc0000101
msr[c0000101] = fffff801`7f876000
kd> dt _KPCR fffff801`7f876000
nt!_KPCR
   +0x000 NtTib            : _NT_TIB
   +0x000 GdtBase          : 0xfffff801`87a77fb0 _KGDTENTRY64
   +0x008 TssBase          : 0xfffff801`87a76000 _KTSS64
   +0x010 UserRsp          : 0x000000e6`ab17efa8
   +0x018 Self             : 0xfffff801`7f876000 _KPCR
   +0x020 CurrentPrcb      : 0xfffff801`7f876180 _KPRCB
   +0x028 LockArray        : 0xfffff801`7f876870 _KSPIN_LOCK_QUEUE
   +0x030 Used_Self        : 0x000000e6`aacdd000 Void
   +0x038 IdtBase          : 0xfffff801`87a75000 _KIDTENTRY64
   +0x040 Unused           : [2] 0
   +0x050 Irql             : 0 ''
   +0x051 SecondLevelCacheAssociativity : 0x10 ''
   +0x052 ObsoleteNumber   : 0 ''
   +0x053 Fill0            : 0 ''
   +0x054 Unused0          : [3] 0
   +0x060 MajorVersion     : 1
   +0x062 MinorVersion     : 1
   +0x064 StallScaleFactor : 0xc7a
   +0x068 Unused1          : [3] (null) 
   +0x080 KernelReserved   : [15] 0
   +0x0bc SecondLevelCacheSize : 0x1000000
   +0x0c0 HalReserved      : [16] 0xbe609280
   +0x100 Unused2          : 0
   +0x108 KdVersionBlock   : (null) 
   +0x110 Unused3          : (null) 
   +0x118 PcrAlign1        : [24] 0
   +0x180 Prcb             : _KPRCB
```

---

**权限切换**

1. 系统调用：只有一张ssdt表，32位先进入64位，然后再调用。64位就直接调用
2. 中断：idt表，进入中断服务例程

ntdll里面既有32位也有64位代码，32位进入64位主要是根据一个远跳指令，将cs修改成33，直接执行64位代码

---

我们先来分析一下64位系统中，32位如何调用64位

我们先随便下一个断点

![image-20220206174151858](x64%20lab/image-20220206174151858.png)

跟进之后发现

![image-20220206174223765](x64%20lab/image-20220206174223765.png)

会远跳一个64位地址，所以下面的代码就不能用32位调试器来看了

但我们可以编辑一下看一下

![image-20220206175238152](x64%20lab/image-20220206175238152.png)

发现是调用了r15寄存器，接下来我们直接上windbg，下断点到ntopenprocess处

```
kd> .process /i ffffcd8dc4b48080
You need to continue execution (press 'g' <enter>) for the context
to be switched. When the debugger breaks in again, you will be in
the new process context.
kd> g
Break instruction exception - code 80000003 (first chance)
nt!DbgBreakPointWithStatus:
fffff802`257ff140 cc              int     3
kd> bp 77a22c30
breakpoint 2 redefined
kd> u 77a22c30
00000000`77a22c30 b826000000      mov     eax,26h
00000000`77a22c35 ba7088a377      mov     edx,77A38870h
00000000`77a22c3a ffd2            call    rdx
00000000`77a22c3c c21000          ret     10h
00000000`77a22c3f 90              nop
00000000`77a22c40 b827000000      mov     eax,27h
00000000`77a22c45 ba7088a377      mov     edx,77A38870h
00000000`77a22c4a ffd2            call    rdx
kd> g
The context is partially valid. Only x86 user-mode context is available.
WOW64 breakpoint - code 4000001f (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
00000000`77a22c30 b826000000      mov     eax,26h
32.kd:x86> t
The context is partially valid. Only x86 user-mode context is available.
Breakpoint 2 hit
00000000`77a22c35 ba7088a377      mov     edx,77A38870h
32.kd:x86> t
The context is partially valid. Only x86 user-mode context is available.
00000000`77a22c3a ffd2            call    edx
32.kd:x86> t
The context is partially valid. Only x86 user-mode context is available.
00000000`77a38870 ff252892ad77    jmp     dword ptr ds:[77AD9228h]
32.kd:x86> t
The context is partially valid. Only x86 user-mode context is available.
wow64cpu!KiFastSystemCall:
00000000`779a7000 ea09709a773300  jmp     0033:779A7009
32.kd:x86> t
wow64cpu!KiFastSystemCall+0x9:
0033:00000000`779a7009 41ffa7f8000000  jmp     qword ptr [r15+0F8h]

```

![image-20220207140127167](x64%20lab/image-20220207140127167.png)

我们可以发现r12寄存器是没有改变的

---

那么我们就可以以此为实验，在32位下转换到64位，然后将数据读入r12中，之后将r12写到一个内存中，之后在读取出来。

由于调试只能在32位下，所以64位里面的操作是完全看不见的。

我们需要手写一些硬编码

```
49 c7 c4 78 56 34 12 	-->		mov r12, 0x12345678
b8 18 30 40 00			--> 	mov eax, 403018
48 ff 28				--> 	jmp rax
//这个需要算相对偏移
4c 89 25 41 23			--> 	mov ds:[0x403388h], r12
48 ff 28				--> jmp far tword ds:[rax]
```

![image-20220207160638540](x64%20lab/image-20220207160638540.png)

这里可以看到已经修改成功了

![image-20220207165830945](x64%20lab/image-20220207165830945.png)

但好像出现了问题，他的远跳指令是无法执行的，根本跳不到401096这个地方。没看懂，但windbg中确实是已经跳转了

```c
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>

char far_jmp[10] = { 0x00, 0x10, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x23, 0x00 };//403018
char secret[8] = { 0, }; //403388

void __declspec(naked) go_read() // 0x401040
{
	__asm {
		//__emit 0x49		//mov r12, 0x12345678
		//__emit 0xc7
		//__emit 0xc4
		//__emit 0x78
		//__emit 0x56
		//__emit 0x34
		//__emit 0x12

		//__emit 0x90
		//__emit 0x90
		//__emit 0x90
		//__emit 0x90
		//__emit 0x90
		//__emit 0x90
		//__emit 0x90
		// 
		//403388-7-401040
		__emit 0x4c		//mov qword ptr ds:[0x403388], r12
		__emit 0x89
		__emit 0x25
		__emit 0x41
		__emit 0x23
		__emit 0x00
		__emit 0x00		//40100d

		__emit 0xb8		// mov eax, 403018h
		__emit 0x18
		__emit 0x30
		__emit 0x40
		__emit 0x00

		__emit 0x48
		__emit 0xff
		__emit 0x28

	}
}

void __declspec(naked) go_write() // 0x401050
{
	__asm {
		//__emit 0xcc
		__emit 0x49		//mov r12, 0x12345678
		__emit 0xc7
		__emit 0xc4
		__emit 0x78
		__emit 0x56
		__emit 0x34
		__emit 0x12

		__emit 0xb8		// mov eax, 403018h
		__emit 0x18
		__emit 0x30
		__emit 0x40
		__emit 0x00

		__emit 0x48
		__emit 0xff
		__emit 0x28

	}
}

void main()
{
	printf("go_read: %p\n", go_read);
	printf("go_write: %p\n", go_write);
	*(unsigned int*)far_jmp = 0x00401096;
	__asm {
		int 3
		__emit 0xea
		__emit 0x50
		__emit 0x10
		__emit 0x40
		__emit 0x00
		__emit 0x33
		__emit 0x00
	}
L1://00401095
	printf("write data ok\n");
	system("pause");
	//printf("L: %p\n");
	*(unsigned int*)far_jmp = 0x004010bf;
	__asm {
		__emit 0xea
		__emit 0x00
		__emit 0x10
		__emit 0x40
		__emit 0x00
		__emit 0x33
		__emit 0x00
	}
L2: //004010be

	printf("back to x86\n");
	printf("%p\n", *(int*)secret);
	system("pause");
}
```

# 0x02 Lab 02 SMAP and SMEP

## 1. goal

1. 自己构造中断函数
2. 理解SMAP和SMEP的实现

## 2. step

我们需要自己构造一个中断函数去实现，就像xp中的一样

这里需要构造21号中断

`main.cpp`

**smap.exe**

```c++
#include<stdio.h>
#include<Windows.h>
#include<stdlib.h>



extern "C" void idtEntry();
extern "C" void go();
extern "C" ULONG64 x;
ULONG64 x;
//idtr fffff80060075000
//
void main()
{
	if ((ULONG64)idtEntry != 0x00000001400010f0) {
		printf("[+] idtEntry is %p\n", idtEntry);
		system("pause");
		exit(-1);
	}
	system("pause");
	go();
	printf("[+] The information is %p\n", x);
	system("pause");
}
```

`main.asm`

```asm
option casemap:none


.data

EXTERN x:qword
PUBLIC idtEntry
PUBLIC go
.code
idtEntry proc
		stac
		mov rax, 1
		iretq
idtEntry endp

go proc
	int 21h
	ret
go endp
end
```

```
kd> !idt 21

Dumping IDT: fffff80060075000

kd> dq fffff80060075000
fffff800`60075000  5c408e00`00101d00 00000000`fffff800
fffff800`60075010  5c408e04`00102040 00000000`fffff800
kd> eq fffff807`0496e210 4000ee00`001010f0
kd> eq fffff807`0496e218 1
kd> !idt 21

Dumping IDT: fffff8070496e000

21:	00000001400010f0 
```

这里出现了问题，我的win10是无法进入中断的，也就是说无法执行内核代码，可以看后面的问题记录，下面的内容就记录一下

好没问题，是我自己写得有问题

接下来继续实验，将gdtr的数据读出来，然后并在终端中打印

这里我们首先需要考虑SMAP和SMEP

SMAP和SMEP主要是内核层不能访问用户层的代码和数据，也不能执行

具体实现是cr4寄存器

![image-20220207173518403](x64%20lab/image-20220207173518403.png)

解决方法就是直接windbg中改，或者硬编码一下

```
0f 01 cb 	-->		stac
```

具体解释看这里

![image-20220208003416669](x64%20lab/image-20220208003416669.png)

我们可以设置AC标志位，然后就会略过检查。

**这里我们读取的数据还是有限的，是什么意思，就是我们无法读取.text段的数据，具体可以自己去检验**

可算是成功了，调了很长时间，CE，x32，windbg都上来，发现还是windbg最好调，x32比较适合用户层。

windbg具体调试就是开一个进程，然后断下来，这里并不是int3断点，而是让程序一直执行，可以用`system("pause")`。

直接挂载到进程上去调。

代码如下

**smap1.exe**

```c
#include<stdio.h>
#include<Windows.h>
#include<stdlib.h>



extern "C" void idtEntry();
extern "C" void go();
extern "C" ULONG64 x;
ULONG64 x;
//idtr fffff80060075000
//
void main()
{
	if ((ULONG64)idtEntry != 0x00000001400010f0) {
		printf("[+] idtEntry is %p\n", idtEntry);
		system("pause");
		exit(-1);
	}
	system("pause");
	go();
	printf("[+] The information is %p\n", x);
	system("pause");
}
```

```asm
option casemap:none


.data

EXTERN x:qword
PUBLIC idtEntry
PUBLIC go
.code
idtEntry proc
		mov rax, [0fffff8054c677fc0h]
		db 0fh, 01h, 0cbh
		mov x, rax
		iretq
idtEntry endp

go proc
	int 21h
	ret
go endp
end
```



![image-20220208004012045](x64%20lab/image-20220208004012045.png)

视频中还说了一个问题就是有些段是读不出来的，但我这里实验了是读出来了

![image-20220208005520199](x64%20lab/image-20220208005520199.png)

这里页表没有隔离？等下一节

我的AMD开不了KPTI，我是废物

![image-20220208005853098](x64%20lab/image-20220208005853098.png)

好像windbg preview的!pte还不能用

# 0x03 Lab 03 分页

## 1. goal

1. 理解四级页表
2. 能够编写内核模块去查找PTE等表项



## 2. step

四级页表，具体表现如下

一个表项是占8字节

```
PXE	-> 	PPE	->	PDE	->	PTE	->物理页面
9	    9	   9		9		12
```

![image-20220208141645605](x64%20lab/image-20220208141645605.png)

1个PXE管理512G

1个PPE管理1G

1个PDE管理2M

1个PTE管理4k

接下来我们实际跟一个就知道了

**以gdt为例**

```windbg
kd> dq gdtr
fffff803`33a77fb0  00000000`00000000 00000000`00000000
fffff803`33a77fc0  00209b00`00000000 00409300`00000000
fffff803`33a77fd0  00cffb00`0000ffff 00cff300`0000ffff
fffff803`33a77fe0  0020fb00`00000000 00000000`00000000
fffff803`33a77ff0  33008ba7`60000067 00000000`fffff803
fffff803`33a78000  0040f300`00003c00 00000000`00000000
fffff803`33a78010  00000000`00000000 00000000`00000000
fffff803`33a78020  00000000`00000000 00000000`00000000
kd> .formats fffff803`33a77fb0
Evaluate expression:
  Hex:     fffff803`33a77fb0
  Decimal: -8782341505104
  Octal:   1777777600146351677660
  Binary:  11111111 11111111 11111000 00000011 00110011 10100111 01111111 10110000
  Chars:   ....3..
  Time:    ***** Invalid FILETIME
  Float:   low 7.79977e-008 high -1.#QNAN
  Double:  -1.#QNAN


1 1111000 0		-->		1f0
0 000011 00		-->		00c
1 10011 101		-->		19d
0 0111 0111		-->		077
1111 1011 0000	-->		fb0

PXE
kd> !dq 0000000123a49000+1f0*8
#123a49f80 00000000`04909063 00000000`00000000
#123a49f90 00000000`00000000 00000000`00000000
#123a49fa0 00000000`00000000 00000000`00000000
#123a49fb0 00000000`00000000 00000000`00000000
#123a49fc0 00000000`00000000 0a000001`067fc863
#123a49fd0 00000000`00000000 00000000`00000000
#123a49fe0 00000000`00000000 00000000`00000000
#123a49ff0 00000000`00000000 00000000`04b25063

PPE
kd> !dq 04909000+c*8
# 4909060 00000000`0490a063 00000000`00000000
# 4909070 00000000`00000000 00000000`00000000
# 4909080 00000000`00000000 00000000`00000000
# 4909090 00000000`00000000 00000000`00000000
# 49090a0 00000000`00000000 00000000`00000000
# 49090b0 00000000`00000000 00000000`00000000
# 49090c0 00000000`00000000 00000000`00000000
# 49090d0 00000000`00000000 00000000`00000000

PDE
kd> !dq 490a000+19d*8
# 490ace8 00000000`04b24063 0a000001`3cd38863
# 490acf8 0a000000`04c84863 0a000000`04c85863
# 490ad08 0a000000`04c86863 0a000000`00245863
# 490ad18 0a000000`00246863 0a000001`0131a863
# 490ad28 0a000001`0131b863 0a000001`0131c863
# 490ad38 0a000001`0131d863 0a000001`00d1e863
# 490ad48 0a000001`00d1f863 0a000001`00d20863
# 490ad58 00000000`00000000 00000000`00000000

PTE
kd> !dq 04b24000+77*8
# 4b243b8 89000000`06877963 89000000`06878963
# 4b243c8 89000000`06879963 89000000`0687a963
# 4b243d8 00000000`00000000 89000000`0687c963
# 4b243e8 89000000`0687d963 89000000`0687e963
# 4b243f8 89000000`0687f963 89000000`06880963
# 4b24408 89000000`06881963 00000000`00000000
# 4b24418 89000000`06883963 89000000`06884963
# 4b24428 89000000`06885963 89000000`06886963

physical address
kd> !dq 06877000+fb0
# 6877fb0 00000000`00000000 00000000`00000000
# 6877fc0 00209b00`00000000 00409300`00000000
# 6877fd0 00cffb00`0000ffff 00cff300`0000ffff
# 6877fe0 0020fb00`00000000 00000000`00000000
# 6877ff0 33008ba7`60000067 00000000`fffff803
# 6878000 0040f300`00003c00 00000000`00000000
# 6878010 00000000`00000000 00000000`00000000
# 6878020 00000000`00000000 00000000`00000000

```

可以看到真实的物理地址

---

在32位中，我们会把pte放在虚拟地址里面，64位也不例外，但64位的pte不会像32位固定在0xc0000000，而是变化的，具体怎么看呢？

```
kd> !pte 0
                                           VA 0000000000000000
PXE at FFFF81C0E0703000    PPE at FFFF81C0E0600000    PDE at FFFF81C0C0000000    PTE at FFFF818000000000
```

那么我们怎么算任意地址的pte呢？

```
pte = ((addr&0xffffffffffff >> 12) << 3) + g_pte_base

kd> !pte fffff803`33a77fb0
                                           VA fffff80333a77fb0
PXE at FFFF81C0E0703F80    PPE at FFFF81C0E07F0060    PDE at FFFF81C0FE00CCE8    PTE at FFFF81FC0199D3B8
contains 0000000004909063  contains 000000000490A063  contains 0000000004B24063  contains 8900000006877963
pfn 4909      ---DA--KWEV  pfn 490a      ---DA--KWEV  pfn 4b24      ---DA--KWEV  pfn 6877      -G-DA--KW-V

kd> ? ((f803`33a77fb0 >> c) << 3) + FFFF818000000000
Evaluate expression: -138555618110536 = ffff81fc`0199d3b8
```

c代码

```
PULONG64 GetPteAddress(PVOID addr){
	return (PULONG64)(((((ULONG64)addr & 0xffffffffffff)>>12)<<3) + g_pte_base);
}
```

其他PDE和PXE已经PPE呢？

首先我们需要知道的是他们的基址

这里给出了一个很实用的方法

1. 指向PTE所在物理页的PTE是PDE
2. 指向PDE所在物理页的PTE是PPE
3. 指向PPE所在物理页的PTE是PXE

具体看个例子

```
kd> !pte 0
                                           VA 0000000000000000
PXE at FFFF81C0E0703000    PPE at FFFF81C0E0600000    PDE at FFFF81C0C0000000    PTE at FFFF818000000000
contains 0A00000121D64867  contains 0000000000000000
pfn 121d64    ---DA--UWEV  contains 0000000000000000
kd> ? (((FFFF818000000000 & 0xffffffffffff) >> c) << 3) + (FFFF818000000000)
Evaluate expression: -138810121781248 = ffff81c0`c0000000

pde = (pte >> 12) << 3) + g_pde_base)
```

接下来我们就可以得到其他的

```
pte = (((addr & 0xffffffffffff) >> 12) << 3) + (g_pte_base)
g_pde_base = (((pte & 0xffffffffffff) >> 12) << 3) + (g_pte_base)
g_ppe_base = (((pde & 0xffffffffffff) >> 12) << 3) + (g_pte_base)
g_pxe_base = (((ppe & 0xffffffffffff) >> 12) << 3) + (g_pte_base)
```

最后就是求各个地址的PDE PPE和PXE

```
kd> !pte fffff803`33a77fb0
                                           VA fffff80333a77fb0
PXE at FFFF81C0E0703F80    PPE at FFFF81C0E07F0060    PDE at FFFF81C0FE00CCE8    PTE at FFFF81FC0199D3B8
contains 0000000004909063  contains 000000000490A063  contains 0000000004B24063  contains 8900000006877963
pfn 4909      ---DA--KWEV  pfn 490a      ---DA--KWEV  pfn 4b24      ---DA--KWEV  pfn 6877      -G-DA--KW-V

kd> ?(((fffff803`33a77fb0 & 0xffffffffffff) >> 15) << 3) + (FFFF81C0C0000000)
Evaluate expression: -138809081541400 = ffff81c0`fe00cce8
```

公式

```
pte = (((addr & 0xffffffffffff) >> 12) << 3) + (g_pte_base)
g_pde_base = (((pte & 0xffffffffffff) >> 12) << 3) + (g_pte_base)
g_ppe_base = (((pde & 0xffffffffffff) >> 12) << 3) + (g_pte_base)
g_pxe_base = (((ppe & 0xffffffffffff) >> 12) << 3) + (g_pte_base)
pde = (((addr & 0xffffffffffff) >> 21) << 3) + (g_pde_base)
ppe = (((addr & 0xffffffffffff) >> 30) << 3) + (g_ppe_base)
pxe = (((addr & 0xffffffffffff) >> 39) << 3) + (g_pxe_base)
```

**最后最重要的就是如何找pte_base，视频中给出了直接搜内存**

```
kd> s -q fffff803`2d600000 l100000 FFFF818000000000
fffff803`2d81c0f8  ffff8180`00000000 4819e0c1`48c08b49
fffff803`2d821290  ffff8180`00000000 0008e1b8`19e6c149
fffff803`2d821ac0  ffff8180`00000000 4d8366ff`fff74fe9
fffff803`2d82c580  ffff8180`00000000 ffffc7c7`49f10348
fffff803`2d833860  ffff8180`00000000 5b75c085`48c28b48
fffff803`2d8385d8  ffff8180`00000000 00002025`348b4c65
fffff803`2d838e00  ffff8180`00000000 38244c8b`48c08b48

kd> u fffff803`2d81c0f2
nt!MiFillSystemPtes+0x462:
fffff803`2d81c0f2 48c1e119        shl     rcx,19h
fffff803`2d81c0f6 49b8000000008081ffff mov r8,0FFFF818000000000h
fffff803`2d81c100 498bc0          mov     rax,r8
fffff803`2d81c103 48c1e019        shl     rax,19h
fffff803`2d81c107 482bc8          sub     rcx,rax
fffff803`2d81c10a 48c1f910        sar     rcx,10h
fffff803`2d81c10e 498bc0          mov     rax,r8
fffff803`2d81c111 483bc8          cmp     rcx,rax

kd> ? fffff803`2d81c0f8-fffff803`2d600000
Evaluate expression: 2212088 = 00000000`0021c0f8
```

可行的办法就是找到具体函数然后遍历找到硬编码

最后我们知道了偏移就需要找到ntbase

可以用ZwQuerySystemInformation

```c
#include "ntddk.h"
#include <WinDef.h>

ULONG64 NT_BASE;
ULONG64 PTE_BASE;
ULONG64 PDE_BASE;
ULONG64 PXE_BASE;
ULONG64 PPE_BASE;
typedef enum
{
    MmTagTypeQS = 'M1KE'             //PrintAllLoadedMoudleByDriverSection
}MmTagType;

typedef enum _SYSTEM_INFORMATION_CLASS
{
    SystemModuleInformation = 11
} SYSTEM_INFORMATION_CLASS;

typedef struct _SYSTEM_MODULE_INFORMATION_ENTRY64 {
    ULONG Reserved[4];
    PVOID Base;
    ULONG Size;
    ULONG Flags;
    USHORT Index;
    USHORT Unknown;
    USHORT LoadCount;
    USHORT ModuleNameOffset;
    CHAR ImageName[256];
} SYSTEM_MODULE_INFORMATION_ENTRY64, * PSYSTEM_MODULE_INFORMATION_ENTRY64;

EXTERN_C NTSTATUS  ZwQuerySystemInformation(
    IN SYSTEM_INFORMATION_CLASS SystemInformationClass,
    OUT PVOID SystemInformation,
    IN ULONG SystemInformationLength,
    OUT PULONG ReturnLength);
typedef struct _SYSTEM_MODULE_INFORMATION
{
    ULONG Count;//内核中以加载的模块的个数
#ifdef _AMD64_
    SYSTEM_MODULE_INFORMATION_ENTRY64 Module[1];
#else
    SYSTEM_MODULE_INFORMATION_ENTRY32 Module[1];
#endif

} SYSTEM_MODULE_INFORMATION, * PSYSTEM_MODULE_INFORMATION;
PVOID PrintAllLoadedMoudleByZwQuerySystemInformation()
{
    ULONG ulInfoLength = 0;
    PVOID pBuffer = NULL;
    NTSTATUS ntStatus = STATUS_UNSUCCESSFUL;
    do
    {
        ntStatus = ZwQuerySystemInformation(SystemModuleInformation,
            NULL,
            NULL,
            &ulInfoLength);
        if ((ntStatus == STATUS_INFO_LENGTH_MISMATCH))
        {
            pBuffer = ExAllocatePoolWithTag(PagedPool, ulInfoLength, MmTagTypeQS);
            if (pBuffer == NULL)
            {
                DbgPrint("[*] Allocate Memory Failed\r\n");
                break;
            }
            ntStatus = ZwQuerySystemInformation(SystemModuleInformation,
                pBuffer,
                ulInfoLength,
                &ulInfoLength);
            if (!NT_SUCCESS(ntStatus))
            {
                DbgPrint("[*] ZwQuerySystemInformation Failed\r\n");
                break;
            }

            PSYSTEM_MODULE_INFORMATION pModuleInformation = (PSYSTEM_MODULE_INFORMATION)pBuffer;
            if (pModuleInformation)
            {
                DbgPrint("[+] Image:%-20s Base:0x%p\r\n",
                    pModuleInformation->Module[0].ImageName, pModuleInformation->Module[0].Base);
                NT_BASE = (ULONG64)pModuleInformation->Module[0].Base;
                
            }
        }
    } while (FALSE);

    if (pBuffer)
    {
        ExFreePoolWithTag(pBuffer, MmTagTypeQS);
    }
    return STATUS_SUCCESS;
}

PULONG64 GetPxeAddr(PVOID64 addr) {
    return (PULONG64)(((((ULONG64)addr & 0xffffffffffff) >> 39) << 3) + (PXE_BASE));
}

PULONG64 GetPpeAddr(PVOID64 addr) {
    return (PULONG64)(((((ULONG64)addr & 0xffffffffffff) >> 30) << 3) + (PPE_BASE));
}
PULONG64 GetPdeAddr(PVOID64 addr) {
    return (PULONG64)(((((ULONG64)addr & 0xffffffffffff) >> 21) << 3) + (PDE_BASE));
}
PULONG64 GetPteAddr(PVOID64 addr) {
    return (PULONG64)(((((ULONG64)addr & 0xffffffffffff) >> 12) << 3) + (PTE_BASE));
}

EXTERN_C NTSTATUS DriverEntry(
    PDRIVER_OBJECT DriverObject,
    PUNICODE_STRING RegistryPath
)
{
    if (RegistryPath != NULL) {
        DbgPrint("[%ws]Driver RegistryPath:%wZ\n", __FUNCTIONW__, RegistryPath);
    }
    if (DriverObject != NULL) {
        DbgPrint("[%ws]Driver Object Address:%p\n", __FUNCTIONW__, DriverObject);
    }
    DbgPrint("[*] Enter...\n");
    PrintAllLoadedMoudleByZwQuerySystemInformation();

    PTE_BASE = *(PULONG64)(NT_BASE + 0x21c0f8);
    PDE_BASE = (ULONG64)GetPteAddr((PVOID64)PTE_BASE);
    PPE_BASE = (ULONG64)GetPteAddr((PVOID64)PDE_BASE);
    PXE_BASE = (ULONG64)GetPteAddr((PVOID64)PPE_BASE);
    DbgPrint("[+] PTE_BASE: %p\n", (PVOID64)PTE_BASE);
    DbgPrint("[+] PDE_BASE: %p\n", (PVOID64)PDE_BASE);
    DbgPrint("[+] PPE_BASE: %p\n", (PVOID64)PPE_BASE);
    DbgPrint("[+] PXE_BASE: %p\n", (PVOID64)PXE_BASE);
    DbgPrint("[+] PXE: %p\n", GetPxeAddr((PVOID)(0xfffff8054c677fb0)));
    DbgPrint("[+] PPE: %p\n", GetPpeAddr((PVOID)(0xfffff8054c677fb0)));
    DbgPrint("[+] PDE: %p\n", GetPdeAddr((PVOID)(0xfffff8054c677fb0)));
    DbgPrint("[+] PTE: %p\n", GetPteAddr((PVOID)(0xfffff8054c677fb0)));
    return STATUS_SUCCESS;
}
```

![image-20220208232918478](x64%20lab/image-20220208232918478.png)

```
kd> !pte 0
                                           VA 0000000000000000
PXE at FFFFF6FB7DBED000    PPE at FFFFF6FB7DA00000    PDE at FFFFF6FB40000000    PTE at FFFFF68000000000
contains 8A00000002987867  contains 0000000000000000
pfn 2987      ---DA--UW-V  contains 0000000000000000
not valid
kd> !pte 0xfffff8054c677fb0
                                           VA fffff8054c677fb0
PXE at FFFFF6FB7DBEDF80    PPE at FFFFF6FB7DBF00A8    PDE at FFFFF6FB7E015318    PTE at FFFFF6FC02A633B8
contains 0000000004909063  contains 000000000490A063  contains 0000000004B24063  contains 8900000006877963
pfn 4909      ---DA--KWEV  pfn 490a      ---DA--KWEV  pfn 4b24      ---DA--KWEV  pfn 6877      -G-DA--KW-V

```

好家伙终于弄好了，主要是网上找遍历模块找得时间挺长的，**pte0.sys**

# 0x04 Lab 04 KPTI

## 1. goal

1. 理解KPTI
2. 完成内核环境的初始化

## 2. step

先来说一下KPTI，**本质上就是用户和内核用两个页表，用不同的cr3，如下图所示**

KPTI 全称叫做内核页表隔离。原理是在内存页面映射中进行处理。剔除了普通进程中的内核空间页表映射，只映射很小一部分内核与应用层交互必须的内存空间。使得即使成功利用也无法读取更多敏感数据。**并且在系统调用/中断处理中修改入口地址，添加KVAS段，来做相应的对接工作**。

![image-20220210162957599](x64%20lab/image-20220210162957599.png)

我们先写一个代码来看一下现在我们中断提权的cr3是多少

![image-20220210173514116](x64%20lab/image-20220210173514116.png)

```
kd> r cr3
cr3=000000001df30000
```

实际上两个cr3是不一样的，但我是AMD，好像是开不了KPTI的

后面就研究不下去了....😂😂

接下来就看一下过程把，有些原理还是得研究清楚的

**从上面那个图我们可以看到，KVASCODE是同一块内存映射，所以从用户转换到内核说明就通过这一块区域**

![image-20220210205402318](x64%20lab/image-20220210205402318.png)

我们先来看上面这个，可以看到其中逻辑

1. 先切换gs
2. 然后判断shadowflags
3. 之后转变rsp指针，以及cr3寄存器，也就是将cr3赋值成内核cr3
4. 之后就把rsp弄成一个内核rsp
5. 最后就是将内容存到idtbase+tablestate（0x4200）的位置

![image-20220210210013683](x64%20lab/image-20220210210013683.png)

之后就是进入正常的内核区域

```
swapgs
mov     rsp, gs:[9000h]
mov     cr3, rsp
mov     rsp, gs:[9008h]
mov     gs:[10h], rsi
mov     rsi, gs:[38h]
add     rsi, 4200h
push    qword ptr [rsi-8]
push    qword ptr [rsi-10h]
push    qword ptr [rsi-18h]
push    qword ptr [rsi-20h]
push    qword ptr [rsi-28h]
mov     rsi, gs:10h
and     qword ptr gs:[10h], 0
iretq
```

这一节姑且研究到这吧，实在是弄不了

# 0x05 Lab 05 CFG

# Some question

## 硬编码

```
//32位转换64位
ea 09709a77 3300		--> 		jmp far 33:779a7009
//64位跳转32位
48 ff 28				--> 		jmp ds:[rax]
//不检测smap
0f 01 cb 				--> stac
```

## 自己构建的IDT表无法跳转

pg应该不是问题所在，经过测试后发现int 4是可以中断的，说明中断是发出去了，但为什么不会走0x21自己构建的IDT表呢？

后来测试发现自己构造DPL时写错了，我是废物。

## windbg preview的!pte不能用

出现什么Levels not implemented....

不知道什么情况

但windbg旧版可以用，真的神奇

## 数据长度问题

ULONG64和PVOID64以及PULONG64做运算时好像是会出问题

需要看清楚数据长度，记住一定要加括号

## 如何手动开启KPTI

[How to enable KvaShadow(KPTI) manually? 如何手动开启内核页表隔离？ (microsoft.com)](https://social.technet.microsoft.com/Forums/ie/zh-CN/7fc82174-0cd9-4690-b043-6ba3285fbcd9/how-to-enable-kvashadowkpti-manually?forum=winserver8zhcn)

[A Deep Dive Analysis of Microsoft’s Kernel Virtual Address Shadow Feature (fortinet.com)](https://www.fortinet.com/blog/threat-research/a-deep-dive-analysis-of-microsoft-s-kernel-virtual-address-shadow-feature)

[Windows 10 KVAS and Software SMEP - wumb0in'](https://wumb0.in/windows-10-kvas-and-software-smep.html)

amd好像是开启不了KPTI的

```
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v FeatureSettingsOverride /t REG_DWORD /d 0 /f
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v FeatureSettingsOverrideMask /t REG_DWORD /d 3 /f

Install-Module SpeculationControl
Import-Module SpeculationControl
Set-ExecutionPolicy RemoteSigned -Scope Currentuser
Get-SpeculationControlSettings
```

![image-20220210203315138](x64%20lab/image-20220210203315138.png)
