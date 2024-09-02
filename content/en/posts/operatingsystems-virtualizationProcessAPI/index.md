---
title: Operating systems - Virtualization - Processes API
date: '2020-09-15'
summary: 
---

Unix creates a process with a pair of system calls:

*   `fork()`
*   `exec()`

A third system call can be used by a process to wait until a process it has created to complete.

*   `wait()`

fork()
------

> It is used to create a new process

In the following example:

    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    
    int main(int argc, char* argv[]) {
      printf("Hello world (pid:%d)\n", (int) getpid());
      // rc = return code
      int rc = fork();
      if (rc < 0) {
        // Fork failed
        fprintf(stderr, "Fork failed\n");
        exit(1);
      } else if (rc == 0) {
        // Child (new process)
        printf("Hello, I'm a child (pid:%d)\n", (int) getpid());
      } else {
        // Parent goes down this path
        printf("Hello, I am a parent of %d (pid:%d)\n", rc, (int) getpid());
      }
    return 0;
    }
    

A new child process is created when `fork()` is called. To the OS, now there a two nearly identical programs running alongside each other. Both about to return from the `fork()` system call. The new process comes into life as it had just called `fork()` itself.

The child’s return code is 0 (if the new process is created correctly) and the parent’s return code is the child’s PID. This is done in order to simplify the code that handles both processes.

Now, there are two different processes running alongside each other, with different Address space, registers, PC, etc… This means that this code’s output is **non deterministic**.

wait() or waitpid()
-------------------

> Enables the parent to wait until the child’s process has finished executing, then resumes parent process execution.

    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <sys/wait.h>
    
    int main(int argc, char* argv[]) {
      printf("Hello world (pid:%d)\n", (int) getpid());
      // rc = return code
      int rc = fork();
      if (rc < 0) {
        // Fork failed
        fprintf(stderr, "Fork failed\n");
        exit(1);
      } else if (rc == 0) {
        // Child (new process)
        printf("Hello, I'm a child (pid:%d)\n", (int) getpid());
      } else {
        // Parent goes down this path
        int rc_wait = wait(NULL);
        printf("Hello, I am a parent of %d (rc_wait:%d) (pid:%d)\n", rc, rc_wait, (int) getpid());
      }
    return 0;
    }
    

This enables us to convert a non deterministic output into a deterministic one.

exec()
------

> Enables us to run a program that is different from the calling program.

This system call and its variants (execl(), execlp(), execle(),execv(), execvp(), and execvpe()) overwrite the code and static segments of the process calling it, re-initializing the heap, stack and other parts of memory space.

A successful call to `exec()` never returns.

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <unistd.h>
    #include <sys/wait.h>
    
    int main(int argc, char* argv[]) {
      printf("Hello world (pid:%d)\n", (int) getpid());
      // rc = return code
      int rc = fork();
      if (rc < 0) {
        // Fork failed
        fprintf(stderr, "Fork failed\n");
        exit(1);
      } else if (rc == 0) {
        // Child (new process)
        printf("Hello, I'm a child (pid:%d)\n", (int) getpid());
        char* myargs[3];
        myargs[0] = strdup("wc");     // Word count (Program name)
        myargs[1] = strdup("p3.c");   // Current program
        myargs[2] = NULL;             // Marks end of array
        execvp(myargs[0], myargs);    // Runs word count
        printf("This shouldn't print out");
      } else {
        // Parent goes down this path
        int rc_wait = wait(NULL);
        printf("Hello, I am a parent of %d (rc_wait:%d) (pid:%d)\n", rc, rc_wait, (int) getpid());
      }
    return 0;
    }
    

In this example, we create a copy of the current program, with `fork()` and overwrite it’s code segment (Which is the behavior of the exec() system call) to inject the program we told it to execute `wc` in the `execvp()` system call.

Why?
----

Although the process creation interface might seem complex, it is a cornerstone in building a UNIX shell.

The separation of fork() and exec() is essential in building a UNIX shell, because it lets the shell run code after the call to fork() but before the call to exec(); this code can alter the environment of the about-to-be-run program, and thus enables a variety of interesting features to be readily built.

In the UNIX shell, the main program forks itself, to then exec the command executed by the user while waiting for the child program to finish execution.
