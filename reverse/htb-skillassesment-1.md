---
platform: hack the box
category: reverse
difficulty: easy
date: 2026-11-16
---
# Task 1 - Skill assessment Assembly module

---
## Context
We have a suspicious binary file that is loading an encoded shellcode onto the stack and storing the `xor` decoding key in `rbx`. 

---
## Goal
We need to decode the shellcode once it is loaded onto the stack and run the shellcode to get the flag.  

---
## Walkthrough
First, we need to disassemble the binary to get the assembly code. 
I used the `disas` command of `gdb` to do so. 

Here it is :  
```asm
mov rax, 0xa284ee5c7cde4bd7
push rax
mov rax,0x935add110510849a
push rax
mov rax,0x10b29a9dab697500
push rax
mov rax, 0x200ce3eb0d96459a
push rax
mov rax, 0xe64c30e305108462
push rax
mov rax, 0x69cd355c7c3e0c51
push rax
mov rax, 0x65659a2584a185d6
push rax
mov rax, 0x69ff00506c6c5000
push rax
mov rax, 0x3127e434aa505681
push rax
mov rax, 0x6af2a5571e69ff48
push rax
mov rax, 0x6d179aaff20709e6
push rax
mov rax, 0x9ae3f152315bf1c9
push rax
mov rax, 0x373ab4bb0900179a
push rax
mov rax, 0x69751244059aa2a3
push rax
mov rbx, 0x2144d2144d2144d2
```

We can see that the encoded shellcode is pushed onto the stack 8 bytes by 8 bytes. That means that, in order to decode it, we need to decode each 8-byte block on the stack. 

This will not be hard because XOR is its own inverse. Applying the same key again decodes the encoded block. Thus, we will take the first block which is pointed by `rsp` and decode it, then decode the block pointed by `rsp + 8`, then `rsp + 16` and so on. 

To do so, we just need to add a loop at the end of the code that implements this logic.

Now, we should be able to execute the shellcode somehow. 
At first, I tried to simply make the code jump to `rsp` so the shellcode can be directly executed from the stack. But, unfortunately, I wasn't able to make this jump work (the reason is explained in **Beyond the solution**). 

The other option was crafting the shellcode by hand. To do so, I retrieved each block from the stack using `gdb/gef` and the following command, displaying the first 14 hex `qwords` of the stack:
```bash
gef➤ x/14gx $rsp  
0x7fffffffddc0: 0x4831c05048bbe671 0x167e66af44215348  
0x7fffffffddd0: 0xbba723467c7ab51b 0x4c5348bbbf264d34  
0x7fffffffdde0: 0x4bb677435348bb9a 0x10633620e7711253  
0x7fffffffddf0: 0x48bbd244214d14d2 0x44214831c980c104  
0x7fffffffde00: 0x4889e748311f4883 0xc708e2f74831c0b0  
0x7fffffffde10: 0x014831ff40b70148 0x31f64889e64831d2  
0x7fffffffde20: 0xb21e0f054831c048 0x83c03c4831ff0f05
```

Here we can find all the decoded blocks of the shellcode. 
Concatenated, the shellcode becomes : 
```
4831c05048bbe671167e66af44215348bba723467c7ab51b4c5348bbbf264d344bb677435348bb9a10633620e771125348bbd244214d14d244214831c980c1044889e748311f4883c708e2f74831c0b0014831ff40b7014831f64889e64831d2b21e0f054831c04883c03c4831ff0f05
```

---
## Solution
Here is the decoding loop I added at the end of the existing code:
```asm
mov rcx, 0x0E
xor r8, r8
decode_loop:
	xor [rsp + r8], rbx
	add r8, 8
	loop decode_loop
```

Also, we will use the following custom Python script executing a shellcode :
```python
import sys
from pwn import *

context(os="linux", arch="amd64", log_level="error")

run_shellcode(unhex(sys.argv[1])).interactive()
```

---
## Result
Then we use this command:
```bash
(pwn_venv) ➜ python loader.py 4831c05048bbe671167e66af44215348bba723467c7ab51b4c5348bbbf264d344bb677435348bb9a10633620e771125348bbd244214d14d244214831c980c1044889e748311f4883c708e2f74831c0b0014831ff40b7014831f64889e64831d2b21e0f054831c04883c03c4831ff0f05
```

It will return the flag, which is : **`HTB{4553mbly_...}  `**

---
## Beyond the solution 
I struggled when I tried to jump directly to `rsp` to execute the shellcode on the stack for 2 main reasons. 

The first issue was that the shellcode bytes were written as 64-bit integers. Since x86 stores integers in little-endian, the bytes of each 8-byte block end up reversed in memory. So, when the CPU tried to execute the shellcode, everything was just nonsense. 

For example, the block at the top of the stack should read: 
```
48 31 c0 50 48 bb e6 71
```

but it is actually stored like this:
```
71 e6 bb 48 50 c0 31 48
```

The solution here is to swap the 8 bytes of each blocks inside the loop. So here we have the new loop:
```asm
decode_loop:
	mov r9, [rsp + r8]
	xor r9, rbx
	bswap r9 
	mov [rsp + r8], r9
	add r8, 8
	loop decode_loop
	
	jmp rsp              ; Here we add the jump to the top of the stack
```

The last issue occurs at runtime: by default, the stack is mapped as non-executable (NX), so the CPU refuses to execute code from it. To fix this, we add `-z execstack` to the linking command.

```bash
(pwn_venv) ➜ nasm -f elf64 loaded_shellcode_rw.asm 
(pwn_venv) ➜ ld -z execstack -o loaded_shellcode_rw loaded_shellcode_rw.o
(pwn_venv) ➜ ./loaded_shellcode_rw
HTB{4553mbly_...}
```

We can see that using this method allow us to find the flag directly without having to craft the shellcode by hand, which is not very efficient. 

---

## Takeaways

First, I understood that the way memory is displayed (for example with gdb/gef) can differ from how it’s actually stored. Now I will remember to always consider endianness. 

I also learned that the stack is non-executable by default (NX). If I encounter a similar situation where I need to execute something on the stack, I will know why it fails and that the stack has to be made executable.