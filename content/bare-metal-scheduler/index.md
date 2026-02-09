
+++
title = "Building a Bare Metal Scheduler on an STM32H7"
date = 2026-02-08
description = "Notes on writing a simple round-robin scheduler from scratch"
[extra]
toc = true
+++

# Starting situation

As a weekend project I decided to give bare metal programming a shot. What started with some basic blinking of an led
and firing an interrupt when a button is pressed turned into writing a small task switcher / scheduler.
In this post I hope to take you along the path I took, the challgenges I encountered and how i overcame them.

In order to appreciate this post you should probably be familiar with the C programming language. I am not talking about crazy macro expansions
and the like but declaring structs, dealing with pointers and  passing functions as arguments to other functions shouldn't be too unfamiliar.
Knowing a thing or two about computer architecture and what a register is would be helpful too.

// TODO
getting in contact

errata
















# The problem
Bare metal programming is fun. It is just you, your development board and a 3357 page reference manual written in dense and technical prose.

























# Building and running code on the ST32H7
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





## Helpful information
Most if not all of the things I convey in this post I have read in the documentation and datasheets regarding the hardware in question. I would like to point
out the relevant documents once more because knowing where to find relevant and original information is what will help you understand most.

The [Nucleo-H723ZG user manual](https://www.st.com/resource/en/user_manual/um2407-stm32h7-nucleo144-boards-mb1364-stmicroelectronics.pdf) contains plenty of useful
information regarding the development board. Here we can find information regarding the flashing process, pin connections and more.

The [STM32H723 reference manual](https://www.st.com/resource/en/reference_manual/rm0468-stm32h723733-stm32h725735-and-stm32h730-value-line-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
was the most daunting document for me when starting out because of its 3000+ page count. Register addresses, offsets, usage, the functional description of certain hardware blocks,
documentation on all peripherals, the clock tree. This document has it all. Knowing where to look in order to find relevant information can be a bit challengeing but once gets used to it.

Next we have the [Cortex-M7 generic user guide](https://documentation-service.arm.com/static/61efd6602dd99944d051417b?token=) and the [Cortex-M7 technical reference manual](https://documentation-service.arm.com/static/5e906b038259fe2368e2a7bb?token=).
Whenever we need to find processor core specific documentation, this is the go to source.

Lastly, knowing the differences between the Cortex-M7 and the STM32H723 can be a bit confusing at first. The Cortex-M7 is the processor core, doing the computation. The STM32H7
is our **System on Chip (SoC)**. The SoC contains the Cortex-M7, furthermore it contains FLASH storage that can hold our executable code. RAM that serves as the working memory
for the core. Different functional blocks like the USART block for transceiving serial data or the EXTI block for interrupt handling.

When the need arises, we will go much deeper on each individual topic.
For now, having a grasp on the different levels of abstraction we are dealing with and where to find relevant information should suffice.





## Target architecture
Compiling our code for the target platform is the next step. In the world of embedded we usually talk about **cross-compilation** because the computer
we write and compile the code on is usually somewhat different from the platform we are targetting. As a little experiment, running *lscpu* give the following output:

```c
jan@jan:~$ lscpu
Architecture:                x86_64
  CPU op-mode(s):            32-bit, 64-bit
  Address sizes:             39 bits physical, 48 bits virtual
  Byte Order:                Little Endian
CPU(s):                      8
  On-line CPU(s) list:       0-7
Vendor ID:                   GenuineIntel
  Model name:                11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz
  ...

```
Now we can see that the architecture is x86_64. Taking a look at the [ARM Cortex-m7 reference manual](https://documentation-service.arm.com/static/5e906b038259fe2368e2a7bb?token=)
we find under section *1.4.1* that this processor is implementing the Armv7E-M architecture. That is quiet different from my pc! The architecture, more formally known as
an **Instruction Set Architecture** is the interface between the software we write and the hardware that is going to execute it. We need to find a compiler that translates
our already architecture specific assembly code, and later also our architecture agnostic C code, into machine that can be executed by the STM32H7.





## Writing the code that brings our MCU to life
The code I am talking about is so called startup code. This is the code that runs first and configures our MCU in a usable way.
Taking a look in the [generic user guide](https://documentation-service.arm.com/static/61efd6602dd99944d051417b?token=) for our
ARM Cortex-M7, we can find some words regarding the reset behaviour of our chip in section 2.1.3. Skipping past the core registers
we find the following text:

*The Program Counter (PC) is register R15. It contains the current program address. On reset, the processor loads the PC with the value
of the reset vector, which is at the initial value of the Vector Table Offset Register (VTOR) plus 0x00000004.
Bit[0] of the value is loaded into the EPSR T-bit at reset and must be 1*.

Now this is interesting and tells us exactly what to do! The second we give electrical power to our MCU, it looks at a specific memory address,
loads the data at that memory address into the **Program Counter** and then continues executing from there. Now the program counter tells us which command the
 MCU is currently executing. The text also mentioned the **Reset Vector**. The reset vector is a routine whose address we would like to store in that specific
memory address, so that it runs right away when the system first receives power

A bit further down under section 2.3.4 we can read about the **Vector Table**. The vector table is an area in memory that stores a bunch of important addresses
to routines the MCU would like to execute in certain scenarios. Those scenarios include system reset, different fault conditions we might want to recover from
but also repeating time intervals that might remember us to do something. 

**Vector Table of the Cortex-M7**

![Vector Table](vector_table.png)
*[generic user guide P. 36](https://documentation-service.arm.com/static/61efd6602dd99944d051417b?token=)*

In the vector table we can see that our reset vector is at offset 0x0004. That is 32 bit above the Initial Stack Pointer Value entry. Now the inital SP value is of importance too!
The stack is continuous memory serving as our programs scratch space. Whenever we need to remember important data we can put it there. The SP points to the head of our stack. Pushing
i.e. 4 bytes onto our stack moves our SP by ... exactly, 4 bytes. Popping those same 4 bytes again moves our SP. The processor does this automatically for us. Under section 2.1.2 in the
Cortex-M7 generic user guide, we find that the stack pointer decreases on push and increases on pop. This implies that later, when figuring out an appropriate value for our initial SP
assignment, we should take the last address of our stack address space.

The initial SP is located at offset 0x0000. *Offset?* you might wonder. Yeah exactly, we now know the location of our reset vector and initial SP relative to the vector table
but the vector table itself is located at a different location.

In order to know at which memory location the vector table lives, we need to take a look at the STM32H7 reference manual again. Remember? Flash and Ram are part of our SoC, information
regarding the positioning and boot address thus can be found there. Under section 2.6 in the reference manual we find the following table:
![Boot modes](boot_modes.png)
Here we get to know about the boot process of the STM32H7 and that the standard boot address for this specific MCU is 0x08000000. That is exactly where we want to put our vector
table once it is done.

Writing what we know so far as code gives us the following glorious three lines of assembly (asm).

**main.s**
```C
.section .vtable
  .word _estack
  .word reset_handler
```

Now a couple questions arise that I would like to point out at this moment in time.
We have not yet defined our inital SP value called *_estack*. We don't know what a *.section* is.
There is no *reset_handler* yet and lastly, how do we put all of this at the right position in Flash so
that our MCU can actually find it.

I would like to start with our two yet undefined variables. Compiling our *main.s* program with *arm-none-eabi-as*
yields a file called *a.out*. Further inspecting *a.out* with *arm-none-eabi-nm* we see that both *_estack* and *reset_handler*
are marked as *U*, meaning Undefined.

```C
jan@jan:~$ arm-none-eabi-as main.s 
jan@jan:~$ ls
a.out  main.s
jan@jan:~$ arm-none-eabi-nm a.out 
         U _estack
         U reset_handler
```
*Compiling and inspecting main.s*

*_estack* we will define at a later stage as we will first need to know where in memory our stack should reside. We can, however,
provide a *reset_handler* at this point in time. *reset_handler* is the routine that will eventually launch our application,
call other routines that set up our execution environment and eventually yield to our trusty **main()** function. For now though,
*reset_handler* will do nothing.

```C
.section .vtable
        .word _estack
        .word reset_handler

.section .text
.thumb_func
reset_handler:
loop:
        nop
        b loop
```
*main.s is now equipped with a simple yet useless reset handler*

We are now providing a routine that executes a bunch of *nop* (no operation) instructions in a loop, effectively idling.
Describing what is happening here we have a symbol *reset_handler* that is also the second entry in our *vector table*.
We have another symbol *loop* right past *reset_handler* so that the control flow is falling through *reset_handler*
to *loop*. *loop* contains two instructions, our *nop* and a *b* (branch) back to the loop label. *.thumb_func* might not be so obvious
at the first glance. *.thumb_func* is an assembler directive telling the assembler that the next function is a thumb function. *Shocker, I know*.
You might remember the excerpt from the generic user guide at the start of this chapter saying that *Bit[0] of the value is loaded into
the EPSR T-bit at reset and must be 1*. This is what *thumb_func* takes care of for us. At this point in time I prefer to not go depth on
ARM & Thumb mode but further reading can be done [here](https://azeria-labs.com/arm-instruction-set-part-3/). I would just like to say that this
directive is not optional and your code will not run should you forget adding the label.

I want to cover sections next. Sections let us group similar information together. 
Grouping data with data and code with code turns out to be handy for a number of reasons. A more thorough
walkthrough of the **Executable and Linkable Fileformat** (ELF) can be found [here](https://www.linuxjournal.com/article/1059).
For our case it will be important that our uninitialized and initialized data reside in different sections, namely *.bss* and *.data*.
Some special handling is needed when we eventually want to level up from asm to C as C has some requirements on what has to happen before
*void main(void)* is called.

```C
int uninitialized_data;
int initialized_data = 42;
```
*Depending on how variables are declared in C they will reside in different sections of the binary*


### Inspecting the code we just created
I would like to take the time to look at what we have got so far. Running our Assembler *arm-none-eabi-as* on the latest version
of *main.s* gave us a file called *a.out*. We can inspect *a.out* with a tool called *arm-none-eabi-objdump*. Here we find our
*reset_handler* again. Noteworthy is the address of 00000000 right next to *reset_handler*. Earlier we have noted that,
in order for our program to run on an STM32H7 we need to put *reset_handler* at location 0x08000000. Yet right now it is at
location 0x00000000 Earlier we have noted that,
in order for our program to run on an STM32H7 we need to put *reset_handler* at location 0x08000000. Yet right now it is at
location 0x00000000. *This ain't gonna work then!* I hear you say. *You're god damn right* I reply. In the next chapter we will
learn how we can link and relocate our relocatable object file into an exectuable file that an STM32H7 can understand.

```C
jan@jan:~$ arm-none-eabi-objdump -dS a.out 

a.out:     file format elf32-littlearm


Disassembly of section .text:

00000000 <reset_handler>:
   0:   46c0            nop                     @ (mov r8, r8)
   2:   e7fd            b.n     0 <reset_handler>
```
*Inspecting the contents of a.out with arm-none-eabi-objdump*


## Linking
relocatable code

basic linker script

basic sections (.text .bss. data)

## Flashing




# Blinky example
building a small blinky application


## your three best friends
memory map / clock tree / gdb


## Blinking an led
reading the reference sheet

configuring and writing to some basic registers


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
