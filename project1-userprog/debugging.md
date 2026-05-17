
---

# `project1-userprog/debugging.md`

```md
# Debugging Stories

## 1. Wrong Exit Status in `wait-simple`

### Symptom

A parent waiting on a child sometimes observed the wrong exit status, or saw values that appeared to belong to a different process.

### Root Cause

The child’s lifecycle state was being freed too early. In particular, parent-visible status information was not guaranteed to outlive child thread termination, which led to reuse or corruption of memory that the parent still expected to read during `wait()`.

### Diagnosis Process

The issue showed up as an unexpected exit code during `wait-simple`. Inspection of the parent-child relationship revealed that the status object needed to survive longer than the child execution context itself. This made it clear that process exit and exit-status lifetime could not be treated as the same thing.

### Fix

I introduced an explicit `child_status` object, stored in the parent’s `children` list and referenced by the child through `self_status`. This object tracks:
- exit status
- whether the child has exited
- whether the parent has already waited
- reference count for safe lifetime management

### Lesson Learned

Exit status is not just thread-local termination information. It is a shared parent-child contract, and therefore needs its own object with its own lifetime rules.

---

## 2. FPU Support: Why the First Attempt Broke Everything

### Symptom

Early FPU integration attempts caused severe instability:
- basic userprog tests hung or panicked
- kernel floating-point tests did not pass
- enabling save/restore in the scheduler sometimes caused the system to stop making progress

### Root Cause

There were two root causes at different stages:

1. **The CPU was not yet configured for normal FPU instruction execution**
2. **The FPU initialization strategy was more complicated than necessary**

At first, the redesign focused on save/restore logic before confirming that the CPU control-register configuration allowed those instructions to run safely.

### Diagnosis Process

The key insight was that context-switch code could not be trusted until the FPU itself was correctly enabled at boot. Once the low-level CPU state became the debugging focus, the higher-level failures started to make sense.

### Fix

The final fix had three parts:

1. Enable real FPU execution at boot
2. In `thread_init()`, build one clean global FPU template using:
   - `fninit`
   - `fnsave(initial_fpu_state)`
3. In `schedule()`, preserve FPU state exactly like thread context:
   - save current thread’s FPU state before switching out
   - restore next thread’s FPU state before switching in

### Lesson Learned

When low-level CPU features are involved, correctness must be established from the bottom up. Scheduler-level logic cannot be trusted until the hardware execution model is correct.

---

## 3. `fork-tree` Failed Even After Basic `fork()` Worked

### Symptom

After `fork-simple`, `fork-nested`, and `fork-bad` were already passing, `fork-tree` still failed with an unexpected `fork()` failure deep in the process tree.

### Root Cause

The original page-directory duplication routine scanned the entire user virtual address space page by page from `0` to `PHYS_BASE`. That worked for shallow fork cases, but in deep fork trees it became both:
- too slow
- too resource-intensive

This amplified memory pressure because many child processes were simultaneously mid-duplication and holding newly allocated pages.

### Diagnosis Process

The key observation was that `fork-tree` was not exposing a semantic mistake in parent/child return behavior. Instead, it was exposing an algorithmic inefficiency in address-space copying. Once the duplication routine was examined from a complexity perspective, the bottleneck became obvious.

### Fix

I rewrote `duplicate_pagedir()` to iterate only over:
- present page-directory entries
- present page-table entries

This changed the routine from “scan all possible user pages” to “scan only actually mapped pages,” which dramatically reduced fork cost and stabilized deep fork trees.

### Lesson Learned

Passing simple correctness tests does not guarantee a sound design. Some failures are not about semantics but about algorithmic scaling under realistic process structure.

---

## 4. Why File Descriptors Needed a Shared `open_file` Layer

### Symptom

A direct `fd -> struct file *` design was enough for ordinary file syscalls, but became conceptually wrong once `fork()` was implemented. The challenge was especially visible in inherited file offset semantics.

### Root Cause

After `fork()`, parent and child should have:
- separate descriptor table entries
- but shared underlying open-file state

A single-layer mapping could not naturally represent both facts at once.

### Diagnosis Process

The turning point was reasoning about `fork-offset`. Reopening the file in the child would create independent offset state, while blindly sharing the descriptor object would break independent close behavior.

### Fix

I split file state into two layers:

- `file_descriptor`: per-process descriptor binding
- `open_file`: shared underlying open-file object with reference count

This preserved:
- descriptor-number inheritance
- shared file offset behavior
- independent close semantics

### Lesson Learned

A good OS design often appears when one state object is split into two different lifetime domains.