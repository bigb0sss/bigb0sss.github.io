---
title: SLAE32 - Assignment#7 [Custom Cryptor]
author: bigb0ss
date: 2021-04-27 23:53:00 +0800
categories: [SLAE32, Assignment_7_Cryptor]
tags: [slae32, assembly, x86, Custom Cryptor]
image: /assets/img/post/slae32/slae32.png
---

<b>This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:</b>

<b>http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/</b>

<b>Student ID: SLAE-1542</b>

[SLAE32 Assignemt#7 Github](https://github.com/bigb0sss/SLAE32)

# Assignement #7 
* Create a custom crypter like the one shown in the "crypters" video
* Free to use any existing encrpytion schema
* Can use any programming language

# What is Crypter?
A crypter is a software that can encrypt, obfuscate and manipulate malware or a RAT (Remote Access Tool) tool to potentially bypass security products such as antiviruses. 

# Encryption Process
For creating a simple crpyter, I will be using the following process:
* Generate a key with random characters & seed (32 characters hard-coded as of now)
* AES Encrypt #1 - Initialize the state array with the block data using the key
* AES Encrypt #2 - Generate IV (Initialization Vector) using block size + length of shellcode
* AES Encrypt #3 - Run the encrpytion process using the block and IV
* Base64 encode the results

# Decryption Process
* Base64 decode the results
* AES Decrypt #1 - Initialize the state array with the block data using the key 
* AES Decrypt #2 - Check if length IV is equal to the block size
* AES Decrypt #3 - Run the decryption process using the block and IV
* Return the decrypted string

I chose `Go` programming language to create the crypter. 

## Generate Key Code
```go
// Random Key Generator (128 bit)
var chars = []rune("0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz")

func randKeyGen(n int) string {
	charSet := make([]rune, n)
	for i := range charSet {
		charSet[i] = chars[math.Intn(len(chars))]
	}
	return string(charSet)
}
```

## Encryption Code
```go
// Encrpyt: Original Text --> Add IV --> Encrypt with Key --> Base64 Encode
func Encrypt(key []byte, text []byte) string {
	block, err := aes.NewCipher(key)
	if err != nil {
		panic(err)
	}

	// Creating IV
	ciphertext := make([]byte, aes.BlockSize+len(text))
	iv := ciphertext[:aes.BlockSize]
	if _, err := io.ReadFull(rand.Reader, iv); err != nil {
		panic(err)
	}

	// Encrpytion Process
	stream := cipher.NewCFBEncrypter(block, iv)
	stream.XORKeyStream(ciphertext[aes.BlockSize:], text)

	// Encode to Base64
	return base64.URLEncoding.EncodeToString(ciphertext)
}
```

## Decryption Code
```go
// Decrypt: Encrypted Text --> Base64 Decode --> Decrypt with IV and Key
func Decrypt(key []byte, encryptedText string) string {
	ciphertext, _ := base64.URLEncoding.DecodeString(encryptedText)

	block, err := aes.NewCipher(key)
	if err != nil {
		panic(err)
	}

	// Using IV
	iv := ciphertext[:aes.BlockSize]

	// Checking BlockSize = IV
	if len(iv) != aes.BlockSize {
		panic("[Error] Ciphertext is too short!")
	}

	ciphertext = ciphertext[aes.BlockSize:]

	// Decryption Process
	stream := cipher.NewCFBDecrypter(block, iv)
	stream.XORKeyStream(ciphertext, ciphertext)

	return string(ciphertext)
}
```

I put everything together and created the final code: `goBase64Crypter.go`:
```go
package main

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/base64"
	"fmt"
	"io"
	math "math/rand"
	"os"
	"time"
)

// Random Key Generator (128 bit)
var chars = []rune("0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz")

func randKeyGen(n int) string {
	charSet := make([]rune, n)
	for i := range charSet {
		charSet[i] = chars[math.Intn(len(chars))]
	}
	return string(charSet)
}

// Encrpyt: Original Text --> Add IV --> Encrypt with Key --> Base64 Encode
func Encrypt(key []byte, text []byte) string {
	block, err := aes.NewCipher(key)
	if err != nil {
		panic(err)
	}

	// Creating IV
	ciphertext := make([]byte, aes.BlockSize+len(text))
	iv := ciphertext[:aes.BlockSize]
	if _, err := io.ReadFull(rand.Reader, iv); err != nil {
		panic(err)
	}

	// Encrpytion Process
	stream := cipher.NewCFBEncrypter(block, iv)
	stream.XORKeyStream(ciphertext[aes.BlockSize:], text)

	// Encode to Base64
	return base64.URLEncoding.EncodeToString(ciphertext)
}

// Decrypt: Encrypted Text --> Base64 Decode --> Decrypt with IV and Key
func Decrypt(key []byte, encryptedText string) string {
	ciphertext, _ := base64.URLEncoding.DecodeString(encryptedText)

	block, err := aes.NewCipher(key)
	if err != nil {
		panic(err)
	}

	// Using IV
	iv := ciphertext[:aes.BlockSize]

	// Checking BlockSize = IV
	if len(iv) != aes.BlockSize {
		panic("[Error] Ciphertext is too short!")
	}

	ciphertext = ciphertext[aes.BlockSize:]

	// Decryption Process
	stream := cipher.NewCFBDecrypter(block, iv)
	stream.XORKeyStream(ciphertext, ciphertext)

	return string(ciphertext)
}

func main() {

	if len(os.Args) != 2 {
		fmt.Println("[INFO] Usage:", os.Args[0], "<Shellcode>")
		return
	}

	// Getting Original Shellcode as an Argument
	originalText := string(os.Args[1])
	fmt.Printf("[INFO] Original Text\t: %v\n", originalText)

	// Creating a Random Seed
	math.Seed(time.Now().UnixNano())

	// Generating a Random Key
	key := []byte(randKeyGen(32)) //Key Size: 16, 32
	fmt.Printf("[INFO] Random Key\t: %v\n", string(key))

	// Encrypt value to base64
	encryptedText := Encrypt(key, []byte(originalText))
	fmt.Printf("[INFO] Encrypted Base64\t: %v\n", encryptedText)

	// Decrypt base64 to original value
	text := Decrypt(key, encryptedText)
	fmt.Printf("[INFO] Decrypted Text\t: %v\n", text)
}

```

# Final Testing
I tested the crypter with a string "bigb0ss". It worked as expected to encrypt and decrypt the string, and the key was randomly generated as well. 

```bash
# go run goBase64Crypter.go "bigb0ss"
[INFO] Original Text    : bigb0ss
[INFO] Random Key       : Fnf4Ml3Z6MhP11qMQ0zehdp9dEj5j5Pn
[INFO] Encrypted Base64 : JpbQCdofLYb7aceWzkb2i0xNakO8aAg=
[INFO] Decrypted Text   : bigb0ss

# go run goBase64Crypter.go "bigb0ss"
[INFO] Original Text    : bigb0ss
[INFO] Random Key       : CQroxMZRxOZlXWlU5v6yQdpzz8ibYUhp
[INFO] Encrypted Base64 : PJHvYqoEK1K5thBrPMIRqndRI8kH0f4=
[INFO] Decrypted Text   : bigb0ss
```

I used the `/bin/sh` shellcode to run the crypter again.
```bash
# python compilerX86.py -f bin-sh

[+] Assemble: bin-sh.nasm
[+] Linking: bin-sh.o
[+] Shellcode: "\xeb\x1a\x5e\x31\xdb\x88\x5e\x07\x89\x76\x08\x89\x5e\x0c\x8d\x1e\x8d\x4e\x08\x8d\x56\x0c\x31\xc0\xb0\x0b\xcd\x80\xe8\xe1\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68\x41\x42\x42\x42\x42\x43\x43\x43\x43"
```

```bash
# go run goBase64Crypter.go "\xeb\x1a\x5e\x31\xdb\x88\x5e\x07\x89\x76\x08\x89\x5e\x0c\x8d\x1e\x8d\x4e\x08\x8d\x56\x0c\x31\xc0\xb0\x0b\xcd\x80\xe8\xe1\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68\x41\x42\x42\x42\x42\x43\x43\x43\x43"
[INFO] Original Text    : \xeb\x1a\x5e\x31\xdb\x88\x5e\x07\x89\x76\x08\x89\x5e\x0c\x8d\x1e\x8d\x4e\x08\x8d\x56\x0c\x31\xc0\xb0\x0b\xcd\x80\xe8\xe1\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68\x41\x42\x42\x42\x42\x43\x43\x43\x43
[INFO] Random Key       : XhVkv3RK89cTGx63Cv7YPkcwdA3v1qWE
[INFO] Encrypted Base64 : blSC0D5MO-tFgq9afP-R6Jb5lcQNIo3Z4LTeSyl3tJALs_Mq_W8IYDlkZ3MOro-YjknSad6HzLEmqlQLgRkdPQSrCTGGq_NcWoPGbvCqIpey1wEqfiPnxwiljlfGTxBzXXV-UQzkiSqDidgYwVaND2qCE66_vHuMW7Chq_CP62BPjGvOozVoz_uTUs9POF0jW6Y2qXY9e_Ykw5wvly_aKPOVxWPIPdCctn3uFF7XXoUD50KmEa-VLxadgG2fx02ky2iuKvZF0AvABxtOCL1pQio1NDM=
[INFO] Decrypted Text   : \xeb\x1a\x5e\x31\xdb\x88\x5e\x07\x89\x76\x08\x89\x5e\x0c\x8d\x1e\x8d\x4e\x08\x8d\x56\x0c\x31\xc0\xb0\x0b\xcd\x80\xe8\xe1\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68\x41\x42\x42\x42\x42\x43\x43\x43\x43
```

I used the following `C` code to run the decrypted shellcode:

```c
#include<stdio.h>
#include<string.h>

unsigned char code[] = \ 
"\xeb\x1a\x5e\x31\xdb\x88\x5e\x07\x89\x76\x08\x89\x5e\x0c\x8d\x1e\x8d\x4e\x08\x8d\x56\x0c\x31\xc0\xb0\x0b\xcd\x80\xe8\xe1\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68\x41\x42\x42\x42\x42\x43\x43\x43\x43";
main()
{
printf("Shellcode Length:  %d", strlen(code));
	int (*ret)() = (int(*)())code;
	ret();
}
```

Compile and run it:

```bash
# gcc -fno-stack-protector -z execstack -o shellcode shellcode.c -w

# ./shellcode 
# id
uid=0(root) gid=0(root) groups=0(root)
# exit
```

<b>This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:</b>

<b>http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/</b>

<b>Student ID: SLAE-1542</b>

[SLAE32 Assignemt#7 Github](https://github.com/bigb0sss/SLAE32)