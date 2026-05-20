# Ordinary Thread Exit Flow

```mermaid
flowchart TD
    A[User thread calls pthread_exit or returns from thread function] --> B[Kernel enters pthread_exit]
    B --> C[Mark current pthread_status as exited]
    C --> D[Signal join_sema]
    D --> E[Close thread local file descriptors]
    E --> F[Acquire thread_count_lock]
    F --> G[Decrement thread_count]
    G --> H{Is this the last live thread}

    H -->|No| I[Call thread_exit]
    H -->|Yes| J[Compute final status]
    J --> K[Call process_cleanup_and_exit]
    K --> L[Last thread performs final process cleanup]