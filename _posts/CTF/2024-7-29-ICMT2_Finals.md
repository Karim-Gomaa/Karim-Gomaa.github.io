---
title: "ICMT CSC2 Finals"
classes: wide
header:
  teaser: /assets/images/CTF/ICMT/icmt.png
ribbon: Red
description: "Writeup for ICMT Reverse challenges ..."
categories:
  - CTF
toc: true
---

# OperationQak

**Description**
OperationQak is an easy challenge "The notorious OperationQak is planning yet another phishing attack!"

**OperationQak** is a PE32+ executable (console) x86-64, for MS Windows.

There were no interesting strings so let's run and see.

running it found a simple message asking for the flag:

![dynamic](/assets/images/CTF/ICMT_Finals/Qak/run.png)

I tried some random flag, but the screen just closed.

So it's **IDA time ;)**.

As expected from an easy challenge it has been straight through:


![first look](/assets/images/CTF/ICMT_Finals/Qak/ASM.png)

you can find easily in the main that there is a **string compare** function so it simply takes the input and the flag and compares them. So set Breakpoint and check the passed argument and here your flag resides in the memory:

![first look](/assets/images/CTF/ICMT_Finals/Qak/flag.png)


rerun the debugger and pass the right flag then you have the flag format to submit:
![first look](/assets/images/CTF/ICMT_Finals/Qak/success.png)


# SimpleObfuscator

**Description**
SimpleObfuscator is a medium challenge "Reverse the obfuscation and find your flag!"

**SimpleObfuscator** is a  PE32 executable (console) Intel 80386 Mono/.Net assembly, for MS Windows.

the first thing I thought about as this challenge is .NET is to use **dnspy**.
once loaded to dnspy you have the source code and you find the program arch:

![first look](/assets/images/CTF/ICMT_Finals/Simple/structure.png)

as you see there are two classes one of them is a pure user-defined and also obfuscated. The program class contains the main.

The main only has one call to strange function:

![first look](/assets/images/CTF/ICMT_Finals/Simple/main.png)

looking at this function from an abstract view you see it contains 3 parts:

![first look](/assets/images/CTF/ICMT_Finals/Simple/abs_view.png)

1- Block number one contains some strings deobfuscation and is not interesting.

2- Block number 2 is anti-debugging tech which will be examined further.

3- The last block is our treasure where the flag is but in encrypted form so it would be very easy to just debug the process and see the flag value.

let's examine 2 and 3 in detail.

![first look](/assets/images/CTF/ICMT_Finals/Simple/anti.png)

The program protects itself with two methods:

1- `IsDebuggerPresent()` which checks for debuggers and sets the boolean value `flag`.

2- `ProcessName` simply loops on all the processes in the system and then checks if there is any process with the program name **"SimpleObfuscator"**

If one of those flags matches the true case then the process is killed and you can't debug.

so let's set two break points at `flag` and `flag2` and debug once one of them is true just manually change it to false :).
Now we are ready but let's see how the check function operates.

![first look](/assets/images/CTF/ICMT_Finals/Simple/flag%20processing.png)

As you see at number one the bool to the true case which will print the flag format, number two is where the two strings `text` and `b` are checked but number three shows that text is our input so the flag is the variable `b`.

running the debugger with the behaviour discussed before and looking at the `b` var value you see the flag:

![first look](/assets/images/CTF/ICMT_Finals/Simple/flag.png)

when the program takes input in the next line I pass this value and the magic starts.

![first look](/assets/images/CTF/ICMT_Finals/Simple/success.png)


# Doma 

**Description**
Doma is a Hard challenge "Welcome to the infinity castle."

**Doma** is a PE32+ executable (console) x86-64, for MS Windows.

so as they say it's boss fight time ;(

running the file only a poor message asking for the flag, once I entered the random flag the program closed.

![run](/assets/images/CTF/ICMT_Finals/doma/run.png)

to be honest basic analysis has been disappointing so I moved through with code review.

opening ida it could not load any graph so I decided to use a cutter this time.

![cutter](/assets/images/CTF/ICMT_Finals/doma/Cutter.png)

first, I tried to search for printed messages and found them stored in strings -don't get really happy- but there is no cross reference for it so now I need just find how the strings are accessed, where the input is stored and how it checks for the flag.

first, I see from the cutter that the app compares `rax` with some values and then decides how to behave.

Now I think it's a good time to go dynamically with the IDA debugger. and search for our three targets as the code is hard to read statically.

I set a breakpoint at the main and found it simple to debug but time-consuming thanks I found the printed message so now I understand how does it access strings.

![str](/assets/images/CTF/ICMT_Finals/doma/str_acc.png)

it simply takes some address subtracts some value and then the wanted string address is stored at rcx, matching this with what I saw in cutter I found that the `rax` moves to the location to load the message to be printed and calculate those addresses found that rax 3 is the case leading to success message let's keep going.

from here I got to the place that stores the input but I detected that the program trying to hide the real code and modified the instruction pointer to keep moving.

![store](/assets/images/CTF/ICMT_Finals/doma/store.png)

at number 1 the program accesses the input char by char.
at 2, it performs some arithmetic operations to encrypt it.

![check](/assets/images/CTF/ICMT_Finals/doma/check.png)

at 3 it compares the encrypted form stored at `edx` with dw stored at memory `[r9]`.

So `[r9]` store our flag encrypted form.

![r9](/assets/images/CTF/ICMT_Finals/doma/flag%20raw.png)

I found it impossible to reverse the process so I thought to run the algorithm against all chars by the following code

```c++
#include <iostream>
#include <iomanip>
#include <cstdint>

uint32_t encryptCharacter(uint8_t input) {
    uint32_t encrypted;
 __asm {
 movzx   edx, input
 inc     edx
 mov     eax, edx
 shl     eax, 0xA
        xor eax, edx
 mov     ecx, eax
 shr     ecx, 1
 push    eax
        not eax
 sub     ecx, eax
 pop     eax
 sub     ecx, 1
 lea     edx, [ecx * 8]
 xor edx, ecx
 mov     ecx, edx
 shr     ecx, 5
 push    edx
 not edx
 sub     ecx, edx
 pop     edx
 sub     ecx, 1
 mov     edx, ecx
 shl     edx, 4
 xor edx, ecx
 mov     eax, edx
 shr     eax, 0x11
 push    edx
 not edx
 sub     eax, edx
 pop     edx
 sub     eax, 1
 mov     edx, eax
 mov     ecx, eax
 shr     ecx, 6
 and edx, 0x7F
 shl     edx, 0x13
 xor edx, ecx
 mov     ecx, eax
 shl     ecx, 0x19
 xor ecx, eax
 push    ecx
 not ecx
 sub     edx, ecx
 pop     ecx
 sub     edx, 1
 mov     encrypted, edx
 }
    return encrypted;
}

int main() {
    for (uint8_t i = 0; i < 128; ++i) {
        uint32_t encrypted = encryptCharacter(i);
        std::cout << "Character: " << (char)i << ", Encrypted: 0x"
            << std::hex << std::setw(8) << std::setfill('0') << encrypted << std::dec << std::endl;
 }
    return 0;
}

```
the code gave me a dictionary for each character.
output:

```
Character: , Encrypted: 0x5553595a
Character: ☺, Encrypted: 0x9e76b331
Character: ☻, Encrypted: 0xd55108fe
Character: ♥, Encrypted: 0x1a756653
Character: ♦, Encrypted: 0x6fc7ed71
Character: ♣, Encrypted: 0xb8ea1184
Character: ♠, Encrypted: 0xf3da4c8b
Character: , Encrypted: 0x597accb8
Character:, Encrypted: 0x49391e10
Character:      , Encrypted: 0x1267dc85
Character:
, Encrypted: 0xefd4c853
Character:
, Encrypted: 0x6fdc2309
Character:
, Encrypted: 0x4537f500
, Encrypted: 0xb90c9984
Character: , Encrypted: 0xc32c1f25
Character: , Encrypted: 0xb2f59971
Character: ►, Encrypted: 0x22bd3bb9
Character: ◄, Encrypted: 0xa4ba3db0
Character: ↕, Encrypted: 0x8c6d33d2
Character: ‼, Encrypted: 0x085fb8fc
Character: ¶, Encrypted: 0xb70a0355
Character: §, Encrypted: 0x2cf99154
Character: ▬, Encrypted: 0xcd7e7770
Character: ↨, Encrypted: 0xe1c04614
Character: ↑, Encrypted: 0x6be9cf65
Character: ↓, Encrypted: 0x3d3fec64
Character: →, Encrypted: 0xc35d6125
Character: encrypted: 0x92a9331b
Character: ∟, Encrypted: 0x84708464
Character: ↔, Encrypted: 0x96a83ed7
Character: ▲, Encrypted: 0xcf8e715b
Character: ▼, Encrypted: 0x437332d3
Character:  , Encrypted: 0xa2f7afb1
Character: !, Encrypted: 0x9aca7617
Character: ", Encrypted: 0xc362b39b
Character: #, Encrypted: 0x477c7b61
Character: $, Encrypted: 0xd9b9f3b2
Character: %, Encrypted: 0xcbaa677d
Character: &, Encrypted: 0x2b22472b
Character: ', Encrypted: 0x10bf71f8
Character: (, Encrypted: 0xc791a1ef
Character: ), Encrypted: 0x3d640716
Character: *, Encrypted: 0x621f4884
Character: +, Encrypted: 0x59f322a8
Character: ,, Encrypted: 0x704fa167
Character: -, Encrypted: 0xee54ef8e
Character: ., Encrypted: 0x009e2b73
Character: /, Encrypted: 0xc1888c2a
Character: 0, Encrypted: 0xf2475372
Character: 1, Encrypted: 0xea1b9f56
Character: 2, Encrypted: 0x8288d321
Character: 3, Encrypted: 0x9d07d8da
Character: 4, Encrypted: 0xcbba9c51
Character: 5, Encrypted: 0xb78ac2e7
Character: 6, Encrypted: 0xcfef5b0b
Character: 7, Encrypted: 0x45e26648
Character: 8, Encrypted: 0x930b26e3
Character: 9, Encrypted: 0xdc310a37
Character: :, Encrypted: 0x60212047
Character: ;, Encrypted: 0x29507dae
Character: <, Encrypted: 0xde176e5a
Character: =, Encrypted: 0x4dece416
Character: >, Encrypted: 0x2b4fb7a1
Character: ?, Encrypted: 0x6a766598
Character: @, Encrypted: 0x973fc315
Character: A, Encrypted: 0x174760d3
Character: B, Encrypted: 0x37b76e13
Character: C, Encrypted: 0x3194ec2e
Character: D, Encrypted: 0x84db9614
Character: E, Encrypted: 0xb99d68d8
Character: F, Encrypted: 0x41dedf66
Character: G, Encrypted: 0x8ef8f6c3
Character: H, Encrypted: 0xc7dd6143
Character: I, Encrypted: 0xc1bbe6ec
Character: J, Encrypted: 0x33e1ad79
Character: K, Encrypted: 0xb5dccf0c
Character: L, Encrypted: 0x072fd51f
Character: M, Encrypted: 0x480c8dcd
Character: N, Encrypted: 0xa173825e
Character: O, Encrypted: 0x050ee3e2
Character: P, Encrypted: 0x78d77863
Character: Q, Encrypted: 0x3ffb43b9
Character: R, Encrypted: 0x33af2d13
Character: S, Encrypted: 0x7ac80e2c
Character: T, Encrypted: 0x70a549c3
Character: U, Encrypted: 0xf70690a0
Character: V, Encrypted: 0x0962e22e
Character: W, Encrypted: 0xd66e4562
Character: X, Encrypted: 0xe8b4cda3
Character: Y, Encrypted: 0xb5f743be
Character: Z, Encrypted: 0x6cdb9395
Character: [, Encrypted: 0xd8a9df1d
Character: \, Encrypted: 0xa1a9628e
Character: ], Encrypted: 0xb2145848
Character: ^, Encrypted: 0xa9f9db7a
Character: _, Encrypted: 0x7f111854
Character: `, Encrypted: 0x93642e87
Character: a, Encrypted: 0x115ea782
Character: b, Encrypted: 0x70c1a0e1
Character: c, Encrypted: 0xf4c73ebf
Character: d, Encrypted: 0x030fe761
Character: e, Encrypted: 0xd659a5a8
Character: f, Encrypted: 0x052181b4
Character: g, Encrypted: 0x199fb1a6
Character: h, Encrypted: 0x48543605
Character: i, Encrypted: 0x46453a02
Character: j, Encrypted: 0x7f591866
Character: k, Encrypted: 0x4ea585c0
Character: l, Encrypted: 0x9fe23cf9
Character: m, Encrypted: 0x8da6b692
Character: n, Encrypted: 0xce94c074
Character: o, Encrypted: 0x6f54cc82
Character: p, Encrypted: 0xa013ceb3
Character: q, Encrypted: 0x56e64d5e
Character: r, Encrypted: 0xc05f9a27
Character: s, Encrypted: 0xb66a1470
Character: t, Encrypted: 0x6526c7c9
Character: u, Encrypted: 0x1192413c
Character: v, Encrypted: 0x999e0a64
Character: w, Encrypted: 0x54a8fb5d
Character: x, Encrypted: 0x0f827b9f
Character: y, Encrypted: 0xa9f6de34
Character: z, Encrypted: 0x858ac814
Character: {, Encrypted: 0x7f69c81e
Character: |, Encrypted: 0x303097c9
Character: }, Encrypted: 0xabef6fef
Character: ~, Encrypted: 0xdcad763f
Character: , Encrypted: 0xd6f4cb32
```
mapping with a flag stored in memory at [r9] I got the flag.


`EGCERT{6d944e4c857b10dc9abb20e9550334503e996cd4}`

testing with the app itself:

![suc](/assets/images/CTF/ICMT_Finals/doma/success.png)

and here is where the happy moments come.

![sub](/assets/images/CTF/ICMT_Finals/doma/submission.png)

stay tuned for more in Reverse Engineering and Malware Analysis.

#### Written by

## *Karim Gomaa*

