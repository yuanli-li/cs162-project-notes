# Testing

## 1. Testing goals

The testing strategy for the Project 2 threads portion focused on four major goals:

1. verify user-thread lifecycle correctness
2. verify synchronization correctness
3. verify process-exit correctness
4. verify memory-pressure robustness

Because many bugs only appeared under particular interleavings or under partial-failure conditions, the testing process relied on:

- existing Pintos multithreading tests
- repeated focused reruns of failing tests
- full regression runs
- one custom test covering an important under-tested scenario

---

## 2. Existing multithreading test coverage

The full `tests/userprog/multithreading/*.ck` suite was run and passed.

This suite covers the following categories.

### 2.1 User lock tests

- `lock-simple`
- `lock-init-fail`
- `lock-dbl-acq`
- `lock-dbl-rel`
- `lock-acq-fail`
- `lock-data`
- `lock-ll`

These verify:

- initialization
- invalid initialization
- repeated acquire/release rejection
- correct mutual exclusion
- lock ordering behavior

### 2.2 User semaphore tests

- `sema-simple`
- `sema-init-fail`
- `sema-up-fail`
- `sema-wait`
- `sema-wait-many`

These verify:

- initialization behavior
- invalid initialization handling
- signaling and waiting semantics
- multiple waiters

### 2.3 Combined synchronization tests

- `synch-many`
- `pcb-syn`

These stress:

- many user sync objects in one process
- synchronization correctness under shared PCB state

### 2.4 Thread creation and reuse

- `create-simple`
- `create-many`
- `create-reuse`
- `reuse-stack`

These verify:

- basic creation
- creation under load
- repeated creation/destruction cycles
- user-stack reuse correctness

### 2.5 Join / exit tests

- `join-fail`
- `join-recur`
- `join-exit-1`
- `join-exit-2`
- `exit-simple`
- `exit-clean-1`
- `exit-clean-2`

These verify:

- invalid join rejection
- recursive / repeated join behavior
- thread exit propagation
- process-wide exit correctness
- cleanup after exit

### 2.6 Interaction tests

- `file-join`
- `wait-fail`
- `exec-thread-1`
- `exec-thread-n`

These verify:

- join interacting with file activity
- wait semantics
- thread/process interactions with exec

### 2.7 Stress / robustness tests

- `multi-oom-mt`
- `arr-search`

These stress:

- repeated creation and destruction
- recursion depth under pressure
- memory cleanup robustness

---

## 3. Full multithreading regression result

A full regression over all multithreading tests passed.

This confirmed that the implementation was not just passing isolated single tests, but was also stable across the entire user multithreading suite.

---

## 4. Broader regression results

After pthread and user sync support were stabilized, `make check` was run.

Results:

- threads tests under FIFO / PRIO passed
- userprog tests passed
- fork tests passed
- multithreading tests passed
- current filesys/base tests passed

The only remaining failures in the broader check were the `smfs-*` fair-scheduler tests, which fail because the fair scheduler path is not implemented. Those failures are not caused by the pthread/user-multithreading implementation.

---

## 5. Custom test requirement

The project requires one additional student-written test that covers functionality not fully covered by the existing suite.

A custom test was added:

- `main-ptexit`

### Why this test was chosen
A particularly fragile and important scenario is:

- main thread explicitly calls `pthread_exit()`
- worker thread must continue running
- process must **not** terminate immediately
- final process exit must still happen correctly after the last worker exits

This path was important in the implementation and had required dedicated fixes, but no existing test name directly and explicitly targeted that scenario.

---

## 6. Custom test: `main-ptexit`

## 6.1 Purpose

This test verifies that:

1. the main thread can explicitly call `pthread_exit()`
2. the process remains alive while another user thread is still running
3. the worker thread continues normally
4. the final process exit occurs only after the worker finishes
5. the final exit status is correct

## 6.2 Scenario

The test:

1. prints `"main start"`
2. creates a worker thread
3. prints `"main exiting"`
4. calls `pthread_exit()`
5. worker prints `"worker start"` and `"worker done"`
6. process finally exits with status `0`

## 6.3 Why it matters

This test directly targets the correctness of:

- main-thread exit semantics
- distinction between thread exit and process exit
- last-thread cleanup responsibility
- final exit printing / parent notification path

---

## 7. Debug-driven targeted reruns

During development, targeted reruns were used heavily.

Typical examples included:

- rerunning `create-reuse` after fixing start-argument lifetime
- rerunning `join-exit-*` after fixing join wakeups
- rerunning `exit-clean-*` after fixing process-exit coordination
- rerunning `pcb-syn`, `synch-many`, and `multi-oom-mt` together while refining sync-object storage design

This allowed each design change to be validated against the specific class of bug it was intended to fix.

---

## 8. What the testing process established

The final testing process established that the implementation is correct in the following important senses:

### Thread lifecycle
- creation works
- join works
- thread exit works
- main-thread explicit exit works

### Process lifecycle
- process-wide exit is coordinated
- parent wait semantics are preserved
- exit status is reported correctly

### Synchronization
- user locks work
- user semaphores work
- invalid sync-object calls fail safely

### Resource management
- stacks are reclaimed
- sync objects are reclaimed
- startup failures are cleaned up
- OOM-sensitive tests remain correct

---

## 9. Testing summary

The testing strategy combined:

- full official multithreading regression
- broader system regression
- targeted reruns for debugging
- one custom test for a subtle but important lifecycle scenario

This provided confidence that the final pthread implementation is both functionally correct and robust under stress.