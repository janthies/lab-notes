
+++
title = "Building a Bare Metal Scheduler on an STM32H7"
date = 2026-02-08
description = "Notes on writing a simple round-robin scheduler from scratch"
+++

# Starting situation

As a weekend project I decided to give bare metal programming a shot. What started with some basic blinking of an led
and firing an interrupt when a button is pressed turned into writing a small task switcher / scheduler.
By the end of this post I hope you have a rough idea on what path I took, what challenges I encountered and how I overcame them.

As a computer science student myself, I am by no means an authoritative source lalilu. feel free to get in contact 

# The problem
Bare metal programming is fun. It is just you, your development board and a 3357 page reference manual written in dense and technical prose.


# toolchain / build process
If you have done something with an Arduino before you might be familiar with the term **flashing**.
Flashing describes the process of writing executable program code to the Read Only Memory (ROM) of our Microcontroller Unit (MCU). Now there are two major
problems to solve. How do we get executable code and how do we flash it to our MCU?

## Target Board
The board I use is an STM32 [Nucleo-H723ZG](https://www.st.com/en/evaluation-tools/nucleo-h723zg.html) as shown in the photo below.
![test](nucleo.png)
Now inside the red box is
what we care about. It is the [STM32H723 MCU](https://www.st.com/en/microcontrollers-microprocessors/stm32h723-733.html). The MCU itself then contains
a single Arm Cortex-M7 core running at up to 550MHz. Apart from that the MCU also contains the FLASH memory that, for all intents and purposes serves as our ROM.
Yes, we write to Read Only Memory, welcome to the world of embedded programming.

Now inside the blue box is something interesting, it is another MCU! *Wait!* you might say. *Didn't you just tell me that there's only a single core processor?
But now two processors? That doesn't make sense!* Well, I'm sure this is obvious to some but being precise with the language being used is of importance here.
The white Printed Circuit Board (PCB) you can see is the **Development Board**. It contains our MCU (red rectangle) but also a whole bunch of other peripherals.
In the green rectangle are some LEDs that are connected to the MCU and can be lit up on demand. The MCU itself now contains a processing
core, namely an ARM Cortex-M7. In order for us to have an easier time talking to our MCU, the MCU in the blue rectangle is running a software called **ST-Link**.
At a later stage, ST-Link will help us flash some executable code from our computer to the MCU in the red rectangle, our target chip. If the second
MCU were not there, we would need special hardware for communicating with our MCU.

## Compiling


## Linking



## Building an Executable 
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
