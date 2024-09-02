---
title: Operating systems - Loading a simple kernel written in C
date: '2020-07-19'
summary: 
---

In this post, we are describing how to write a simple C program and loading it from a 32-bit boot sector code.

In this post, our kernel is as simple as it can get. It consists of a stored variable in a specific location in the code.

Boot sector
-----------

The code we are going to use in this section has nothing new.

*   We are changing from 16-bit to 32-bit to be able to access more memory range.
*   We are jumping from boot sector code to memory code, stored in the next 512 bytes of the boot sector.

    ; A boot sector that boots a C Kernel in 32-bit protected mode
    [org 0x7c00]
    KERNEL_OFFSET equ 0x1000 ; The memory offset to which we will load the kernel
    
      mov [BOOT_DRIVE], dl   ; BIOS stores our boot drive in DL,
                             ; So it's best to remember this for later
      mov bp, 0x9000         ; Set-up the stack
      mov sp, bp
    
      mov bx, MSG_REAL_MODE  ; Announce that we have entered first boot phase
      call print_string      ; 16-bit real mode
    
      call load_kernel       ; Load our kernel
    
      call switch_to_pm      ; Switch to protected mode
                             ; from which we will not return
    
      jmp $
    
    ; Include routines
    %include "print_string.asm"
    %include "disk_load.asm"
    %include "gdt.asm"
    %include "print_string_pm.asm"
    %include "switch_to_pm.asm"   
    
    [bits 16]
    
    ; load_kernel
    load_kernel:
      mov bx, MSG_LOAD_KERNEL   ; Print a message to say we are loading the kernel
      call print_string
    
      mov bx, KERNEL_OFFSET     ; Setup the parameter for our disk_loaded routine, so
      mov dh, 15                ; that we load the first 15 sectors (Excluding boot sector)
      mov dl, [BOOT_DRIVE]      ; from the boot disk to address KERNEL_OFFSET
      call disk_load
      ret
    
    [bits 32]
    
    ; We arrive to this point after switching to and initialising protected mode
    
    BEGIN_PM:
      mov ebx, MSG_PROMPT_MODE  ; Print welcome string in 32-bits
      call print_string_pm
    
      call KERNEL_OFFSET        ; Jump to the address of our loaded kernel code
                                ; If we have done things wright, the loaded, compiled
                                ; kernel code will be at the defined location and we
                                ; will have been able to go from asm to c code.
    
      jmp $                     ; Hang. This happens when we exit the previous
                                ; call, which, sould never happen
      
    
      ; Global variables
      BOOT_DRIVE      db 0
      MSG_REAL_MODE   db "Loaded 16-bit real mode!", 0
      MSG_PROMPT_MODE db "Landed on 32-bit protected mode!", 0
      MSG_LOAD_KERNEL db "Loading Kernel into memory", 0
    
      ; Bootsector padding
      times 510-($-$$) db 0
      dw 0xaa55
    
    

Kernel
------

Kernel code:

    void main(){
      // Create a pointer to a char and point it to the first text cell of
      // video memory (i.e. the top-left of the screen)
      char *video_memory = (char*) 0xb8000;
      // at the address pointed to by the video_memory, store a character 'X'
      // (i.e. display X on the top-left of the screen)
      *video_memory = 'X';
    }
    

This code overwrites the video memory to contain a simple X.

To compile the above code, we are going to need to use these commands:

`gcc -ffreestanding -c kernel.c -o kernel.o`

We generate the `.o` file, which is the intermediate step in a C compilation routine. If we had more than one file, we would have to link each `.o` file together to create the final executable.

`ld -o kernel.bin -Ttext 0x1000 kernel.o --oformat binary`

Here is where we are telling the compiler that our code origin will start at section `0x1000`.

We are jumping from the boot sector to the location 0x1000, which contains the compiled C code we have written.

The BIOS only loads the first 512 bytes of disk (Boot sector)

Kernel image
------------

To simplify the problem of which disk and from which sectors to load the kernel code, the boot sector and kernel of an operating system can be grafted together into a kernel image, which can be written to the initial sectors of the boot disk, such that the boot sector code is always at the head of the kernel image.

To generate the kernel image, we concatenate both binary files created into a single file. There is a _GNU/Linux_ command called `cat` that allows us to do this very easily.

`cat asm/rose-os_boot_sector.bin kernel/kernel.bin > os-image`

Now we have created the kernel image that we are going to load with _Bochs_.

Bochs
-----

The emulator I was using until now was called `qemu`, but in this section it started bailing on me, so I decided I was going to use **Bochs** from now on.

The initial configuration was somewhat cumbersome because I had to generate a `.boshrc` file and specify boot disk and other parameters, but in the long run, it provides a lot more control and granularity, which is very welcome.

The configuration file:

    floppya: 1_44=os-image, status=inserted
    log: ./bochs/bochsout.txt
    debugger_log: ./bochs/debugger.out
    boot: floppy
    

Once setting everything up, execute _Bochs_ with the generated configuration file:

`bochs -qf 'bochs/.bochsrc'`

And in the command line, allow the code to continue execution (Write `c` in the prompt)

Result
------

    Xanded on 32-bit protected mode!03 Jan 2020
    This VGA/VBE Bios is released under the GNU LGPL
    
    Please visit :
     . http://bochs.sourceforge.net
     . http://www.nongnu.org/vgabios
    
    Bochs VBE Display Adapter enabled
    
    Bochs 2.6.10.svn BIOS - build: 01/05/20
    $Revision: 13752 $ $Date: 2019-12-30 14:16:18 +0100 (Mon, 30 Dec 2019) $
    Options: apmbios pcibios pnpbios eltorito rombios32
    
    
    Press F12 for boot menu.
    
    Booting from Floppy...
    Loaded 16-bit real mode!
    Loading Kernel into memory
    

Notice the `X` at the upper left of our result. This is the character referenced by our C-compiled code, and the last step of execution.

It is the proof of the jump we have made from boot sector to the second disk sector, which contains our `C` code (At location `0x1000`.

Conclusion
----------

Now, we have successfully jumped from the boot sector into a C-compiled section of code.

We can start writing our kernel, to allow communication from our hardware to our higher-level software, but this is still a long way away.
