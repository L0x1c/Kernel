[toc]
# 保护模式

## 段寄存器介绍

当我们用汇编写某一个地址的时候：`mov dword ptr ds:[0x123456],eax`我们真正读写的地址是：ds.base + 0x123456

段寄存器ES CS SS DS FS GS LDTR TR 一共是8个，因为之前看的16位的时候知道一些，CS是code代码base，DS是data数据base，SS是stack栈的base，实模式的时候是没有**段选择子**的，之前是base*16 + 地址，因为有20根线，因为有些地址并不能取到所以运用了这个方式取，但是进入了保护模式也不能将这些段寄存器丢掉所以就有了段选择子的概念

![image-20210315213038267](windows内核/image-20210315213038267.png)

像这些ES，CS，SS，DS...都是

我们用里面的DS开始举例子，DS = 0x2B这个只是段选择，段寄存器一共是96位

![image-20210315213714054](windows内核/image-20210315213714054.png)

```
struct SegMent
{
	WORD Selector;	//16位的Selector
	WORD Attribute;	//16位的Attribute
	DWORD Base;	//32位的Base
	DWORD Limit;	//32位的Limit
}
```

0x2B拆解：0010 1 011 后面的两位代表了 Requested Privilege Level(RPL)，后面的两位代表的是请求的特权级别在几环 11 就是在3环，倒数第三位代表了 0 -> 查询GDT表  1 -> 查询ldt表

![image-20210315214803678](windows内核/image-20210315214803678.png)

因为gtr表的宽度是8所以要乘8

```
gdt = gdtr + index * 8
```

解释GDT表和LDT表比较好的图：

LDT的话索引是在LDT表中找，首先之前是要lldt的，然后再找ldtr，再索引

![image-20210315221409588](windows内核/image-20210315221409588.png)

火哥这里取出来的数据是00cff300-0000ffff

```
base: 00000000
Attribute: 0cf3
limit: fffffffff
```

![image-20210315221920971](windows内核/image-20210315221920971.png)

G位这里是1，代表粒度，1代表4K，0代表字节    C -> 1100进行拆分，因为G位是1，所以段限长：(0xfffff+1) * 0x1000 - 1 = ffffffff，因为有一个地方没用到所以Attribute补上0，因为段限长的3个字节补齐，以及Attribute的一个0的补齐，所以 是增加了16位，所以我们看到的是80位，但是真正是96位的原因

在整个系统中，全局描述符表GDT只有一张(一个处理器对应一个GDT)，GDT可以被放在内存的任何位置，但CPU必须知道GDT的入口，也就是基地址放在哪里，Intel的设计者门提供了一个寄存器GDTR用来存放GDT的入口地址，程序员将GDT设定在内存中某个位置之后，可以通过LGDT指令将GDT的入口地址装入此寄存器，从此以后，CPU就根据此寄存器中的内容作为GDT的入口来访问GDT了。GDTR中存放的是GDT在内存中的基地址和其表长界限



自己测试一下：

![image-20210316185902834](windows内核/image-20210316185902834.png)

可以看到 ds = 0x2B -> 00101 0 11 说明了现在是在3环查gdt表索引是5

![image-20210316190418766](windows内核/image-20210316190418766.png)

可以看到gdtr表的地址是0xfffff80000b95000，长度为0x7f

![image-20210316190856273](windows内核/image-20210316190856273.png)

看到的是 00cff300 0000ffff -> base = 0x00000000 Attribute = 0x0cf3 limit = 0xffffffff 

## 段寄存器探测

![image-20210316192908915](windows内核/image-20210316192908915.png)

数据可以不一样的，红色的部分是我们能看到，但是如何证明剩下的这些是存在的

可以来写代码来证明一下这个是存在的，自己本机的段寄存器的数值

![image-20210316202905410](windows内核/image-20210316202905410.png)

用代码实现看看权限是否存在就可以将段寄存器复制之后，查看是否报错，就知道是否可以读写

```c
	int var = 0;
00411838  mov         dword ptr [ebp-0Ch],0  
	__asm
	{
		mov ax,ss
0041183F  mov         ax,ss  
		mov ds,ax
00411842  mov         ds,ax  
		mov dword ptr ds:[var],1
00411845  mov         dword ptr [ebp-0Ch],1  
	}
	printf("%d", var);
0041184C  mov         eax,dword ptr [ebp-0Ch]  
0041184F  push        eax  
```

cs的时候会触发异常

![image-20210316203551474](windows内核/image-20210316203551474.png)

判断段寄存器是否存在段的base

```c
#include <stdio.h>


void main()
{
	__asm
	{
		xor eax, eax
		xor ebx, ebx
		mov bx, ds
		mov ax, fs
		mov es, ax
		mov ebx,dword ptr ds:[0xBA3000]
		mov eax,es:[0]

	}
}
```

测试是否存在limit的属性

```c
#include <stdio.h>


void main()
{
	int var = 0;
	__asm
	{
		mov ax, fs;
		mov es, ax;
		mov eax, dword ptr es : [0xffc] ;
		mov var, eax;
	}
	printf("%x", var);
}
```

如果是fs:[0xfff] 我们就多出了三个fff  ffe ffd这三个字节，因为给eax是4个字节给的，所以多了三个，每次给的时候应该是4倍数开始取，一直到limit的结束

## 段描述符与段选择子

![image-20210318232334083](windows内核/image-20210318232334083.png)

LES( load ES)指令是把内存中指定位置的双字操作数的低位字装入指令中指定的寄存器、高位字装入ES寄存器的功能，les一次操作了6位（为啥：internel白皮书也没有说为什么）

```c
#include <stdio.h>

int main()
{
	unsigned char buffer[6] = { 0x78,0x56,0x34,0x12,0x1B,0x00 };
	__asm
	{
		les eax, fword ptr ds : [buffer] 
		les eax, dword ptr ds : [buffer] 
	}
	return 0;
}
```

当时学从实模式到保护模式的时候，因为数据段只有一个ds去找的话比较繁琐，所以加了一个es段寄存器去找值

CPL：当前正在执行程序或任务的特权级，存放在代码段寄存器CS和堆栈段寄存器SS的最低两位中。

DPL：段或门的特权级，存放在段或门描述符的DPL字段中。

RPL：赋予段选择符的超越特权级，存放在选择符的最低两位中，访问非一致代码段时，DPL数值必须不大于CPL和RPL的最大值



当 S=1 时TYPE中的4个二进制位情况：
3 2 1 0
执行位 一致位 读写位 访问位

执行位：置1时表示可执行，置0时表示不可执行
一致位：置1时表示一致码段，置0时表示非一致码段
读写位：置1时表示可读可写，置0时表示只读
访问位：置1时表示已访问，置0时表示未访问

![image-20210318232525612](windows内核/image-20210318232525612.png)

当S位为0的时候我们的Type域可以通过下表来查看其含义

![image-20210318232652785](windows内核/image-20210318232652785.png)



```c
int var = 0;
int main(int argc,char * argv[])
{
    __asm
   {
      mov ax,0x48;修改为P=0的段 为es选择子
      mov es,ax
      mov ebx,0x1000;
      mov dword ptr es:[var],ebx; //查看结果
   }

   return 0;   
}
```





## R3读写操作系统

段描述符中的 P: 段存在位，该位为 0 表示该段不存在，为 1 表示存在

操作系统正常在R3层的时候不能去修改和读取R0层的东西，也就是高2GB的空间的东西

![image-20210320162908888](windows内核/image-20210320162908888.png)

控制的位置就是U/S位，U/S = 0的时候是超级用户（但是这里的不是指操作系统的那个特权的用户，这里是CPU的超级用户）, U/S = 1的时候代表了超级用户和普通用户都可以操作物理页，但是要满足的是PDE的U/S位和PTE的U/S位进行相与的结果是1的时候才可以对物理页进行操作

32位系统，分页模式有两种10-10-12分页和2-9-9-12分页

2 9 9 12：

![image-20210320172322822](windows内核/image-20210320172322822.png)

![image-20210320172931341](windows内核/image-20210320172931341.png)

10 10 12

![image-20210320172421600](windows内核/image-20210320172421600.png)

这里有三个表比较好看的我就直接复制下来了：PDPTE

![image-20210320181544702](windows内核/image-20210320181544702.png)

PDE

![image-20210320181621513](windows内核/image-20210320181621513.png)

PTE

![image-20210320181633870](windows内核/image-20210320181633870.png)

实验一下可以在R3读写操作系统的高2GB的内存空间

将地址：0x8003f048拆分成2 9 9 12结构

```
2: 10	2*8      
9: 000000000000	0

9: 000000111111	3f * 8

12: 000001001000 48
```

找到了CR3的地址：

![image-20210320180817722](windows内核/image-20210320180817722.png)

查找PDPT

![image-20210320181648405](windows内核/image-20210320181648405.png)

查找PDT

![image-20210320181740962](windows内核/image-20210320181740962.png)

查找PTT

![image-20210320181822250](windows内核/image-20210320181822250.png)

查找physical type

![image-20210320182139312](windows内核/image-20210320182139312.png)

如果在我们什么都没有修改的情况我们写代码去修改这个地方的内存会出现错误的因为我们在r3层并不能去读写r0的值，所以我们要改U/S位

```c
#include "stdafx.h"
#include "stdio.h"
int main(int argc, char* argv[])
{
	int *getaddress = NULL;
	getchar();
	getaddress = (int*)0x8003f048;
	*getaddress = 2;
	printf("%x,%x",getaddress,*getaddress);
	return 0;
}
```

![image-20210320182421533](windows内核/image-20210320182421533.png)

修改U/S为1测试：

![image-20210320182830547](windows内核/image-20210320182830547.png)

可以看到可以读写了

![image-20210320183028553](windows内核/image-20210320183028553.png)

## 段描述符属性S_TYPE

S位 = 1 代码段或者数据段描述符

= 0 系统段描述符

![image-20210321140604758](windows内核/image-20210321140604758.png)

首先11位代表的是数据段和代码段
数据段中A代表了如果被加载到段选择子里面会被置1，否则 就是0，W代表是否为可读可写，E段代表了扩展的方向，如果是0的话就是向上增长，如果是1的话就是向下增长

![image-20210321140954689](windows内核/image-20210321140954689.png)

代码段中的A同上，但是R代表了是否为可读可执行，C位代表了如果是0（非一致代码段），那么R0层不可以调用R3层的代码，同样R3也不可以调用R0，但是如果是1的话，那么R3可以调用R0，C如果是1（一致代码段），那么R0中的函数，R3可以调用

做一下实验：

差一下段描述符的表

![image-20210321142417801](windows内核/image-20210321142417801.png)

我们在0x8003f048构造一个E位为1的段00cff700`0000ffff 我们让DPL = 3 ，TYPE的E = 1，我们直接赋值给ds做一下试验

![image-20210321142718143](windows内核/image-20210321142718143.png)

ds: 01001 011 = 0x4B

这个是我们看到的结果，没有出错，我们已经把ds换成了0x4B，而且base+limit已经没有空间了，为什么不报错，是因为我们这里默认了使用了ss段，因为var在这里是局部变量使用了ss段寄存器

![image-20210321143632194](windows内核/image-20210321143632194.png)

将var设置成全局变量测试一下，就会报错

![image-20210321143903357](windows内核/image-20210321143903357.png)

那么我们试一下构造limit不是0xffffffff 把段描述符改成：00c0f700`00000001就可以让程序执行了：

![image-20210321144354835](windows内核/image-20210321144354835.png)

可以执行成功

![image-20210321144509603](windows内核/image-20210321144509603.png)

段描述符TYPE，还有一种属性是系统的：

![image-20210321144626755](windows内核/image-20210321144626755.png)

（这个后面会讲，写不了笔记啦，大概就是什么值就构造了什么门那种）

## D/B位详解

在代码段的时候，D/B位叫做D，数据段的时候叫做B

CS段的影响：

D = 1 采用32位的寻址方式，D = 0 采用16位的寻址方式，32位的时候，用前缀67（硬编码）改变寻址方式

SS段的影响：

D = 1 隐式堆栈访问指令（push，call，pop）使用了32位堆栈指针寄存器ESP D = 0 是默认使用了16位堆栈指针寄存器SP

向下拓展的数据段：

B = 1（4GB）

B = 0（64KB）

![image-20210322111205616](windows内核/image-20210322111205616.png)

如果base > limit 那么就是 base ~ base + limit （向上拓展） base + limit ~ B=(1/0) （向下拓展）

如果是base < limit 那么就是 base ~ limit （向上拓展）limit ~ B=(1/0) （向下拓展）

**这里介绍了个指令：dg**

![image-20210322112359216](windows内核/image-20210322112359216.png)



## 段权限检查

火哥这里讲的没太懂，但是自己总结了一下

![image-20210323103845641](windows内核/image-20210323103845641.png)

这里火哥的图自己总结一下，首先指令mov ds,ax要检车cs代码段的cpl是否权限大于等于cs的dpl，如果大于dpl证明这句话可以执行，然后判断ds = ax，如果ds == ax啥也不做，但是如果数据段的rpl权限大于等于dpl，且cpl权限大于等于dpl，这样就可以访问数据区域，就可以赋值成功，刷新缓存

总结权限：cpl（代码段） >= dpl（代码段）&& cpl（代码段） >= dpl （数据段）&& rpl （数据段）>= dpl（数据段）

写一下R0层的和R3层的测试一下

vs修改一下配置

![image-20210323110458498](windows内核/image-20210323110458498.png)

![image-20210323110513226](windows内核/image-20210323110513226.png)

```c
#include <ntddk.h>
#include <ntstatus.h>


VOID DriverUnload(PDRIVER_OBJECT pDriver)
{
	UNREFERENCED_PARAMETER(pDriver);
	KdPrint(("驱动卸载\n"));
}

int g_value = 0;

NTSTATUS DriverEntry(PDRIVER_OBJECT pDriver, PUNICODE_STRING pReg)
{
	UNREFERENCED_PARAMETER(pReg);
	__asm
	{
		int 3;
		mov ax, 0x4B;
		mov ds, ax;
		mov ebx, 0x64;
		mov dword ptr ds : [g_value] , ebx;
		mov ax, 0x20;
		mov ds, ax;

	}
	KdPrint(("%X\n", g_value));
	pDriver->DriverUnload = DriverUnload;
	return STATUS_SUCCESS;
}
```

![image-20210323111431405](windows内核/image-20210323111431405.png)

如果我们把 mov ax, 0x4B; 我们构造成0x48，还构造成原来的段的样子就可以通过了不会蓝屏了

![image-20210323112208912](windows内核/image-20210323112208912.png)

测试cpl

![image-20210323113619437](windows内核/image-20210323113619437.png)

```
r @寄存器 = 0 //就可以修改寄存器的值
```



## 跳转流程

这节课主要介绍了跳转的流程和一致代码段好像没啥用 - -

`JMP 0x20:0x12345678`流程：

RPL >= DPL & CPL >= DPL （0x20的DPL）然后查找GDT表，CS = 0x20，代码段 base+0x12345678

主要做实验来看一致代码段是否有用：

因为书上说的是一致代码段可以R3去调用R0的函数，但是实际上这个中间要通过很多的过程并不是直接可以调用的

```c
#include <ntddk.h>


VOID DriverUpload(PDRIVER_OBJECT pDriver)
{
	DbgPrint("upload sucessful!\n");
}


__declspec(naked) void test()
{
	__asm
	{
		int 3;
		ret;
	}
}

NTSTATUS DriverEntry(PDRIVER_OBJECT pDriver, PUNICODE_STRING pReg)
{
	DbgPrint("welcome!\n");
	DbgPrint("%x\n", test);
	return STATUS_SUCCESS;
}
```

我们构建一致代码段，看是否在r3 可否调用0环的代码

![image-20210323151153970](windows内核/image-20210323151153970.png)

![image-20210323151557951](windows内核/image-20210323151557951.png)

![image-20210323151708765](windows内核/image-20210323151708765.png)

发现并不可以

![image-20210323151740670](windows内核/image-20210323151740670.png)

一般分页模式后三位都是0的情况下是 10 10 12 分页 不是0的情况下是 2 9 9 12分页

![image-20210323153355606](windows内核/image-20210323153355606.png)

地址是f78de040

```
0xf78de040
11				3*8
1 1011 1100		1BC * 8			
0 1101 1110		DE * 8
0000 0100 0000	40
```

看一下PDPTE PDE PTE 物理页

![image-20210323154116309](windows内核/image-20210323154116309.png)

修改完：

![image-20210324110227704](windows内核/image-20210324110227704.png)

这里发现了个问题！群里问了一下！

![image-20210323161126524](windows内核/image-20210323161126524.png)

![image-20210323161141214](windows内核/image-20210323161141214.png)

小路哥哥这边说是cow机制，CopyOnWrite，写时复制

改了内存属性，系统会在你修改内存的时候，帮你复制然后拷贝一个新的物理页面出来，那时候pte的内容就不一样了

测试：

```c
// 1234.cpp : Defines the entry point for the console application.
//

#include "stdafx.h"

int var = 0;

int main(int argc, char* argv[])
{
	__asm
	{
		mov ax,0x48;
		mov ds,ax;
		mov ebx , 0x68;
		mov dword ptr ds:[var],ebx;
	}
	printf("%x\n",var);
	return 0;
}

```

`!vtop cr3 va`

![img](windows内核/BX33]]29%WP6I26NS3Z5RV4.png)

发触发了写时复制的情况

![image-20210323170116270](windows内核/image-20210323170116270.png)



## 段跳转实验-一致代码段分析

这里主要是实验了 一致代码段的问题，这里设置成一致代码段的条件之外，还需要把PDE，PTE的权限U/S改成1，这样就可以在r3层去访问r0的函数代码

实验代码：

```c
#include <stdio.h>
#include <stdlib.h>


int gupdate_value = 0;
int main(int argc,char * argv[])
{
	char buf[]={0x0,0,0,0,0x90,0};
	unsigned int value = 0;
	*((unsigned int *) &buf[0])=0xF8AD1060;
	printf("%X\n",&gupdate_value);
	system("pause");
	__asm
	{
		mov eax,0xF8AD1060;
		mov eax,[eax];
		mov value,eax;
		call fword ptr ds:[buf]
	}
	printf("%X\n",gupdate_value);
	printf("%X\n",value);
	system("pause");
	return 0;
}

```

```c
#include <ntddk.h>

VOID DriverUpload(PDRIVER_OBJECT pDriver)
{
	KdPrint(("卸载完成\n"));
}

int g_value = 10;

void  __declspec(naked) test()
{
	__asm
	{
		int 3;
		mov eax, 0x429C78; 
		mov ebx, 0x100;
		mov [eax], ebx;
		
		retf;

	}
	
	
	
}
NTSTATUS DriverEntry(PDRIVER_OBJECT pDriver, PUNICODE_STRING pReg)
{
	KdPrint(("welcome to driver world\n"));
	KdPrint(("%X\n", test));
	pDriver->DriverUnload = DriverUpload;
	return STATUS_SUCCESS;
}
```

![image-20210324135640141](windows内核/image-20210324135640141.png)

![image-20210324135621607](windows内核/image-20210324135621607.png)



## 长调用与短调用

















































