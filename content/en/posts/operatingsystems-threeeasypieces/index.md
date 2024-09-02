--- 
title: Operating systems - Three easy pieces - introduction
date: '2020-09-09'
summary: 
---

This series contains my notes on the free on line book [Operating Systems: Three easy pieces](http://pages.cs.wisc.edu/~remzi/OSTEP/). I will create a entry on each topic or on anything I feel worth remembering/mentioning. This entry is the first one, consisting on a introduction to the book, and a few resources I found to be handy.

Links and references
--------------------

*   [XV6](https://github.com/mit-pdos/xv6-public)
*   [Advanced XV6](https://github.com/nextbytes/Advanced-xv6)
*   [OSTEP projects](https://github.com/remzi-arpacidusseau/ostep-projects)

Main focus
----------

There are three main topics on operating systems development.

*   Virtualization
*   Concurrency
*   Persistence

### Virtualization

The main goal of the operating system is to figure out a way to virtualize hardware resources in a efficient and easy-to-use way.

We sometimes call a Operating System a virtual machine because, when we implement the hardware API, we are creating a extended driver for this hardware. This create a more powerful version of the hardware. Imagine if all the programs had to know how to manage the screen layout. This is where the OS comes into action. It manages drivers that make the specific hardware products easier to interact with. This is the basis of **Virtualization**

There are two different types of virtualization:

*   Memory virtualization
*   CPU virtualization

### Concurrency

Is the concurrent running of processes. The issue with OS is that we have to be running multiple programs at once, and as the most low-level instructions are not atomic, we may run into problems.

Given a program that runs concurrently adding 1 to a shared variable, if we want the program to execute 100000 times, we will most likely run into concurrency problems. As said above, the most lower-level instructions (one to load the value of the counter from memory into a register, one to increment it, and one to store it back into memory) are not executed atomically, therefore problems may arise.

### Persistence

In modern OS, we have to have a way to store user’s data in the disk. This is called a file system. To write to it, the OS has to perform a load of different calls, to figure out where the data is stored, how to access it, and what to write. It is usually a complicated process with data structures such as lists or b-trees.

### Other goals

**Abstraction** is vital in computer science, as it enables us to write a program in a high-level language like C without thinking about assembly, to write code in assembly without thinking about logic gates, and to build a processor out of gates without thinking too much about transistors.

**Performance**. In order to attain it, we have to minimize overhead (Reduce system time and disk space).

**Protection**: We want to make sure that a malicious program can’t damage another. This is achieved through process isolation.

**Reliability**: Is key, as a user has to be able to not care about it’s system breaking.

