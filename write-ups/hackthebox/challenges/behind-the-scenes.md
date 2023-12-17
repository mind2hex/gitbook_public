---
description: Very Easy
---

# Behind the scenes

## Análisis básico

***

### Usando  file

```
$ file behindthescenes 
behindthescenes: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=e60ae4c886619b869178148afd12d0a5428bfe18, for GNU/Linux 3.2.0, not stripped
```

El archivo es un binario ELF de 64 BITS enlazado dinamicamente y **no esta strippeado**.

El binario al no estar strippeado, puede contener mucha mas información útil, por lo que vamos a utilizar el comando `strings` para revisar las secuencias de carácteres imprimibles.

***

### Usando strings

<pre><code>$ strings behindthescenes     
/lib64/ld-linux-x86-64.so.2
libc.so.6
<a data-footnote-ref href="#user-content-fn-1">strncmp</a>
puts
__stack_chk_fail
printf
<a data-footnote-ref href="#user-content-fn-2">strlen</a>             
sigemptyset
memset
sigaction
__cxa_finalize
__libc_start_main
...
./challenge &#x3C;password>
> HTB{%s}
...
strncmp@@GLIBC_2.2.5
_ITM_deregisterTMCloneTable
puts@@GLIBC_2.2.5
sigaction@@GLIBC_2.2.5
strlen@@GLIBC_2.2.5
__stack_chk_fail@@GLIBC_2.4
printf@@GLIBC_2.2.5
memset@@GLIBC_2.2.5
...
segill_sigaction
sigemptyset@@GLIBC_2.2.5
...
</code></pre>

Como se puede ver en la salida anterior,  el programa utiliza funciones como `strlen` y `strncmp` por lo que es muy probable que espere unargumento de línea  de comandos.

Ahora vamos a ejecutar el binario para saber como funciona.

***

### Ejecutando el binario

```
┌──(kali㉿kali)-[~/…/Challenges/Reversing/Behind_the_scenes/rev_behindthescenes]
└─$ ./behindthescenes             
./challenge <password>
                                                                                                                     
┌──(kali㉿kali)-[~/…/Challenges/Reversing/Behind_the_scenes/rev_behindthescenes]
└─$ ./behindthescenes s3cr3t 

```

Lo único que podemos examinar del binario, es que este mismo espera un argumento, sin embargo no hay más que podamos deducir así que vamos a utilizar **ltrace** y **strace** para ver que llamadas al sistema se realizan y para ver como usa las librerías dinámicas.

***

### Usando ltrace

```
$ ltrace ./behindthescenes       
--- SIGILL (Illegal instruction) ---
--- SIGILL (Illegal instruction) ---
./challenge <password>
--- SIGILL (Illegal instruction) ---
+++ exited (status 1) +++
                                                                                                                                                                                                                                              
┌──(kali㉿kali)-[~/…/Challenges/Reversing/Behind_the_scenes/rev_behindthescenes]
└─$ ltrace ./behindthescenes s3cr3t
--- SIGILL (Illegal instruction) ---
--- SIGILL (Illegal instruction) ---
--- SIGILL (Illegal instruction) ---
+++ exited (status 0) +++

```

Al usar ltrace, vemos que durante la ejecución nos encontramos con la señal SIGILL (Illegal Instruction). Aunque no podemos ver más información, debemos tomar en cuenta lo siguiente:

**SIGILL** Es una señal enviada a un proceso en sistemas Unix y Linux cuando intenta ejecutar una instrucción de máquina que no es válida o no tiene permiso para ejecutar.

***

### Usando strace

```
$ strace ./behindthescenes s3cr3t
execve("./behindthescenes", ["./behindthescenes", "s3cr3t"], 0x7ffec6dd0b08 /* 56 vars */) = 0
brk(NULL)                               = 0x55b632832000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f121ef8d000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=91307, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 91307, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f121ef76000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220x\2\0\0\0\0\0"..., 832) = 832
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
newfstatat(3, "", {st_mode=S_IFREG|0755, st_size=1926256, ...}, AT_EMPTY_PATH) = 0
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
mmap(NULL, 1974096, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f121ed94000
mmap(0x7f121edba000, 1396736, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x26000) = 0x7f121edba000
mmap(0x7f121ef0f000, 344064, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x17b000) = 0x7f121ef0f000
mmap(0x7f121ef63000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1cf000) = 0x7f121ef63000
mmap(0x7f121ef69000, 53072, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f121ef69000
close(3)                                = 0
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f121ed91000
arch_prctl(ARCH_SET_FS, 0x7f121ed91740) = 0
set_tid_address(0x7f121ed91a10)         = 224573
set_robust_list(0x7f121ed91a20, 24)     = 0
rseq(0x7f121ed92060, 0x20, 0, 0x53053053) = 0
mprotect(0x7f121ef63000, 16384, PROT_READ) = 0
mprotect(0x55b631a17000, 4096, PROT_READ) = 0
mprotect(0x7f121efbf000, 8192, PROT_READ) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
munmap(0x7f121ef76000, 91307)           = 0
rt_sigaction(SIGILL, {sa_handler=0x55b631a15229, sa_mask=[], sa_flags=SA_RESTORER|SA_SIGINFO, sa_restorer=0x7f121edd0510}, NULL, 8) = 0
--- SIGILL {si_signo=SIGILL, si_code=ILL_ILLOPN, si_addr=0x55b631a152e6} ---
rt_sigreturn({mask=[]})                 = 0
--- SIGILL {si_signo=SIGILL, si_code=ILL_ILLOPN, si_addr=0x55b631a1530b} ---
rt_sigreturn({mask=[]})                 = 0
--- SIGILL {si_signo=SIGILL, si_code=ILL_ILLOPN, si_addr=0x55b631a15432} ---
rt_sigreturn({mask=[]})                 = 6
exit_group(0)                           = ?
+++ exited with 0 +++

```

A pesar de que strace nos muestra un monton de información, no vemos nada nuevo que nos pueda indicar del porque de las señales o como funciona el programa por lo que hay que analizar la herramienta más a fondo.

***

## Análisis estático

***

### Usando radare2

```
$ radare2 ./behindthescenes
Warning: run r2 with -e bin.cache=true to fix relocations in disassembly
[0x00001140]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Finding and parsing C++ vtables (avrr)
[x] Type matching analysis for all functions (aaft)
[x] Propagate noreturn information (aanr)
[x] Use -AA or aaaa to perform additional experimental analysis.
[0x00001140]> afl
0x00001140    1 47           entry0
0x00001170    4 41   -> 34   sym.deregister_tm_clones
0x000011a0    4 57   -> 51   sym.register_tm_clones
0x000011e0    5 57   -> 54   sym.__do_global_dtors_aux
0x000010b0    1 11           sym..plt.got
0x00001220    1 9            entry.init0
0x00001000    3 27           sym._init
0x000014c0    1 5            sym.__libc_csu_fini
0x000014c8    1 13           sym._fini
0x00001229    1 56           sym.segill_sigaction
0x00001450    4 101          sym.__libc_csu_init
0x00001261    1 135          main
0x00001120    1 11           sym.imp.memset
0x00001130    1 11           sym.imp.sigemptyset
0x000010e0    1 11           sym.imp.sigaction
0x000010c0    1 11           sym.imp.strncmp
0x000010d0    1 11           sym.imp.puts
0x000010f0    1 11           sym.imp.strlen
0x00001100    1 11           sym.imp.__stack_chk_fail
0x00001110    1 11           sym.imp.printf
[0x00001140]> 
```

El comando `aaa` analiza el binario a fondo y el comando `afl` sirve para listar las funciones detectadas del análisis realizado. En el resultado anterior vemos las funciones del binario incluyendo main así que vamos a examinar la función main para tener una idea general de como funciona el binario.

```
[0x00001140]> pdf @main
            ; DATA XREF from entry0 @ 0x1161
┌ 135: int main (int argc, char **argv);
│           ; var char **var_b0h @ rbp-0xb0
│           ; var int64_t var_a4h @ rbp-0xa4
│           ; var void *s @ rbp-0xa0
│           ; var int64_t var_18h @ rbp-0x18
│           ; var int64_t var_8h @ rbp-0x8
│           ; arg int argc @ rdi
│           ; arg char **argv @ rsi
│           0x00001261      f30f1efa       endbr64
│           0x00001265      55             push rbp
│           0x00001266      4889e5         mov rbp, rsp
│           0x00001269      4881ecb00000.  sub rsp, 0xb0
│           0x00001270      89bd5cffffff   mov dword [var_a4h], edi    ; argc
│           0x00001276      4889b550ffff.  mov qword [var_b0h], rsi    ; argv
│           0x0000127d      64488b042528.  mov rax, qword fs:[0x28]
│           0x00001286      488945f8       mov qword [var_8h], rax
│           0x0000128a      31c0           xor eax, eax
│           0x0000128c      488d8560ffff.  lea rax, [s]
│           0x00001293      ba98000000     mov edx, 0x98               ; size_t n
│           0x00001298      be00000000     mov esi, 0                  ; int c
│           0x0000129d      4889c7         mov rdi, rax                ; void *s
│           0x000012a0      e87bfeffff     call sym.imp.memset         ; void *memset(void *s, int c, size_t n)
│           0x000012a5      488d8560ffff.  lea rax, [s]
│           0x000012ac      4883c008       add rax, 8
│           0x000012b0      4889c7         mov rdi, rax
│           0x000012b3      e878feffff     call sym.imp.sigemptyset
│           0x000012b8      488d056affff.  lea rax, [sym.segill_sigaction] ; 0x1229
│           0x000012bf      48898560ffff.  mov qword [s], rax
│           0x000012c6      c745e8040000.  mov dword [var_18h], 4
│           0x000012cd      488d8560ffff.  lea rax, [s]
│           0x000012d4      ba00000000     mov edx, 0
│           0x000012d9      4889c6         mov rsi, rax
│           0x000012dc      bf04000000     mov edi, 4
│           0x000012e1      e8fafdffff     call sym.imp.sigaction
└           0x000012e6      0f0b           ud2

```

Al inspeccionar la función main, vemos que en ningún momento se muestra la llamada a la función `printf` o `puts`, además la última instrucción que se muestra de la función es `ud2` lo cuál no es lo usual ya que normalmente las funciones terminan con una instrucción `ret`.

Según [https://mudongliang.github.io/x86/html/file\_module\_x86\_id\_318.html](https://mudongliang.github.io/x86/html/file\_module\_x86\_id\_318.html), la instrucción `UD2` levanta un opcode invalido, es decir, lla instrucción `UD2` está diseñada específicamente para generar de manera intencionada una excepción de "Instrucción Inválida" o "Undefined Instruction". Esta excepción se traduce en sistemas operativos basados en Unix y Linux como la señal `SIGILL` (señal de instrucción ilegal).

Ahora vamos a analizar nuevamente la función main, solo que esta vez vamos a indicar la cantidad de lineas a desensamblar.

```
[0x55bdc35eb261]> pd 200
            ;-- rax:
            ;-- rip:
            ; DATA XREF from entry0 @ 0x55bdc35eb161
┌ 135: int main (int argc, char **argv, char **envp);
│           ; var char **var_b0h @ rbp-0xb0
│           ; var int64_t var_a4h @ rbp-0xa4
│           ; var void *s @ rbp-0xa0
│           ; var int64_t var_18h @ rbp-0x18
│           ; var int64_t var_8h @ rbp-0x8
│           ; arg int argc @ rdi
│           ; arg char **argv @ rsi
│           0x55bdc35eb261      f30f1efa       endbr64
│           0x55bdc35eb265      55             push rbp
│           0x55bdc35eb266      4889e5         mov rbp, rsp
│           0x55bdc35eb269      4881ecb00000.  sub rsp, 0xb0
│           0x55bdc35eb270      89bd5cffffff   mov dword [var_a4h], edi ; argc
│           0x55bdc35eb276      4889b550ffff.  mov qword [var_b0h], rsi ; argv
│           0x55bdc35eb27d      64488b042528.  mov rax, qword fs:[0x28]
│           0x55bdc35eb286      488945f8       mov qword [var_8h], rax
│           0x55bdc35eb28a      31c0           xor eax, eax
│           0x55bdc35eb28c      488d8560ffff.  lea rax, [s]
│           0x55bdc35eb293      ba98000000     mov edx, 0x98           ; 152 ; size_t n
│           0x55bdc35eb298      be00000000     mov esi, 0              ; int c
│           0x55bdc35eb29d      4889c7         mov rdi, rax            ; void *s
│           0x55bdc35eb2a0      e87bfeffff     call sym.imp.memset     ; void *memset(void *s, int c, size_t n)
│           0x55bdc35eb2a5      488d8560ffff.  lea rax, [s]
│           0x55bdc35eb2ac      4883c008       add rax, 8
│           0x55bdc35eb2b0      4889c7         mov rdi, rax
│           0x55bdc35eb2b3      e878feffff     call sym.imp.sigemptyset
│           0x55bdc35eb2b8      488d056affff.  lea rax, [sym.segill_sigaction] ; 0x55bdc35eb229
│           0x55bdc35eb2bf      48898560ffff.  mov qword [s], rax
│           0x55bdc35eb2c6      c745e8040000.  mov dword [var_18h], 4
│           0x55bdc35eb2cd      488d8560ffff.  lea rax, [s]
│           0x55bdc35eb2d4      ba00000000     mov edx, 0
│           0x55bdc35eb2d9      4889c6         mov rsi, rax
│           0x55bdc35eb2dc      bf04000000     mov edi, 4
│           0x55bdc35eb2e1      e8fafdffff     call sym.imp.sigaction
└           0x55bdc35eb2e6      0f0b           ud2
            0x55bdc35eb2e8      83bd5cffffff.  cmp dword [rbp - 0xa4], 2
        ┌─< 0x55bdc35eb2ef      741a           je 0x55bdc35eb30b
        │   0x55bdc35eb2f1      0f0b           ud2
        │   0x55bdc35eb2f3      488d3d0a0d00.  lea rdi, str.._challenge__password_ ; 0x55bdc35ec004 ; "./challenge <password>"
        │   0x55bdc35eb2fa      e8d1fdffff     call sym.imp.puts       ; int puts(const char *s)
        │   0x55bdc35eb2ff      0f0b           ud2
        │   0x55bdc35eb301      b801000000     mov eax, 1
       ┌──< 0x55bdc35eb306      e92e010000     jmp 0x55bdc35eb439
       ││   ; CODE XREF from main @ +0x8e
       │└─> 0x55bdc35eb30b      0f0b           ud2
       │    0x55bdc35eb30d      488b8550ffff.  mov rax, qword [rbp - 0xb0]
       │    0x55bdc35eb314      4883c008       add rax, 8
       │    0x55bdc35eb318      488b00         mov rax, qword [rax]
       │    0x55bdc35eb31b      4889c7         mov rdi, rax
       │    0x55bdc35eb31e      e8cdfdffff     call sym.imp.strlen     ; size_t strlen(const char *s)
       │    0x55bdc35eb323      4883f80c       cmp rax, 0xc            ; 12
       │┌─< 0x55bdc35eb327      0f8505010000   jne 0x55bdc35eb432
       ││   0x55bdc35eb32d      0f0b           ud2
       ││   0x55bdc35eb32f      488b8550ffff.  mov rax, qword [rbp - 0xb0]
       ││   0x55bdc35eb336      4883c008       add rax, 8
       ││   0x55bdc35eb33a      488b00         mov rax, qword [rax]
       ││   0x55bdc35eb33d      ba03000000     mov edx, 3
       ││   0x55bdc35eb342      488d35d20c00.  lea rsi, [0x55bdc35ec01b] ; "Itz"
       ││   0x55bdc35eb349      4889c7         mov rdi, rax
       ││   0x55bdc35eb34c      e86ffdffff     call sym.imp.strncmp    ; int strncmp(const char *s1, const char *s2, size_t n)
       ││   0x55bdc35eb351      85c0           test eax, eax
      ┌───< 0x55bdc35eb353      0f85d0000000   jne 0x55bdc35eb429
      │││   0x55bdc35eb359      0f0b           ud2
      │││   0x55bdc35eb35b      488b8550ffff.  mov rax, qword [rbp - 0xb0]
      │││   0x55bdc35eb362      4883c008       add rax, 8
      │││   0x55bdc35eb366      488b00         mov rax, qword [rax]
      │││   0x55bdc35eb369      4883c003       add rax, 3
      │││   0x55bdc35eb36d      ba03000000     mov edx, 3
      │││   0x55bdc35eb372      488d35a60c00.  lea rsi, [0x55bdc35ec01f] ; "_0n"
      │││   0x55bdc35eb379      4889c7         mov rdi, rax
      │││   0x55bdc35eb37c      e83ffdffff     call sym.imp.strncmp    ; int strncmp(const char *s1, const char *s2, size_t n)
      │││   0x55bdc35eb381      85c0           test eax, eax
      │││   0x55bdc35eb383      0f8597000000   jne 0x55bdc35eb420
      │││   0x55bdc35eb389      0f0b           ud2
      │││   0x55bdc35eb38b      488b8550ffff.  mov rax, qword [rbp - 0xb0]
      │││   0x55bdc35eb392      4883c008       add rax, 8
      │││   0x55bdc35eb396      488b00         mov rax, qword [rax]
      │││   0x55bdc35eb399      4883c006       add rax, 6
      │││   0x55bdc35eb39d      ba03000000     mov edx, 3
      │││   0x55bdc35eb3a2      488d357a0c00.  lea rsi, [0x55bdc35ec023] ; "Ly_"
      │││   0x55bdc35eb3a9      4889c7         mov rdi, rax
      │││   0x55bdc35eb3ac      e80ffdffff     call sym.imp.strncmp    ; int strncmp(const char *s1, const char *s2, size_t n)
      │││   0x55bdc35eb3b1      85c0           test eax, eax
      │││   0x55bdc35eb3b3      7562           jne 0x55bdc35eb417
      │││   0x55bdc35eb3b5      0f0b           ud2
      │││   0x55bdc35eb3b7      488b8550ffff.  mov rax, qword [rbp - 0xb0]
      │││   0x55bdc35eb3be      4883c008       add rax, 8
      │││   0x55bdc35eb3c2      488b00         mov rax, qword [rax]
      │││   0x55bdc35eb3c5      4883c009       add rax, 9
      │││   0x55bdc35eb3c9      ba03000000     mov edx, 3
      │││   0x55bdc35eb3ce      488d35520c00.  lea rsi, [0x55bdc35ec027] ; "UD2"
      │││   0x55bdc35eb3d5      4889c7         mov rdi, rax
      │││   0x55bdc35eb3d8      e8e3fcffff     call sym.imp.strncmp    ; int strncmp(const char *s1, const char *s2, size_t n)
      │││   0x55bdc35eb3dd      85c0           test eax, eax
      │││   0x55bdc35eb3df      752d           jne 0x55bdc35eb40e
      │││   0x55bdc35eb3e1      0f0b           ud2
      │││   0x55bdc35eb3e3      488b8550ffff.  mov rax, qword [rbp - 0xb0]
      │││   0x55bdc35eb3ea      4883c008       add rax, 8
      │││   0x55bdc35eb3ee      488b00         mov rax, qword [rax]
      │││   0x55bdc35eb3f1      4889c6         mov rsi, rax
      │││   0x55bdc35eb3f4      488d3d300c00.  lea rdi, str.__HTB_s_n  ; 0x55bdc35ec02b ; "> HTB{%s}\n"
      │││   0x55bdc35eb3fb      b800000000     mov eax, 0
      │││   0x55bdc35eb400      e80bfdffff     call sym.imp.printf     ; int printf(const char *format)
      │││   0x55bdc35eb405      0f0b           ud2
      │││   0x55bdc35eb407      b800000000     mov eax, 0
      │││   0x55bdc35eb40c      eb2b           jmp 0x55bdc35eb439
      │││   ; CODE XREF from main @ +0x17e
      │││   0x55bdc35eb40e      0f0b           ud2
      │││   0x55bdc35eb410      b800000000     mov eax, 0
      │││   0x55bdc35eb415      eb22           jmp 0x55bdc35eb439
      │││   ; CODE XREF from main @ +0x152
      │││   0x55bdc35eb417      0f0b           ud2
      │││   0x55bdc35eb419      b800000000     mov eax, 0
      │││   0x55bdc35eb41e      eb19           jmp 0x55bdc35eb439
      │││   ; CODE XREF from main @ +0x122
      │││   0x55bdc35eb420      0f0b           ud2
      │││   0x55bdc35eb422      b800000000     mov eax, 0
      │││   0x55bdc35eb427      eb10           jmp 0x55bdc35eb439
      │││   ; CODE XREF from main @ +0xf2
      └───> 0x55bdc35eb429      0f0b           ud2
       ││   0x55bdc35eb42b      b800000000     mov eax, 0
       ││   0x55bdc35eb430      eb07           jmp 0x55bdc35eb439
       ││   ; CODE XREF from main @ +0xc6
       │└─> 0x55bdc35eb432      0f0b           ud2
       │    0x55bdc35eb434      b800000000     mov eax, 0
       │    ; XREFS: CODE 0x55bdc35eb306  CODE 0x55bdc35eb40c  CODE 0x55bdc35eb415  CODE 0x55bdc35eb41e  CODE 0x55bdc35eb427  CODE 0x55bdc35eb430  
       └──> 0x55bdc35eb439      488b4df8       mov rcx, qword [rbp - 8]
            0x55bdc35eb43d      6448330c2528.  xor rcx, qword fs:[0x28]
            0x55bdc35eb446      7405           je 0x55bdc35eb44d
            0x55bdc35eb448      e8b3fcffff     call sym.imp.__stack_chk_fail
            ; CODE XREF from main @ +0x1e5
            0x55bdc35eb44d      c9             leave
            0x55bdc35eb44e      c3             ret
            0x55bdc35eb44f      90             nop

```



[^1]: 

[^2]: 
