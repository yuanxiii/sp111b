# 期中作業：理解[GNU Assembler Examples](https://cs.lmu.edu/~ray/notes/gasexamples/)
## 參考GNU Assembler範例與陳鍾誠老師的系統程式課程教材，能理解部分程式碼與執行並進行註解


>1.堆疊（Stack）：一種資料結構，遵循先進後出（FILO）的原則。
>
>2.幾個組合語言指令的作用：
>- push：將資料存入堆疊中，同時將堆疊指標向低位移動一個單位。
>- pop：從堆疊中取出資料，同時將堆疊指標向高位移動一個單位。
>- call：將目前的位址壓入堆疊中，然後跳轉到某個程序。
>- ret：從堆疊中彈出資料，並跳轉到該位址。
<br>

Assembly | C 語言
-----|-------------------
push  rbp | push(rbp);
mov   rbp,rsp  | rbp = rsp;
sub   rsp,0x20 | rsp -= 0x20;
call  401710      |f_401710();
call  402c40  | f_402c40();
pop   rbp  | pop(&rbp);
ret   |  return;

對比

[表格資料來源](https://ithelp.ithome.com.tw/articles/10227112)

<br>

>GAS, the GNU Assembler, is the default assembler for the GNU Operating System.  
It works on many different architectures and supports several assembly language syntaxes.   
這些範例只能在作業系統 Linux 、 x86-64 processor 使用

## 開始 - Getting Started-hello.s
>Hello World program that uses Linux System calls, for a 64-bit installation:

```
# ----------------------------------------------------------------------------------------
# Writes "Hello, World" to the console using only system calls. Runs on 64-bit Linux only.
# To assemble and run:
#
#     gcc -c hello.s && ld hello.o && ./a.out
#
# or
#
#     gcc -nostdlib hello.s && ./a.out
# ----------------------------------------------------------------------------------------

        .global _start

        .text
_start:
        # write(1, message, 13)  和函數呼叫差不多
        mov     $1, %rax                # system call 1 is write 把1移到rax
        mov     $1, %rdi                # file handle 1 is stdout  把1移到rdi
        mov     $message, %rsi          # address of string to output 把message移到rsi
        mov     $13, %rdx               # number of bytes 把13移到rdx
        syscall                         # invoke operating system to do the write （用組合語言方式呼叫#syscall : 先把參數移到暫存器，再呼叫syscall

        # exit(0)  0=正常 非0=不正常
        mov     $60, %rax               # system call 60 is exit 把60移到rax
        xor     %rdi, %rdi              # we want return code 0 把自己跟自己xor --> 歸 0
        syscall                         # invoke operating system to exit 再呼叫syscall
        
message:
        .ascii  "Hello, world\n"

```

先組譯再連結才能執行
```
gcc -c hello.s       #編出hello.o
ld hello.o -o hello  #hello.o連結變成hello才是執行檔  
./hello              #執行
```
組譯出有格式的碼 在linux叫elf 還沒跟別的函式庫連接 ld 才會連接

#  Working with the C Library-hola.s : 
>GCC而言可以自動連接 (gcc -c hola.s 時只編譯不連接 所以把-c拿掉會自動連接(既編譯又連結) C Library
```
# ----------------------------------------------------------------------------------------
# Writes "Hola, mundo" to the console using a C library. Runs on Linux or any other system
# that does not use underscores for symbols in its C library. To assemble and run:
#
#     gcc hola.s && ./a.out
# ----------------------------------------------------------------------------------------

        .global main

        .text
main:                                   # This is called by C library's startup code
        mov     $message, %rdi          # First integer (or pointer) parameter in %rdi 這裡mov message到rdi
        call    puts                    # puts(message) 呼叫puts函數(gcc時候自動連結C函式庫)
        ret                             # Return to C library code (回傳值)
message:
        .asciz "Hola, mundo"            # asciz puts a 0 byte at the end
```
執行 hola.s :   
```
gcc -no-pie hola.s -o hola     
./hola
```
>-no-pie 代表 no position independent executable  --> 別編成與位置無關的目的檔

    call    puts      # puts(message) 呼叫puts函數(c函式)



# Calling Conventions for 64-bit C Code-fib.s (呼叫約定、呼叫慣例)

>64-bit calling conventions 在 [AMD64 ABI Reference](http://www.x86-64.org/documentation/abi.pdf) 裡有更詳細的解釋. [維基](https://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_Calling_Conventions)也有關於他的資訊. 更重要的是，給 64-bit Linux用,不是 Windows):

>以下翻譯參照參考資料使用chatGPT翻譯



參數在函數調用中按從左到右的順序依次傳遞，並盡可能存放在適合的暫存器中。整數和指針類型的參數會被分配到一組暫存器中，按優先順序為rdi、rsi、rdx、rcx、r8和r9。浮點數（float和double）類型的參數會被分配到另一組暫存器中，按順序為xmm0、xmm1、xmm2、xmm3、xmm4、xmm5、xmm6和xmm7。

參數被從右到左的順序推入堆疊中。在調用指令之後，返回地址位於%rsp（堆疊指標）的位置，第一個內存參數位於8（%rsp）的位置，依此類推。

在進行函數調用之前，堆疊指標%RSP必須按16位元邊界對齊。由於在進行函數調用時會將返回地址（8個字節）壓入堆疊，因此當函數獲得控制權時，%rsp可能未對齊。因此，你需要額外創建空間，例如通過推動某些內容或將8減去%rsp的值。

被調用的函數需要保留的暫存器（調用保存暫存器）包括rbp、rbx、r12、r13、r14和r15。其他暫存器可以由被調用的函數自由更改。被調用者還應該保存XMCSR的控制位和x87控制字，但在64位代碼中很少使用x87指令。

整數返回值存儲在rax或rdx:rax中，浮點數返回值存儲在xmm0或xmm1:xmm0中。

<Br>

```
# -----------------------------------------------------------------------------
# A 64-bit Linux application that writes the first 90 Fibonacci numbers.  It
# needs to be linked with a C library. 必須跟C函式庫連接
#
# Assemble and Link:
#     gcc fib.s
# -----------------------------------------------------------------------------

        .global main

        .text
main:
        push    %rbx                    # we have to save this since we use it (先資料存進stack)

        mov     $90, %ecx               # ecx will countdown to 0
        xor     %rax, %rax              # rax will hold the current number (rax存目前資料)
        xor     %rbx, %rbx              # rbx will hold the next number (rbx存下個)
        inc     %rbx                    # rbx is originally 1
print:
        # We need to call printf, but we are using eax, ebx, and ecx.  printf
        # may destroy eax and ecx so we will save these before the call and (先暫存起來避免衝突
        # restore them afterwards.

        push    %rax                    # caller-save register (先將caller-save register的值存入stack frame。)
        push    %rcx                    # caller-save register (先將caller-save register的值存入stack frame。)

        mov     $format, %rdi           # set 1st parameter (format) 型態範圍
        mov     %rax, %rsi              # set 2nd parameter (current_number) 現在的值
        xor     %rax, %rax              # because printf is varargs (printf是可變動函數 所以xor清0)

        # Stack is already aligned because we pushed three 8 byte registers
        call    printf                  # printf(format, current_number)

        pop     %rcx                    # restore caller-save register (把存入的值取出)
        pop     %rax                    # restore caller-save register (把存入的值取出)

        mov     %rax, %rdx              # save the current number
        mov     %rbx, %rax              # next number is now current (下個值變成當前值)
        add     %rdx, %rbx              # get the new next number
        dec     %ecx                    # count down
        jnz     print                   # if not done counting, do some more

        pop     %rbx                    # restore rbx before returning
        ret
format:
        .asciz  "%20ld\n"
```
呼叫慣例（Calling convention）定義了函式之間的互動方式，包括如何傳遞參數給函式、如何獲取函式結束後的返回值，以及哪些暫存器的內容需要在函式呼叫前後保持一致，哪些暫存器的值在函式呼叫結束後仍然需要，必須在函式呼叫前先儲存起來。如果一個函式被編譯成目標檔案（object file）後，並與其他函式進行外部連結，則必須嚴格遵守呼叫慣例，以確保與其他函式之間的正確溝通。呼叫慣例通常會區分為兩種暫存器：

Caller-save 暫存器：由呼叫者（Caller）負責清理或將其存入堆疊框架（stack frame）。
- 在呼叫者調用被呼叫者（Callee）之前，呼叫者必須先將Caller-save暫存器的值存入堆疊框架。
- 因此，被呼叫者可以直接使用Caller-save暫存器中的值，而無需將其存入堆疊框架。


>

> [Calling convention參考資料](http://redbug0314.blogspot.com/2007/06/calling-convention.html)  
  [Calling convention參考資料2](https://tclin914.github.io/77838749/)

執行 fib.s :
```
gcc -no-pie fib.s -o fib
./fib
```
<br>

# Mixing C and Assembly Language -maxofthree.s(將c跟組合語言融合) :
>  They will have been pushed on the stack so that on entry to the function, they will be in rdi, rsi, and rdx, respectively. The return value is an integer so it gets returned in rax.
>這個64位的程式非常簡單，它是一個接收三個64位整數參數並返回最大值的函式。該函式展示了如何取得整數參數：這些參數將被推入堆疊，進入函式後分別儲存在rdi、rsi和rdx暫存器中。返回值是一個整數，因此以rax暫存器的形式返回。
```
# -----------------------------------------------------------------------------
# A 64-bit function that returns the maximum value of its three 64-bit integer
# arguments.  The function has signature:
#
#   int64_t maxofthree(int64_t x, int64_t y, int64_t z)
#
# Note that the parameters have already been passed in rdi, rsi, and rdx.  We
# just have to return the value in rax.
# -----------------------------------------------------------------------------

        .globl  maxofthree
        
        .text
maxofthree:
        mov     %rdi, %rax              # result (rax) initially holds x (初始先存x(rdi))
        cmp     %rsi, %rax              # is x less than y? (x和y(rsi)進行比較)
        cmovl   %rsi, %rax              # if so, set result to y (如果y較大,將結果改變成y(rsi))
        cmp     %rdx, %rax              # is max(x,y) less than z? (和z(rdx)進行比較)
        cmovl   %rdx, %rax              # if so, set result to z (如果z較大,將結果改成z)
        ret                             # the max will be in eax (將最後結果回傳)
```
>c語言呼叫組合語言(maxofthree)的部分 (C program that calls the assembly language function ) :
>maxofthree是組合語言寫出來的
```
 /*
 * callmaxofthree.c
 *
 * A small program that illustrates how to call the maxofthree function we wrote in
 * assembly language.
 */

#include <stdio.h>
#include <inttypes.h>

int64_t maxofthree(int64_t, int64_t, int64_t);  #先定義一個函數原型才能讓他編譯 之後就會連到maxofthree

int main() {
    printf("%ld\n", maxofthree(1, -4, -7));
    printf("%ld\n", maxofthree(2, -6, 1));
    printf("%ld\n", maxofthree(2, 3, 1));
    printf("%ld\n", maxofthree(-2, 4, 3));
    printf("%ld\n", maxofthree(2, -6, 5));
    printf("%ld\n", maxofthree(2, 4, 6));
    return 0;
}
```
執行callmaxofthree.c maxofthree.s :
```
gcc -std=c99 callmaxofthree.c maxofthree.s 
./a.out
```

<br><br> 
# Command Line Arguments-echo.s   (了解如何存取argc argv) :
>在 C 語言裡，main 只是一個普通的函數，有自己的參數：  
int main(int argc, char** argv)　　


>一個程式傳回每個命令列參數：

```
# -----------------------------------------------------------------------------
# A 64-bit program that displays its commandline arguments, one per line.
#
# On entry, %rdi will contain argc and %rsi will contain argv.
# -----------------------------------------------------------------------------

        .global main

        .text
main:
        push    %rdi                    # save registers that puts uses
        push    %rsi
        sub     $8, %rsp                # must align stack before call

        mov     (%rsi), %rdi            # the argument string to display
        call    puts                    # print it   (呼叫puts)

        add     $8, %rsp                # restore %rsp to pre-aligned value
        pop     %rsi                    # restore registers puts used
        pop     %rdi

        add     $8, %rsi                # point to next argument
        dec     %rdi                    # count down
        jnz     main                    # if not done counting keep going

        ret
format:
        .asciz  "%s\n"

```
執行echo.s : 
```
gcc echo.s
./a.out 25782 dog huh $$ 
./a.out 25782 dog huh '$$'
```
>$$ 顯示出shell 的 PID 
>'$$' 印出$$


<br><br> 

# 這個程式碼 `power.s` 是將傳入的參數（字串）視為整數（x的y次方）來處理。

1. 就C Library（標準函式庫）而言，命令列參數始終是以字串（string）的形式呈現。
2. 如果我們希望將這些參數視為整數，就需要使用 `atoi` 函式（ASCII to Integer）來將字串轉換為整數。
3. 這個範例的另一個特點是將這些整數值限制在32位元（bit）的範圍內。也就是說，可能會有某些參數超出了32位元整數所能表示的範圍，因此可能需要額外的處理或錯誤檢查來確保處理結果的正確性。

```
# -----------------------------------------------------------------------------
# A 64-bit command line application to compute x^y.
#
# Syntax: power x y
# x and y are integers
# -----------------------------------------------------------------------------

        .global main

        .text
main:
        push    %r12                    # save callee-save registers
        push    %r13
        push    %r14
        # By pushing 3 registers our stack is already aligned for calls

        cmp     $3, %rdi                # must have exactly two arguments
        jne     error1

        mov     %rsi, %r12              # argv

# We will use ecx to count down form the exponent to zero, esi to hold the
# value of the base, and eax to hold the running product.

        mov     16(%r12), %rdi          # argv[2]
        call    atoi                    # y in eax
        cmp     $0, %eax                # disallow negative exponents
        jl      error2
        mov     %eax, %r13d             # y in r13d

        mov     8(%r12), %rdi           # argv
        call    atoi                    # x in eax   (呼叫atoi把字串轉成整數)
        mov     %eax, %r14d             # x in r14d

        mov     $1, %eax                # start with answer = 1
check:
        test    %r13d, %r13d            # we're counting y downto 0
        jz      gotit                   # done
        imul    %r14d, %eax             # multiply in another x
        dec     %r13d
        jmp     check
gotit:                                  # print report on success
        mov     $answer, %rdi
        movslq  %eax, %rsi
        xor     %rax, %rax
        call    printf
        jmp     done
error1:                                 # print error message
        mov     $badArgumentCount, %edi
        call    puts
        jmp     done
error2:                                 # print error message
        mov     $negativeExponent, %edi
        call    puts
done:                                   # restore saved registers
        pop     %r14
        pop     %r13
        pop     %r12
        ret

answer:
        .asciz  "%d\n"
badArgumentCount:
        .asciz  "Requires exactly two arguments\n"
negativeExponent:
        .asciz  "The exponent may not be negative\n"
```
atoi用法 : 
>int atoi(const char *str)  
>將字串裡的數字字元(參數str)轉化為整數（int型）。並傳回  
>注意：轉化時跳過前面的空格字元，直到遇上數字或正負符號才開始做轉換，而再次遇到非數字或字串結束時('/0')才結束轉換，並將結果傳回。

執行power.s
```
gcc -no-pie power.s -o power
./power 1 3  (印出1)
./power 2 4  (印出16)
```

參考資料 : 

https://poli.cs.vsb.cz/edu/soj/down/soj-syllabus.pdf

https://hackmd.io/@PJChiou/HJGLKrbYQ/https%3A%2F%2Fhackmd.io%2Fs%2FS1lP1tfF7?type=book
