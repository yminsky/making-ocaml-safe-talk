% Making OCaml Safe for Performance Engineering

# What's the issue?

(stealing from Dolan's talk)

- Memory access is very expensive
  - control over memory representation matters
      - Packed data into fewer cache lines
      - Overall, touch less memory
      - use memory in predictable ways
- Paralllism is critical
  - CPUs aren't getting much faster
    - but we're still getting more of them!
  - Power considerations push yet more towards slower, more numerous
    cores.

* What do we want?

- Control over performance-critical aspects of program behavior
- In a safe way
- Which is convenient for users
- And which doesn't complicate the lives of programmers who don't
  care.

*
