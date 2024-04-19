# Introduction to Assembly language

Assembly is the closest programming language  to machine code executed by computer processors. While doing software reversing, we end up working with assembly code. Understand assembly allow us to comprehend exactly what a program is doing at hardware level. In this blog, i resumed all knowledge i learn of the assembly language from the book **Practical Binary Analysis** from **nostarchpress** editorial.

## Design of a program in assembly <a href="#diseno-de-un-programa-en-assembly" id="diseno-de-un-programa-en-assembly"></a>

Comparative between C code and assembly code.

<table><thead><tr><th width="372">C Code</th><th>Assembly Code</th></tr></thead><tbody><tr><td><pre class="language-c"><code class="lang-c">#include &#x3C;stdio.h>

int main(int argc, char *argvp[]){

    printf("Hola mundo\n");

    return 0;
}
</code></pre></td><td><pre class="language-nasm"><code class="lang-nasm">    .file      "hello.c"
    .intel_syntax noprefix
    .section   .rodata
.LC0:
    .string    "Hola mundo\n"
    .text
    .global    main
    .type      main, @function
main:
    push       rbp
    mov        rbp, rsp
    sub        rsp, 16
    mov        DWORD PTR [rbp-4], edi
    mov        QWORD PTR [rbp-16], rsi
    mov        edi, OFFSET FLAT:.LC0
    call       puts
    mov        eax, 0
    leave
    ret
    .size      main, .-main
    .ident     "GCC: (Ubuntu 5.4.0-6ubuntu1..)"
    .section   .note.GNU-stack,""
</code></pre></td></tr></tbody></table>

C code consist of a `main` function that is the entry point of every C program. Inside the `main` function there is a call to `printf` function to show the message "Hola mundo" to screen. At assembly level, the program consist of 4 types of components:

* Instrucions
* Directives
* Labels
* Comments

In the assembly code, the directive `.section .rodata` tells the assembler that the next content has to be located in the `.rodata` section, which is dedicated to store constant data, which means read-only data. The directive `.section` has mentioned before, tells the assembler to store the following content in the specified section.&#x20;

The `.string` directive allows the definition of ASCII strings. There is different directives to define different types of data like `.byte`, `.word`, `.long` and `.quad`.

The `main` function is located inside `.text` section, which is dedicated to store executable code. The directive `.text` is a short way of `.section .text`.

The instruction `main:` define a symbolic label to `main` function. After defining `main` label, the program continue with the instructions inside `main` function. These instructions can refer symbolically to prior information like `.LC0` (is the symbolic name that `gcc` choose to string "Hola mundo").

***

### Instructions, directives, labels and comments <a href="#instrucciones-directivas-etiquetas-y-comentarios-de-lenguaje-ensamblador" id="instrucciones-directivas-etiquetas-y-comentarios-de-lenguaje-ensamblador"></a>

| Type        | Example               | Description                                       |
| ----------- | --------------------- | ------------------------------------------------- |
| Instruction | mov eax, 0            | eax=0                                             |
| Directive   | .section .text        | Store the following content inside .text section  |
| Directive   | .string “foobar”      | Define an ASCII string which contains "foobar"    |
| Directive   | .long 0x12345678      | Define a doubleword with the value 0x12345678     |
| Label       | foo: .string “foobar” | Define the string “foobar” with symbolic name foo |
| Comment     | ; this is a comment   | A comment                                         |

The **instructions** are operations executed by CPU.&#x20;

**Directives** are commands that tells the assembly to:

* Produce a particular piece of data.
* Store instructions or information inside a particular section.
* etc

**Labels** are simply names that we can use to referer to an instruction or data inside the assembly program.

**Comments** are text  to document the code.

***

### Separation between code and data <a href="#separacion-entre-codigo-y-datos" id="separacion-entre-codigo-y-datos"></a>

In the assembly source code above, is easy to distinguish code and data because the code is separated in different sections. However, this is not always the case because there is nothing that prevents the x86 architecture to mix the code and data in the same section and in the practice, some compilers or assemblers, do this.

***

### AT\&T Syntax vs Intel Syntax <a href="#sintaxis-att-vs-sintaxis-intel" id="sintaxis-att-vs-sintaxis-intel"></a>

When analyzing data, is important to distinguish between different assembly syntaxes.

For example, the AT\&T syntax explicitly puts the prefix `%` to registers and for constants it puts the prefix `$` .

The operands order also changes.

| AT\&T Syntax                               | Intel Syntax                             |
| ------------------------------------------ | ---------------------------------------- |
| <pre><code>mov    $0x6, %edi
</code></pre> | <pre><code>mov    edi, 0x6
</code></pre> |

***

## Structure of an x86 instruction <a href="#estructura-de-una-instruccion-x86" id="estructura-de-una-instruccion-x86"></a>

At assembly level, the instructions generally have the following structure:

```
mnemonic    destination,    source
```

The `mnemonic` is the representation of the machine instruction, and the `source` and `destination` are operands for the `mnemonic`.

Not all instructions require operands.

***

### Machine level x86 instructions structure <a href="#estructura-a-nivel-de-maquina-de-las-instrucciones-x86" id="estructura-a-nivel-de-maquina-de-las-instrucciones-x86"></a>

x86 Instruction Set Architecture (ISA) use variable length instructions. It menas that, there are instructions which consist in 1 byte, multiple bytes and there are also instructions with 15 bytes long.

Besides of that, instructions can start at any memory  address. This means that the CPU does not force any alignment in the code, although some compilers some times perform alignment to improve performance.

<table><thead><tr><th>Prefix</th><th>Opcode</th><th>Offset</th><th>Inmediate</th><th data-hidden>Addressing mode</th><th data-hidden>SIB Byte</th></tr></thead><tbody><tr><td>0-4 bytes</td><td>1-3 bytes</td><td>0/1/2/4 bytes</td><td>0/1/2/4 bytes</td><td>0-1</td><td>0-1</td></tr></tbody></table>

An instruction consists of a **prefix** (optional), an **opcode** and zero or more **operands**. All these parts are optionals except for the **opcode.**&#x20;

The **opcode** is the main part of the instruction, meanwhile the prefix can modify the behavior of an instruction.

Some instructions have implicit operands. That is, they are not explicitly shown in the instruction since they are innate to the opcode. For example, the recipient operand of opcode 0x05 (an add instruction) is always rax and only the source operand (src) is variable and needs to be explicitly specified.&#x20;

Another example of implicit operands is the push instruction which implicitly updates rsp (stack pointer register).&#x20;

Instructions can have different types of operands:&#x20;

* register operands&#x20;
* memory operands&#x20;
* immediate operands

***

### Register Operands&#x20;

Registers are small but very fast pieces of storage stored in the CPU. Some registers have special purposes, such as the instruction pointer that stores the current address of execution or the stack pointer that stores the address of the top of the stack.&#x20;

**General purpose registers:** In the 8086 instruction set, the registers were 16 bits. The x86 ISA extended registers to 32 bits, and x86-64 extended them further to 64 bits. To maintain compatibility, the registers used in new instruction sets are supersets of the old registers.&#x20;

To specify a register operand in assembler, the register name is used. For example, `mov rax, 64` what it does is move the value 64 to the `rax` register. In the previous example we are using the 64-bit `rax` register, if we wanted to use the 32-bit part we would have to specify the name `eax` and if we wanted to use the 16-bit part we would use the name `ax` and finally, if we wanted to use the upper 8 bits for `ax` we would use the name `ah` and to access the lower 8 bits we would use the name `al` .

<figure><img src="https://zerotrustoffsec.com/assets/img/blog/reversing/introduccion_al_lenguaje_ensamblador/register_subdivision.png" alt=""><figcaption><p>subdivisión del registro x86-64 rax</p></figcaption></figure>

Other registries such as `rbx, rcx, rdx` among others, follow the same naming system to access their lower parts.&#x20;

Registers `r8-r15` were added in x86-64 and are not available in earlier variants of x86.

<figure><img src="https://zerotrustoffsec.com/assets/img/blog/reversing/introduccion_al_lenguaje_ensamblador/general_purpose_registers.png" alt=""><figcaption><p>Registros de proposito general</p></figcaption></figure>

### Other Registers

In addition to the registers mentioned above, there are also other registers such as `rip` (`eip` in 32 bit x86 and `ip` in 8086) and `rflags` (called `eflags` or `flags`). The rip register always points to the address of the next instruction to be executed and is automatically updated by the CPU. The status flags register is used for comparisons and conditionals as well as tracking things like whether the last operation returned zero or resulted in overflow, etc.

***

### Memmory  Operands <a href="#operandos-de-memoria" id="operandos-de-memoria"></a>

Memory operands specify a memory address where the CPU should look for one or more bytes. The x86 ISA supports only one explicit memory operand per instruction. This means that you cannot move bytes from one memory location to another location in the same instruction. To do this, registers must be used as intermediate storage.&#x20;

On x86, memory operands are specified as follows:

```
[base + index*scale + displacement]
```

Base and index are 64-bit registers, scale is an integer with the value 1, 2, 4, or 8, and displacement is a 32-bit constant or symbol.&#x20;

For example, you can use a statement like `mov eax, DWORD PTR [rax*4 + arr]` to access an array, where `arr` is the offset containing the starting address of the array, `rax` contains the index of the element you want access from the array and each element of the array is 4 bytes, that is why rax \* 4 is multiplied. DWORD PTR tells the assembler that we want 4 bytes (a doubleword or DWORD) of memory.

***

### Inmediate Operands <a href="#operandos-inmediatos" id="operandos-inmediatos"></a>

These operands are simply constants encoded in the instruction. For example, in the `add rax, 42` instruction, the value 42 is the immediate one.&#x20;

On x86, immediates are encoded in little-endian format.

***

## Common x86 Instructions <a href="#instrucciones-comunes-x86" id="instrucciones-comunes-x86"></a>

List of instructions online:

* [ref.x86asm.net](http://ref.x86asm.net/)&#x20;
* [Intel SDM](https://software.intel.com/en-us/articles/intel-sdm/)

```
+--------------------------------------------------------------------+
|                 Data transfer                                      | 
+----------------------+---------------------------------------------+
|    Instruccion       |       Descripcion                           |
+----------------------+---------------------------------------------+
|  mov dst, src        |  dst = src                                  |
|  xchg dst1, dst2     |  swap dst1 and dst2                         |
|  push src            |  push src onto stack and decrement rsp      |
|  pop dst             |  pop value from stack into dst and inc rsp  |
+--------------------------------------------------------------------+
|                 Aritmetic                                          | 
+----------------------+---------------------------------------------+
|    Instruccion       |       Descripcion                           |
+----------------------+---------------------------------------------+
|  add dst, src        |  dst += src                                 |
|  sub dst, src        |  dst -= src                                 |
|  inc dst             |  dst += 1                                   |
|  dec dst             |  dst -= 1                                   |
|  neg dst             |  dst = -dst                                 |
|  cmp src1, src2      |  set status flag based on src1 - src2       |
+--------------------------------------------------------------------+
|                 Logical/bitwise                                    | 
+----------------------+---------------------------------------------+
|    Instruccion       |       Descripcion                           |
+----------------------+---------------------------------------------+
|  and dst, src        |  dst &= src                                 |
|  or  dst, src        |  dst |= src                                 |
|  xor dst, src        |  dst ^= src                                 |
|  not dst             |  dst = ~dst                                 |
|  test src1, src2     |  set status flag based on src1 & src2       |
+--------------------------------------------------------------------+
|                 unconditional branches                             | 
+----------------------+---------------------------------------------+
|    Instruccion       |       Descripcion                           |
+----------------------+---------------------------------------------+
|  jmp addr            |  jump to addr                               |
|  call addr           |  push return address on stack, then return  |
|                      |  function at addr                           |
|  ret                 |  pop return address from stack and return   |
|                      |  to that address                            |
|  syscall             |  Enter the kernel to perform a system call  |
+--------------------------------------------------------------------+
|                 conditional branches                               | 
+----------------------+---------------------------------------------+
|    Instruccion       |       Descripcion                           |
+----------------------+---------------------------------------------+
|  je addr/jz addr     |  jump to addr if zero flag is set           |
|                      |  for example, operands were equal on the    |
|                      |  last cmp                                   |
|  ja addr             |  jump if dst is above src in the last cmp   |
|  jb addr             |  jump if dst is below src in the last cmp   |
|  jg addr             |  jump if dst is greater thn src in the cmp  |
|  jl addr             |  jump if dst is less than  src in the cmp   |
+----------------------+---------------------------------------------+

```

***

### Operands Comparasion (status flags) <a href="#comparando-operandos-status-flags" id="comparando-operandos-status-flags"></a>

The `cmp` instruction subtracts the second operand from the first operand, and according to the result of this operation, it establishes several status flags in the `rflags` register that we can work with later. The most important flags are the following:&#x20;

* zero flag (ZT) if the result of the subtraction is zero, this means that the operands are equal or rather, they have equal values.&#x20;
* sign flag (SF) if the result was negative, this means that in the `cmp src1, src2` operation the operand src2 is greater than src1.&#x20;
* overflow flag (OF) the result was an overflow&#x20;

The `test` instruction does the same thing on `rflags`, only instead of performing a subtraction, it performs an addition.

***

### System Calls <a href="#implementando-system-calls" id="implementando-system-calls"></a>

To make a system call, the `syscall` instruction is used, but before using it, the system call must be prepared by selecting a number and establishing its operands as specified by the system. For example, to make a read system call in Linux, the value 0 is loaded in `rax`, then the file descriptor is loaded in `rdi`, buffer address in `rsi` and number of bytes to read in `rdx`

```
section .data
    buffer db 256    ; Allowcate a buffer of 256 bytes
    len equ 256      ; length of the buffer 

section .text
global _start

_start:
    ; Syscall to read from stdin
    mov rax, 0          ; syscall number for read (0)
    mov rdi, 0          ; file descriptor 0 (stdin)
    mov rsi, buffer     ; buffer address
    mov rdx, len        ; n bytes to read (buffer length)
    syscall             ; perform syscall
    mov rbx, rax        ; save the return value in rbx

    ; Syscall to write to stdout
    mov rax, 1          ; syscall number for write (1)
    mov rdi, 1          ; file descriptor 1 (stdout)
    mov rsi, buffer     ; buffer address
    mov rdx, rbx        ; 
    syscall             ; 

    ; Syscall to finish program
    mov rax, 60         ; syscall number for exit (60)
    xor rdi, rdi        ; código de salida 0
    syscall             ; 
```

The previous example is a simple assembly program that reads a message from stdin and print it to stdout. To execute it we need first to execute the following commands:

```bash
# usamos el ensamblador nasm
nasm -f elf64 -o syscall_example.o syscall_example.asm 

# enlazamos el programa
ld -o syscall_example syscall_example.o

# lo ejecutamos
./syscall_example
```

***

### Conditional Jumps <a href="#implementando-conditional-jumps" id="implementando-conditional-jumps"></a>

As mentioned above, conditional jumps work thanks to state flags which are modified by instructions such as `cmp` or `test` . These jumps are made to specific addresses or labels if the condition is met and if it is not met, the jump will simply be ignored and the instruction that follows it will be executed.

```nasm
cmp rax, rbx
jb label
```

In the previous example, a comparasion is performed and subsequently a `jb` (jump if below). This means that if rax < rbx (unsigned comparison) then the jump is performed. In the following example, the jump jnz (jump if not zero) is performed if rax is not equal to 0

```nasm
test rax, rax
jnz label
```

***

### Loading memory addresses  <a href="#cargando-direcciones-de-memoria" id="cargando-direcciones-de-memoria"></a>

Instruction `lea` (load effective address) computes the result direction of a memory operand and store it in a registry. Is almost the same as the `&` operator in C language.

```
lea r12, [rip+0x2000]
```

In the example above, `lea` instruction loads the memory address of the result of `rip+0x2000` in the register `r12`.

***

## The stack <a href="#la-pila-stack" id="la-pila-stack"></a>

The stack is a reserved memory region to store data related with the function calls like the return address, function arguments and the local variables. The stack is called that way cause of the way data is accessed from it. Instead of writing data in random addresses of the stack, the stack store data using the LIFO order (Last in First out). This way, data can be written with a push from the top.

As data is pushed onto the stack, the `rsp` register (`rsp` is the registar that points to the top of the stack) will decrement and this is because the stack aument to lower memory addresses.&#x20;

<figure><img src="https://zerotrustoffsec.com/assets/img/blog/reversing/introduccion_al_lenguaje_ensamblador/stack_1.png" alt=""><figcaption></figcaption></figure>

It should be noted that when we perform a push, as mentioned before, the value is stored in the lowest address of the stack (`rsp`) and when we perform a push, the `rsp` increases until it is at the memory address it had previously, however , the value that we popped is still in memory so it is important to know that if we have sensitive information on the stack and we want to clean it completely, we must overwrite it or delete it explicitly since a pop is not going to do it.

***

## Function calls and Function frames <a href="#function-calls-y-function-frames" id="function-calls-y-function-frames"></a>

| C source code                                                                                                                                                                | Assembly source code                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <pre><code>#include &#x3C;stdio.h>
#include &#x3C;stdlib.h>

int main(int argc, char *argv[]){
  printf("%s=%s\n", argv[1], getenv(argv[1]));
  
  return 0;
}
</code></pre> | <pre><code>Contents of section .rodata:
.LC0:
        .string "%s=%s\n"

Contents of section .text:
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-4], edi
        mov     QWORD PTR [rbp-16], rsi
        mov     rax, QWORD PTR [rbp-16]
        add     rax, 8
        mov     rax, QWORD PTR [rax]
        mov     rdi, rax
        call    getenv
        mov     rdx, rax
        mov     rax, QWORD PTR [rbp-16]
        add     rax, 8
        mov     rax, QWORD PTR [rax]
        mov     rsi, rax
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf
        mov     eax, 0
        leave
        ret
</code></pre> |

The C source code aboce, shows two function calls. The first function call is `getenv` which is a function to get the value of a specified environment variable. The environment variable is specified with `argv[1]` which is the firs command line argument. After the call to `getenv`, the second function call is to `printf` function which simply prints a message to the terminal.

Comparing the C source code with the assembly code, we can identify a lot of things. First, the string "%s=%s" is stored as a constant in the section `.rodata` (read-only data) and the symbol for that constant is `.LC0`.

Every function have its own frame in the stack, limited by `rbp` registar which points to the bottom of the function frame and `rsp` points to the top of the frame.

<figure><img src="https://zerotrustoffsec.com/assets/img/blog/reversing/introduccion_al_lenguaje_ensamblador/stack_2.png" alt=""><figcaption></figcaption></figure>

In the example we see of the previous assembly code, the first thing the main function does is execute the prologue, which basically what it does is establish the function frame.

{% code title="function prologue" %}
```nasm
push rbp      ; Saving the content of rbp to the stack
mov rbp, rsp  ; Now rbp is equal to rsp
```
{% endcode %}

This prologue is so common, that a shortcut instruction was created specifically to do this and is called `enter`.

On Linux x86-64, registers `rbx` and `r12-r15` must not be contaminated during the execution of a function, that is, if a function makes use of these registers, said function must restore them to the original value before returning (`ret`) . This is achieved by storing the values of said registers that need to be restored on the stack at the beginning of the function execution, and popping said values to restore the registers before returning.&#x20;

After executing the function prologue, the `rbp` register is decremented by `0x10` (16) bytes to reserve space for local variables on the stack (4 bytes for argc and 8 bytes for the argv pointer and the rest of the bytes used for padding) .&#x20;

On x86-64 Linux systems, the first 6 arguments to a function are passed using the rdi, rsi, rdx, rcx, r8, and r9 registers. If the function receives more than 6 arguments or some arguments do not fit in the 64-bit registers, then the remaining arguments are stored on the stack in reverse order.

```
; Storing parameters in variables before calling a function
mov rdi, param1
mov rsi, param2
mov rdx, param3
mov rcx, param4
mov r8, param5
mov r9, param6
push param9
push param8
push param7
...
call function
```

This can vary depending on the convention used to pass parameters to functions, that is, if the cdecl convention is used, all arguments are passed on the stack using reverse order without using any registers. Another convention is fastcall which passes some arguments into registers.

***

### Red Zone <a href="#la-zona-roja" id="la-zona-roja"></a>

The red zone is a 128-byte area (in the x86-64 ABI) below the stack pointer. Programmers and compilers can use this area to store temporary data without needing to modify the stack pointer. This can be useful to optimize certain operations and avoid additional instructions to adjust the stack.&#x20;

A key characteristic of the red zone is that the operating system does not preserve it when handling interrupts or signals. This means that if an interrupt occurs, the interrupt handler could overwrite the data in the red zone.

***

### Preparing arguments and function calls <a href="#preparando-argumentos-y-llamando-funciones" id="preparando-argumentos-y-llamando-funciones"></a>

In the source code at the start of this blog, after the the execution of the prologue function, the following is done:

```nasm
; Store the memory address of the start of argv in the register rax
mov  rax, QWORD PTR [rbp-16]

; The C source code used the first index of argv (argv[1]), then 
; we need to add 8 bytes to rax (8 bytes is the length of a pointer) to point to
; argv[1]
add  rax, 8

; rax points to argv[1], however what we want is the value of the address
; rax is pointing to so we need to execute the following instruction
mov  rax, QWORD PTR [rax]

; now we need to move the value of rax to rdi because rdi is the register 
; used to send parameters to functions
mov  rdi, rax

; Now we simply call getenv function and rdi is passed as parameter
call getenv 
```

***

### Reading return values from functions <a href="#leyendo-valores-retornados-por-funciones" id="leyendo-valores-retornados-por-funciones"></a>

If a function returns a value, this value will be stored in `rax` register:

```
call getenv ; function call

; storing the result value of the function call to rax
mov rdx, rax

mov rax, QWORD PTR [rbp-16] ; rax = &argv[0]
add rax, 8                  ; rax = &argv[1]
mov rax, QWORD PTR [rax]    ; rax = argv[1]
mov rsi, rax                ; rsi = rax

; now edi points to .LC0 
mov edi, OFFSET FLAT:.LC0

; al llamar a una funcion variadica, el registro rax cumple la funcion de especificar
; la cantidad de argumentos float pasados a la funcion, sin embargo en este caso no
; hay ningun argumento de tipo float por lo que rax (eax) es igual a 0 
mov eax, 0

; en este punto, los siguientes registros estan asi
; PARAMETRO 1: rdi = "%s=%s\n"
; PARAMETRO 2: rsi = argv[1]
; PARAMETRO 3: rdx = getenv(argv[1])
; ahora se llama a la funcion printf
call printf ; printf(rdi, rsi, rdx); -> printf("%s=%s\n", argv[1], getenv(argv[1]));
```

***

### Returning from a function <a href="#retornando-de-una-funcion" id="retornando-de-una-funcion"></a>

After the `printf` function call is completed, `rax` is set to 0 because as mentioned before, this is the registry used to store the return value of a function call. Then `leave` instruction is executed.&#x20;

The `leave` instruction is a fast way of:

```
mov rsp, rbp
pop rbp
```

This is the `epilogue` of a function and it is done to restore the stack to the initial state before the function call.

Lastly, `ret` instruction is executed. The `ret` instruction executes a `pop` to get the address where the execution should continue.

***

## Conditional Branches <a href="#ramas-condicionales" id="ramas-condicionales"></a>

| C Source Code                                                                                                                                                                                                                                                          | Assembly Source Code                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <pre class="language-c"><code class="lang-c">#include &#x3C;stdio.h>
#include &#x3C;stdlib.h>

int main(int argc, char *argv[]){

    if (argc > 5){
        printf("argc > 5\n");
    }else{
        printf("argc &#x3C;= 5\n");
    }

    return 0;
}
</code></pre> | <pre class="language-nasm"><code class="lang-nasm">.LC0:
        .string "argc > 5"
.LC1:
        .string "argc &#x3C;= 5"
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-4], edi
        mov     QWORD PTR [rbp-16], rsi
        cmp     DWORD PTR [rbp-4], 5
        jle     .L2
        mov     edi, OFFSET FLAT:.LC0
        call    puts
        jmp     .L3
.L2:
        mov     edi, OFFSET FLAT:.LC1
        call    puts
.L3:
        mov     eax, 0
        leave
        ret
</code></pre> |

In the source code above, after the prologue, variables `argc` and `argv` are stored in the stack, after  this a comparison is made between `argc` and the constant `5`.

```nasm
; prologue
push rbp
mov rbp, rsp

; The values of edi and rsi are stored in the stack
mov DWORD PTR [rbp-4], edi  ; storing argc
mov QWORD PTR [rbp-16], rsi ; storing argv

; comparison
cmp DWORD PTR [rbp-4], 5    ; compare(argc, 5)

jle .L2 ; if argc <= 5, then jump to .L2

; The value of .LC0 is stored in edi to use it as a parameter to a function call
mov edi, OFFSET FLAT:.LC0 

call puts ; calling puts function and using edi as parameter to the function
jmp .L3   ; jump to .L3

.L2:
mov edi, OFFSET FLAT:.LC1 ; the string "argc > 5" is stored in edi 

call puts

.L3:
mov eax, 0 ; eax=0 (return value)
leave  ; restoring stack
ret    ; returning to the point where this function was called
```

***

## Loops <a href="#bucles" id="bucles"></a>

|                                                                                                                                                                                                            |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| <pre><code>#include &#x3C;stdio.h>
#include &#x3C;stdlib.h>

int main(int argc, char *argv[]){

    while (argc > 0){
        printf("%s\n", argv[(unsigned)--argc]);
    }

    return 0;
}
</code></pre> | <pre><code>main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-4], edi
        mov     QWORD PTR [rbp-16], rsi
        jmp     .L2
.L3:
        sub     DWORD PTR [rbp-4], 1
        mov     eax, DWORD PTR [rbp-4]
        mov     eax, eax
        lea     rdx, [0+rax*8]
        mov     rax, QWORD PTR [rbp-16]
        add     rax, rdx
        mov     rax, QWORD PTR [rax]
        mov     rdi, rax
        call    puts
.L2:
        cmp     DWORD PTR [rbp-4], 0
        jg      .L3
        mov     eax, 0
        leave
        ret
</code></pre> |

The C program above simply prints to the terminal all arguments passed to the program from command line.

```
$ exec-cpp hola mundo prueba 1 2 3
3
2
1
prueba
mundo
hola
./prog.out

```

Assembly code explanation:

```
main:
        ; epilogue
        push    rbp
        mov     rbp, rsp
        
        ; allocating memory in the stack 
        sub     rsp, 16
        
        ; storing argc and argv in the stack
        mov     DWORD PTR [rbp-4], edi  ; argc
        mov     QWORD PTR [rbp-16], rsi ; argv
        
        ; unconditional jump to a .L2
        jmp     .L2
.L3:
        ; decrementing 1 to argc 
        sub     DWORD PTR [rbp-4], 1
        
        ; eax = argc
        mov     eax, DWORD PTR [rbp-4]
        
        ; ??? what the ...
        mov     eax, eax
        
        ; rdx = argv[rax]
        lea     rdx, [0+rax*8]
        
        ; rax = &argv[0]
        mov     rax, QWORD PTR [rbp-16]
        
        ; rax += rdx
        add     rax, rdx
        
        ; rax = argv[rdx]
        mov     rax, QWORD PTR [rax]
        
        ; rdi = rax
        mov     rdi, rax
        
        ; puts(rdi);
        call    puts
.L2:
        ; comparando argc con el valor inmediato 0
        cmp     DWORD PTR [rbp-4], 0
        
        ; si argc > 0, entonces saltar a .L3
        jg      .L3
        
        ; este codigo se ejecuta unicamente cuando el valor de argc sea igual a 0
        mov     eax, 0
        leave
        ret
```

***
