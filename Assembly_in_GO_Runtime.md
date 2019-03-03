## Assembly in GO Runtime

### Why use assembly

-   [算法加速](https://github.com/minio/sha256-simd)，golang编译器生成的机器码基本上都是通用代码，而且 优化程度一般，远比不上C/C++的gcc/clang生成的优化程度高，毕竟时间沉淀在那里。因此通常我们需要用到特 殊优化逻辑、特殊的CPU指令让我们的算法运行速度更快，如sse4_2/avx/avx2/avx-512等。
    
-   摆脱golang编译器的一些约束，如[通过汇编调用其他package的私有函数](https://sitano.github.io/2016/04/28/golang-private/)。
    
-   进行一些hack的事，如[通过汇编适配其他语言的ABI来直接调用其他语言的函数](https://github.com/petermattis/fastcgo)。
    
-   利用//go:noescape进行内存分配优化，golang编译器拥有逃逸分析，用于决定每一个变量是分配在堆内存上 还是函数栈上。但是有时逃逸分析的结果并不是总让人满意，一些变量完全可以分配在函数栈上，但是逃逸分析将其 移动到堆上，因此我们需要使用golang编译器的[go:noescape](https://golang.org/cmd/compile/#hdr-Compiler_Directives) 将其转换，强制分配在函数栈上。当然也可以强制让对象分配在堆上，可以参见[这段实现](https://github.com/golang/go/blob/d1fa58719e171afedfbcdf3646ee574afc08086c/src/reflect/value.go#L2585-L2597)。

### Basic operation
```
LEA 和 MOV

LEA：操作地址；  
MOV：操作数据

如：  
LEAQ 8(SP), SI // argv 把 8(SP)地址放入 SI 寄存器中  
MOVQ 0(SP), DI // argc 把0(SP)内容放入 DI 寄存器中
```
### Plan9 register
- PC: program counter
	- PC counts instructions, not bytes of data
	- 2(PC) => 跳过一条指令
- Pseudo FP: frame pointer, 引數和本地變數 
	- a+0(FP) 表示函数的第一个参数；b+4(FP)表示第二个参数等
	- 取 parent caller 的 arg 和 存(回傳)  給 parent 的 return value
	- MOVQ	buf+0(FP), AX		// gobuf
- Pseudo SB: static base pointer:
	- 全域性符號取 global func or var
	- JMP	runtime·rt0_go(SB)
- Pseudo SP: stack pointer: 棧頂
	- 保存局部变量。0(SP)表示第一个局部变量，4(SP)表示第二个局部变量等
	- 存取 current function 的 local variables
	- x-8(SP) => local var 0
- Hardware SP
	- 用來存(傳遞) callee arg 和 取 callee function 的回傳 return value

### About Stack
![enter image description here](https://s3-ap-northeast-1.amazonaws.com/steven-dev/presentation/golang/stack.png)




```
                              参数及返回值大小
                                  | 
 TEXT pkgname·add(SB),NOSPLIT,$32-32
       |        |               |
      包名     函数名         栈帧大小(局部变量+可能需要的额外调用函数的参数空间的总大小，但不包括调用其它函数时的 ret address 的大小)

```
#### 底下為一實例 (main.go & output.s 皆在跟專案根目錄)

output.s:
```
#include "textflag.h"

// func output(a,b int) int
TEXT runtime·output(SB), NOSPLIT, $24-8
    MOVQ a+0(FP), DX // arg a
    LEAQ test+0(FP), DI // 測試 FP 位址
    LEAQ test+0(SP), SI // 測試 SP 位址
    MOVQ DX, 0(SP) // arg x
    MOVQ b+8(FP), CX // arg b
    MOVQ CX, 8(SP) // arg y
    CALL ·add(SB) // 在调用 add 之前，已经把参数都通过物理寄存器 SP 搬到了函数的栈顶
    MOVQ 16(SP), AX // add 函数会把返回值放在这个位置
    MOVQ AX, ret+16(FP) // return result
    RET


```
main.go:
```
package main

import (
	"fmt"
	_ "unsafe"
)

func add(x, y int) int {
	return x + y
}

//go:linkname linkOutput runtime.output
func linkOutput(a, b int) int

func main() {
	s := linkOutput(10, 13)
	fmt.Println(s)
}

```
```
$ go build
$ ./hello
$ 23
```
- output 對於 add 是 caller，對於 main 為 callee
先用 FP+offset 取記憶體 callee main func 提供的 arg a & b，然後 hardware SP 用來存塞入 call add 所要傳遞的 arg (a -> x & b -> y)，最後在調用 CALL ·add(SB)，在用 hardware SP 取 add 回傳的 int ret，最後用 FP+offset 再存到 callee func main 的 ret

### Debug with gdb
```shell
$ go build -gcflags "-N -l" main.go
$ gdb main

(gdb) source /usr/local/go/src/runtime/runtime-gdb.py
Loading Go Runtime support.
(gdb) b asm_amd64.s:15
Breakpoint 1 at 0x44e670: file /usr/local/go/src/runtime/asm_amd64.s, line 15.
(gdb) b runtime.rt0_go
(gdb) r
Starting program: /home/stevenchiu/hello/main

Breakpoint 1, _rt0_amd64 () at /usr/local/go/src/runtime/asm_amd64.s:15
15              MOVQ    0(SP), DI       // argc
(gdb) c
Continuing.

Breakpoint 2, runtime.rt0_go () at /usr/local/go/src/runtime/asm_amd64.s:89
89              MOVQ    DI, AX          // argc

```
以下為另一條 debug 的路線，來看 output.s 裡面的記憶體內容
x/6 $sp 可以看到一個基於 hardware SP 指向位址以下  4 byte * 6 的記憶體內容 [Call add args (10, 13) & return value (23) ]  
```
(gdb) b runtime.output
Breakpoint 2 at 0x484db0: file /home/stevenchiu/hello/output.s, line 4.

Thread 1 "hello" hit Breakpoint 2, runtime.output () at /home/stevenchiu/hello/output.s:4
4       TEXT runtime·output(SB), NOSPLIT, $24-8
(gdb) n
5           MOVQ a+0(FP), DX // arg a
(gdb) n
6           MOVQ DX, 0(SP) // arg x
(gdb) n
7           MOVQ b+8(FP), CX // arg b
(gdb) n
8           MOVQ CX, 8(SP) // arg y
(gdb) n
9           CALL ·add(SB) // 在调用 add 之前，已经把参数都通过物理寄存器 SP 搬到了函数的栈顶
(gdb) n
10          MOVQ 16(SP), AX // add 函数会把返回值放在这个位置
(gdb) n
11          MOVQ AX, ret+16(FP) // return result

// 這裡的 $sp 會是 hardware sp，pseudo register 無法從 $sp $fp 得到，經由 LEAQ 存到 SI DI 的結果，才可以明確知道 pueso sp fp 實際位址，如果要看的話必須存到 register 裡。
(gdb) x/20 $sp
0xc0000406e0:   10      0       13      0
0xc0000406f0:   23      0       264072  192
0xc000040700:   4738037 0       10      0
0xc000040710:   13      0       23      0
0xc000040720:   582064  192     264008  192


```
```
Real assembly
```shell
$ objdump -d hello > ori.asm
```

Intermediate assembly (Plan 9)

```shell
$ go tool objdump hello > hello.asm
