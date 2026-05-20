# Project 2 Threads

## Overview

This directory documents the **Project 2 user-level multithreading / pthread support** built on top of Pintos. The implementation extends the original "one user process = one thread" model into a model where:

- one process owns one **PCB**
- multiple user threads share that PCB and the same address space
- each user thread has its own:
  - kernel thread
  - user stack
  - `pthread_status`
- user threads may synchronize using user-visible `lock` and `sema` objects implemented by kernel-backed synchronization primitives

The implementation supports:

- creating user threads
- joining user threads
- ordinary thread exit
- main-thread explicit `pthread_exit()`
- correct process-wide cleanup when the **last** user thread exits
- user-level locks
- user-level semaphores
- stack-slot reuse
- cleanup under memory pressure

## What was implemented

### 1. User thread lifecycle
Implemented the full lifecycle for user threads:

- `pthread_execute()`
- `start_pthread()`
- `setup_thread()`
- `pthread_join()`
- `pthread_exit()`
- `pthread_exit_main()`

### 2. Per-process thread tracking
Extended the PCB to track:

- live thread count
- all active `pthread_status` records
- reusable user stack slots
- process exit state
- user lock/semaphore tables

### 3. Process-exit coordination
Implemented a process-exit protocol so that:

- the first thread that triggers a process-wide exit decides the final exit status
- later threads do not re-run full process cleanup
- the last surviving thread frees the PCB and process-owned resources
- parent `wait()` receives the correct exit code exactly once

### 4. User synchronization objects
Implemented user-visible:

- `lock_init`
- `lock_acquire`
- `lock_release`
- `sema_init`
- `sema_down`
- `sema_up`

using kernel synchronization objects stored in per-process tables.

### 5. Memory-pressure robustness
Refined the user sync storage model multiple times and ended with:

- pointer-table design
- per-slot allocation
- lazy / lightweight process-owned storage
- cleanup on all major failure paths

This was necessary to pass both:

- synchronization-heavy tests such as `synch-many` / `pcb-syn`
- memory-pressure tests such as `multi-oom-mt`

## Final status

The entire `tests/userprog/multithreading/*.ck` suite passes.

This includes:

- thread creation tests
- join / exit tests
- user lock tests
- user semaphore tests
- combined synchronization tests
- OOM pressure tests
- stack reuse tests
- PCB synchronization tests

In the broader `make check` run, the remaining failures were only the `smfs-*` fair-scheduler tests, which belong to the unimplemented fair scheduler path rather than the pthread/user-multithreading implementation.

## Directory contents

- `README.md`  
  High-level overview and status.

- `design.md`  
  Detailed design of the pthread and user synchronization implementation.

- `debugging.md`  
  Most important bugs encountered and how they were fixed.

- `testing.md`  
  Testing strategy, coverage, and custom test notes.

- `diagrams/`  
  Mermaid diagrams showing the major execution flows.

## Core idea in one sentence

The final design treats **user multithreading as "multiple kernel threads sharing one PCB and one address space, with per-thread status objects and per-process synchronization tables coordinated through carefully staged exit and cleanup logic."**