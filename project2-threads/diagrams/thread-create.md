# `project2-threads/diagrams/thread-create.md`

# Thread Creation Flow

```mermaid
flowchart TD
    A[User thread calls pthread_create] --> B[Kernel enters pthread_execute]
    B --> C[Allocate start argument structure]
    C --> D[Allocate pthread_status]
    D --> E[Insert pthread_status into pcb pthread_status_list]
    E --> F[Create kernel thread with start_pthread]
    F --> G[Increment thread_count]
    G --> H[Parent waits on loaded semaphore]

    H --> I[Child runs start_pthread]
    I --> J[Bind child to shared PCB]
    J --> K[Initialize child thread fields]
    K --> L[Call setup_thread]
    L --> M[Allocate or reuse user stack]
    M --> N[Lay out user stack for stub thread function and argument]
    N --> O[Child records success or failure and signals loaded]
    O --> P[Child waits on start_sema]

    H --> Q[Parent wakes and checks success]
    Q --> R[If failure rollback pthread_status and thread_count]
    Q --> S[If success signal start_sema]
    S --> T[Child enters user mode through intr_exit]
    T --> U[User stub calls target thread function]