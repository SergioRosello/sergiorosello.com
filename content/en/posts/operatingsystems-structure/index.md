---
title: Operating systems - Structure
date: '2020-11-09'
summary: 
---

Below are some notes on Operating Systems Structure from the book Modern Operating Systems.

Different approaches suit different intended usages, below is a list of the most common structures and their explanation.

* Monolithic systems
* Layered systems
* Microkernels
* Client-server systems
* Virtual Machines
* Exokernels

## Monolithic Systems

* The entire OS runs as a single program in Kernel Mode.
* There is no privacy, as every procedure is visible to every other procedure.
* For every system call, there is one service procedure that takes care of it and executes it.
* In addition to the core, they normally provide support for loadable extensions (Shared Libraries (UNIX) - DLL’s (Windows))

## Layered Systems

The first Layered system produced was the THE system. A popular system that used a layered approach was the MULTICS system.
The layers

* Layer 0 provided the basic multiprogramming of the CPU
* Layer 1 did the memory management
* Layer 2 handled communication between each process and the operator console (User)
* Layer 3 took care of the I/O devices
* Layer 4 was where the user programs where found
* Layer 5 contained the system operator process

## Microkernels

Based on the idea that the larger program, the more bugs it contains, the idea behind Microkernels is to achieve high reliability by splitting the Operating System into small, well-defined modules with only the Microkernel running in kernel mode and the rest of the programs running in user mode. As a direct consequence, a bug in a device driver can cause that device to crash, but not the entire system.

The tasks the Microkernel is responsible for are: Handling Interrupts, Processes, Scheduling and Inter-process communication. The other low level programs, for example, device drivers, don’t have physical access to I/O port, and therefore have to write to a structure, what ports it wants to write or read, and make a kernel call so that it handles the operation. This way, the kernel can decide if the driver is accessing a protected location or not.

Above the drivers, there is another layer, which holds the servers. They do most of the work in the Operating System. One or more file servers manage the filesystem, so programs tell the servers what to do, and they make the POSIX system calls. For example, a process needing to do a read, tells the server what to read.

One technique used minimize the kernel is to provide the mechanisms but not the policy.
Client-Server Model

It distinguishes two classes of processes: The Client and the Server. Often the lowest-level is a Microkernel, but this is not required.

This idea relies on message-passing to get the job done. One Client tells the server what it needs, and the server responds with the answer. Note this can be done over a message bus, communicating several clients (which are different machines) to several servers. As network speeds are getting faster, Client-Server models can become more popular. Imagine a phone that only needs to make requests to the server to operate. (Of course, privacy is an issue here)
Virtual Machines

Provide another abstraction layer from the hardware. They directly replicate the hardware. By doing this, they can simulate different machines in a single physical one.

Virtual Machines provide a cost-efficient solution for companies that run web servers. If they had to use a dedicated machine, the cost would be much higher, and if they have to use a pre-configured server, they can’t configure the whole stack, and have to stick to the programs installed by the server rental company.

During the VM development process, there was a leap forward in terms of speed.

* Hypervisor Type 1 (Emulate the entire system)
* Hypervisor Type 2 (Have a specific Kernel module for virtualization techniques and makes use of it’s host operating system)

## Exokernels

Are virtual machines managed by a program called Exokernel, which ensures they don’t overwrite each other. The advantage of these implementations is that the architecture looses one entire mapping. In the other designs, the Virtual Machine thinks it has it’s own disks, so the Virtual Machine Monitor has to map specific disk partitions to specific Virtual Machines, but with the Exokernel, it only needs to know which Virtual Machine has been assigned which resource.

