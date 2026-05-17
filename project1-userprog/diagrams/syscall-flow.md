
```mermaid
flowchart TD
    A[User program executes syscall] --> B[Interrupt 0x30]
    B --> C[syscall_handler]
    C --> D[Read syscall number from user stack]
    D --> E[Validate syscall stack pointer]
    E --> F[Validate each argument before use]
    F --> G[Dispatch to syscall implementation]
    G --> H[Perform syscall-specific work]
    H --> I[Store return value in f->eax]
    I --> J[Return to user mode]