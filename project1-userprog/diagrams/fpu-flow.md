
```mermaid
flowchart TD
    A[Boot: enable FPU in low-level startup] --> B[thread_init]
    B --> C["fninit + fnsave(initial_fpu_state)"]
    C --> D[init_thread copies initial_fpu_state into thread fpu_state]
    D --> E[Thread runs]
    E --> F[Scheduler saves current thread FPU state]
    F --> G[Scheduler restores next thread FPU state]
    G --> H[Next thread resumes with its own floating-point context]
```
For the full detailed control flow, see design.md