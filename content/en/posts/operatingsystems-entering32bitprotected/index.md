---
title: Operating systems - Entering 32 bit protected mode
date: '2020-07-17'
summary: 
---

In this post, we will cover how to enter 32-bit protected mode. This is a requirement if we want to be able to create a OS with the features we are used to. Otherwise, we will have very little storage space allowed due to the 16-bit reference limit.

Boot sector
===========

To enter 32-bit protected mode, we first have to bootstrap our code from a 16-bit sector. This is necesary because the initial code location is stored in a space-restricted zone.

Steps
=====

1.  Create “Main flow”
2.  Load GDT (Global Descriptor Table)
3.  Switch to 32-bit protected Mode

Main flow
---------

We want to load the GDT structure, so that the processor knows where each code block is going to be.

    ; A boot sector that enters 32-bit protected mode
    [org 0x7c00]
      mov bp, 0x9000          ; Set the stack
      mov sp, bp
    
      mov bx, MSG_REAL_MODE
      call print_string
    
      call switch_to_pm       ; We won't return from here
    
    

In the above code segment, we print a message (In 16-bit real mode) to make sure we are on the right track. Then we execute the code that switches to protected mode (32-bit)

Load DGT
--------

We define two variables based on the DGT created. These variables will become very handy, because they are the start of the sections they describe.

    ; Define some handy constants for the GDT segment descriptor offset, which
    ; are what segment registers must contain when in protected mode. For example,
    ; when we set DS = 0x10 im PM, the CPU knows that we mean it to use the
    ; segment described at offset 0x10 (i.e. 16 bytes) in our GDT, which in our
    ; case is the DATA segment (0x0 -> NULL; 0x08 -> CODE; 0x10 -> DATA)
    CODE_SEG equ gdt_code - gdt_start
    DATA_SEG equ gdt_data - gdt_start
    

When we load the DGT, we can then jump the the 32-bit code which is stored in another memory location than the 16-bit code.

    switch_to_pm:
      cli                     ; We must switch of interrupts until we have
                              ; set-up the protected mode interrupt vector
                              ; otherwise interrupts will run riot.
    
      lgdt [gdt_descriptor]   ; Load our GDT, which defines the protected
                              ; mode segments (e.g. for code and data)
    
      mov eax, cr0            ; To make the switch to protected mode, we set
      or eax, 0x1             ; the first bit of CR0, a control register
      mov cr0, eax
    
    
      jmp CODE_SEG:init_pm    ; Make a far jump (i.e. to a nre segment) to our 32-bit
    

Now, we have loaded the Global Descriptor Table.

Once the CPU has been switchedinto 32-bit protected mode, the process by which it translates logical addresses (i.e.the combination of a segment register and an offset) to physical address is completelydifferent: rather than multiply the value of a segment register by 16 and then add to itthe offset, a segment register becomes an index to a particular segment descriptor(SD) in the GDT.

32-bit protected mode
=====================

    [bits 32]
    ; Initialise registers and the stack once in PM
    init_pm:
      mov ax, DATA_SEG        ; Now in PM, our old segments are meaningless,
      mov ds, ax              ; so we point our segment registers to the
      mov ss, ax              ; data selector we defined in our GDT
      mov es, ax
      mov fs, ax
      mov gs, ax
    
      mov ebp, 0x90000 ; Update our stack position so it is right
      mov esp, ebp     ; at the top of the free space.
    
      call BEGIN_PM    ; Finally, call some well-known label
    

We initialise the segment registers for our new 32-bit structured code. Then we call our first 32-bit code segment.

    [bits 32]
    ; This is where we arrive after switching to and initialising protected mode.
    BEGIN_PM:
      mov ebx, MSG_PROT_MODE
      call print_string_pm    ; Use our 32-bit print routine
    
      jmp $                   ; Hang
    

In this ocasion, we are only printing a string, without BIOS help, like we had in 16-bit real mode. But in the future, we will be able to load our compiled C code.

These code segments require further code to be run successfully. The rest of the code needed for the segmemts to work can be obteined in my [GitHub repository](https://github.com/SergioRosello/Rose-OS/tree/OS/os-dev).
