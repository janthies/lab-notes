
+++
title = "Building a Bare Metal Scheduler"
date = 2026-02-08
description = "Notes on writing a simple round-robin scheduler from scratch"
+++

# content/bare-metal-scheduler.md

## The Problem

When running on bare metal, there's no OS to schedule tasks for you...

Your normal markdown content goes here. Code blocks work as expected:
yes this is cool
oh yeah
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
