# Project 1: User Programs

## Problem

This project extends Pintos from a threads-only teaching kernel into one that can safely run user programs.

The major challenges were:

- building a safe syscall boundary
- validating untrusted user pointers
- implementing process launch, exit, and wait semantics
- managing file descriptors correctly
- protecting running executables from writes
- preserving floating-point state across thread switches
- implementing `fork()` with correct parent/child semantics, address-space duplication, and file-descriptor inheritance

## Design

I approached this project as a full redesign rather than a patchwork of test-specific fixes.

The final design centered on a few major ideas:

- separating **process state** from **thread state**
- using an explicit **child lifecycle object** for startup, wait, and exit coordination
- validating all user memory before dereferencing it in the kernel
- treating floating-point state as part of the thread context
- implementing `fork()` incrementally:
  - startup synchronization
  - register-state continuation
  - address-space duplication
  - file-descriptor inheritance

## Key Data Structures

The most important structures in the final design were:

- **`struct process`**  
  Owns the user page directory, process name, and parent-child metadata.

- **`struct child_status`**  
  Bridges parent and child for load synchronization, wait semantics, exit status, and lifetime management.

- **`struct file_descriptor`**  
  Represents a process-local fd binding.

- **`struct open_file`**  
  Represents a shared underlying open-file object with reference counting, used to preserve fork offset semantics.

- **`thread->fpu_state`**  
  Stores each thread’s saved floating-point context.

## Execution Flow

Detailed writeups and diagrams:

- [Design](./design.md)
- [Debugging Stories](./debugging.md)
- [Validation](./validation.md)

Diagrams:

- [Process Lifecycle](./diagrams/process-lifecycle.md)
- [System Call Flow](./diagrams/syscall-flow.md)
- [Fork Flow](./diagrams/fork-flow.md)
- [FPU Flow](./diagrams/fpu-flow.md)

## Highlights

- Designed a robust parent/child lifecycle model using explicit child status objects
- Implemented full `fork()` semantics including address-space duplication and inherited file-descriptor behavior
- Added floating-point support with per-thread FPU context save/restore
- Added extra tests beyond the provided public suite

## Skills Practiced

- Kernel boundary design
- Synchronization and lifecycle reasoning
- Memory validation and defensive programming
- Low-level debugging
- Systems testing and correctness reasoning