

# `project2-threads/diagrams/main-ptexit.md`

# Main Thread Explicit Pthread Exit Scenario

```mermaid
flowchart TD
    A[Program starts in tests main] --> B[Print begin]
    B --> C[test_main prints main start]
    C --> D[Main thread creates worker thread]
    D --> E[thread_count becomes 2]
    E --> F[Main thread prints main exiting]
    F --> G[Main thread calls pthread_exit]
    G --> H[Main thread is marked exited and removed]
    H --> I[thread_count becomes 1]
    I --> J[Process does not exit because worker still lives]
    J --> K[Worker thread runs and prints worker start]
    K --> L[Worker thread prints worker done]
    L --> M[Worker thread exits]
    M --> N[thread_count becomes 0]
    N --> O[Worker is now the last thread]
    O --> P[Worker performs final process cleanup]
    P --> Q[Print exit 0 and destroy process resources]