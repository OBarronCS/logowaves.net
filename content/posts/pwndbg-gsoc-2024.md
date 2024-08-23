+++
title = 'Google Summer of Code Report - Pwndbg 2024'
date = 2024-08-22T15:33:12-07:00
draft = false
toc = false
# toc_sticky = true
+++

This summer, I worked on [Pwndbg](https://github.com/pwndbg/pwndbg/), a GDB dashboard popular among reverse engineers and exploit developers, and something I use nearly every weekend for [CTFs](https://ctftime.org/team/12858/). Pwndbg allows you to quickly see the state of CPU registers and stack memory, provides a view of the disassembled machine instructions near the instruction pointer, and offers powerful context and control while debugging binaries.

My project aimed to enhance the disassembly view of Pwndbg while debugging RISC-V, ARM, and MIPS processes. Combining emulation and binary instrumentation, I built a system to annotate the assembly instructions with text to indicate the action each one takes. For mathematical operations, we display the values of the operands as well as the result, and for load/store instructions we indicate the memory address in use and the value being moved.


{{< image src="/images/riscv_annotations.png" caption="Disassembly view of a RISC-V program">}}


This project expands upon previous contributions of mine which created this annotation feature for x86.

# The details

First, let’s start with some examples of instruction annotations. The following is an example of a RISC-V `add` instruction with its annotation. The text indicates the values of the source operands, `a5` and `a2`, which are used in the add operation, as well as the resulting value that the instruction will place into register `a4`.

```asm
add a4, a5, a2         A4 => 0x555555558000 (0x4000 + 0x555555554000)
```

Here’s a MIPS store instruction - it takes the value in the `v1` register, treats it as a word (32-bits) and places it into the memory location defined by the second operand. The annotation indicates the concrete memory address used in the operation (`0x4040c0`), and the value placed into it (`0x203`).


```asm
sw $v1, 0x20($v0)      [0x4040c0] => 0x203
```

And here’s a load instruction found in AArch64:
```asm
ldrsh w1, [x0]         W1, [0x555555576fc8] => 0x19
```
To determine the value that will be loaded in `w1`, we read 2 bytes from the memory location `0x555555576fc8` and sign-extend it to 4 bytes. This is just one example of AArch64’s over 20 unique load instructions!

![insert screenshot here](insert_screenshot_here)

For this project, I needed to dig into the internals of RISC-V, ARM, and MIPS to understand what types of instructions are present, what kinds of actions they take, and how operands are used. Every architecture has unique aspects that require special care - such as Arm's Thumb mode or MIPS's delay slots - and there are edge cases until the eye can see.

The first step in creating an annotation is identifying the instruction and resolving the concrete values of the operands. Using the [Capstone Engine](https://www.capstone-engine.org/), we disassemble instructions and get programmatic access to the operand details and other metadata. We need to resolve the values of the operands, which can involve reading a register or dereferencing a memory address.

For example, take AArch64, which has memory operands that allow a dizzying array of modifiers.
```assembly
str w2, [x0, w4, sxtw #2]
```
To resolve the concrete memory address used in this store instruction, we need to read the `w4` register, sign-extend it, apply a left bitshift of 2, and add it to `x0`. You can also apply these shifts and extends to [register operands](https://devblogs.microsoft.com/oldnewthing/20220729-00/?p=106915) and even [immediates](https://dinfuehr.github.io/blog/encoding-of-immediate-values-on-aarch64/)!

For each architecture, I dug into the Instruction Set Architecture (ISA) manuals, found how each operand is resolved, and wrote parsers to manually resolve the actual value used in the operation.

{{< image src="/images/aarch64_example.png" caption="Example AArch64 Disassembly">}}

After identifying the instruction and determining the values of its operands, then we can go about creating an annotation. Instructions are handled case-by-case, although due to patterns in instruction types, we can process many in the same way. Consider arithmetic operations that act on registers - things like `add`, `sub`, `xor`, `and`, `cmp`, bit shifts, and more. We want to display the value that the instruction will put into the destination register. To do this, we need emulation.

We use [Unicorn Engine](https://www.unicorn-engine.org/) for emulation. We copy the process's memory and CPU registers into the emulator and start stepping it instruction-by-instruction. At each step, we query the emulator for the memory and register values relevant to the current instruction. This is how we can show the results of mathematical operations, determine the outcome of branches, and know the contents of memory in the future. Emulation allows us to display annotations for instructions that the CPU is about to execute, providing a great level of context.

A lot of hours went into getting the emulator to work nicely with all the architectures. A highlight was [getting Arm Thumb mode to work](https://github.com/pwndbg/pwndbg/pull/2292) and finding an intricacy of the Arm architecture - banked registers - [that caused the stack pointer to always be zero](https://github.com/pwndbg/pwndbg/pull/2337s). MIPS delay slots also threw a wrench into the system. 

{{< image src="/images/mips_delay_slots.png" caption="MIPS has delay slots!">}}


Stepping through tens of thousands of instructions is bound to bring up edge cases. Many instructions have aliases that need to be considered, have a varying number of operands, or have interesting behaviors under the right conditions. A bug that took a while to track down was in Arm `load` operations - if and only if the base operand is `PC`, then the resolved value is aligned to a 32-bit boundary (after all shifts and modifiers), which is only relevant in Thumb-2 mode when the instruction is not-aligned to a word boundary. You really have to read the footnotes of the ISA manuals.

Emulation is also used to disassemble instructions along the predicted flow of program execution. I wrote code to manually determine the targets of branches - taking into account if it’s a conditional branch - to ensure the disassembly follows the correct line of execution in these architectures. Other conditional instructions, such as AArch64's `cset`, get little green checkmarks to indicate that the action was taken, and the same goes with Arm, where nearly every instruction can be made conditional. 

Some annotations can be resolved statically, like ones that move a constant into a register, or load an address relative to the instruction address. When possible, annotations were written to work even in the absence of emulation, so that people debugging in an environment where they don't have enough RAM to run Unicorn benefit from this feature.

{{< image src="/images/aarch_conditional_instruction.png" caption="We determine if conditional instructions are executed">}}


# Current State

Around two hundred instructions across Arm, MIPS, and RISC-V now have annotations. The most common general-purpose instructions now automatically display the result of the instruction, providing users of Pwndbg with insight into instructions being executed, and adding information to the dashboard that otherwise you would need to fish out manually using a variety of intricate GDB commands that vary depending on the context. While I'm debugging through a binary with Pwndbg nowadays, I often nearly ignore the instructions themselves, and focus on the annotations, and they greatly speed up reverse engineering efforts.

{{< image src="/images/arm_instructions.png" caption="Arm instructions - we follow transitions to and from Thumb mode!">}}

!(insert another screenshot here)[]

The testing code was updated to allow us to validate the non-x86 Pwndbg experience using QEMU for emulation, and a suite of tests was added to the codebase so that I can sleep well at night. 


# Final thoughts

My hope is that these contributions encourage people to use Pwndbg to debug and reverse engineer in these new architectures, and most of all to help you CTFers to get more flags! Not only do these annotations provide much-needed context to aid in debugging, but they provide insight to anyone learning about assembly languages - you can visually see what each instruction is doing to get a better grasp of low level programming.

Finally, a huge thank you to my mentors, Dominik 'Disconnect3d' Czarnota and Gulshan Singh!

# Future work

You can categorize instructions into 5 broad groups: branch instructions, load instructions, store instructions, arithmetic instructions, and everything else. This project focused on the first four groups, providing annotations on general-purpose instructions. Future work could integrate annotations for SIMD and floating point operations.

Additionally, there are always more architectures to support! With the state of the current codebase, adding new architectures should hopefully go smoothly - fill in some functions and separate the instructions into groups, and you should be good to go!

Lastly, as more people start to use Pwndbg for these new architectures, bugs will arise. I'll be continuing to contribute to Pwndbg, fixing bugs, adding support for more instructions, and writing new features, and I encourage others to do the same!


Architecture References:
- "Arm Assembly Internals and Reverse Engineering" by Maria (Azeria) Markstedter
- See MIPS Run (The Morgan Kaufmann Series in Computer Architecture and Design)
- RISC-V Assembly Language Manual - https://shakti.org.in/docs/risc-v-asm-manual.pdf


List of Pull Requests:
- Fix virtual memory mappings on remote targets - https://github.com/pwndbg/pwndbg/pull/2386
- Fix ARM Cortex-M detection - https://github.com/pwndbg/pwndbg/pull/2381
- Update developer documentation - https://github.com/pwndbg/pwndbg/pull/2377
- Test suite for annotations - https://github.com/pwndbg/pwndbg/pull/2374
- AArch64 conditional instructions + others - https://github.com/pwndbg/pwndbg/pull/2368
- Miscellaneous annotations improvements - https://github.com/pwndbg/pwndbg/pull/2364
- Store instruction annotations - https://github.com/pwndbg/pwndbg/pull/2363
- Fix crash in argument resolution while reading register - https://github.com/pwndbg/pwndbg/pull/2357
- Arithmetic instruction annotations - https://github.com/pwndbg/pwndbg/pull/2356
- Fix annotation crash on Arm - https://github.com/pwndbg/pwndbg/pull/2346
- Correctly initialize Arm mode in Unicorn Engine - https://github.com/pwndbg/pwndbg/pull/2337
- Pause GDB event handlers while exploring memory - https://github.com/pwndbg/pwndbg/pull/2328
- x86 fs/gs register support in Unicorn and annotations - https://github.com/pwndbg/pwndbg/pull/2317
- Load instruction annotations - https://github.com/pwndbg/pwndbg/pull/2309
- CMP-like instructions in AArch64, Arm - https://github.com/pwndbg/pwndbg/pull/2303
- Fix tests in Ubuntu 24.04 - https://github.com/pwndbg/pwndbg/pull/2295
- Arm Thumb support with Unicorn + Capstone - https://github.com/pwndbg/pwndbg/pull/2292
- Display Arm mode in banner - https://github.com/pwndbg/pwndbg/pull/2281
- Fix edge case in stepuntilasm - https://github.com/pwndbg/pwndbg/pull/2279
- Bitwise math helper functions - https://github.com/pwndbg/pwndbg/pull/2278
- Rehaul testing structure to allow for QEMU user space unit tests - https://github.com/pwndbg/pwndbg/pull/2275
- Support MIPS delay slots - https://github.com/pwndbg/pwndbg/pull/2262
- Simplify detection of function-call-like instructions - https://github.com/pwndbg/pwndbg/pull/2261
- Reimport disassembly code - https://github.com/pwndbg/pwndbg/pull/2260
- Manually resolve AArch64 conditional branches - https://github.com/pwndbg/pwndbg/pull/2259
- Display link register by default in register view - https://github.com/pwndbg/pwndbg/pull/2251
- Add ABI for 64-bit MIPS - https://github.com/pwndbg/pwndbg/pull/2241
- Display names of syscalls that occur in the future - https://github.com/pwndbg/pwndbg/pull/2205
- Indicate private API functions - https://github.com/pwndbg/pwndbg/pull/2193
- x86 Annotations - https://github.com/pwndbg/pwndbg/pull/2001


