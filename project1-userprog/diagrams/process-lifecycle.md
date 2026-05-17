# Process Lifecycle

```mermaid
flowchart TD
    A[Parent calls exec or fork] --> B[Allocate child_status]
    B --> C[Insert child_status into parent children list]
    C --> D[Create child thread]
    D --> E[Parent waits on load_sema]
    E --> F[Child startup completes]
    F --> G[Child publishes load_done / load_success]
    G --> H[Parent returns pid or -1]

    H --> I[Child runs in user mode]
    I --> J[Child exits]
    J --> K[Child writes exit_status into self_status]
    K --> L[Child sets exited = true]
    L --> M[Child signals wait_sema]

    N[Parent calls wait(pid)] --> O[Find matching child_status]
    O --> P[If child not exited, block on wait_sema]
    P --> Q[Read exit_status]
    Q --> R[Mark waited = true]
    R --> S[Release parent reference]