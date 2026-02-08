
+++
title = "Building a Bare Metal Scheduler on an STM32H7"
date = 2026-02-08
description = "Notes on writing a simple round-robin scheduler from scratch"
+++

# Starting situation

As a weekend project I decided to give bare metal programming a shot. What started with some basic blinking of an led and firing an interrupt when a button is pressed turned into writing a small task switcher / scheduler.
By the end of this post I hope you have a rough idea on what path I took, what challenges I encountered and how I overcame them.

As a computer science student myself, I am by no means an authoritative source lalilu. feel free to get in contact 

# The problem
Bare metal programming is fun. It is just you, your development board and a 3357 page reference manual written in dense and technical prose.


# toolchain / build process
how to get code running on an embedded device

basic sections

reset handler

# Blinky example
reading the reference sheet

configuring and writing to some basic registers

# Your three best friends
memory map / clock tree / gdb

# Leveling up | switching from asm to C
more about sections

crt0

# Execution context
what does a task need to execute

how to configure a tasks context

how to load and store a task



# Looking forward




```c
void scheduler_init(void) {
    // ...
}
```
```

**For images**, put them next to your post by using a "page directory" instead of a standalone `.md` file:
```
content/
  bare-metal-scheduler/
    index.md          ← your post
    scheduler-diagram.png
    memory-layout.png
