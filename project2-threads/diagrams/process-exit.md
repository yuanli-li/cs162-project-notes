

# `project2-threads/diagrams/process-exit.md`

# Process Wide Exit Flow

```mermaid
flowchart TD
    A[A thread triggers process wide exit through exit status or exception] --> B[Kernel enters process_exit_with_status]
    B --> C[Mark current pthread_status as exited]
    C --> D[Signal join_sema]
    D --> E[Acquire thread_count_lock]
    E --> F{Is process_exiting already set}

    F -->|No| G[Current thread becomes first exiter]
    G --> H[Set process_exiting true]
    H --> I[Record final exit_status]

    F -->|Yes| J[Reuse previously recorded exit_status]

    I --> K[Decrement thread_count]
    J --> K
    K --> L{Is this the last live thread}

    L --> M[Call process_cleanup_and_exit with exit_status last and is_first]
    M --> N[If responsible print exit status]
    N --> O[If responsible wake parent wait]
    O --> P[If responsible destroy pagedir]
    P --> Q[If last thread free PCB owned resources]
    Q --> R[Call thread_exit]