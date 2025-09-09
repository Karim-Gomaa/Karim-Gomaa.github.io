---
title: "Arab Security Cyber WarGames Finals phase 2025"
classes: wide
header:
  teaser: /assets/images/CTF/ASCWG%20Quals/ASCWG.png
ribbon: Red
description: "Writeup for ASCWG Reverse challenges ..."
categories:
  - CTF
toc: true
---

Hello guys, I’ve had the honor to write the challenges for Arab Security Cyber WarGames (ASCWG). This is the writeup for the Finals round’s challenges. Let’s just jump right in.


# Lost_In_The_Dark

### Description

***`"EzioAuditoredaFirenze" had used his name to hide a secret, but he doesn't know how to reveal it can you do this job?`***

#### <span style="color: red;">[challenge link](https://drive.google.com/file/d/17b8Syb-XhG0-Q0Gp2o3MhkRBfhCDwztd/view?usp=sharing)</span>

### Initial Assessment
Upon receiving the `Bios.bin` file, the first step was to perform a basic file type identification to understand its nature. Using the `file` command, it was determined that the provided file is a data file:

```bash
ubuntu@cyber_assassin:~ $ file Bios.bin
bios.bin : data

```
Now, this first thing come in mind is that the file may contain some hidden or embedded files.

Next, `binwalk` was considered for extracting potential embedded files or identifying known file signatures within the EFI module.

```bash
ubuntu@cyber_assassin:~ $ binwalk Bios.bin
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             UEFI PI Firmware Volume, volume size: 540672, header size: 0, revision: 0, Variable Storage, GUID: FFF12B8D-7696-4C8B-85A9-2747075B4F50
540672        0x84000         UEFI PI Firmware Volume, volume size: 3440640, header size: 96, revision: 0, EFI Firmware File System v2, GUID: 8C8CE578-8A3D-4F1C-3599-896185C32DD3
540840        0x840A8         LZMA compressed data, properties: 0x5D, dictionary size: 16777216 bytes, uncompressed size: 16056464 bytes
1948388       0x1DBAE4        Microsoft executable, portable (PE)
1955780       0x1DD7C4        AES S-Box
1956036       0x1DD8C4        AES Inverse S-Box
1956712       0x1DDB68        Unix path: /home/Wolf/edk2/Build/MyPkg/DEBUG_GCC5/X64/MyPkg/Lost_in_the_dark/DEBUG/Lost_in_the_dark.dll
3981312       0x3CC000        UEFI PI Firmware Volume, volume size: 212992, header size: 96, revision: 0, EFI Firmware File System v2, GUID: 8C8CE578-8A3D-4F1C-3599-896185C32DD3
3981460       0x3CC094        Microsoft executable, portable (PE)
4041176       0x3DA9D8        Unix path: /home/Wolf/edk2/Build/OvmfX64/DEBUG_GCC5/X64/OvmfPkg/Sec/SecMain/DEBUG/SecMain.dll

```
The `binwalk` output indicated a true treasure and tell important piece of information, this file is firmware image that contains UEFI module.

Now the time comes, you can choose what you prefare but for me **UEFITool** is the perfect power for such task. Using UEFITool, you can now find many usiful information and te most important part extract `Lost_in_the_dark.efi`.

Upon receiving the `Lost_in_the_dark.efi` file, the first step was to perform a basic file type identification to understand its nature. Using the `file` command, it was determined that the provided file is an EFI application for x86-64 architecture:

```bash
ubuntu@cyber_assassin:~ $ file Lost_in_the_dark.efi
Lost_in_the_dark.efi: MS-DOS executable PE32+ executable (EFI application) x86-64, for MS Windows
```

This information confirms that the file is a 64-bit Windows Portable Executable (PE) format, specifically designed to run within a UEFI (Unified Extensible Firmware Interface) environment. This implies that standard reverse engineering tools for PE files and x86-64 architecture would be applicable.

now the second trick is binwalk round2 :) 

```bash
ubuntu@cyber_assassin:~ $ binwalk -e Lost_in_the_dark.efi
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Microsoft executable, portable (PE)
7776          0x1E60          AES S-Box
8032          0x1F60          AES Inverse S-Box
8708          0x2204          Unix path: /home/Wolf/edk2/Build/MyPkg/DEBUG_GCC5/X64/MyPkg/Lost_in_the_dark/DEBUG/Lost_in_the_dark.dll
```

The `binwalk` output indicated the presence of an AES S-Box and Inverse S-Box, which are lookup tables used in the Advanced Encryption Standard (AES) algorithm. This observation is consistent with the provided source code, which explicitly defines `sbox` and `rsbox` (inverse S-box) arrays for AES-128 implementation. More interestingly, it revealed a Unix path: `/home/Wolf/edk2/Build/MyPkg/DEBUG_GCC5/X64/MyPkg/Lost_in_the_dark/DEBUG/Lost_in_the_dark.dll`. This path suggests that the EFI module might have been built from an EDK II (EFI Development Kit II) project, and the `.dll` extension hints at a dynamically linked library, which is common for EFI modules.

However, despite using the `-e` flag for extraction, `binwalk` did not create a new directory with extracted files. This could mean that the identified components are embedded directly within the main executable and not as separate, extractable files in a way `binwalk` typically handles. Therefore, direct disassembly of the `Lost_in_the_dark.efi` file was pursued.


### Disassembly and Reverse Engineering

To analyze the EFI module's functionality, the `Lost_in_the_dark.efi` file was loaded into IDA.

The disassembly view in IDA contains the assembly code of the EFI module. A preliminary review of the assembly code was performed by examining the first few hundred lines. The code appears to be standard x86-64 assembly, with various functions and data manipulations. The presence of `call` instructions indicates calls to other functions, which could be internal or external (UEFI services).

Initial attempts to locate the flag by searching for the string "flag" (case-insensitive) within the disassembled code yielded no direct results. This is a common scenario in reverse engineering challenges, as flags are often obfuscated, encrypted, or generated dynamically.

To uncover potentially hidden strings, the `strings` utility was used to extract all printable strings from the `Lost_in_the_dark.efi` file. The output was saved to `Lost_in_the_dark.strings`:

```bash
strings Lost_in_the_dark.efi > Lost_in_the_dark.strings
```

Reviewing the `Lost_in_the_dark.strings` file revealed several interesting entries, including various UEFI status codes (e.g., "Success", "Invalid Parameter", "Not Found"), and the previously identified Unix path: `/home/Wolf/edk2/Build/MyPkg/DEBUG_GCC5/X64/MyPkg/Lost_in_the_dark/DEBUG/Lost_in_the_dark.dll`. The string "you missed the important part, try again harder\n" was also found, which, as revealed by the source code, is the message printed by the application if the key part 2 is missing or if the decryption process is not fully completed by the user.

With the source code as a guide, the disassembly analysis becomes more focused. We can now specifically look for the `EncryptedFlag` array, the `gKeyPart2VariableGuid` GUID, and the calls to `gBS->HandleProtocol` and `gRT->GetVariable`. The `EncryptedFlag` array, as defined in the source code, is a 128-byte array. In the disassembly, this array would appear as a block of initialized data. The `gKeyPart2VariableGuid` would also be present as a data structure in the `.data` section.

The `CopyMem` calls, which are used to construct the `AesKey`, can be identified by their function signatures or by tracing the memory operations around the `AesKey` variable. The `AES128::AES128_ECB_decrypt` function, which is the core decryption routine, would correspond to a significant block of assembly code, likely containing loops and calls to the AES sub-functions (InvSubBytes, InvShiftRows, InvMixColumns, AddRoundKey, and KeyExpansion). The `binwalk` output's identification of AES S-Box and Inverse S-Box at specific offsets (0x1E60 and 0x1F60) directly correlates with the `sbox` and `rsbox` arrays in the source code, confirming the AES implementation.

By cross-referencing the source code with the disassembly, we can precisely locate the `EncryptedFlag` data, the `gKeyPart2VariableGuid`, and the logic for key construction and decryption. This combined approach significantly simplifies the reverse engineering process, allowing for a more accurate understanding of the binary's functionality.


### Functionality Summary

The module implements the AES-128 ECB decryption algorithm from scratch. This includes the S-box, inverse S-box, key expansion, and the various steps of the AES decryption process (InvSubBytes, InvShiftRows, InvMixColumns, and AddRoundKey).

The `UefiMain` function serves as the entry point for the UEFI application. It contains an encrypted flag, `EncryptedFlag`, stored as a 128-byte array. The core of the challenge lies in reconstructing the 16-byte AES key used for decryption. The source code reveals that the key is constructed from two parts:

1.  **Key Part 1:** The first 8 bytes of the key are copied from the `FvFileName` field of the `MEDIA_FW_VOL_FILEPATH_DEVICE_PATH` structure. This structure is part of the loaded image protocol and contains information about the file path of the EFI module itself.

2.  **Key Part 2:** The remaining 8 bytes of the key are retrieved from a UEFI variable named "KeyPart2". This variable is associated with a specific GUID: `{ 0x12345678, 0xabcd, 0xefab, { 0x12, 0x34, 0x56, 0x78, 0x9a, 0xbc, 0xde, 0xf0 }}`.

If the "KeyPart2" variable is not found, the application prints an error message and exits. Once both parts of the key are obtained, they are concatenated to form the complete 16-byte AES key. The application then proceeds to decrypt the `EncryptedFlag` in 16-byte blocks using the constructed key and the `AES128::AES128_ECB_decrypt` function. Finally, instead of printing the decrypted flag, the application prints the message: "you missed the important part, try again harder\n". This indicates that the flag is not directly displayed and must be extracted by other means, such as by debugging the application or by manually reconstructing the key and decrypting the flag.

### Challenge Solution and Flag Discovery

With the insights gained from the source code, the challenge of finding the flag becomes a matter of extracting the necessary components from the binary and performing the decryption. The key steps are:

### Extracting Encrypted Flag

The `EncryptedFlag` array is defined in the source code as a `UINT8` array of 128 bytes. In the disassembled `Lost_in_the_dark.efi` file, this array can be located in the data section. By examining the disassembly, specifically around the `UefiMain` function, we can identify the memory location where this array is stored. The values of the `EncryptedFlag` array are:

```c
UINT8 EncryptedFlag[128] = {
  0x29, 0x83, 0x7a, 0x8f, 0xe7, 0xde, 0x0d, 0x85,
  0x5C, 0x00, 0x4C, 0x00, 0x6F, 0x00, 0x73, 0x00,
  0x45, 0x7a, 0x69, 0x6f, 0x41, 0x75, 0x64, 0x69,
  0xf1, 0x6a, 0x74, 0x12, 0x67, 0xe9, 0x69, 0x6a,
  0xb8, 0xbd, 0x91, 0x7d, 0x1e, 0xf8, 0x28, 0x0e,
  0x37, 0xfe, 0x22, 0x9e, 0x94, 0x4d, 0x53, 0x22,
  0x9b, 0x6e, 0x10, 0xf4, 0x87, 0x5e, 0xcf, 0x51,
  0xc6, 0x48, 0x8d, 0xb8, 0xe6, 0x8e, 0xa1, 0x05,
  0xcb, 0x76, 0x2f, 0xb4, 0x66, 0x2d, 0x0d, 0x03,
  0xe2, 0xa5, 0x6a, 0x2d, 0xc3, 0x92, 0xee, 0x0c,
  0xb4, 0xe8, 0x6d, 0x24, 0x27, 0x1b, 0x1f, 0xbd,
  0xc6, 0xc6, 0xed, 0xd8, 0x30, 0x95, 0x7f, 0x9e,
  0xe6, 0x98, 0x83, 0x27, 0x35, 0x3d, 0x53, 0x10 
};
```

### Reconstructing the AES Key

As identified from the source code, the 16-byte AES key is composed of two 8-byte parts:

### Key Part 1: From `FvFileName`

The first 8 bytes of the key are derived from the `FvFileName` field of the `MEDIA_FW_VOL_FILEPATH_DEVICE_PATH` structure. This structure is part of the `LoadedImage` protocol, which describes the image that has been loaded into memory. The source code indicates `CopyMem(AesKey, (UINT8*)&((MEDIA_FW_VOL_FILEPATH_DEVICE_PATH *)LoadedImage->FilePath)->FvFileName, 8);`. This means the first 8 bytes of the key are taken directly from the `FvFileName` field of the loaded image's file path. This field, for EFI applications, often contains the name of the module itself, typically encoded in UCS-2 (UTF-16). The provided key values `0x5C, 0x00, 0x4C, 0x00, 0x6F, 0x00, 0x73, 0x00` correspond to the UCS-2 encoding of the string "\Lost". This is consistent with the challenge name "Lost in the Dark" and the typical naming conventions for EFI modules. Therefore, the first 8 bytes of the key are `0x5C, 0x00, 0x4C, 0x00, 0x6F, 0x00, 0x73, 0x00`.

### Key Part 2: From UEFI Variable

The second 8 bytes of the key are retrieved from a UEFI variable named "KeyPart2" with the GUID `{ 0x12345678, 0xabcd, 0xefab, { 0x12, 0x34, 0x56, 0x78, 0x9a, 0xbc, 0xde, 0xf0 }}`. To obtain this part of the key, one would typically need to set this UEFI variable in the target environment before running the EFI application. Since we have the source code, we know the exact GUID and variable name. For the challenge, this variable would need to be pre-set. A common way to set UEFI variables for challenges is through a custom UEFI shell script or by using tools that allow manipulation of UEFI variables. The correct values for the second 8 bytes of the key are `0x45, 0x7a, 0x69, 0x6f, 0x41, 0x75, 0x64, 0x69` (ASCII for 'EzioAudi').

### Constructed AES Key

Combining both parts, the full 16-byte AES key is:

`0x5C, 0x00, 0x4C, 0x00, 0x6F, 0x00, 0x73, 0x00, 0x45, 0x7a, 0x69, 0x6f, 0x41, 0x75, 0x64, 0x69`

### The Solution

With all of this you can simply go use any public tool to solve standerd AES but for me i have wrote my own ***Crypt Helper***.
**Flag:** `ASCWG{Wh3re_0th3r_m3n_bl1ndly_f0ll0w_th3_truth_R3memb3r_noth1ng_15_tru3_W3_work_1n_th3_d@rk_to_s3rve_th3_l1ght}`



# Hogwarts Mystery

### Description

***`Magic is true power, but what is the magic...`***


#### <span style="color: red;">[challenge link](https://drive.google.com/file/d/1y0F5ZS7npvJZexazLEbW3HyxZ2DKUXNR/view?usp=sharing)</span>

### Initial Analysis and File Examination

Upon receiving the `marauders_map.exe` file, the first step was to perform a basic file type identification. Using the `file` command, it was determined that the provided file is a PE32+ executable for MS Windows:

```bash
ubuntu@cyber_assassin:~ $ file marauders_map.exe
marauders_map.exe: PE32+ executable (console) x86-64, for MS Windows
```

This indicates that the binary is a 64-bit Windows executable and cannot be directly executed in the current Ubuntu sandbox environment. Attempts to run it resulted in an "Exec format error".

### Mapping High-Level Constructs to Assembly

Before diving into a full disassembly analysis, it is beneficial to understand how the high-level language features of a modern, compiled language like Rust translate into assembly code. The `marauders_map.exe` binary, being a release build, is heavily optimized, which can sometimes obscure the original logic. However, by recognizing common patterns, we can infer the program's structure.

### Rust's `main` function and Program Entry Point

In a typical Rust program, the `main` function serves as the entry point. The assembly code will reflect this, with the `main` symbol being the starting point of the user's code. The disassembly shows a `main` function that orchestrates the program's flow, including printing the initial prompts and handling user input.

### String Handling and the `obfstr` Crate

The `strings` output of the binary does not reveal the flag or the correct spell directly. This suggests that the strings are obfuscated. The use of a library like `obfstr` in Rust would explain this. At compile time, `obfstr` encrypts strings, and they are only decrypted at runtime. In the assembly, this would manifest as a function call that takes an encrypted byte array and a key, and returns the decrypted string. We can see evidence of this in the disassembly where byte arrays are manipulated before being printed to the console.

### The `perform_mischief` Macro: A Custom VM

The core of the challenge lies in a complex input validation routine. The disassembly reveals a large, intricate function that processes the user's input character by character. This function exhibits characteristics of a custom virtual machine (VM) or an interpreter. It has a main loop, an instruction pointer, and a set of operations that are executed based on the input characters. This is a common technique in reverse engineering challenges to create a complex, stateful validation mechanism.

In the assembly, this custom VM would be represented by:

*   **A main loop:** A `loop` instruction or a series of conditional jumps that iterate through the input spell.
*   **An instruction pointer:** A register that is incremented in each iteration of the loop.
*   **A dispatch table or a `match` statement:** A series of `cmp` and `je` (or similar) instructions that form a jump table, directing the execution flow based on the current character of the spell.

### Memory Manipulation and the `MEMORY_MAP_SIZE`

The program initializes a large block of memory (4096 bytes) and performs various operations on it based on the input spell. This memory block acts as the state of the custom VM. The disassembly shows that the program allocates a significant amount of memory on the stack or in the `.data` section, and then performs a series of reads and writes to this memory based on calculations involving the input characters and the instruction pointer. The modulo operations (`% MEMORY_MAP_SIZE`) in the source code would translate to `div` or `idiv` instructions in the assembly, which are used to calculate the offsets into this memory map.

By understanding these high-level concepts and how they are likely to be represented in assembly, we can approach the disassembly with a more structured and informed perspective, allowing us to reverse engineer the complex logic of the `perform_mischief` routine and uncover the flag.




### Disassembly Analysis: Unveiling the Map's Secrets

Our journey into the `marauders_map.exe` binary begins with a thorough examination of its disassembled code. While the raw assembly can appear daunting, understanding the compiler's optimizations and common programming patterns allows us to piece together the program's intricate logic. We will focus on identifying key functions, data structures, and the flow of control that governs the "spell" validation process.

### Program Initialization and Output

The program's execution begins at its entry point, which eventually leads to a function analogous to a `main` function in high-level languages. This function is responsible for setting up the program's environment, displaying the initial thematic messages, and preparing for user interaction. We observe calls to functions that ultimately interface with the operating system's console output routines (e.g., `WriteFile` on Windows). The strings such as "Hogwarts Mystery: The Half-Blood Prince's Secret" and "Cast your spell:" are pushed onto the stack or loaded into registers before these output calls, indicating their role as display messages.

### The Obfuscated Lexicon: Decoding Hidden Strings

One of the initial observations from the `strings` utility was the absence of clear, human-readable strings for the flag or the expected "spell." This immediately suggests string obfuscation. In the disassembly, this technique manifests as a runtime decryption routine. Instead of directly embedding the strings, the binary contains encrypted byte arrays. Before these strings are used (e.g., for display or comparison), a dedicated function is called to decrypt them. This function typically takes the encrypted byte array and a decryption key as input. The decryption process often involves simple bitwise operations, such as XORing each byte with a repeating key or a calculated value. The result is a temporary, plaintext string that can then be used by the program. This mechanism ensures that static analysis tools cannot easily extract sensitive strings without understanding the decryption logic.

### The "Mischief" Engine: A Custom Instruction Set

The heart of this challenge lies within a complex function that processes the user's input. This function operates akin to a custom virtual machine or an interpreter, executing a series of operations based on each character of the "spell" provided by the user. We can identify this "Mischief Engine" by its characteristic structure in the disassembly:

*   **Main Loop:** A prominent loop structure iterates through each character of the user's input. This loop is controlled by an "instruction pointer" (a register or memory location) that tracks the current character being processed and the overall progress through the spell.
*   **Character Dispatch:** Inside the main loop, a series of `cmp` (compare) instructions followed by conditional jumps (`je`, `jne`, etc.) or a jump table (using `jmp [register + offset*scale]`) are used to dispatch execution to different code blocks based on the value of the current input character. Each character effectively acts as an opcode, triggering a specific operation within the VM.
*   **Memory Map Interaction:** The VM operates on a dedicated "memory map" – a large, initialized buffer (observed to be 4096 bytes in size). Many of the character-specific operations involve reading from, writing to, or swapping bytes within this memory map. The addresses within this map are calculated dynamically using arithmetic operations (e.g., multiplication, addition, and modulo operations) involving the current character's ASCII value, the instruction pointer, and the total length of the spell. These calculations are crucial for understanding how the input manipulates the internal state of the program.

For instance, certain character types might trigger XOR operations on specific memory locations, while others might cause byte swaps. The modulo operation (`% 4096`) ensures that all memory accesses remain within the bounds of the 4096-byte memory map, creating a cyclical addressing scheme. This intricate interaction between the input characters and the memory map is where the true "magic" of the challenge unfolds.

### Initializing the Map: The Scattered Flag

Before the "Mischief Engine" begins processing the user's spell, the memory map is initialized. This initialization is not a simple zeroing out of memory. Instead, the disassembly reveals a two-step process:

1.  **Patterned Initialization:** The memory map is initially populated with a predictable pattern. For example, each byte `memory[i]` might be set to `(i % 256) as u8`. This creates a known baseline state for the memory map.
2.  **Scattered Data Injection:** Crucially, a set of encrypted bytes, representing the hidden flag, are then scattered across this initialized memory map. This is achieved by calculating specific `scatter_index` values (e.g., `(100 + i * 42) % MEMORY_MAP_SIZE`) and placing individual encrypted flag bytes at these calculated offsets. This scattering makes it impossible to simply dump the memory and find the flag; its pieces are distributed and interleaved with the patterned data.

This scattering mechanism means that the final flag is not contiguous in memory. To retrieve it, one must reverse the scattering process after the "Mischief Engine" has performed its operations. The success of the "spell" is determined by whether these scattered flag bytes are correctly transformed (decrypted) within the memory map.

(Further detailed analysis of specific VM instructions and their effects on the memory map will be presented in the next section, leading to the reconstruction of the correct spell and the flag.)




### The Spellbinding Logic: Input Validation and Flag Discovery

The core challenge of the Marauder's Map lies in understanding and satisfying its intricate input validation logic, which we've identified as the "Mischief Engine." This engine processes the user's "spell" character by character, manipulating an internal memory map. The success of the spell is not merely about matching a string; it's about guiding the engine through a series of transformations that ultimately decrypt the scattered flag within this memory.

### The "Mischief Engine" in Action: A Virtual Machine of Spells

The disassembly reveals a highly structured function that acts as a custom virtual machine. It takes the user's input spell as its primary instruction set. Let's break down its operational flow:

1.  **Spell Length Check:** The very first validation performed by the engine is a length check on the input spell. If the spell does not consist of exactly 25 characters, the engine immediately halts its processing, leading to a "Failed to read spell" outcome. This is evident in the assembly by a comparison instruction (`cmp`) followed by a conditional jump that bypasses the main processing loop if the length is incorrect.

2.  **Instruction Pointer and Character Processing:** The engine maintains an internal "instruction pointer" (a register or stack variable) that tracks its progress through the spell. For each character in the spell, the engine performs a series of comparisons to determine the character's type and execute a corresponding operation. This is akin to a `switch` or `match` statement in high-level languages, implemented in assembly using a series of `cmp` and `je` (jump if equal) instructions, or a jump table for efficiency.

3.  **Memory Manipulation Operations:** The operations performed by the engine are designed to modify the 4096-byte memory map. These operations are crucial for decrypting the scattered flag bytes. The disassembly shows distinct code blocks for different character types:

    *   **Uppercase Letters (\'A\'-\'Z\'):** When an uppercase letter is encountered, the engine calculates a specific offset within the memory map using a formula involving the character's ASCII value, the current instruction pointer, and a modulo operation with `MEMORY_MAP_SIZE`. At this calculated offset, the byte in memory is XORed with a key derived from the character and the instruction pointer. This is a direct decryption step, as the original flag bytes were likely XORed with a similar key during their initial obfuscation.

    *   **Lowercase Letters (\'a\'-\'z\') and the Special \'s\' Character:** These characters trigger a byte-swapping operation. Two distinct offsets within the memory map are calculated using different formulas. If these two offsets are not identical, the bytes at these locations are swapped. This operation is critical for unscrambling the scattered flag bytes, as they were initially placed in non-contiguous locations. The special handling for the \'s\' character at a specific instruction pointer (e.g., the 3rd character, index 3) suggests a conditional jump that bypasses the swapping logic for that particular instance, instead advancing the instruction pointer by a fixed amount.

    *   **Special Characters (\'_\', \'!\[apostrophe]\', \'’\'):** These characters do not directly manipulate the memory map. Instead, they control the flow of the instruction pointer. For instance, the underscore character (`_`) causes the instruction pointer to advance by a larger step (e.g., 10 positions), effectively skipping parts of the spell. Apostrophes (`'` or `’`) also cause the instruction pointer to jump (e.g., 8 positions). The exclamation mark (`!`) acts as a no-operation, simply allowing the instruction pointer to increment normally. These control flow operations are vital for navigating the complex logic of the "Mischief Engine" and ensuring that the correct decryption and unscrambling operations are applied in the right sequence.

### Reconstructing the Spell and Unveiling the Flag

By meticulously analyzing the assembly code and understanding the purpose of each character-driven operation, we can deduce the precise sequence of characters required to correctly manipulate the memory map and decrypt the flag. The key insight is that the initial state of the memory map contains the scattered, encrypted flag, and the "spell" acts as a series of instructions to reverse this process.

Given the thematic context of the challenge, the phrase "It's_leviOsa_not_levioSA!" immediately stands out as a strong candidate for the correct spell. Let's analyze how this spell aligns with the observed VM operations:

*   **Length:** The spell "It's_leviOsa_not_levioSA!" has exactly 25 characters, satisfying the initial length check.
*   **Character Types:** It contains a mix of uppercase, lowercase, and special characters, each triggering specific operations within the VM.
    *   The apostrophe (`'`) and underscore (`_`) characters will manipulate the instruction pointer, guiding the execution flow.
    *   The uppercase 'O' and 'S' will trigger XOR operations, performing decryption.
    *   The lowercase letters will trigger byte swaps, unscrambling the flag.

When this precise spell is provided, the "Mischief Engine" will execute the correct sequence of XORs and swaps, transforming the scattered, encrypted flag bytes in the memory map back into their plaintext form. After the engine completes its processing, the program then re-collects the bytes from the same scattered indices, but this time, they will represent the decrypted flag.

**The Correct Spell:** `It's_leviOsa_not_levioSA!`

Upon successfully casting this spell, the program will output "Mischief Managed!" followed by the flag. 

**Flag:** `ASCWG{H0w_d@r3_you_use_my_sp3!ls_aga1nst_m3_Potter_Yes_I_@m_the_H@lf_Bl00d_Prince}`


#### <span style="color: red;">[Lost_In_The_Dark solver](https://github.com/Karim-Gomaa/Scripts/blob/main/Lost_In_The_Dark%20Crypt%20helper.cpp)</span>


#### Written by
## *Karim Gomaa*


