

# `project2-threads/diagrams/user-sync.md`

# User Lock and Semaphore Flow

```mermaid
flowchart TD
    A[User program calls lock_init or sema_init] --> B[Kernel validates user pointer nonfatally]
    B --> C[Acquire pcb user_sync_lock]
    C --> D[Ensure user sync pointer tables exist]
    D --> E[Find a free ID slot]
    E --> F[Allocate one kernel lock or semaphore object for that slot]
    F --> G[Initialize the kernel object]
    G --> H[Store pointer in pcb user_locks or user_semas]
    H --> I[Write ID back into user object]
    I --> J[Release user_sync_lock]

    K[User program later calls acquire release down or up] --> L[Kernel validates user pointer]
    L --> M[Read ID from user object]
    M --> N[Check that ID is valid and mapped]
    N --> O[Look up kernel object in PCB table]
    O --> P[Invoke real kernel lock or semaphore operation]