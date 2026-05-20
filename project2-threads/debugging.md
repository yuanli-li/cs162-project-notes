# Debugging

This file records the most important bugs encountered during the Project 2 threads implementation, ordered roughly by importance and impact.

---

## 1. Start-argument lifetime bug: `args` freed too early

### Symptom
Tests such as:

- `create-reuse`
- `create-many`

would hang or time out.

### Root cause
The parent thread signaled the child and then freed the thread-start argument structure too early.  
The child thread had not yet finished consuming that structure.

As a result:

- the memory could be reused
- semaphores inside the structure could be overwritten
- the child could block forever on corrupted synchronization state

### Fix
The lifetime of the start-argument structure was transferred to the child:

- parent no longer frees it on the success path
- child frees it only after:
  - setup completed
  - parent bookkeeping completed
  - `start_sema` allowed it to continue

### Why this mattered
This was one of the most important correctness fixes for thread creation reuse tests.

---

## 2. `process_exit_with_status()` did not wake `pthread_join()` waiters

### Symptom
Tests such as:

- `exec-thread-1`
- `file-join`
- some join/exit combinations

would time out.

### Root cause
`pthread_exit()` correctly marked the thread's status as exited and signaled `join_sema`, but `process_exit_with_status()` did not do the same.

So if a thread died via:

- `exit(status)`
- process-killing exception

then:

- `pthread_status->exited` stayed false
- joiners slept forever on `join_sema`

### Fix
Made `process_exit_with_status()` mirror the join-notification portion of `pthread_exit()`:

- mark `tstatus->exited = true`
- `sema_up(&join_sema)`

### Why this mattered
This unified thread-exit semantics across both thread-local and process-wide exit paths.

---

## 3. Pure `pthread_exit()` shutdown path never notified the parent

### Symptom
Some tests involving thread-only termination behavior would hang at the very end even though all user threads were gone.

### Root cause
Process-level cleanup originally happened only when a thread entered the explicit process-exit path.  
If all threads ended via `pthread_exit()` and no one ever called `exit(status)`, then:

- no `"exit(status)"` line was printed
- parent `wait()` was never awakened
- the process could finish internally but still never complete externally

### Fix
Added logic so that the **last** thread may also perform process-level cleanup if:

- it is freeing the PCB
- and `process_exiting` was never set

This allowed "all threads ended normally" to still behave like a real process termination.

### Why this mattered
This was essential for the `"main thread exits first, worker finishes later"` scenario.

---

## 4. Process exit repeated many times after pagedir destruction

### Symptom
Observed earlier incorrect behavior such as:

- the same exit status printed multiple times
- many page-fault diagnostics after one thread already started process exit
- parent receiving inconsistent final exit information

### Root cause
After one thread destroyed the pagedir, the remaining threads could still run briefly and fault when touching user memory.  
Without a coordinated first-exiter protocol, those later threads would re-enter full process-exit logic.

### Fix
Introduced a strict process-exit coordination rule:

- the **first** exiting thread sets:
  - `process_exiting`
  - final `exit_status`
- later threads do not re-run full process cleanup
- later user faults during teardown are handled quietly

### Why this mattered
This stabilized process exit and prevented duplicated cleanup behavior.

---

## 5. PCB / pagedir lifetime bug: freeing process-owned state too early

### Symptom
Mixed crashes, incorrect exits, or instability after one thread initiated process exit while other threads still existed.

### Root cause
The PCB or pagedir could be destroyed while other threads still had pointers into process-owned structures.

That meant later accesses by surviving threads could become:

- use-after-free
- stale pagedir access
- invalid synchronization table access

### Fix
Separated:

- thread exit
- process-wide cleanup
- final PCB free

Only the last surviving thread is allowed to free process-owned structures.

### Why this mattered
This was a foundational lifecycle fix that made all other exit logic reliable.

---

## 6. Main thread had no `pthread_status`

### Symptom
Tests such as:

- `join-exit-1`
- `pcb-syn`

behaved inconsistently.

### Root cause
Worker threads had `pthread_status`, but the main thread in some paths did not.  
That meant the main thread was not fully represented in the process-wide thread-status model.

### Fix
Created a `pthread_status` for the main thread in:

- `userprog_init()`
- `start_process()`
- `start_fork()`

### Why this mattered
It made the main thread behave uniformly under join/exit/cleanup rules.

---

## 7. User sync storage design fought against both sync-heavy tests and OOM tests

### Symptom
The implementation oscillated between failing:

- `synch-many`
- `pcb-syn`

or failing:

- `multi-oom-mt`

depending on the storage design.

### Root cause
Several versions were tried:

1. small embedded arrays  
   not enough capacity for sync-heavy tests

2. large embedded arrays  
   made the PCB too large and harmed OOM depth

3. large dynamic blocks  
   reduced PCB size but still caused coarse-grained memory pressure / fragmentation

### Final fix
Settled on:

- pointer-table design
- per-slot allocation
- lightweight process-owned base tables
- allocate actual lock/sema objects only when initialized

### Why this mattered
This was the key design change that allowed both synchronization-heavy and OOM-heavy tests to pass together.

---

## 8. Invalid user lock/semaphore pointers should return `false`, not kill the process

### Symptom
Tests like:

- `lock-init-fail`
- `sema-init-fail`

initially caused process termination instead of graceful failure.

### Root cause
The lock/sema syscalls originally reused the same "fatal invalid pointer" path used by regular file syscalls.

But user sync syscalls are specified to behave like API operations:

- bad sync-object input should usually return failure
- not necessarily kill the process

### Fix
Added / used nonfatal user-pointer validation for lock/sema syscalls so that bad sync-object arguments return `false`.

### Why this mattered
This was necessary for the failure-mode tests and for clean API semantics.

---

## 9. Missing ID loads in release/down/up paths

### Symptom
Some early sync operations behaved unpredictably even when initialization appeared correct.

### Root cause
A few syscall handlers declared an `id` variable but forgot to actually load the ID value from the user object before checking it.

This affected paths like:

- lock release
- semaphore down
- semaphore up

### Fix
Explicitly loaded the ID from user memory in all handlers before range and validity checks.

### Why this mattered
This was a direct logic bug that caused incorrect sync-object validation.

---

## 10. Failure-path leaks in process/thread startup

### Symptom
OOM-sensitive tests were off by a small but persistent amount, and some failed-startup cases leaked state.

### Root cause
Failure paths in:

- `start_process()`
- `start_fork()`
- `process_execute()`
- `pthread_execute()`

did not consistently release all partially created objects such as:

- `child_status`
- page directories
- `pthread_status`
- start-argument structures
- user sync tables
- stack-slot metadata

### Fix
Audited and repaired failure cleanup so that each startup path frees everything allocated so far before exiting.

### Why this mattered
This cleanup discipline was required to pass the most memory-sensitive tests.

---

## Summary

The most important debugging theme in this project was not "one big wrong algorithm", but rather **resource lifetime correctness under concurrency**.

The hardest bugs all came from one of the following categories:

- who owns an object
- who is allowed to free it
- who wakes whom
- when process-wide cleanup begins
- when process-wide cleanup is allowed to finish

Once those ownership and lifecycle rules were made explicit, the implementation became stable and the full multithreading suite passed.