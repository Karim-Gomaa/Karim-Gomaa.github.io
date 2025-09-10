---
title: "Arab Security Cyber WarGames Qualification phase 2025"
classes: wide
header:
  teaser: /assets/images/CTF/ASCWG%20Quals/ASCWG.png
ribbon: Red
description: "Writeup for ASCWG Reverse challenges ..."
categories:
  - CTF
toc: true
---

Hello guys, I’ve had the honour to write the challenges for Arab Security Cyber WarGames (ASCWG). This is the write-up for the Qualifications round’s challenges. Let’s just jump right in.

# The Leaky Configuration

### Description

***`Even simple trick can give you a headache`***

#### <span style="color: red;">[challenge link](https://drive.google.com/file/d/1Zyc_p-4-PhL2_BUufpTKXaihxZiEQ9kH/view?usp=drive_link)</span>

### Initial Assessment

The challenge file is a 64-bit PE32+ executable for Windows. Initial attempts to run the file result in a console window briefly appearing before closing. Executing it from the command line reveals its proper usage: `chall.exe <flag>`. However, providing a random flag yields no output, suggesting a more complex internal validation process. Static analysis proves unhelpful, as the binary is stripped and contains no revealing strings.

### Bypassing Anti-Debugging

Loading the executable into a disassembler like IDA shows that the `main` function checks for the correct number of command-line arguments before proceeding. The primary logic involves over 40 functions that manipulate the user's input flag. Analysis indicated that the most critical operations occur at runtime, specifically within the final function call, `sub_403930`.

An attempt to set breakpoints at the beginning of `main` and at `sub_403930` failed, with the program terminating prematurely. This behaviour pointed to an anti-debugging check occurring before the `main` function is reached. The likely culprits were a TLS (Thread Local Storage) callback or an `initterm` function calling `exit()`.

To locate the anti-debugging code, cross-references to the `exit` function were examined.

![Cross-references to the exit function](/assets/images/CTF/ASCWG%20Quals/Exit%20xref.png)

This revealed three calls to `exit`. By placing breakpoints at these locations, the anti-debugger trap could be isolated. The check was identified as a common implementation that inspects the `BeingDebugged` flag in the Process Environment Block (PEB).

```
sub_40B900 proc near
...
mov     rax, gs:60h
cmp     byte ptr [rax+2], 0
jnz     short loc_40B937
...
retn

loc_40B937:
xor     ecx, ecx
call    exit
sub_40B900 endp
```

![The anti-debugger check code](/assets/images/CTF/ASCWG%20Quals/Anti-debugger%20check.png)

### Deconstructing the Encryption Routine

With the anti-debugger check bypassed, the core function `sub_403930` could be analysed. This function is responsible for a multi-stage encryption process.

The process unfolds as follows:

* **Key Generation**: The function first decrypts four internal keys: "EvadeDebuuger", "RuntimeMisslead", "SEHisAwesome", and "Cyber_Assassin!!".

* **XOR Encryption**: The user-provided input is then encrypted by performing a series of XOR operations with the first three keys in sequence.

* **RC5 Encryption**: The result of the XOR operations is subsequently encrypted using the RC5 algorithm, with the fourth key ("Cyber_Assassin!!") serving as the encryption key.

* **Final Transformation**: Each character of the RC5 encrypted output is manipulated by adding 16 and then XORing the result with `0xAA`.

### The Final Validation

The transformed data is passed to a final function that performs two distinct checksum calculations on the input and compares the results against the checksums of a hard-coded, pre-encrypted flag. If the checksums match, the program prints a success message.

### The Solution

By reversing this entire process—inverting the final transformation, decrypting the RC5 ciphertext, and reversing the multi-layer XOR operations—the correct input flag was recovered.

**Flag:** `ASCWG{Ev3n_S1mple_X0r_C@n_be_Tricky_and_form_R3@l_ch@ll3ng3_Whenused_by_Cyber_Assassin}`
![The anti-debugger check code](/assets/images/CTF/ASCWG%20Quals/success%20message.png)



# Nightmare

### Description

***`The IR Team thinks this file is stealing data. Can you find the folder path and C2?`***

**flag format** ***`{IP:Port_FilePath}`***

**<span style="color: red;"> Note!!</span>**
`This file is real malware sample, don't try to run on you host machine.`

#### <span style="color: red;">[challenge link](https://drive.google.com/file/d/1u0o0jOuVHoF37y6_RmMmb9_RTX43IGmo/view?usp=drive_link)</span>

I would like to mention that this challenge is insanely difficult and needs a skilled player to solve. In this walk-through, I am not going to solve the challenge; I will just provide the main idea, but you will need to get your hands dirty to fully solve the challenge.

### Scenario

```
After an incident, the DFIR team recovered a file from the hard disk; they believed this was what had initiated the infection. As a Malware Analyst, you are asked to analyse the file and find the C2, Port, and the dropped 2nd-stage full path.
```

### Solution Guide

Nightmare is recognised as `data`; the file utility can’t find out its arch.

That’s logic, as the file is supposed to be retrieved from a memory dump after DF investigation, and the file is corrupted.

The only way to deal with the file is static analysis.

Again, I will choose IDA to disassemble the file. On examining strings, there are 2 useful strings, the first one “DOS mode.\r\r\n$” which means this can be an executable Windows binary, the second one “IsDebuggerPresent” which can lead to the main function.

Trying to trace the file, there are no imports, exports or cross-references, so no solution except manually examining each function.

Spending enough time analysing the code, you will find that only 10 ~ 15 functions are important. On analysing the file, a skilful reverse engineer and experienced malware analyst would recognise that the file is performing some form of API Hashing. At this point, you remember the flag format `{IP: Port_FilePath}`, so you think that you need to resolve all Hashes in the file and determine which API is useful for your case.

The first trick is that you need to reimplement the whole resolution function, as the algorithm is not reversible. The second one is that you must encrypt all APIs in the `C:\Windows\System32` directory, as you can’t just try to traverse the PEB and find the loaded module; this way will not give you any network API hash.

The final stage is just to locate the target hashes:

- `0x778d1c36` – WININET.DLL  
- `0x831034c4` – InternetConnectW  
  *`This will take the IP and port as an argument.`*

- `0x387c4f86` – KERNEL32.DLL  
- `0xa5854b25` – CreateFileW  
  *`And this will take the file path as an argument.`*

The last thing to note is that all arguments are obfuscated using a simple XOR operation (four-character-length hard-coded key) and a sum operation.

Note that the hashing algorithm and Deobfuscation function are hard-coded; you just need to locate and deeply understand them and then reimplement them.

### The Solution
**Flag:** `ASCWG{198.4.109.11:5555_C:\Users<User>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\}`

#### <span style="color: red;">[The Leaky Configuration solver](https://github.com/Karim-Gomaa/Scripts/tree/main/The_Leaky_Configuration)</span>
#### <span style="color: red;">[Nightmare Solvers](https://github.com/Karim-Gomaa/Scripts/tree/main/Nightmare)</span>

#### Written by
## *Karim Gomaa*





