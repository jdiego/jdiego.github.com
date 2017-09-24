---
layout: post
title:  "How debuggers work"
date:   2017-06-24 21:06
categories: c++ debugger programming
---


# How debuggers work: The Basics
This is the first part in a series of articles on how debuggers work. 
I'm still not sure how many articles the series will contain and what topics it will cover, 
but I'm going to start with the basics. 

In this part, we are going to present the main building block of a debugger's implementation on Linux - 
the ptrace system call. All the code in this article is developed on a 64-bit Linux machine. Note that 
the code is very much platform specific, although porting it to other POSIX platforms shouldn't be 
too difficult.

## Motivation

To understand where we're going, try to imagine what it takes for a debugger to do its work. A debugger 
can start some process and debug it, or attach itself to an existing process. It can single-step through 
the code, set breakpoints and run to them, examine variable values and stack traces. Many debuggers have 
advanced features such as executing expressions and calling functions in the debbugged process's 
address space, and even changing the process's code on-the-fly and watching the effects.

Although modern debuggers are complex beasts, it's surprising how simple is the foundation on which they 
are built. Debuggers start with only a few basic services provided by the operating system and the 
compiler/linker, all the rest is just a simple matter of programming.


## Linux debugging - ptrace

The Swiss army knife of Linux debuggers is the ptrace system call. It's a versatile and rather complex 
tool that allows one process to control the execution of another and to peek and poke at its innards [3]. 
Let's dive right in.

### Stepping through the code of a process

I'm now going to develop an example of running a process in "traced" mode in which we're going to 
single-step through its code - the machine code (assembly instructions) that's executed by the CPU. 
I'll show the example code in parts, explaining each, and in the end of the article you will find a 
link to download a complete C file that you can compile, execute and play with.

The high-level plan is to write code that splits into a child process that will execute a 
user-supplied command, and a parent process that traces the child. We'll do this with the
classic fork/exec pattern. 
```c++
int main(int argc, char* argv[]) {
    if (argc < 2) {
        std::cerr << "Program name not specified\n";
        return -1;
    }
    auto prog = argv[1];
    auto pid = fork();
    
    if (pid == 0) 
    {
        //we're in the child process, then execute debugee
        run_target(prog);
    }
    else if (pid >= 1)  
    {
        //we're in the parent process, then execute debugger
        run_debbuger(pid);
    }
    else
    {
        perror("fork");
        return -1;
    }
    return 0;
```

We call fork and this causes our program to split into two processes. If we are in the child process, 
fork retuns 0, and if we are in the parent process, it returns the process ID of the child process.
If we’re in the child process, we want to replace whatever we’re currently executing with the program 
we want to debug.

