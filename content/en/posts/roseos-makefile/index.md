---
title: Rose-OS - Makefile teardown
date: '2020-07-29'
summary: 
---

<p>In this post, I will be describing programs and resources used to build Rose-OS and how we combine then into a single, automated Makefile.</p>

<h1 id="programs-used-to-generate-the-kernel">Programs used to generate the kernel</h1>

<ul>
<li>gcc</li>
<li>nasm</li>
<li>bochs (Emulator)</li>
<li>ld</li>
<li>cat</li>
<li>rm</li>
</ul>

<h2 id="gcc">gcc</h2>

<p>Is the compiler we use to compile our code.
Normally, this command preprocesses, compiles, assembles and links our code, but we can stop it from doing this with some command-line options.</p>

<p>The command:</p>

<p><code>gcc -m32 -ffreestanding -c kernel/kernel.c -o kernel/kernel.o -fno-pie</code></p>

<p>With the parameters:</p>

<ul>
<li><code>-m32</code>: Compile 32-bit objects instead of the default 64-bit.</li>
<li><code>-c</code>: Do not run the linker, instead, output object files outputted from the assambler.</li>
<li><code>-ffreestanding</code>: Directs the compiler to not assume that standard functions have their usual definitions.</li>
<li><code>-fno-pie</code>: Negative form of <code>-fpie</code>, which stands for Position Independent Code. This option disables the random code positioning security feature.</li>
<li><code>-o</code>: Describes the name and location of the object file being generated.</li>
</ul>

<h2 id="ld">ld</h2>

<p><code>ld</code> combines a number of object and archive files, relocates their data and ties up symbol references.
Usually the last step in compiling a program is to run ld.</p>

<p>The command:</p>

<p><code>ld -m elf_i386 -o kernel/kernel.bin -Ttext 0x1000 asm/kernel_entry.o kernel/kernel.o --oformat binary</code></p>

<p>With the parameters:</p>

<ul>
<li><code>-m elf_i386</code>: The emulation mode. In our case, we want to emulate for the Intel 80386 architecture.</li>
<li><code>-Ttext 0x1000</code>: Locate a section in the output file at the absolute address given by org. Start the .text section at address 0x1000.</li>
<li><code>--oformat binary</code>: The output format of the file being generated.</li>
</ul>

<p>Here we are providing initially the kernel_entry object file because this file contains the definition for the main kernel function. This is the first method we are going to execute.</p>

<p>In our Makefile, we provide a list of <code>.o</code> files after specifying the kernel_entry.</p>

<h2 id="nasm">nasm</h2>

<p>The Netwide Assembler, a portable 80x86 assembler</p>

<p>The commands:</p>

<p><code>nasm rose-os_boot_sector.asm -f bin -o rose-os_boot_sector.bin</code></p>

<p><code>nasm asm/kernel_entry.asm -f elf -o asm/kernel_entry.o</code></p>

<p>With the parameters:</p>

<ul>
<li><code>-f bin | elf</code>: Specifies the output file format.</li>
<li><code>-I './boot/'</code>: Adds a directory to the search path for include files</li>
</ul>

<h2 id="bochs">Bochs</h2>

<p>Bochs is a 32-bit emulator that also serves as a debugger.</p>

<p>We have chosen to work with this emulator because it provides greater control over the executables being run.
It provides out-of-the-box features to break, continue, step into, or step out of the run binary code.</p>

<p>The configuration of this program is managed by a <code>.boshrc</code> file located in the directory the program is being run from.</p>

<p>In my case, the <code>.boshrc</code> file looks like this:</p>

<pre><code>floppya: 1_44=os-image, status=inserted
log: ./bochs/bochsout.txt
debugger_log: ./bochs/debugger.out
boot: floppy
</code></pre>

<p>In this file, we are stating that we are loading our system with a inserted floppy disk containing the binary <code>os-image</code>.
The log and debugger_log files are saved to a custom directory, to keep things organised.</p>

<h2 id="cat">Cat</h2>

<p>Although being a very basic GNU/Linux command, this command is invaluable for kernel development, because it lets us create a kernel image binary file.</p>

<p>This file contains the boot sector and the kernel in one file.</p>

<p>The command:</p>

<p><code>cat boot/rose-os_boot_sector.bin kernel/kernel.bin &gt; os-image</code></p>

<h2 id="other-commands">Other commands</h2>

<p>They are used to clean the working directory.</p>

<h1 id="combining-these-commands">Combining these commands</h1>

<h2 id="make">Make</h2>

<p>With the make command, we automate the build, execution and clean process.</p>

<p>Otherwise, when we change a file, we have to execute a bunch of commands to compile, build and link our code.
Make takes care of all of this in an automated way.</p>

<p>My Makefile is as follows:</p>

<pre><code class="language-Makefile"># Automatically  generate  lists  of  sources  using  wildcards.
C_SOURCES = $(wildcard  kernel/*.c drivers/*.c)
HEADERS = $(wildcard  kernel/*.h drivers/*.h)

# TODO: Make sources dep on all header files.

# Convert  the *.c filenames  to *.o to give a list of  object  files  to  build

OBJ = ${C_SOURCES:.c=.o}

# Defaul  build  target

all: os-image
# Run  bochs  to  simulate  booting  of our  code.

run: all
	bochs

# This is the  actual  disk  image  that  the  computer  loads
# which  is the  combination  of our  compiled  bootsector  and  kernel

os-image: boot/rose-os_boot_sector.bin kernel/kernel.bin
	cat $^ &gt; os-image

# This  builds  the  binary  of our  kernel  from  two  object  files:
#   - the kernel_entry, which jumps to main() in our kernel
#   - the compiled C kernel
kernel/kernel.bin: boot/kernel_entry.o ${OBJ}
	ld -m elf_i386 -o $@ -Ttext=0x1000 $^ --oformat binary

# Generic  rule  for  compiling C code to an  object  file
# For  simplicity, we C files  depend  on all  header  files.
%.o : %.c ${HEADERS}
	gcc -ffreestanding -fno-pie -m32 -c $&lt; -o $@

# Assemble the kernel_entry.
%.o : %.asm
	nasm $&lt; -f elf -o $@

%.bin : %.asm
	nasm $&lt; -f bin -I  ’./boot/’ -o $@

clean:
	rm -fr *.bin *.dis *.o os-image
	rm -fr  kernel/*.o kernel/*.bin boot/*.bin  drivers/
</code></pre>

<p>Upon inspecting this file, we can see the commands listed above are in this file, but we use regular expressions and macros to shorten the needed commands further.</p>

<p>If we run <code>make</code> without arguments, then the first instruction will be run, this is why the first instruction</p>

<p><code>all: os-image</code></p>

<p>has as a dependency another instruction,</p>

<p><code>os-image: boot/rose-os_boot_sector.bin kernel/kernel.bin</code></p>

<p>that triggers the whole make process.</p>

<h1 id="project-structure">Project structure</h1>

<p>I have decided, based of the <a href="https://www.cs.bham.ac.uk/~exr/lectures/opsys/10_11/lectures/os-dev.pdf">os-dev</a> book, to create several directories.</p>

<ul>
<li><strong>boot</strong>: All the assambly code needed to boot the OS is located here</li>
<li><strong>kernel</strong>: The C code that compiles the kernel is stored here</li>
<li><strong>drivers</strong>: Any hardware-specific code</li>
</ul>

<h2 id="bibliography">Bibliography:</h2>

<ul>
<li><a href="https://cs107e.github.io/guides/gcc/">ffreestanding</a></li>
<li><a href="https://linux.die.net/man/1/ld">ld</a></li>
<li><a href="https://linux.die.net/man/1/nasm">nasm</a></li>
</ul>


