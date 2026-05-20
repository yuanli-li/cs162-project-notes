# Design

## 1. Goals

The goal of the Project 2 threads portion is to support **multiple user threads inside one user process** while preserving correct process semantics.

The design must support:

- creating additional user threads inside an existing process
- joining terminated user threads
- correct stack allocation and reuse
- main-thread exit without prematurely destroying the process
- process-wide exit when any thread calls `exit(status)` or faults
- parent `wait()` correctness
- user-visible locks and semaphores
- cleanup under failure and out-of-memory conditions

---

## 2. High-level architecture

The implementation separates:

- **process-owned state** in the PCB
- **thread-owned state** in the TCB and `pthread_status`

### Process-owned state
A process owns:

- page directory / address space
- process name
- child-process bookkeeping
- process exit state
- live thread count
- pthread status list
- reusable user stack slot list
- user lock/semaphore tables

### Thread-owned state
A user thread owns:

- kernel thread identity
- user thread ID
- user stack mapping
- pointer to its `pthread_status`
- thread-local file descriptor state

This division allows multiple user threads to share the same process while still exposing per-thread lifecycle operations.

---

## 3. Data structures

## 3.1 `struct process`

The PCB was extended with the following groups of fields.

### Thread lifecycle fields

- `thread_count`  
  Number of live user threads in the process.

- `thread_count_lock`  
  Protects `thread_count` and related process-exit decisions.

- `next_user_stack`  
  Tracks the next candidate virtual stack location for a newly created user thread.

### User-thread bookkeeping

- `pthread_status_list`  
  List of all thread status objects associated with this process.

- `pthread_status_lock`  
  Protects the thread status list.

### User stack reuse

- `free_user_stacks`  
  List of reusable stack slots.

- `free_user_stacks_lock`  
  Protects the reusable stack list.

### User synchronization

Final design:

- `user_locks`
- `user_semas`
- `user_sync_lock`

The final implementation uses pointer tables plus per-slot allocation rather than embedding large lock/semaphore arrays directly in the PCB.

### Process exit coordination

- `exit_status`
- `exit_status_set`
- `process_exiting`

These fields ensure that process-wide exit is decided once and cleaned up exactly once.

---

## 3.2 `struct pthread_status`

Each user thread has a `pthread_status` containing:

- `tid`
- `exited`
- `joined`
- `user_stack_top`
- `join_sema`

This structure is the core object for:

- `pthread_join()`
- exit notification
- stack reclamation

A thread may terminate before it is joined, so the status object must outlive the thread itself until a joiner reaps it.

---

## 3.3 `struct free_stack_slot`

A reusable user stack slot stores:

- `user_stack_top`

Instead of always carving out a new stack address, the implementation reuses old stack virtual regions once threads are joined and their pages are unmapped.

---

## 3.4 `struct child_status`

`child_status` is shared between parent and child process logic and stores:

- `pid`
- `exit_status`
- `exited`
- `waited`
- `load_done`
- `load_success`
- `wait_sema`
- `load_sema`

This structure supports:

- `exec`
- `fork`
- `wait`

It is not a pthread-specific object, but process-exit logic in the pthread design depends on it.

---

## 4. Thread creation design

## 4.1 Entry point: `pthread_execute()`

`pthread_execute(stub_fun, pthread_fun, arg)` is called from the syscall layer when a user process requests a new user thread.

The parent thread performs the following work:

1. allocate a start-argument structure
2. allocate a `pthread_status`
3. initialize `pthread_status`
4. insert the `pthread_status` into `pcb->pthread_status_list`
5. create a kernel thread whose start function is `start_pthread`
6. increment `thread_count`
7. wait for the child thread to report initialization success/failure
8. either:
   - complete creation and let child continue, or
   - roll back thread_count and thread status on failure

The parent uses two semaphores inside the start-argument structure:

- `"loaded"`  
  child -> parent notification that setup finished
- `"start_sema"`  
  parent -> child permission to proceed

This two-step handshake prevents races between:

- child setup
- parent bookkeeping
- failure cleanup

---

## 4.2 Child bootstrap: `start_pthread()`

The newly created kernel thread does not immediately enter user code.

Instead, `start_pthread()`:

1. binds itself to the shared PCB
2. initializes thread-local state
3. attaches its `pthread_status`
4. calls `setup_thread()`
5. reports success/failure through `"loaded"`
6. waits on `"start_sema"`
7. enters user mode through `intr_exit`

This guarantees that the child does not race ahead into user mode before the parent has completed bookkeeping.

---

## 4.3 User-mode setup: `setup_thread()`

`setup_thread()` is responsible for creating the child thread's user execution context.

It does two main jobs:

### A. Stack allocation
The implementation first tries to reuse a stack slot from `free_user_stacks`.  
If none is available, it allocates a new user stack region below `PHYS_BASE`.

Then it:

- allocates a physical page
- maps it into the process page directory
- remembers the stack top in the thread status

### B. Stack layout
The stack is arranged so that user execution begins at the user stub function, with:

- `"arg"`
- `"thread_fun"`
- a fake return address

The child enters user mode at the stub, which then invokes the real user thread function.

---

## 5. Join design

## 5.1 Entry point: `pthread_join(tid)`

`pthread_join()` must:

- reject invalid tids
- reject self-join
- reject duplicate join
- block until the target thread exits
- reclaim thread resources exactly once

The algorithm is:

1. lock `pthread_status_lock`
2. find target `pthread_status`
3. validate it
4. mark `"joined" = true`
5. unlock the list
6. if target not exited, wait on `join_sema`
7. reclaim target user stack
8. insert reusable stack slot into `free_user_stacks`
9. remove the `pthread_status` from the list
10. free it

The important design decision is that **thread exit only marks the thread dead; join performs final reap and stack reclamation**.

---

## 5.2 Why `joined` is set before waiting

The `"joined"` flag is set before sleeping so that only one joiner can claim the target thread.

Without this, two threads could both observe the target as joinable and race into duplicate wait/reap logic.

---

## 6. Ordinary thread exit design

## 6.1 Entry point: `pthread_exit()`

A normal user thread exit performs:

1. mark its own `pthread_status` as exited
2. `sema_up(join_sema)` to wake a joiner
3. close thread-local file descriptors
4. decrement `thread_count`
5. if this was the last thread:
   - invoke process-wide cleanup
6. otherwise:
   - call `thread_exit()`

This preserves the distinction between:

- `"thread exit"`
- `"process exit"`

A thread may disappear while the process remains alive.

---

## 6.2 Main thread special case

The main thread may explicitly call `pthread_exit()`.

That must **not** destroy the process if other user threads are still alive.

The design handles this by letting the main thread use the same thread exit path:

- mark its own thread status exited
- reduce `thread_count`
- if not last, disappear as a thread only

The process itself only ends when the last thread exits.

This behavior is crucial for the `"main-ptexit"` scenario.

---

## 7. Process-wide exit design

## 7.1 Entry point: `process_exit_with_status(status)`

This path is triggered by:

- `exit(status)`
- user exceptions / page faults
- other process-termination events

This is distinct from `pthread_exit()` because `exit(status)` has **process-wide semantics**.

The algorithm is:

1. mark the current thread's `pthread_status` as exited
2. wake possible joiners
3. acquire `thread_count_lock`
4. determine whether this thread is the **first** process-exiting thread
5. if first:
   - set `"process_exiting" = true`
   - record final `"exit_status"`
6. decrement `thread_count`
7. determine whether current thread is the last remaining thread
8. call `process_cleanup_and_exit(exit_status, last, is_first)`

---

## 7.2 First-exiter rule

Only the first thread to trigger process-wide exit is allowed to decide:

- the final exit code
- the parent wakeup
- the exit message
- pagedir destruction

This avoids earlier incorrect behavior such as:

- repeated `"exit(status)"` prints
- later page-faulting threads overwriting the exit code
- duplicated parent notifications

---

## 8. Unified cleanup design

## 8.1 `process_cleanup_and_exit(status, free_pcb_now, is_first)`

This function is the unified final cleanup path used by both:

- `pthread_exit()`
- `process_exit_with_status()`

It performs:

### A. Thread-local cleanup
Always:

- close current thread's file descriptors
- close `exec_file` if present

### B. Decide whether this thread is responsible for process-level cleanup
Process-level cleanup must happen if:

- `"is_first"` is true  
  meaning this thread is the first process-wide exiter

or

- `"free_pcb_now"` is true and `"process_exiting"` is false  
  meaning all threads ended via `pthread_exit()` and the last thread must still finalize the process

### C. Process-level cleanup
If responsible:

- print `"exit(status)"`
- write exit status to `self_status`
- wake parent's `wait()`
- destroy pagedir

### D. Final PCB cleanup
If `free_pcb_now` is true:

- free all remaining `pthread_status` records
- free all stack-slot records
- free user lock/semaphore storage
- free the PCB itself

Finally:

- `thread_exit()`

---

## 8.2 Why this unified design matters

This unified cleanup solves two difficult cases:

### Case 1: explicit process exit from one thread
Only the first exiter does process-wide work.  
Later threads quietly disappear.

### Case 2: all threads end via `pthread_exit()`
No thread ever called `exit(status)`, so the **last** thread must still:

- print the process exit message
- notify the parent
- destroy the pagedir

Without this second case, tests involving main-thread `pthread_exit()` and clean final shutdown fail.

---

## 9. User lock / semaphore design

## 9.1 Core idea

User-level synchronization objects do not contain the actual kernel object.  
Instead, the user object stores a small ID, and the PCB maps that ID to a real kernel lock/semaphore.

Final design:

- `"0"` is reserved as invalid / uninitialized
- valid IDs begin at `"1"`
- the PCB holds:
  - `struct lock** user_locks`
  - `struct semaphore** user_semas`

Each slot is allocated on demand.

---

## 9.2 Why not embed large arrays directly in the PCB

The design evolved through multiple versions.

### Embedded arrays
Pros:
- simple indexing

Cons:
- PCB became too large
- `multi-oom-mt` lost depth
- scaling capacity harmed memory pressure tests

### Large dynamic blocks
Pros:
- smaller PCB

Cons:
- still required coarse-grained large allocations
- memory fragmentation hurt stability after OOM-heavy tests

### Final design: pointer table + per-slot allocation
Pros:
- small PCB
- lightweight per-process base allocation
- only allocate actual sync objects when used
- robust under pressure
- passes both `synch-many` and `multi-oom-mt`

---

## 9.3 `lock_init` / `sema_init`

Initialization performs:

1. validate user pointer non-fatally
2. acquire `user_sync_lock`
3. ensure the pointer table exists
4. find a free slot ID
5. allocate one kernel sync object for that slot
6. initialize it
7. write the slot ID back into the user object
8. release `user_sync_lock`

Bad initialization input returns `false` rather than killing the process.

---

## 9.4 `lock_acquire` / `lock_release`

Acquire/release perform:

1. validate the user pointer
2. read the ID byte from the user object
3. check that the ID is valid and mapped
4. for acquire:
   - reject reacquiring a lock already held by the current thread
5. for release:
   - reject releasing a lock not held by the current thread
6. invoke the real kernel lock operation

---

## 9.5 `sema_down` / `sema_up`

Semaphore operations are similar:

1. validate pointer
2. read ID
3. verify the slot exists
4. call kernel `sema_down()` / `sema_up()`

---

## 10. Failure-path cleanup

A major design concern was cleaning up partially initialized process/thread state.

The implementation now cleans up correctly on failure in:

- `start_process()`
- `start_fork()`
- `pthread_execute()` failure paths

This includes:

- page directories
- child-status structures
- `pthread_status` objects
- user sync tables
- stack-slot lists
- start-argument lifetime

These cleanups were necessary to pass OOM-sensitive tests.

---

## 11. Fork interaction

Although this document focuses on the pthread portion, fork correctness is tightly connected.

The fork path:

- allocates a new PCB
- initializes child process thread structures
- duplicates page tables
- duplicates fd state
- attaches child `self_status`
- returns `"0"` in the child via `if_.eax = 0`

The child main thread is also represented by a `pthread_status`, so pthread logic stays uniform inside fork-created processes.

---

## 12. Summary of final design decisions

The final implementation is built on the following principles:

1. one PCB per process
2. one `pthread_status` per user thread
3. join reaps thread resources
4. `pthread_exit()` only kills the current thread unless it is the last one
5. `process_exit_with_status()` has process-wide semantics
6. only the first process-exiting thread chooses final process exit state
7. the last remaining thread frees process-owned resources
8. user lock/sema objects use ID mapping into PCB-owned pointer tables
9. sync storage is lightweight and per-slot allocated
10. all major failure paths clean up partially allocated resources

---

## 13. Design summary in one paragraph

The final pthread design extends Pintos from a single-threaded user-process model into a shared-PCB multithreaded model where each user thread has its own kernel thread, user stack, and join status, while all threads in the process share one address space and one PCB. Thread creation uses a two-phase parent/child handshake, joining reaps thread state and stack pages, normal thread exit only removes the current thread, process-wide exit is coordinated by a first-exiter rule, and the final thread is responsible for freeing process-owned resources. User-visible locks and semaphores are implemented as ID-mapped kernel synchronization objects stored in PCB-owned pointer tables with per-slot allocation to balance scalability and memory-pressure robustness.