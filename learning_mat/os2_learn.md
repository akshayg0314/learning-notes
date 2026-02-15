# Operating Systems Part 2 — Explained Simply

---

## 1. System Calls — How Your App Talks to the OS

Your app runs in **User Mode** (restricted — can't touch hardware).
The OS kernel runs in **Kernel Mode** (full power — controls everything).

A **system call** is how your app **asks the kernel** to do something it can't do itself.

```
  YOUR APP (User Mode)          KERNEL (Kernel Mode)
  ┌──────────────────┐          ┌──────────────────────────┐
  │                  │          │                          │
  │  "I want to      │  SYSTEM  │  "OK, I'll do it for     │
  │   read a file"   │──CALL──► │   you safely"            │
  │                  │          │                          │
  │  Can't touch     │          │  CAN touch disk,         │
  │  disk directly   │          │  RAM, hardware           │
  └──────────────────┘          └──────────────────────────┘
```

### What Happens During a System Call? (Step by Step)

```
  Your app calls:  read(fd, buffer, 100)

  ┌─────────────────────────────────────────────────────────────┐
  │ STEP 1: App prepares arguments                              │
  │                                                             │
  │   Your code:  read(fd, buffer, 100);                       │
  │   The C library (glibc) puts:                              │
  │     - System call NUMBER (e.g. read = 0) into register RAX │
  │     - fd into register RDI                                 │
  │     - buffer address into register RSI                     │
  │     - 100 into register RDX                                │
  └─────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ STEP 2: TRAP instruction (mode switch)                      │
  │                                                             │
  │   CPU executes a special instruction:  syscall (x86-64)    │
  │                                                             │
  │   This does THREE things:                                   │
  │     1. Switches CPU from User Mode → Kernel Mode           │
  │     2. Saves the app's return address                      │
  │     3. Jumps to the kernel's system call handler           │
  │                                                             │
  │   ┌─────────────┐         ┌─────────────┐                  │
  │   │  USER MODE  │ ──────► │ KERNEL MODE │                  │
  │   │ (restricted)│  TRAP   │ (full power)│                  │
  │   └─────────────┘         └─────────────┘                  │
  └─────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ STEP 3: Kernel handles the request                          │
  │                                                             │
  │   Kernel looks at RAX → "ah, system call #0 = read"        │
  │   Kernel reads the file from disk safely                   │
  │   Kernel copies data into the app's buffer                 │
  └─────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ STEP 4: Return to User Mode                                 │
  │                                                             │
  │   Kernel puts the result (bytes read) in RAX               │
  │   CPU switches back: Kernel Mode → User Mode               │
  │   Your app continues as if nothing happened                │
  │                                                             │
  │   ┌─────────────┐         ┌─────────────┐                  │
  │   │ KERNEL MODE │ ──────► │  USER MODE  │                  │
  │   │(done working)│ RETURN │ (continues) │                  │
  │   └─────────────┘         └─────────────┘                  │
  └─────────────────────────────────────────────────────────────┘
```

### Categories of System Calls

```
  ┌──────────────────────┬────────────────────────────────────────┐
  │  CATEGORY            │  EXAMPLES                              │
  ├──────────────────────┼────────────────────────────────────────┤
  │  Process Control     │  fork(), exec(), wait(), exit()       │
  │  File Management     │  open(), read(), write(), close()     │
  │  Device Management   │  ioctl(), read(), write()             │
  │  Information         │  getpid(), alarm(), sleep()           │
  │  Communication (IPC) │  pipe(), shmget(), mmap()             │
  └──────────────────────┴────────────────────────────────────────┘
```

---

### fork() — Creating a New Process

`fork()` creates an **exact copy** of the current process.

```
  BEFORE fork():
  ┌──────────────────────────┐
  │  Parent Process (PID 100)│
  │                          │
  │  code, data, stack, heap │
  │  open files, variables   │
  └──────────────────────────┘

  AFTER fork():
  ┌──────────────────────────┐     ┌──────────────────────────┐
  │  Parent Process (PID 100)│     │  Child Process (PID 101) │
  │                          │     │                          │
  │  SAME code, data, stack  │     │  COPY of code, data,     │
  │  fork() returned 101     │     │  stack. fork() returned 0│
  │  (child's PID)           │     │  (means "I'm the child") │
  └──────────────────────────┘     └──────────────────────────┘
```

**Copy-on-Write (COW):**
```
  fork() doesn't ACTUALLY copy all memory immediately.
  That would be way too slow for a large process!

  Instead, parent and child SHARE the same physical pages.
  The OS marks these pages as "read-only."

  AFTER fork() (both share same physical pages):
  ┌────────────┐
  │  Parent    │──────┐
  └────────────┘      ▼
                ┌──────────────┐
                │ Shared Pages │   ← both POINT to same RAM
                │ (read-only)  │
                └──────────────┘
  ┌────────────┐      ▲
  │  Child     │──────┘
  └────────────┘

  When either one WRITES to a page:
    → PAGE FAULT! OS intercepts
    → OS makes a COPY of just THAT page
    → Writer gets the copy, other keeps the original
    → Only the modified page is duplicated

  AFTER child writes to page 2:
  ┌────────────┐    ┌──────────────┐
  │  Parent    │───►│ Page 0, 1, 3 │  ← still shared (unchanged)
  └────────────┘    └──────────────┘
                    ┌──────────────┐
  ┌────────────┐───►│ Page 0, 1, 3 │  ← same pages
  │  Child     │    └──────────────┘
  │            │───►┌──────────────┐
  └────────────┘    │ Page 2 COPY  │  ← child got its own copy
                    └──────────────┘

  This saves TONS of memory! Most pages are never written to.
```

---

### exec() — Replace a Process with a New Program

`fork()` makes a copy. `exec()` **replaces** that copy with a **different program**.

```
  BEFORE exec():
  ┌──────────────────────────┐
  │  Child Process (PID 101) │
  │                          │
  │  Running: copy of parent │
  │  (same code as parent)   │
  └──────────────────────────┘

  child calls: exec("/bin/ls")

  AFTER exec():
  ┌──────────────────────────┐
  │  Child Process (PID 101) │  ← SAME PID!
  │                          │
  │  Running: /bin/ls        │  ← DIFFERENT program!
  │  Old code is GONE        │
  └──────────────────────────┘

  The PID stays the same, but the PROGRAM inside is completely replaced.
```

**fork() + exec() together = how Linux starts every program:**
```
  You type "ls" in the terminal:

  1. Shell (bash) calls fork()   → creates a child process
  2. Child calls exec("/bin/ls") → child becomes "ls"
  3. Parent (bash) calls wait()  → waits for child to finish
  4. "ls" finishes, child exits
  5. Shell gets control back, shows you the prompt again

  ┌──────────┐  fork()  ┌──────────┐  exec("ls")  ┌──────────┐
  │  bash    │────────►│  bash    │─────────────►│  ls      │
  │ (parent) │         │ (child)  │              │ (child)  │
  └──────────┘         └──────────┘              └──────────┘
       │                                              │
       │              wait()                          │ runs, prints
       │◄─────────────────────────────────────────────│ files, exits
       │
    shows prompt again
```

---

### wait() — Parent Waits for Child to Finish

```
  Parent calls: wait(&status)
  Parent BLOCKS (pauses) until the child exits.
  When child exits, wait() returns the child's PID.

  WHY is this important?

  Without wait():
    Child finishes → becomes a ZOMBIE process 🧟
    (entry stays in process table, wastes resources)

  With wait():
    Child finishes → parent reads exit status → child fully cleaned up ✅
```

### Zombie & Orphan Processes

```
  ZOMBIE PROCESS 🧟:
    Child finished, but parent HASN'T called wait() yet.
    The child is DEAD but still has an entry in the process table.
    It exists ONLY so the parent can read its exit status.

    Parent (busy)          Child (finished)
    ┌──────────┐           ┌──────────────────┐
    │ running  │           │ ZOMBIE 🧟         │
    │ hasn't   │           │ "I'm done but    │
    │ called   │           │  nobody collected │
    │ wait()   │           │  my exit status"  │
    └──────────┘           └──────────────────┘

    Fix: parent should call wait() or waitpid()


  ORPHAN PROCESS 👶:
    Parent DIED before the child finished.
    The child is still running but has no parent.
    Linux solution: init (PID 1) ADOPTS the orphan.

    Parent (died!)         Child (still running)
    ┌──────────┐           ┌──────────────────┐
    │  DEAD ☠  │           │ "My parent died! │
    └──────────┘           │  Who will wait()  │
                           │  for me?"         │
    ┌──────────┐           │                  │
    │ init     │ adopts ──►│  init is my new  │
    │ (PID 1)  │           │  parent now"     │
    └──────────┘           └──────────────────┘

    init periodically calls wait() → cleans up orphans. No zombies.
```

---

## 2. Context Switching & PCB

### Process Control Block (PCB)

Every process has a **PCB** — a data structure the OS uses to track everything about it.
Think of it as a **"resume" or ID card** for the process.

```
  ┌─────────────────────────────────────────────┐
  │          PCB (Process Control Block)         │
  │          ═══════════════════════════         │
  │                                             │
  │  PID:            1234                       │
  │  State:          RUNNING / READY / WAITING  │
  │  Program Counter: 0x00401A3F  (next instr.) │
  │  CPU Registers:  RAX=5, RBX=0, RSP=...     │
  │  Priority:       3                          │
  │  Memory Info:    page table pointer         │
  │  Open Files:     [fd0=stdin, fd3=data.txt]  │
  │  Scheduling Info: CPU time used, arrival    │
  │  Parent PID:     1100                       │
  │  Child PIDs:     [1235, 1236]               │
  │                                             │
  └─────────────────────────────────────────────┘

  The OS keeps ONE PCB per process.
  All PCBs are stored in a PROCESS TABLE (a big list of PCBs).

  When a process is created → OS creates a PCB.
  When a process dies     → OS deletes its PCB.
```

### Context Switching — What Happens When the CPU Switches Processes

```
  The CPU can only run ONE process at a time (per core).
  When it's time to switch from Process A to Process B,
  the OS must:
    1. SAVE everything about A
    2. LOAD everything about B

  This is called a CONTEXT SWITCH.
```

**Step-by-step:**

```
  Process A is RUNNING on the CPU.
  Timer interrupt fires! (or A does I/O, or higher priority process arrives)

  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 1: SAVE Process A's state into its PCB                  │
  │                                                                │
  │  CPU Registers → PCB of A                                     │
  │  Program Counter → PCB of A  (so we know where to resume)    │
  │  Stack Pointer → PCB of A                                     │
  │                                                                │
  │  Process A's PCB:                                              │
  │  ┌────────────────────────────────────┐                        │
  │  │ PID: 1234                         │                        │
  │  │ State: RUNNING → READY            │                        │
  │  │ PC: 0x00401A3F  (saved!)          │                        │
  │  │ Registers: RAX=5, RBX=7  (saved!) │                        │
  │  └────────────────────────────────────┘                        │
  └────────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 2: LOAD Process B's state from its PCB                  │
  │                                                                │
  │  PCB of B → CPU Registers                                     │
  │  PCB of B → Program Counter                                   │
  │  PCB of B → Stack Pointer                                     │
  │                                                                │
  │  Process B's PCB:                                              │
  │  ┌────────────────────────────────────┐                        │
  │  │ PID: 5678                         │                        │
  │  │ State: READY → RUNNING            │                        │
  │  │ PC: 0x00503B22  (loaded!)         │                        │
  │  │ Registers: RAX=9, RBX=1  (loaded!)│                        │
  │  └────────────────────────────────────┘                        │
  └────────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 3: Switch memory (update page table / flush TLB)        │
  │                                                                │
  │  Tell CPU: "use Process B's page table now"                   │
  │  Flush TLB (old translations are for Process A, not B)        │
  └────────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 4: CPU resumes Process B exactly where it left off       │
  │                                                                │
  │  Process B has NO IDEA it was ever paused.                    │
  │  It continues running from 0x00503B22.                        │
  └────────────────────────────────────────────────────────────────┘
```

**Timeline view:**
```
  ──────────────────────────────────────────────────────► time

  │  Process A  │  CONTEXT  │  Process B  │  CONTEXT  │  Process A  │
  │  running    │  SWITCH   │  running    │  SWITCH   │  running    │
  │             │ (overhead)│             │ (overhead)│             │

  Context switch takes ~1-10 microseconds.
  During the switch, NO useful work happens. It's pure overhead.
  That's why too many switches = slow system (too much time wasted switching).
```

**What TRIGGERS a context switch?**
```
  ┌─────────────────────────────────────────────────────┐
  │  TRIGGER                    │  EXAMPLE               │
  ├─────────────────────────────┼────────────────────────┤
  │  Timer interrupt            │  Time quantum expired  │
  │                             │  (Round Robin)         │
  │  I/O request                │  Process asks to read  │
  │                             │  a file → blocks       │
  │  Higher priority process    │  Urgent task arrives   │
  │  System call                │  Process calls fork()  │
  │  Process terminates         │  Process calls exit()  │
  └─────────────────────────────┴────────────────────────┘
```

---

## 3. TLB (Translation Lookaside Buffer)

### The Problem: Page Table Lookups Are SLOW

```
  Every time the CPU accesses memory, it must:
    1. Take the virtual address
    2. Look up the PAGE TABLE to find the physical frame
    3. Go to the physical address in RAM

  The page table itself is stored IN RAM.
  So every memory access = TWO RAM accesses!
    - One to read the page table (find the frame number)
    - One to read the actual data

  RAM access takes ~100 nanoseconds.
  Two accesses = ~200 ns per memory operation. TOO SLOW!
```

### The Solution: TLB — A Cache for Page Table Entries

```
  TLB = a tiny, SUPER FAST hardware cache inside the CPU.
  It stores RECENT page-to-frame translations.

  Think of it like this:

  PAGE TABLE = a huge phone book (stored in RAM, slow to look up)
  TLB        = a Post-it note with the 5 numbers you call most often
               (right on your desk, instant to check)

  ┌───────────────────────────────────────────────────────┐
  │                        CPU                            │
  │                                                       │
  │  ┌─────────────────────────────────────────────┐      │
  │  │              TLB (tiny, ~64 entries)        │      │
  │  │                                             │      │
  │  │  Page 0  →  Frame 5    ← recently used     │      │
  │  │  Page 3  →  Frame 12   ← recently used     │      │
  │  │  Page 7  →  Frame 2    ← recently used     │      │
  │  │                                             │      │
  │  │  Speed: ~1 nanosecond  (100x faster!)      │      │
  │  └─────────────────────────────────────────────┘      │
  │                                                       │
  └───────────────────────────────────────────────────────┘

  vs.

  ┌───────────────────────────────────────────────────────┐
  │                   RAM (slow)                          │
  │                                                       │
  │  ┌─────────────────────────────────────────────┐      │
  │  │          PAGE TABLE (huge, all entries)     │      │
  │  │                                             │      │
  │  │  Page 0  →  Frame 5                        │      │
  │  │  Page 1  →  Frame 8                        │      │
  │  │  Page 2  →  Frame 0                        │      │
  │  │  Page 3  →  Frame 12                       │      │
  │  │  ... hundreds or thousands of entries ...   │      │
  │  │                                             │      │
  │  │  Speed: ~100 nanoseconds                   │      │
  │  └─────────────────────────────────────────────┘      │
  └───────────────────────────────────────────────────────┘
```

### How a Memory Access Works WITH TLB

```
  CPU wants to access virtual address → Page 3, Offset 42

  ┌──────────────────────────────────────────────────────────────┐
  │  STEP 1: Check TLB first                                     │
  │                                                              │
  │  CPU asks TLB: "Do you have Page 3?"                        │
  │                                                              │
  │  ┌────── YES (TLB HIT) ──────┐  ┌──── NO (TLB MISS) ────┐  │
  │  │                           │  │                         │  │
  │  │  TLB says: Page 3 =      │  │  Must go to RAM and     │  │
  │  │  Frame 12                 │  │  look up the full       │  │
  │  │                           │  │  page table             │  │
  │  │  Go directly to           │  │                         │  │
  │  │  Frame 12, Offset 42     │  │  Then put the result    │  │
  │  │                           │  │  INTO the TLB for       │  │
  │  │  Total time: ~1 ns       │  │  next time              │  │
  │  │  (super fast!)            │  │                         │  │
  │  │                           │  │  Total time: ~100 ns    │  │
  │  └───────────────────────────┘  └─────────────────────────┘  │
  └──────────────────────────────────────────────────────────────┘
```

**Flow chart:**
```
  CPU needs virtual address
         │
         ▼
  ┌──────────────┐
  │ Check TLB    │
  └──────┬───────┘
         │
    ┌────┴────┐
    │         │
  HIT ✅   MISS ❌
    │         │
    ▼         ▼
  Get frame  Go to PAGE TABLE
  from TLB   in RAM (slow)
  (~1 ns)        │
    │            ▼
    │       Get frame number
    │            │
    │            ▼
    │       Put it in TLB
    │       (for next time)
    │            │
    ▼            ▼
  Access physical memory
  at Frame + Offset
```

### TLB and Context Switching

```
  IMPORTANT: TLB entries belong to a SPECIFIC process.

  When the OS switches from Process A to Process B:
    Process A's page table ≠ Process B's page table
    So Process A's TLB entries are WRONG for Process B!

  The OS must FLUSH (clear) the TLB during a context switch.

  ┌─────────────┐   context    ┌─────────────┐
  │ Process A   │   switch     │ Process B   │
  │ TLB is warm │ ──────────►  │ TLB is COLD │
  │ (full of A) │  flush TLB   │ (empty!)    │
  └─────────────┘              └─────────────┘

  After the switch, Process B starts with an EMPTY TLB.
  Every access is a TLB MISS until the TLB warms up again.
  This is one reason context switches are EXPENSIVE.

  Modern CPUs use ASID (Address Space ID) to tag TLB entries
  per-process, so you don't have to flush everything. Faster! ✅
```

### Summary

```
  ┌────────────────┬──────────────────────────────────────────┐
  │  Concept       │  One-liner                               │
  ├────────────────┼──────────────────────────────────────────┤
  │  TLB           │  Hardware cache for page table lookups   │
  │  TLB Hit       │  Translation found in TLB → fast (~1ns) │
  │  TLB Miss      │  Must go to page table in RAM → slow    │
  │  TLB Flush     │  Clear TLB during context switch        │
  │  Why it works  │  Locality — programs reuse same pages   │
  │  ASID          │  Tag entries per-process, avoid flushing│
  └────────────────┴──────────────────────────────────────────┘
```

---
## 4. Interrupt Handling

### What Is an Interrupt?

An interrupt is a **signal to the CPU** that says:
**"STOP what you're doing, something needs attention RIGHT NOW."**

```
  Think of it like this:

  You're writing code (CPU is running a program).
  Your phone rings (INTERRUPT!).
  You pause coding, pick up the phone (handle the interrupt).
  Call ends, you go back to coding (resume the program).

  The CPU does the EXACT same thing:
    Running Process A → INTERRUPT arrives →
    CPU pauses A → handles the interrupt →
    CPU resumes A (or switches to something else)
```

### Types of Interrupts

```
  ┌─────────────────────────────────────────────────────────────┐
  │                     INTERRUPTS                               │
  │                                                             │
  │          ┌──────────────┐        ┌──────────────┐           │
  │          │   HARDWARE   │        │   SOFTWARE   │           │
  │          │  INTERRUPTS  │        │  INTERRUPTS  │           │
  │          └──────┬───────┘        └──────┬───────┘           │
  │                 │                       │                    │
  │     Come from   │            Caused by  │                   │
  │     DEVICES     │            the CPU    │                   │
  │     (external)  │            ITSELF     │                   │
  │                 │            (internal) │                   │
  │                 │                       │                    │
  │    Examples:    │          Examples:    │                   │
  │    - Keyboard   │          - Division   │                   │
  │      key press  │            by zero    │                   │
  │    - Mouse move │          - Invalid    │                   │
  │    - Disk done  │            memory     │                   │
  │      reading    │            access     │                   │
  │    - Network    │          - System     │                   │
  │      packet     │            call       │                   │
  │      arrived    │            (trap)     │                   │
  │    - Timer tick │          - Breakpoint │                   │
  └─────────────────────────────────────────────────────────────┘
```

### How Does the CPU Handle an Interrupt? (Step by Step)

```
  CPU is happily running Process A...
  DING! Keyboard interrupt arrives!

  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 1: CPU finishes the CURRENT instruction                  │
  │                                                                │
  │  CPU won't stop mid-instruction. It finishes whatever it's    │
  │  doing RIGHT NOW, then checks for interrupts.                 │
  └────────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 2: CPU saves the current state                           │
  │                                                                │
  │  Pushes onto stack:                                            │
  │    - Program Counter (where to resume)                        │
  │    - CPU flags / status register                               │
  │    - Sometimes other registers                                 │
  │                                                                │
  │  This is like bookmarking your page before answering the phone.│
  └────────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 3: CPU looks up the INTERRUPT VECTOR TABLE (IVT)         │
  │                                                                │
  │  The IVT is a table stored in memory that maps:               │
  │    Interrupt Number → Address of the handler function         │
  │                                                                │
  │  Interrupt Vector Table:                                       │
  │  ┌──────────────┬──────────────────────────────┐              │
  │  │  INT Number  │  Handler Address             │              │
  │  ├──────────────┼──────────────────────────────┤              │
  │  │  0           │  0x00100000 (divide by zero) │              │
  │  │  1           │  0x00100100 (debug/breakpt)  │              │
  │  │  14          │  0x00101400 (page fault)     │              │
  │  │  33          │  0x00103300 (keyboard)       │  ← this one!│
  │  │  ...         │  ...                         │              │
  │  └──────────────┴──────────────────────────────┘              │
  │                                                                │
  │  "Keyboard = INT 33 → jump to address 0x00103300"             │
  └────────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 4: CPU switches to Kernel Mode and runs the ISR          │
  │                                                                │
  │  ISR = Interrupt Service Routine (the handler function)       │
  │                                                                │
  │  The keyboard ISR does:                                        │
  │    1. Reads the key code from the keyboard controller         │
  │    2. Puts it in a buffer (so your app can read it later)     │
  │    3. Signals the OS: "hey, a key was pressed"                │
  └────────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 5: ISR finishes, CPU restores the saved state            │
  │                                                                │
  │  Pops from stack:                                              │
  │    - Program Counter → resume where Process A left off        │
  │    - CPU flags                                                 │
  │                                                                │
  │  CPU switches back to User Mode.                              │
  │  Process A continues. It has NO IDEA an interrupt happened.   │
  └────────────────────────────────────────────────────────────────┘
```

**Full timeline:**
```
  ────────────────────────────────────────────────────────► time

  │ Process A  │ save │  ISR  │restore│ Process A  │
  │ running    │ state│(handle│ state │ resumes    │
  │            │      │ int)  │       │            │

  The interrupt handling is usually VERY fast (microseconds).
```

### Interrupt Priority & Masking

```
  Not all interrupts are equal. Some are MORE URGENT.

  ┌────────────────────┬────────────────┐
  │  INTERRUPT         │  PRIORITY      │
  ├────────────────────┼────────────────┤
  │  Machine Check     │  HIGHEST ⬆    │  (hardware failure!)
  │  Timer             │  HIGH          │  (scheduling depends on it)
  │  Disk I/O          │  MEDIUM        │
  │  Keyboard          │  LOW           │
  │  Mouse             │  LOWEST ⬇     │
  └────────────────────┴────────────────┘

  INTERRUPT MASKING:
    Sometimes, while handling one interrupt, the CPU can
    DISABLE (mask) lower-priority interrupts temporarily.
    
    "I'm handling a disk interrupt right now.
     Mouse, you can wait. Don't bother me."

  NON-MASKABLE INTERRUPT (NMI):
    Some interrupts CANNOT be disabled. Ever.
    Example: hardware failure, power loss.
    "The building is on fire — you MUST stop everything NOW."
```

---

## 5. Caches in OS

### What Is a Cache?

A cache is a **small, fast storage** that keeps copies of frequently
used data so you don't have to fetch it from a **large, slow storage**.


### The Memory Hierarchy — Speed vs Size vs Cost

```
  The closer to the CPU, the FASTER but SMALLER and more EXPENSIVE.

          FASTER ◄────────────────────────────► SLOWER
          SMALLER ◄───────────────────────────► BIGGER
          COSTLIER ◄──────────────────────────► CHEAPER

  ┌──────────────┐
  │  CPU         │
  │  REGISTERS   │  ~0.3 ns    ~1 KB      Fastest, tiniest
  ├──────────────┤
  │  L1 CACHE    │  ~1 ns      ~64 KB     Per core, split into
  │              │                         instruction + data
  ├──────────────┤
  │  L2 CACHE    │  ~4 ns      ~256 KB    Per core
  ├──────────────┤
  │  L3 CACHE    │  ~10 ns     ~8-32 MB   Shared across all cores
  ├──────────────┤
  │  RAM         │  ~100 ns    ~8-64 GB   Main memory
  ├──────────────┤
  │  SSD         │  ~100 µs    ~500 GB    Persistent storage
  ├──────────────┤
  │  HDD         │  ~10 ms     ~2 TB      Spinning disk, slowest
  └──────────────┘

  The difference is MASSIVE:
    L1 Cache:  1 ns
    RAM:       100 ns      ← 100x slower than L1!
    SSD:       100,000 ns  ← 100,000x slower than L1!
    HDD:       10,000,000 ns ← 10 MILLION times slower!
```

### CPU Cache Levels (L1, L2, L3) — How They're Organized

```
  ┌───────────────────────────────────────────────────────────────┐
  │                        CPU CHIP                               │
  │                                                               │
  │  ┌─────────────────────┐    ┌─────────────────────┐          │
  │  │      CORE 0         │    │      CORE 1         │          │
  │  │                     │    │                     │          │
  │  │  ┌───────────────┐  │    │  ┌───────────────┐  │          │
  │  │  │  L1-I   L1-D  │  │    │  │  L1-I   L1-D  │  │          │
  │  │  │  32KB   32KB  │  │    │  │  32KB   32KB  │  │          │
  │  │  └───────────────┘  │    │  └───────────────┘  │          │
  │  │  ┌───────────────┐  │    │  ┌───────────────┐  │          │
  │  │  │    L2 Cache   │  │    │  │    L2 Cache   │  │          │
  │  │  │    256 KB     │  │    │  │    256 KB     │  │          │
  │  │  └───────────────┘  │    │  └───────────────┘  │          │
  │  └─────────────────────┘    └─────────────────────┘          │
  │                                                               │
  │  ┌───────────────────────────────────────────────────┐       │
  │  │                  L3 Cache (Shared)                 │       │
  │  │                    8 - 32 MB                      │       │
  │  └───────────────────────────────────────────────────┘       │
  └───────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌───────────────────────────────────────────────────────────────┐
  │                         RAM                                    │
  │                      8 - 64 GB                                │
  └───────────────────────────────────────────────────────────────┘

  L1-I = Level 1 Instruction Cache (stores code/instructions)
  L1-D = Level 1 Data Cache        (stores variables/data)
  L2   = Bigger, slightly slower, unified (code + data)
  L3   = Biggest cache, shared across ALL cores
```

### How a Cache Lookup Works

```
  CPU needs to read variable X from memory address 0x1234.

  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ Check L1 │────►│ Check L2 │────►│ Check L3 │────►│ Go to    │
  │          │miss │          │miss │          │miss │   RAM     │
  └──────────┘     └──────────┘     └──────────┘     └──────────┘
       │hit ✅          │hit ✅          │hit ✅          │
       ▼                ▼                ▼                ▼
    Return           Return           Return          Return
    data             data             data            data
    (~1 ns)          (~4 ns)          (~10 ns)        (~100 ns)

  If found in L1 → great, fastest possible!
  If not in L1, check L2. If not there, check L3.
  If not in ANY cache → go all the way to RAM. Slowest.

  The data is then COPIED into the caches for next time.
  (brings it into L3, L2, AND L1)
```

### Cache Lines — How Data Is Stored in Cache

```
  Caches don't store individual bytes. They store CACHE LINES.
  A cache line is a block of 64 bytes (on most CPUs).

  When you access address 0x1234, the cache doesn't just fetch
  that ONE byte. It fetches the entire 64-byte block containing it.

  Memory:  [........][XXXXXXXX][........][........]
                      ^^^^^^^^
                      64 bytes around 0x1234
                      ALL loaded into cache

  WHY? Because of SPATIAL LOCALITY:
    If you access arr[0], you'll probably access arr[1], arr[2], ...
    By loading 64 bytes at once, arr[1] through arr[15] are
    ALREADY in cache when you need them. Free speed! ✅
```

### Types of Caches in an OS

The CPU cache (L1/L2/L3) is the most famous, but there are caches everywhere:

```
  ┌────────────────────┬───────────────────────────────────────────────┐
  │  CACHE TYPE        │  WHAT IT CACHES                               │
  ├────────────────────┼───────────────────────────────────────────────┤
  │  CPU Cache         │  RAM data → into L1/L2/L3 (hardware)        │
  │  (L1, L2, L3)      │  Speeds up memory access                     │
  ├────────────────────┼───────────────────────────────────────────────┤
  │  TLB               │  Page table entries (virtual→physical)       │
  │                    │  Speeds up address translation               │
  │                    │  (we covered this in os2_learn.md!)          │
  ├────────────────────┼───────────────────────────────────────────────┤
  │  Page Cache        │  Disk data → into RAM (OS managed)           │
  │  (Buffer Cache)    │  When you read a file, OS keeps it in RAM   │
  │                    │  so next read is from RAM, not disk          │
  ├────────────────────┼───────────────────────────────────────────────┤
  │  Disk Cache        │  Disk data → small RAM on the disk itself   │
  │                    │  Built into the hard drive hardware          │
  ├────────────────────┼───────────────────────────────────────────────┤
  │  Inode Cache       │  File metadata (inodes) → into RAM          │
  │                    │  Speeds up file lookups (ls, stat, etc.)     │
  ├────────────────────┼───────────────────────────────────────────────┤
  │  DNS Cache         │  Domain→IP mappings → into RAM              │
  │                    │  Speeds up website lookups                   │
  └────────────────────┴───────────────────────────────────────────────┘
```

### Cache Replacement Policies — When Cache Is Full

```
  Cache is SMALL. When it's full and new data needs to come in,
  which old data do we kick out?

  Same idea as Page Replacement (from os_learn.md)!

  ┌────────────┬──────────────────────────────────────────────┐
  │  Policy    │  How it works                                │
  ├────────────┼──────────────────────────────────────────────┤
  │  LRU       │  Kick out the LEAST RECENTLY USED entry     │
  │            │  "Haven't used you in a while? Goodbye."    │
  │            │  Most common in practice.                    │
  ├────────────┼──────────────────────────────────────────────┤
  │  FIFO      │  Kick out the OLDEST entry                  │
  │            │  "First one in, first one out."             │
  │            │  Simple but not the smartest.               │
  ├────────────┼──────────────────────────────────────────────┤
  │  LFU       │  Kick out the LEAST FREQUENTLY USED entry   │
  │            │  "You've been used the fewest times? Bye."  │
  │            │  Good for some workloads.                   │
  ├────────────┼──────────────────────────────────────────────┤
  │  Random    │  Kick out a random entry                    │
  │            │  Surprisingly not terrible! Simple to       │
  │            │  implement in hardware.                     │
  └────────────┴──────────────────────────────────────────────┘
```

### Write Policies — What Happens When You Write to Cache

```
  When the CPU writes data, it updates the cache.
  But what about the copy in RAM? Two strategies:

  ┌───────────────────────────────────────────────────────────────┐
  │  WRITE-THROUGH                                                │
  │                                                               │
  │  Write to cache AND RAM at the same time.                    │
  │                                                               │
  │  CPU writes → Cache updated ✅ → RAM updated ✅              │
  │                                                               │
  │  ✅ RAM is always up-to-date                                 │
  │  ❌ Slower (every write goes to RAM)                         │
  └───────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────┐
  │  WRITE-BACK                                                   │
  │                                                               │
  │  Write to cache ONLY. Mark the line as "dirty."              │
  │  Write to RAM later (when the line is evicted).              │
  │                                                               │
  │  CPU writes → Cache updated ✅ → RAM updated LATER           │
  │                                                               │
  │  ✅ Faster (writes stay in cache, batched to RAM later)      │
  │  ❌ RAM may be out-of-date temporarily                       │
  │  ❌ If power fails, dirty data is LOST                       │
  │                                                               │
  │  Most modern CPUs use WRITE-BACK for speed.                  │
  └───────────────────────────────────────────────────────────────┘
```

### Cache Coherency — Multi-Core Problem

```
  With multiple cores, each has its OWN L1/L2 cache.
  What if Core 0 and Core 1 both cache the SAME memory address?

  Core 0 cache: X = 5         Core 1 cache: X = 5
         (both have a copy of X)

  Core 0 writes: X = 10       Core 1 still sees: X = 5  ← WRONG!

  This is the CACHE COHERENCY problem.

  Solution: SNOOPING / MESI protocol
    When Core 0 writes to X, it broadcasts:
    "Hey everyone! I'm changing X!"
    Core 1 hears this → invalidates its copy of X.
    Next time Core 1 reads X → gets the updated value from Core 0.

  MESI Protocol (4 states for each cache line):
  ┌────────────┬──────────────────────────────────────────────┐
  │  State     │  Meaning                                     │
  ├────────────┼──────────────────────────────────────────────┤
  │  Modified  │  I changed it. My copy is the only correct  │
  │            │  one. RAM is out of date.                    │
  │  Exclusive │  Only I have it. It matches RAM.            │
  │  Shared    │  Multiple cores have it. All match RAM.     │
  │  Invalid   │  My copy is stale/wrong. Don't use it.      │
  └────────────┴──────────────────────────────────────────────┘
```

---

## Quick Reference — All Cache Types Compared

```
  ┌──────────────┬─────────────┬──────────────┬───────────────────┐
  │  Cache       │  Size       │  Speed       │  Managed By       │
  ├──────────────┼─────────────┼──────────────┼───────────────────┤
  │  Registers   │  ~1 KB      │  ~0.3 ns     │  Compiler/CPU     │
  │  L1 Cache    │  ~64 KB     │  ~1 ns       │  CPU Hardware     │
  │  L2 Cache    │  ~256 KB    │  ~4 ns       │  CPU Hardware     │
  │  L3 Cache    │  ~8-32 MB   │  ~10 ns      │  CPU Hardware     │
  │  TLB         │  ~64 entries│  ~1 ns       │  CPU Hardware     │
  │  Page Cache  │  GBs of RAM │  ~100 ns     │  OS Kernel        │
  │  Disk Cache  │  ~64-256 MB │  ~1 µs       │  Disk Hardware    │
  └──────────────┴─────────────┴──────────────┴───────────────────┘
```

---

