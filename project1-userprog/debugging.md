# Debugging Stories

This document records the most important debugging lessons from my redesign of Pintos Project 1: User Programs. I am not treating debugging as a list of random mistakes. Instead, I use each story to explain a specific failure mode, the reasoning process that exposed the root cause, and the design change that fixed it.

---

## 1. Exit Status Could Not Live Inside the Child Thread

### Symptom

Early versions of the process lifecycle logic produced unstable behavior in `wait()`-related tests such as `wait-simple`, `wait-twice`, and `wait-bad-pid`. The parent sometimes observed the wrong exit status, or attempted to wait on a child whose state had already disappeared.

### Root Cause

I initially treated child termination as if the child thread itself were the natural place to store exit-related information. That model was wrong. The parent may need to retrieve the child’s exit status **after the child thread has already exited and released its own resources**. This means the exit status must live in an object with a lifetime longer than the child execution context.

The real issue was not “how to return an integer from `exit()`,” but “how to preserve the parent-child contract across asymmetric lifetimes.”

### Diagnosis Process

The turning point was realizing that `wait()` and `exit()` are not just two separate syscalls. They form a shared protocol between two processes:

* the child must publish an exit status,
* the parent must consume it exactly once,
* and the status object must remain valid until both sides are done with it.

This immediately ruled out any design where exit status lived only in transient thread-local state.

### Fix

I introduced an explicit `child_status` object and made it the shared lifecycle object between parent and child. It stores:

* child pid
* exit status
* whether the child has exited
* whether the parent has already waited
* semaphores for wait/load synchronization
* a reference count for safe deallocation

This object is stored in the parent’s `children` list and linked to the child through `pcb->self_status`.

### Lesson Learned

A process exit code is not just child-local state. It is shared parent-child state and therefore needs its own ownership model and lifetime rules.

---

## 2. `exec()` Needed Explicit Parent-Child Load Synchronization

### Symptom

Even after basic process creation logic worked, `exec()` semantics were still unsafe. The parent could create a child thread and return success before the child had actually finished loading its executable. This created a race: the parent believed process creation succeeded even if the child immediately failed during ELF loading.

### Root Cause

The original flow lacked a synchronization point between:

* parent returning from `exec()`
* child finishing `load()`

This violated the intended contract of `exec()`: success should mean “the child has successfully loaded,” not merely “a child thread was created.”

### Diagnosis Process

The real insight came from reinterpreting the launch process as a two-phase event:

1. child thread creation
2. child executable load success/failure

Those are not the same thing. Without synchronization, the parent observes only phase 1 and misses phase 2.

### Fix

I reused the `child_status` object to hold startup metadata:

* `load_done`
* `load_success`
* `load_sema`

The final flow became:

1. parent allocates `child_status`
2. parent inserts it into `pcb->children`
3. parent creates child thread
4. parent blocks on `load_sema`
5. child performs `load()`
6. child sets `load_done/load_success`
7. child signals parent
8. parent returns pid or `-1`

### Lesson Learned

Creating a thread is not equivalent to creating a process. `exec()` correctness depends on synchronizing process launch completion, not just thread creation.

---

## 3. User Pointer Validation Was Initially Too Weak

### Symptom

System-call tests involving invalid user memory, especially bad-pointer and boundary tests, exposed weaknesses in early syscall validation. Some pointers passed initial checks but still caused unsafe kernel accesses later when the kernel walked a buffer or string beyond the first byte.

### Root Cause

My first validation attempts were too shallow. It is not enough to validate only:

* the pointer itself,
* or the first byte of a buffer,
* or the first character of a string.

The kernel must validate **the full memory region it plans to dereference**.

### Diagnosis Process

The public tests around bad reads, bad writes, and boundary crossing made it obvious that pointer validation is not a single boolean check. It is an access pattern problem:

* for integers, validate the accessed bytes,
* for buffers, validate the entire range,
* for strings, validate until the terminating null byte.

This reframed syscall validation as a per-access safety policy instead of a single pointer filter.

### Fix

I added explicit helpers in `syscall.c`:

* `validate_uaddr()`
* `validate_buffer()`
* `validate_cstring()`
* `get_u32()`
* `get_arg_int()`
* `get_arg_ptr()`

The syscall handler validates arguments **before dereferencing them**, and buffer/string validation walks the relevant memory safely.

### Lesson Learned

At the user-kernel boundary, the kernel must validate the exact memory footprint of an operation, not just the address of its first byte.

---

## 4. ROX Required the Executable to Stay Open for the Entire Lifetime of the Process

### Symptom

Read-only executable (ROX) tests revealed that it was not enough to simply open the ELF file, load its contents, and close it immediately. If the running executable was fully closed too early, other writes could slip through while the process was still alive.

### Root Cause

The mistake was treating the executable like an ordinary input file used only during process load. In reality, the executable must remain protected against writes **while the process is still running**.

### Diagnosis Process

The key realization was that “read-only executable” is not a property of the load step; it is a property of the **running process**. That means the executable file object must remain tracked until process exit.

### Fix

I added `thread->exec_file` and adopted this lifecycle:

1. during load, open the executable
2. call `file_deny_write(exec_file)`
3. keep the file pointer stored in the thread
4. on process exit:

   * call `file_allow_write(exec_file)`
   * close the file

This preserved the intended invariant: a running executable cannot be modified.

### Lesson Learned

Some resources are not just “inputs used during setup.” Their lifetime must match a semantic guarantee that extends across the full process execution window.

---

## 5. FPU Support Failed Until I Solved the CPU-Level Problem First

### Symptom

Early FPU integration attempts caused severe instability:

* simple tests like `do-nothing` and `practice` could hang or panic
* kernel floating-point tests failed
* adding save/restore logic in the scheduler sometimes made the system stop progressing altogether

### Root Cause

I initially focused on scheduler-level FPU state management before fully verifying that the processor was actually configured for safe floating-point execution in the kernel.

The deeper root cause was low-level FPU enablement. Before higher-level save/restore logic could work, the CPU control-register setup had to allow the kernel to execute floating-point instructions at all.

### Diagnosis Process

The debugging process only became productive once I stopped treating FPU failures as purely scheduler bugs and went back to the machine initialization path. Once the CPU enablement issue was treated as the first dependency, the later failures became much easier to reason about.

### Fix

The final design had three parts:

1. enable normal FPU use at boot
2. create a clean initial FPU template in `thread_init()` using:

   * `fninit`
   * `fnsave(initial_fpu_state)`
3. preserve FPU state across context switches in `schedule()`:

   * save current thread’s FPU state before switching out
   * restore next thread’s FPU state before switching in

### Lesson Learned

With low-level CPU features, debugging must proceed bottom-up. Scheduler logic cannot be trusted until the hardware execution model is correct.

---

## 6. `fork()` Had to Be Built in Stages Instead of All at Once

### Symptom

`fork()` was too complex to implement and debug as one giant feature. It combined:

* startup synchronization
* address-space duplication
* register-state continuation
* file-descriptor inheritance
* parent/child lifecycle interactions

Trying to solve all of these at once would have made every failure ambiguous.

### Root Cause

The core problem was not a single bug, but excessive coupling. If `fork()` failed, it was initially unclear whether the issue was in:

* process launch synchronization
* page-table copying
* user register restoration
* file inheritance
* or exit/wait interactions

### Diagnosis Process

The solution was methodological: reduce the number of unknowns. I deliberately built `fork()` in stages so that each phase validated one part of the final behavior.

### Fix

I implemented `fork()` incrementally:

#### Stage 1: startup synchronization skeleton

* parent creates child thread
* parent waits on `load_sema`
* child reports `load_done/load_success`
* child intentionally fails at the end

This validated the launch protocol without requiring full fork semantics.

#### Stage 2: register continuation and address-space duplication

* child allocates PCB
* child duplicates page directory
* child copies parent `intr_frame`
* child sets `eax = 0`
* child returns to user mode through `intr_exit`

#### Stage 3: file-descriptor inheritance

* child duplicates fd table
* shared underlying open-file state is preserved

### Lesson Learned

For large systems features, staged construction is not a workaround. It is often the only practical way to make debugging tractable.

---

## 7. `fork-tree` Exposed an Algorithmic Bottleneck, Not a Semantic Mistake

### Symptom

After `fork-simple`, `fork-nested`, and `fork-bad` were already passing, `fork-tree` still failed with unexpected child startup failures. The core fork semantics seemed right, but deep process trees were not stable.

### Root Cause

The original implementation of `duplicate_pagedir()` walked the entire user virtual address space from `0` to `PHYS_BASE`, page by page. This was functionally acceptable for shallow cases, but much too expensive for fork-heavy process trees.

The result was:

* unnecessary work on unmapped pages
* increased fork latency
* greater overlap between many simultaneously half-constructed child processes
* and more memory pressure during page duplication

### Diagnosis Process

The crucial insight was that `fork-tree` was not disproving the semantics of `fork()`. Parent and child return behavior were already correct. Instead, `fork-tree` exposed a scalability flaw in the duplication algorithm.

### Fix

I rewrote `duplicate_pagedir()` to iterate only over:

* present page-directory entries
* present page-table entries

and to duplicate only actually mapped user pages.

This changed the complexity profile dramatically and stabilized deep process trees.

### Lesson Learned

A design can be semantically correct but still practically wrong if its algorithmic cost explodes under realistic structure.

---

## 8. File-Descriptor Inheritance Required a Two-Layer Design

### Symptom

The original file-descriptor design worked for ordinary file syscalls, but became conceptually wrong once `fork()` needed to preserve inherited file behavior. The most important issue was shared file-offset semantics.

### Root Cause

A one-layer design of the form:

* `fd -> struct file *`

cannot simultaneously model:

* per-process descriptor identity
* shared underlying open-file state
* independent `close()` behavior
* shared offset semantics after `fork()`

Using `file_reopen()` in the child was also wrong, because it would create a new open-file object with its own offset state.

### Diagnosis Process

The turning point came from reasoning about `fork-offset`. The test is really asking for two things at once:

* parent and child should each have their own descriptor entry
* but the descriptor entries should refer to the same underlying open file

That cannot be represented cleanly with a single object.

### Fix

I split the design into two layers:

```c
struct open_file {
  struct file *file;
  int ref_cnt;
};

struct file_descriptor {
  int fd;
  struct open_file *of;
  struct list_elem elem;
};
```

This allowed:

* one `file_descriptor` per process per fd
* shared `open_file` across forked processes
* shared offset behavior
* reference-counted close semantics

This also made the extra `fork-close` test meaningful: one process can close its descriptor without invalidating the other process’s still-live binding.

### Lesson Learned

When one state object is being forced to represent two different lifetime or ownership domains, the correct fix is often to split the abstraction in two.
