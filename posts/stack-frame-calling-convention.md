---
title: 栈帧与调用惯例
date: 2018-01-29 17:09:51
categories: 计算机系统
tags: [编译器, OS, C, ASM, ABI]
---


# 栈与栈帧

要想知道函数是怎么被调用的，需要了解栈帧和调用惯例相关知识。[俞甲子2009](https://book.douban.com/subject/3652388/) 的“**第10章 内存: 栈与堆**”对相关概念有很好的介绍。本文是对相关知识的学习笔记。

<!--more-->

<img width="600" alt="栈与栈帧布局" title="栈与栈帧布局" src="https://static.nullwy.me/stack-frame-layout-zh.png">

附注，栈帧之间的划分边界，其实有两种不一样说法。在有些资料中 [ [wikipedia](https://en.wikipedia.org/wiki/Call_stack#Structure); [俞甲子2009](https://book.douban.com/subject/3652388/) ]，callee 参数被划分在 callee 栈帧，但在 Intel 官方一些权威文档中 [ Intel ASDM, [Vol.1](https://www.intel.cn/content/www/cn/zh/architecture-and-technology/64-ia-32-architectures-software-developer-vol-1-manual.html), Ch.6; Intel [X86-psABI](https://github.com/hjl-tools/x86-psABI/wiki/X86-psABI) ]，callee 参数被划分在 caller 栈帧。

# 调用惯例

调用惯例，规定以下内容：(1) 函数参数的传递顺序和方式；(2) 栈的清理方式；(3) 名称修饰（[name mangling](https://en.wikipedia.org/wiki/Name_mangling)）。常见的 x86 调用惯例列表有：cdecl（C 语言默认）、stdcall（Win32 API 标准）、fastcall、pascal。这些调用惯例如下下表所示（更加全面的列表参见 [wikipedia](https://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions)）：

| 调用惯例 | 栈帧清理 | 参数传递 | 名称修饰 |
| --- | --- | --- | --- |
| cdel | 调用者 caller | 从右至左入栈 RTL | 下划线+函数名，如 _sum |
| stdcall | 被调用者 callee | 从右至左入栈 RTL | 下划线+函数名+@+参数字节数，如 _sum@8 |
| fastcall | 被调用者 callee | 头两参数存入寄存器 ECX 和 EDX，其余参数从右至左入栈 RTL | @+函数名+@+参数字节数，如 @sum@8 |
| pascal | 被调用者 callee | 从左至右入栈 LTR | 较为复杂，参见 pascal 文档 |

下面举例说明，cdecl 和 stdcall 两种调用惯例。

## cdecl 调用惯例

cdecl，调用者负责清理堆栈（caller clean-up），参数从右至左（Right-to-Left，RTL）压入栈。举例说明 [ [ref1](https://en.wikibooks.org/wiki/X86_Disassembly/Calling_Conventions) [ref2](https://www.codeproject.com/Articles/1388/Calling-Conventions-Demystified) ]：

``` c
// cdecl 调用惯例
int __cdecl sum(int a, int b) {
    return a + b;
}
// 调用
int c = sum(2, 3);
```

编译器生成的等价汇编代码：

``` asm
; 调用者清理堆栈（caller clean-up），参数 RTL 入栈
push 3
push 2
call _sum      ; 将返回地址压入栈, 同时 sum 的地址装入 eip
add  esp, 8    ; 清理堆栈, 两个参数占用 8 字节
```

``` asm
; sum 函数等价汇编代码
; // function prolog
push ebp  
mov  ebp, esp
; // return a + b;
mov  eax, [ebp + 12] 
add  eax, [ebp + 8]  ; 返回值规定保存在 eax
; // function epilog
mov  esp, ebp        ; 设置栈顶 esp
pop  ebp             ; 恢复 old ebp
ret                  ; 将栈中保存的返回地址装入 eip
```

## stdcall 调用惯例

stdcall，被调用者负责清理堆栈（callee clean-up），参数从右至左（Right-to-Left，RTL）压入栈。举例说明：

``` c
// stdcall 调用惯例
int __stdcall sum(int a, int b) {
    return a + b;
}
// 调用
int c = sum(2, 3);
```

编译器生成的等价汇编代码：

``` asm
; 被调用者清理堆栈（callee clean-up），参数 RTL 入栈
push 3
push 2
call _sum@8      ; 将返回地址压入栈, 同时 sum 的地址装入 eip
```

``` asm
; sum 函数等价汇编代码
; // function prolog
push ebp  
mov  ebp, esp
; // return a + b;
mov  eax, [ebp + 12] 
add  eax, [ebp + 8]  ; 返回值规定保存在 eax
; // function epilog
mov  esp, ebp        ; 设置栈顶 esp
pop  ebp             ; 恢复 old ebp
ret  8               ; 清理堆栈，并将栈中保存的返回地址装入 eip
```

# gcc 汇编代码

`hello1.c` 文件内容如下：

``` c
int __cdecl sum(int a, int b) {
    return a + b;
}

int main() {
    sum(1, 2);
    sum(3, 4);
    return 0;
}
```

生成汇编代码：

``` shell
$ gcc -m32 -S -masm=intel hello1.c -o hello1.s
$ gcc -m32 hello1.s -o hello
$ ./hello || echo $?
0
```

生成的 `hello1.s`，内容如下：

``` asm
  .section  __TEXT,__text,regular,pure_instructions
  .macosx_version_min 10, 12
  .intel_syntax noprefix
  .globl  _sum
  .p2align  4, 0x90
_sum:                                   ## @sum
## BB#0:
  push  ebp
  mov  ebp, esp
  sub  esp, 8                       ; 预先分配 8 字节栈空间，保存 2 个布局变量
  mov  eax, dword ptr [ebp + 12]    ; 堆栈中读取参数 2
  mov  ecx, dword ptr [ebp + 8]     ; 堆栈中读取参数 1
  mov  dword ptr [ebp - 4], ecx     ; 布局变量 1
  mov  dword ptr [ebp - 8], eax     ; 布局变量 2
  mov  eax, dword ptr [ebp - 4]
  add  eax, dword ptr [ebp - 8]
  add  esp, 8                       ; 清理 8 字节栈空间
  pop  ebp
  ret

  .globl  _main
  .p2align  4, 0x90
_main:                                  ## @main
## BB#0:
  push  ebp
  mov  ebp, esp
  sub  esp, 40                     ; 预先分配 40 字节栈空间
  mov  eax, 1
  mov  ecx, 2
  mov  dword ptr [ebp - 4], 0
  mov  dword ptr [esp], 1
  mov  dword ptr [esp + 4], 2
  mov  dword ptr [ebp - 8], eax ## 4-byte Spill
  mov  dword ptr [ebp - 12], ecx ## 4-byte Spill
  call  _sum
  mov  ecx, 3
  mov  edx, 4
  mov  dword ptr [esp], 3
  mov  dword ptr [esp + 4], 4
  mov  dword ptr [ebp - 16], eax ## 4-byte Spill
  mov  dword ptr [ebp - 20], ecx ## 4-byte Spill
  mov  dword ptr [ebp - 24], edx ## 4-byte Spill
  call  _sum
  xor  ecx, ecx
  mov  dword ptr [ebp - 28], eax ## 4-byte Spill
  mov  eax, ecx
  add  esp, 40                  ; 清理 40 字节栈空间
  pop  ebp
  ret

.subsections_via_symbols
```

GCC 生成的汇编代码并没有使用 `push` 而是通过 `sub  esp, 40` 直接预先分配栈空间，然后使用 `mov` 指令将参数写进栈中，清理栈使用 `add  esp, 40`。逻辑上，还是符合 cdecl 调用惯例，调用者负责清理堆栈（caller clean-up），参数从右至左（Right-to-Left，RTL）压入栈。这样做的好处是，如果同时多次调用 `sum`，清理栈空间的指令，只需要最后的时候调用一次就可以了。统一使用 `sub esp` 和 `add esp` 去操作 esp 值，避免 `push` 指令操作 esp。

现在再来看看，stdcall 调用惯例下，GCC 生成的汇编代码。把 `sum` 函数改为 `__stdcall`，运行下面的命令：

``` shell
$ gcc -m32 -S -masm=intel hello2.c -o hello2.s
$ diff -C1 hello1.s hello2.s
```

``` diff
*** hello1.s    Thu Feb 05 22:43:59 2018
--- hello2.s    Thu Feb 05 22:45:24 2018
***************
*** 18,20 ****
    pop  ebp
!   ret
  
--- 18,20 ----
    pop  ebp
!   ret  8
  
***************
*** 35,36 ****
--- 35,37 ----
    call  _sum
+   sub  esp, 8
    mov  ecx, 3
***************
*** 43,44 ****
--- 44,46 ----
    call  _sum
+   sub  esp, 8
    xor  ecx, ecx
```

# 反汇编代码

反汇编 `objdump`、`gdb`/`lldb`，或者商业工具使用，IDA Pro 或者 Hopper Disassembler [ [wiki](https://en.wikibooks.org/wiki/X86_Disassembly/Disassemblers_and_Decompilers#x86_Disassemblers) ]

## objdump 反汇编

```
$ gobjdump -d -Mintel hello1                # 使用 GNU objdump
$ objdump -d -x86-asm-syntax=intel hello1   # 使用 llvm-objdump
```

``` asm
hello1:	file format Mach-O 32-bit i386

Disassembly of section __TEXT,__text:
__text:
    1f30:	55 	push	ebp
    1f31:	89 e5 	mov	ebp, esp
    1f33:	83 ec 08 	sub	esp, 8
    1f36:	8b 45 0c 	mov	eax, dword ptr [ebp + 12]
    1f39:	8b 4d 08 	mov	ecx, dword ptr [ebp + 8]
    1f3c:	89 4d fc 	mov	dword ptr [ebp - 4], ecx
    1f3f:	89 45 f8 	mov	dword ptr [ebp - 8], eax
    1f42:	8b 45 fc 	mov	eax, dword ptr [ebp - 4]
    1f45:	03 45 f8 	add	eax, dword ptr [ebp - 8]
    1f48:	83 c4 08 	add	esp, 8
    1f4b:	5d 	pop	ebp
    1f4c:	c3 	ret
    1f4d:	0f 1f 00 	nop	dword ptr [eax]
    1f50:	55 	push	ebp
    1f51:	89 e5 	mov	ebp, esp
    1f53:	83 ec 28 	sub	esp, 40
    1f56:	b8 01 00 00 00 	mov	eax, 1
    1f5b:	b9 02 00 00 00 	mov	ecx, 2
    1f60:	c7 45 fc 00 00 00 00 	mov	dword ptr [ebp - 4], 0
    1f67:	c7 04 24 01 00 00 00 	mov	dword ptr [esp], 1
    1f6e:	c7 44 24 04 02 00 00 00 	mov	dword ptr [esp + 4], 2
    1f76:	89 45 f8 	mov	dword ptr [ebp - 8], eax
    1f79:	89 4d f4 	mov	dword ptr [ebp - 12], ecx
    1f7c:	e8 af ff ff ff 	call	-81 <_sum>
    1f81:	b9 03 00 00 00 	mov	ecx, 3
    1f86:	ba 04 00 00 00 	mov	edx, 4
    1f8b:	c7 04 24 03 00 00 00 	mov	dword ptr [esp], 3
    1f92:	c7 44 24 04 04 00 00 00 	mov	dword ptr [esp + 4], 4
    1f9a:	89 45 f0 	mov	dword ptr [ebp - 16], eax
    1f9d:	89 4d ec 	mov	dword ptr [ebp - 20], ecx
    1fa0:	89 55 e8 	mov	dword ptr [ebp - 24], edx
    1fa3:	e8 88 ff ff ff 	call	-120 <_sum>
    1fa8:	31 c9 	xor	ecx, ecx
    1faa:	89 45 e4 	mov	dword ptr [ebp - 28], eax
    1fad:	89 c8 	mov	eax, ecx
    1faf:	83 c4 28 	add	esp, 40
    1fb2:	5d 	pop	ebp
    1fb3:	c3 	ret

_sum:
    1f30:	55 	push	ebp
    1f31:	89 e5 	mov	ebp, esp
    1f33:	83 ec 08 	sub	esp, 8
    1f36:	8b 45 0c 	mov	eax, dword ptr [ebp + 12]
    1f39:	8b 4d 08 	mov	ecx, dword ptr [ebp + 8]
    1f3c:	89 4d fc 	mov	dword ptr [ebp - 4], ecx
    1f3f:	89 45 f8 	mov	dword ptr [ebp - 8], eax
    1f42:	8b 45 fc 	mov	eax, dword ptr [ebp - 4]
    1f45:	03 45 f8 	add	eax, dword ptr [ebp - 8]
    1f48:	83 c4 08 	add	esp, 8
    1f4b:	5d 	pop	ebp
    1f4c:	c3 	ret
    1f4d:	0f 1f 00 	nop	dword ptr [eax]

_main:
    1f50:	55 	push	ebp
    1f51:	89 e5 	mov	ebp, esp
    1f53:	83 ec 28 	sub	esp, 40
    1f56:	b8 01 00 00 00 	mov	eax, 1
    1f5b:	b9 02 00 00 00 	mov	ecx, 2
    1f60:	c7 45 fc 00 00 00 00 	mov	dword ptr [ebp - 4], 0
    1f67:	c7 04 24 01 00 00 00 	mov	dword ptr [esp], 1
    1f6e:	c7 44 24 04 02 00 00 00 	mov	dword ptr [esp + 4], 2
    1f76:	89 45 f8 	mov	dword ptr [ebp - 8], eax
    1f79:	89 4d f4 	mov	dword ptr [ebp - 12], ecx
    1f7c:	e8 af ff ff ff 	call	-81 <_sum>
    1f81:	b9 03 00 00 00 	mov	ecx, 3
    1f86:	ba 04 00 00 00 	mov	edx, 4
    1f8b:	c7 04 24 03 00 00 00 	mov	dword ptr [esp], 3
    1f92:	c7 44 24 04 04 00 00 00 	mov	dword ptr [esp + 4], 4
    1f9a:	89 45 f0 	mov	dword ptr [ebp - 16], eax
    1f9d:	89 4d ec 	mov	dword ptr [ebp - 20], ecx
    1fa0:	89 55 e8 	mov	dword ptr [ebp - 24], edx
    1fa3:	e8 88 ff ff ff 	call	-120 <_sum>
    1fa8:	31 c9 	xor	ecx, ecx
    1faa:	89 45 e4 	mov	dword ptr [ebp - 28], eax
    1fad:	89 c8 	mov	eax, ecx
    1faf:	83 c4 28 	add	esp, 40
    1fb2:	5d 	pop	ebp
    1fb3:	c3 	ret
```

## lldb 反汇编

使用 lldb 反汇编：

```
$ lldb hello1
(lldb) target create "hello1"
Current executable set to 'hello1' (i386).
(lldb) settings set target.x86-disassembly-flavor intel
(lldb) disassemble --name main
hello1`main:
hello1[0x1f50] <+0>:  push   ebp
hello1[0x1f51] <+1>:  mov    ebp, esp
hello1[0x1f53] <+3>:  sub    esp, 0x28
hello1[0x1f56] <+6>:  mov    eax, 0x1
hello1[0x1f5b] <+11>: mov    ecx, 0x2
hello1[0x1f60] <+16>: mov    dword ptr [ebp - 0x4], 0x0
hello1[0x1f67] <+23>: mov    dword ptr [esp], 0x1
hello1[0x1f6e] <+30>: mov    dword ptr [esp + 0x4], 0x2
hello1[0x1f76] <+38>: mov    dword ptr [ebp - 0x8], eax
hello1[0x1f79] <+41>: mov    dword ptr [ebp - 0xc], ecx
hello1[0x1f7c] <+44>: call   0x1f30                    ; sum
hello1[0x1f81] <+49>: mov    ecx, 0x3
hello1[0x1f86] <+54>: mov    edx, 0x4
hello1[0x1f8b] <+59>: mov    dword ptr [esp], 0x3
hello1[0x1f92] <+66>: mov    dword ptr [esp + 0x4], 0x4
hello1[0x1f9a] <+74>: mov    dword ptr [ebp - 0x10], eax
hello1[0x1f9d] <+77>: mov    dword ptr [ebp - 0x14], ecx
hello1[0x1fa0] <+80>: mov    dword ptr [ebp - 0x18], edx
hello1[0x1fa3] <+83>: call   0x1f30                    ; sum
hello1[0x1fa8] <+88>: xor    ecx, ecx
hello1[0x1faa] <+90>: mov    dword ptr [ebp - 0x1c], eax
hello1[0x1fad] <+93>: mov    eax, ecx
hello1[0x1faf] <+95>: add    esp, 0x28
hello1[0x1fb2] <+98>: pop    ebp
hello1[0x1fb3] <+99>: ret

(lldb) disassemble --name sum
hello1`sum:
hello1[0x1f30] <+0>:  push   ebp
hello1[0x1f31] <+1>:  mov    ebp, esp
hello1[0x1f33] <+3>:  sub    esp, 0x8
hello1[0x1f36] <+6>:  mov    eax, dword ptr [ebp + 0xc]
hello1[0x1f39] <+9>:  mov    ecx, dword ptr [ebp + 0x8]
hello1[0x1f3c] <+12>: mov    dword ptr [ebp - 0x4], ecx
hello1[0x1f3f] <+15>: mov    dword ptr [ebp - 0x8], eax
hello1[0x1f42] <+18>: mov    eax, dword ptr [ebp - 0x4]
hello1[0x1f45] <+21>: add    eax, dword ptr [ebp - 0x8]
hello1[0x1f48] <+24>: add    esp, 0x8
hello1[0x1f4b] <+27>: pop    ebp
hello1[0x1f4c] <+28>: ret
hello1[0x1f4d] <+29>: nop    dword ptr [eax]

```




# 参考资料

1. 链接、装载与库，俞甲子，2009：第10章 内存: 栈与堆，[豆瓣](https://book.douban.com/subject/3652388/)
2. IDA Pro权威指南，Eagle，第2版2011：6.2.1 调用约定，[豆瓣](https://book.douban.com/subject/10463039/)
3. 2001-09 Calling Conventions Demystified <https://www.codeproject.com/Articles/1388/Calling-Conventions-Demystified>
4. Calling Conventions <https://en.wikibooks.org/wiki/X86_Disassembly/Calling_Conventions>
5. https://en.wikipedia.org/wiki/X86_calling_conventions


