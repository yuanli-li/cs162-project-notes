# CS162 Project Notes

A portfolio-style collection of design writeups, debugging case studies, validation strategy, and execution-flow diagrams for selected operating systems projects from CS162.

This repository is intentionally focused on **architecture, reasoning, debugging, and testing**, rather than publishing restricted course source code.

## Projects

- [Project 1: User Programs](./project1-userprog/README.md)

> Project 2 and Project 3 will be added later.

## Skills Highlighted

- Operating systems design
- Kernel / user boundary reasoning
- Process lifecycle management
- System call interface design
- Concurrency and synchronization
- File descriptor and file system semantics
- Floating-point context management
- Incremental debugging in low-level systems
- Test design beyond public coverage
- Correctness-driven redesign

## Repository Philosophy

This repository is structured as a set of **case studies**, not as a code dump.

For each project, I document:

1. **Problem** — what the project needed to solve  
2. **Design** — how I decomposed and organized the implementation  
3. **Key Data Structures** — the core state and responsibilities  
4. **Execution Flow** — control flow, lifecycle transitions, and diagrams  
5. **Debugging Stories** — major bugs, root causes, and fixes  
6. **Validation** — how correctness was tested and justified  

## What Is Included

- Design summaries
- Architecture and execution-flow diagrams
- Debugging narratives
- Validation strategy
- Tradeoff and rationale discussion

## What Is Not Included

This repository does **not** include full project source code or restricted course implementation files.

## Why This Repository Exists

I rebuilt these projects with an emphasis on understanding the system end-to-end: not just how to make tests pass, but how to reason about ownership, invariants, synchronization, error paths, and correctness.

My goal in publishing these notes is to present the work as a systems engineering portfolio: a record of how I approached design, diagnosis, and verification in a nontrivial kernel codebase.