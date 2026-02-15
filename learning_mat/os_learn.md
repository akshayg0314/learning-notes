# Operating Systems — Explained Simply

---

## 1. What Is an Operating System?

An OS is the **middleman** between YOU (and your apps) and the HARDWARE.

```
  ┌─────────────────────────────────────────────┐
  │          YOUR APPS                          │
  │   Chrome  |  VS Code  |  Spotify            │
  ├─────────────────────────────────────────────┤
  │          OPERATING SYSTEM                   │
  │   (Windows / Linux / macOS)                 │
  │   - Manages CPU, RAM, Disk, I/O             │
  │   - Decides who runs when                   │
  │   - Keeps apps from crashing each other     │
  ├─────────────────────────────────────────────┤
  │          HARDWARE                           │
  │   CPU  |  RAM  |  Disk  |  Monitor          │
  └─────────────────────────────────────────────┘
```

**What does an OS actually DO?**
- **Process management** — decides which program gets to use the CPU
- **Memory management** — gives each program its own chunk of RAM
- **File management** — organizes your files on disk
- **I/O management** — handles keyboard, mouse, monitor, printer
- **Security** — prevents one program from messing up another

### Types of OS

```
  ┌───────────────────┬──────────────────────────────────────────┐
  │  TYPE             │  WHAT IT MEANS                           │
  ├───────────────────┼──────────────────────────────────────────┤
  │  Batch OS         │  Jobs are collected in a batch and       │
  │                   │  executed one after another.             │
  │                   │  No user interaction during execution.   │
  │                   │  Example: old mainframe systems          │
  ├───────────────────┼──────────────────────────────────────────┤
  │  Time-Sharing OS  │  CPU time is shared among many users.    │
  │                   │  Each gets a small time slice.           │
  │                   │  Feels like everyone runs at once.       │
  │                   │  Example: Unix                           │
  ├───────────────────┼──────────────────────────────────────────┤
  │  Real-Time OS     │  Must respond within strict deadlines.   │
  │  (RTOS)           │  Used where delay = disaster.           │
  │                   │  Example: airbag system, pacemaker       │
  ├───────────────────┼──────────────────────────────────────────┤
  │  Distributed OS   │  Multiple computers work together as     │
  │                   │  one system. Share load.                 │
  │                   │  Example: Google's servers               │
  ├───────────────────┼──────────────────────────────────────────┤
  │  Embedded OS      │  Built into a specific device.           │
  │                   │  Limited resources, fixed purpose.       │
  │                   │  Example: washing machine, smart watch   │
  └───────────────────┴──────────────────────────────────────────┘
```

---

## 2. Process vs Thread vs Program

```
  PROGRAM:  Code sitting on your DISK (a .exe file).
            It's NOT running. It's just a file.
            Like a recipe written in a cookbook.

  PROCESS:  A program that IS running.
            It has its own memory, own CPU time, own resources.
            Like a chef actually cooking using that recipe.

  THREAD:   A lightweight unit INSIDE a process.
            Multiple threads share the same memory.
            Like multiple chefs in the SAME kitchen,
            sharing the same ingredients.
```

```
  PROGRAM (on disk)           PROCESS (in RAM, running)
  ┌──────────────┐            ┌──────────────────────────────┐
  │  main.exe    │  ──run──►  │  Process (PID: 1234)         │
  │  (just a     │            │  ┌────────────────────────┐  │
  │   file)      │            │  │ Code    (instructions) │  │
  └──────────────┘            │  │ Data    (variables)    │  │
                              │  │ Stack   (fn calls)     │  │
                              │  │ Heap    (dynamic mem)  │  │
                              │  └────────────────────────┘  │
                              │                              │
                              │  Thread 1  Thread 2          │
                              │  (main)    (background)      │
                              │   ↓          ↓               │
                              │  Both share Code, Data, Heap │
                              │  Each has its OWN Stack      │
                              └──────────────────────────────┘
```

### Key differences:

```
  ┌─────────────┬──────────────────────┬──────────────────────┐
  │             │  PROCESS             │  THREAD              │
  ├─────────────┼──────────────────────┼──────────────────────┤
  │Memory       │ Own separate memory  │ Shares process mem   │
  │Creation     │ Slow (heavy)         │ Fast (lightweight)   │
  │Communication│ Hard (need IPC)      │ Easy (shared mem)    │
  │Crash        │ Doesn't kill others  │ Can crash all        │
  │Example      │ Chrome vs Spotify    │ Multiple tabs in     │
  │             │ (separate)           │ Chrome (shared)      │
  └─────────────┴──────────────────────┴──────────────────────┘
```

---

## 3. Multiprogramming, Multiprocessing, Multitasking, Multithreading

These words sound similar but mean very different things:

### Multiprogramming

```
  Multiple programs LOADED in memory at the same time.
  CPU switches between them when one is waiting (e.g. for I/O).
  Goal: keep CPU busy, don't waste time waiting.

  ┌─────────────────────────────────────────┐
  │  RAM                                    │
  │  ┌──────────┐ ┌──────────┐ ┌─────────┐  │
  │  │ Program A│ │ Program B│ │Program C│  │
  │  └──────────┘ └──────────┘ └─────────┘  │
  └─────────────────────────────────────────┘

  CPU:  runs A → A waits for disk → switch to B → B waits → switch to C
        (CPU is NEVER idle)
```

### Multiprocessing

```
  Multiple CPUs (or cores) working together.
  Programs ACTUALLY run at the same time (true parallelism).

  ┌─────────┐  ┌─────────┐
  │  CPU 1  │  │  CPU 2  │
  │ runs A  │  │ runs B  │     ← both running SIMULTANEOUSLY
  └─────────┘  └─────────┘
```

### Multitasking

```
  One CPU rapidly switches between tasks.
  Each task gets a tiny time slice (e.g. 10ms).
  FEELS like they run together, but only 1 runs at a time.

  CPU timeline:
  ──[A]──[B]──[C]──[A]──[B]──[C]──[A]──
    10ms  10ms 10ms 10ms ...

  This is what your laptop does — you have 1 CPU
  but Chrome, Spotify, VS Code all "run at once."
  They're actually taking turns very fast.
```

### Multithreading

```
  One PROCESS has multiple threads running.
  The threads share the same memory but do different tasks.

  Example: a browser
  ┌─────────────────────────────┐
  │  Chrome Process             │
  │                             │
  │  Thread 1: render the page  │
  │  Thread 2: download a file  │
  │  Thread 3: play a video     │
  │                             │
  │  All share Chrome's memory  │
  └─────────────────────────────┘
```

### Summary table

```
  ┌──────────────────┬──────────────────────────────────────────┐
  │  Concept         │  One-line meaning                        │
  ├──────────────────┼──────────────────────────────────────────┤
  │ Multiprogramming │  Many programs IN MEMORY, 1 CPU switches │
  │ Multiprocessing  │  Many CPUs, true parallel execution      │
  │ Multitasking     │  1 CPU, fast switching (time slicing)    │
  │ Multithreading   │  Many threads inside 1 process           │
  └──────────────────┴──────────────────────────────────────────┘
```

---

## 4. Various States of a Process

A process goes through these states during its life:

```
                         ┌─────────────┐
         admitted        │             │  dispatch
   ┌────────────────────►│    READY    │──────────────┐
   │                     │  (waiting   │              │
   │                     │   for CPU)  │              ▼
  ┌┴──────────┐          └─────────────┘       ┌──────────────┐
  │           │                ▲               │              │
  │   NEW     │                │ I/O done      │   RUNNING    │
  │ (created) │                │ or event      │  (using CPU) │
  └───────────┘          ┌─────────────┐       │              │
                         │             │       └──────┬───────┘
                         │   WAITING   │◄─────────────┤
                         │  (blocked   │  I/O needed  │
                         │   for I/O)  │              │
                         └─────────────┘              │ exit
                                                      ▼
                                                ┌───────────┐
                                                │TERMINATED │
                                                │  (done)   │
                                                └───────────┘
```

```
  ┌─────────────┬──────────────────────────────────────────────┐
  │  State      │  What's happening                            │
  ├─────────────┼──────────────────────────────────────────────┤
  │  NEW        │  Process just created. Not yet in RAM.       │
  │  READY      │  In RAM, waiting in queue for CPU time.     │
  │  RUNNING    │  Actually executing on the CPU right now.    │
  │  WAITING    │  Paused. Waiting for I/O or some event.     │
  │  TERMINATED │  Done executing. Being cleaned up.           │
  └─────────────┴──────────────────────────────────────────────┘

  Example: you open Notepad
    1. NEW       — OS creates the process
    2. READY     — loaded into RAM, waiting for CPU
    3. RUNNING   — CPU executes Notepad's code
    4. WAITING   — you haven't typed anything, Notepad waits for keyboard
    5. RUNNING   — you typed 'H', CPU processes the keystroke
    6. TERMINATED — you close Notepad
```

---

## 5. CPU Scheduling Algorithms

**Problem:** many processes are READY, but only 1 CPU. **Who goes first?**

### FCFS — First Come, First Served

```
  Whoever arrives first, runs first. Like a queue at a shop.

  Arrival:  P1(0ms)  P2(1ms)  P3(2ms)
  Burst:    P1=6ms   P2=3ms   P3=2ms

  CPU: |  P1 (6ms)  |  P2 (3ms)  |  P3 (2ms)  |
       0            6             9            11

  ✅ Simple
  ❌ Convoy effect: short jobs stuck behind long ones
     P3 needs only 2ms but waits 9ms!
```

### SJF — Shortest Job First

```
  Shortest burst time goes first.

  Ready queue:  P1(6ms)  P2(3ms)  P3(2ms)

  CPU: |  P3 (2ms)  |  P2 (3ms)  |  P1 (6ms)  |
       0            2             5            11

  ✅ Minimum average waiting time
  ❌ How do you KNOW how long a job will take? (you often don't)
  ❌ Long jobs may STARVE (never get to run)
```

### Round Robin (RR)

```
  Each process gets a fixed TIME QUANTUM (e.g. 3ms).
  After 3ms, CPU switches to the next process.

  Time quantum = 3ms
  P1=6ms  P2=3ms  P3=2ms

  CPU: | P1(3ms) | P2(3ms) | P3(2ms) | P1(3ms) |
       0         3         6         8         11

  ✅ Fair — everyone gets a turn
  ✅ Good for time-sharing systems
  ❌ Too small quantum = too much switching overhead
  ❌ Too large quantum = becomes FCFS
```

### Priority Scheduling

```
  Each process has a priority number. Highest priority runs first.
  (lower number = higher priority)

  P1(priority=3)  P2(priority=1)  P3(priority=2)

  CPU: | P2 (prio 1) | P3 (prio 2) | P1 (prio 3) |

  ✅ Important tasks run first
  ❌ STARVATION: low priority tasks may NEVER run
  Fix: AGING — gradually increase priority of waiting processes
```

### Summary

```
  ┌───────────────┬────────────────────┬────────────────────┐
  │  Algorithm    │  Idea              │  Problem           │
  ├───────────────┼────────────────────┼────────────────────┤
  │  FCFS         │  First in line     │  Convoy effect     │
  │  SJF          │  Shortest first    │  Starvation        │
  │  Round Robin  │  Fixed time turns  │  Context switching │
  │  Priority     │  Urgent first      │  Starvation        │
  └───────────────┴────────────────────┴────────────────────┘
```

---

## 6. Critical Section Problem

When two processes/threads share data, the code section that accesses
that shared data is called the **critical section**.

```
  PROBLEM:
  Thread A and Thread B both modify a shared variable "count".

  count = 5

  Thread A: count = count + 1    (should make it 6)
  Thread B: count = count - 1    (should make it 5)

  Expected final value: 5

  BUT what if they INTERLEAVE?

  Thread A: reads count → gets 5
  Thread B: reads count → gets 5       ← BOTH read the OLD value!
  Thread A: writes 5+1=6
  Thread B: writes 5-1=4               ← WRONG! should be 5!

  This is a RACE CONDITION.
```

**A solution must satisfy 3 conditions:**

```
  1. MUTUAL EXCLUSION
     Only ONE thread in the critical section at a time.

  2. PROGRESS
     If nobody is in the critical section, someone should be able to enter.
     Don't block everyone forever.

  3. BOUNDED WAITING
     A thread shouldn't wait FOREVER. It must eventually get its turn.
```

---

## 7. Process Synchronisation

Synchronisation = making processes cooperate so they don't mess up shared data.

**Producer-Consumer Problem (classic example):**

```
  Producer makes items.       Consumer takes items.
  Both share a BUFFER.

  ┌──────────┐    ┌──────────────────┐    ┌──────────┐
  │ PRODUCER │───►│ BUFFER (size=5)  │───►│ CONSUMER │
  │ (makes)  │    │ [X][X][_][_][_]  │    │ (takes)  │
  └──────────┘    └──────────────────┘    └──────────┘

  Problems:
  - Producer shouldn't add when buffer is FULL
  - Consumer shouldn't take when buffer is EMPTY
  - Both shouldn't access buffer at the SAME TIME
```

---

## 8. Process Synchronisation Mechanisms

### Mutex (Lock)

```
  Like a bathroom lock. Only 1 person at a time.

  mutex lock;

  Thread A:                    Thread B:
  lock(mutex)                  lock(mutex) ← BLOCKED (A has it)
  // critical section          // waits...
  unlock(mutex)                // now B can enter
                               // critical section
                               unlock(mutex)
```

### Semaphore

```
  Like a parking lot with N spots.

  Binary Semaphore (N=1) = same as mutex
  Counting Semaphore (N>1) = up to N threads can enter

  sem = 3  (3 spots available)

  Thread A: wait(sem) → sem=2 → enters
  Thread B: wait(sem) → sem=1 → enters
  Thread C: wait(sem) → sem=0 → enters
  Thread D: wait(sem) → sem=0 → BLOCKED! waits for someone to leave
  Thread A: signal(sem) → sem=1 → Thread D can now enter
```
---

## 9. Deadlock

Four processes are all waiting for each other. **Nobody can proceed.**

```
  REAL-WORLD ANALOGY:

  4 cars at a 4-way intersection, all arrived at the same time.
  Each car waits for the car on its right to go first.
  Nobody moves. Everyone waits forever. = DEADLOCK

       │Car B│
  ─────┘     └─────
  Car A         Car C
  ─────┐     ┌─────
       │Car D│

  In OS terms:
  Process A holds Lock 1, wants Lock 2
  Process B holds Lock 2, wants Lock 1
  Both wait forever. Neither can finish.

  Process A ──wants──► Lock 2 ──held by──► Process B
      ▲                                        │
      │                                        │
  held by                                   wants
      │                                        │
  Lock 1 ◄──────────────────────────────────────┘
                    CIRCULAR WAIT!
```

**4 conditions for deadlock (ALL must be true):**

```
  1. MUTUAL EXCLUSION    — resource can only be held by 1 process
  2. HOLD AND WAIT       — holding one resource, waiting for another
  3. NO PREEMPTION       — can't forcibly take a resource from a process
  4. CIRCULAR WAIT       — A waits for B, B waits for C, C waits for A
```

---

## 10. Deadlock Handling Techniques

```
  ┌─────────────────────────────────────────────────────────────┐
  │  STRATEGY 1: PREVENTION  (break one of the 4 conditions)   │
  │                                                             │
  │  Break Circular Wait:                                      │
  │    Force all processes to request resources in a FIXED      │
  │    ORDER. e.g., always lock A before B.                    │
  │                                                             │
  │  Break Hold and Wait:                                      │
  │    Make process request ALL resources at once before        │
  │    starting. If can't get all, get NONE. Wait and retry.   │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │  STRATEGY 2: AVOIDANCE  (be careful before allocating)     │
  │                                                             │
  │  Banker's Algorithm:                                       │
  │    Before giving a resource, check:                        │
  │    "If I give this, can everyone STILL finish?"            │
  │    If YES → safe, give it.                                 │
  │    If NO → unsafe, make the process wait.                  │
  │                                                             │
  │  Like a banker who only gives a loan if they can still     │
  │  serve all other customers.                                │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │  STRATEGY 3: DETECTION + RECOVERY                          │
  │                                                             │
  │  Let deadlocks happen, but detect them and fix.            │
  │                                                             │
  │  Detection: build a resource wait graph, check for cycles. │
  │  Recovery:                                                  │
  │    - Kill one of the deadlocked processes                  │
  │    - OR take a resource from someone forcibly               │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │  STRATEGY 4: IGNORANCE  (Ostrich Algorithm)                │
  │                                                             │
  │  Just pretend deadlocks don't happen.                      │
  │  If one does happen, reboot.                               │
  │  Used by most real systems (Windows, Linux).               │
  │  Deadlocks are RARE, prevention is EXPENSIVE.              │
  └─────────────────────────────────────────────────────────────┘
```

---

## 11. Memory Management

OS must decide **where in RAM** to put each process.

```
  RAM is limited. Multiple processes need to fit.
  OS must:
  1. Allocate memory to new processes
  2. Free memory when processes end
  3. Keep processes from accessing each other's memory
```

### Contiguous vs Non-Contiguous Allocation

```
  CONTIGUOUS: each process gets ONE solid block of RAM.
  ┌──────────────────────────────────────────┐
  │  OS  │ Process A │  Process B  │  FREE   │
  └──────────────────────────────────────────┘
  Simple, but causes FRAGMENTATION.


  NON-CONTIGUOUS: process can be split across multiple blocks.
  ┌──────────────────────────────────────────┐
  │  OS  │ A-part1 │ B │ A-part2 │ A-part3   │
  └──────────────────────────────────────────┘
  More complex, but uses memory better.
  This is what PAGING does (covered later).
```

### Fragmentation

```
  EXTERNAL FRAGMENTATION:
  Free memory exists but is scattered in small pieces.
  No single piece is big enough for a new process.

  │ P1 │ FREE │ P2 │ FREE │ P3 │ FREE │
         10KB        15KB        8KB
  Total free = 33KB, but if a process needs 30KB → WON'T FIT!

  INTERNAL FRAGMENTATION:
  Process gets a block bigger than it needs. The leftover is wasted.

  ┌────────────────────┐
  │ Process (needs 3KB)│ wasted 1KB │
  │     Block = 4KB                 │
  └────────────────────┘
```

---

## 12. First-Fit, Best-Fit, Worst-Fit Algorithms

When a new process needs memory, how do we pick which free block to use?

```
  Available free blocks:  [100KB]  [500KB]  [200KB]  [300KB]  [600KB]
  New process needs:      212KB
```

### First-Fit

```
  Scan from the beginning. Use the FIRST block that fits.

  [100KB]  [500KB]  [200KB]  [300KB]  [600KB]
             ↑
          500 >= 212  ✅  USE THIS ONE!

  After: [100KB]  [288KB free]  [200KB]  [300KB]  [600KB]

  ✅ Fast — stops searching as soon as it finds one
  ❌ May leave awkward leftover fragments at the beginning
```

### Best-Fit

```
  Scan ALL blocks. Pick the SMALLEST block that fits.

  [100KB]  [500KB]  [200KB]  [300KB]  [600KB]
                               ↑
                            300 >= 212 and is the SMALLEST fit

  After: [100KB]  [500KB]  [200KB]  [88KB free]  [600KB]

  ✅ Leaves least wasted space inside the chosen block
  ❌ Slow — must scan everything
  ❌ Creates TINY leftover fragments (88KB) that may be useless
```

### Worst-Fit

```
  Scan ALL blocks. Pick the LARGEST block.

  [100KB]  [500KB]  [200KB]  [300KB]  [600KB]
                                         ↑
                                      600 = largest

  After: [100KB]  [500KB]  [200KB]  [300KB]  [388KB free]

  ✅ Leftover fragment is BIG enough to still be useful
  ❌ Slow — must scan everything
  ❌ Wastes the biggest blocks quickly
```

```
  ┌────────────┬───────────────────────┬──────────────────────┐
  │ Algorithm  │ How it picks          │ Trade-off            │
  ├────────────┼───────────────────────┼──────────────────────┤
  │ First-Fit  │ First block that fits │ Fast, decent results │
  │ Best-Fit   │ Smallest fitting block│ Tiny useless scraps  │
  │ Worst-Fit  │ Largest block         │ Big leftover, wastes │
  └────────────┴───────────────────────┴──────────────────────┘
  In practice, First-Fit is usually the best compromise.
```

---

## 13. Paging

Paging solves external fragmentation by splitting EVERYTHING into
**equal-sized pieces**.

```
  PHYSICAL MEMORY (RAM) is divided into FRAMES  (e.g. 4KB each)
  PROCESS is divided into PAGES                  (same size: 4KB each)

  Each page can go into ANY frame. They don't need to be next to each other.

  Process A (12KB = 3 pages):     RAM (frames):
  ┌────────┐                      ┌─────────┐
  │ Page 0 │──────────────────────│ Frame 0  │ Page 0 of A
  │ Page 1 │──────┐               ├─────────┤
  │ Page 2 │──┐   │               │ Frame 1  │ (some other process)
  └────────┘  │   │               ├─────────┤
              │   └──────────────►│ Frame 2  │ Page 1 of A
              │                   ├─────────┤
              │                   │ Frame 3  │ (free)
              │                   ├─────────┤
              └──────────────────►│ Frame 4  │ Page 2 of A
                                  └─────────┘

  Pages DON'T need consecutive frames!
  No external fragmentation. ✅
```

### Page Table

```
  Each process has a PAGE TABLE that maps pages → frames.

  Process A's Page Table:
  ┌─────────┬──────────┐
  │  Page   │  Frame   │
  ├─────────┼──────────┤
  │  0      │  0       │
  │  1      │  2       │
  │  2      │  4       │
  └─────────┴──────────┘

  When CPU needs Page 1 of Process A:
    Look up page table → Frame 2 → go to Frame 2 in RAM.
```

---

## 14. Virtual Memory

**Problem:** what if a process is BIGGER than RAM?

```
  RAM = 4GB
  Process needs 8GB

  Without virtual memory → can't run!
  With virtual memory    → runs fine ✅
```

**Idea:** don't load the ENTIRE process into RAM.
Only load the parts (pages) you're **actually using right now**.
Keep the rest on disk.

```
  ┌────────────────────────────────────────────────────────┐
  │  VIRTUAL MEMORY                                        │
  │                                                        │
  │  Process thinks it has a HUGE continuous memory.       │
  │  In reality:                                           │
  │    - Some pages are in RAM (fast)                      │
  │    - Other pages are on DISK (slow, loaded when needed)│
  │                                                        │
  │  Process (8GB virtual):       Reality:                 │
  │  ┌────────┐                   RAM (4GB):               │
  │  │ Page 0 │ ─in RAM──────► │ Frame 2 │               │
  │  │ Page 1 │ ─in RAM──────► │ Frame 0 │               │
  │  │ Page 2 │ ─on disk       │         │               │
  │  │ Page 3 │ ─on disk       │         │               │
  │  │ Page 4 │ ─in RAM──────► │ Frame 5 │               │
  │  │ ...    │                                            │
  │  └────────┘                                            │
  │                                                        │
  │  When process accesses Page 2 (on disk):               │
  │    → PAGE FAULT! OS loads Page 2 from disk → RAM      │
  │    → If RAM is full, kick out some other page first   │
  └────────────────────────────────────────────────────────┘
```

---

## 15. Page Replacement Algorithms

When RAM is **full** and a new page needs to come in,
which **existing page** do we kick out?

### FIFO — First In, First Out

```
  Kick out the page that has been in RAM the LONGEST.
  Like a queue — first one in, first one out.

  RAM has 3 frames. Page requests: 7, 0, 1, 2, 0, 3, 0, 4

  Step 1: 7 → [7, _, _]          MISS (loaded 7)
  Step 2: 0 → [7, 0, _]          MISS (loaded 0)
  Step 3: 1 → [7, 0, 1]          MISS (loaded 1)
  Step 4: 2 → [2, 0, 1]          MISS (kicked 7, oldest)
  Step 5: 0 → [2, 0, 1]          HIT  (already in RAM!)
  Step 6: 3 → [2, 3, 1]          MISS (kicked 0, oldest)
  Step 7: 0 → [2, 3, 0]          MISS (kicked 1, oldest)
  Step 8: 4 → [4, 3, 0]          MISS (kicked 2, oldest)

  ✅ Simple
  ❌ May kick out a frequently used page
```

### LRU — Least Recently Used

```
  Kick out the page that was used the LONGEST AGO.
  Logic: if you haven't used it recently, you probably won't soon.

  RAM has 3 frames. Page requests: 7, 0, 1, 2, 0, 3, 0, 4

  Step 1: 7 → [7, _, _]          MISS
  Step 2: 0 → [7, 0, _]          MISS
  Step 3: 1 → [7, 0, 1]          MISS
  Step 4: 2 → [2, 0, 1]          MISS (kicked 7 — used longest ago)
  Step 5: 0 → [2, 0, 1]          HIT  (0 is now "recently used")
  Step 6: 3 → [2, 0, 3]          MISS (kicked 1 — used longest ago)
  Step 7: 0 → [2, 0, 3]          HIT
  Step 8: 4 → [4, 0, 3]          MISS (kicked 2 — used longest ago)

  ✅ Usually performs well
  ❌ Harder to implement (must track when each page was last used)
```

### Optimal (OPT)

```
  Kick out the page that won't be used for the LONGEST TIME in the future.

  ✅ Best possible — fewest page faults
  ❌ IMPOSSIBLE in practice — you can't predict the future!
  Used only as a benchmark to compare other algorithms against.
```

```
  ┌────────────┬──────────────────────────┬──────────────────┐
  │ Algorithm  │ Who gets kicked out      │ Practical?       │
  ├────────────┼──────────────────────────┼──────────────────┤
  │ FIFO       │ Oldest loaded page       │ Yes, simple      │
  │ LRU        │ Least recently used page │ Yes, widely used │
  │ Optimal    │ Page used farthest away  │ No, theoretical  │
  └────────────┴──────────────────────────┴──────────────────┘
```

---

## 16. Thrashing

When the system spends **MORE time swapping pages** in and out
than actually running programs.

```
  NORMAL:
  CPU usage: ████████████████░░░░  (80% doing real work)
  Page swaps: few

  THRASHING:
  CPU usage: ██░░░░░░░░░░░░░░░░░░  (10% doing real work!)
  Page swaps: ████████████████████  (90% swapping pages in/out)

  The CPU is "busy" but not doing any USEFUL work.
  It's just moving pages between RAM and disk, over and over.
```

**Why does it happen?**

```
  Too many processes loaded, each gets too FEW frames.
  Every process keeps page-faulting, kicking out each other's pages.

  Process A needs Page X → kicks out Process B's page
  Process B needs its page → kicks out Process A's page
  Process A needs that page again → kicks out Process B's page
  ... forever!

  ┌────────────────────────────────────────────┐
  │  CPU Utilization vs Number of Processes    │
  │                                            │
  │  CPU%                                      │
  │  100│        ╱──╲                          │
  │     │      ╱     ╲                         │
  │     │    ╱        ╲ ← THRASHING starts     │
  │     │  ╱           ╲   here                │
  │     │╱              ╲                      │
  │   0 └──────────────────── # processes      │
  │     few            too many                │
  └────────────────────────────────────────────┘

  Fix: reduce number of running processes
       or give important processes more frames.
```

---

## 17. Segmentation

Paging splits a process into **equal-sized** pages (4KB each).
Segmentation splits a process into **logical sections** of VARIABLE size.

```
  A process is naturally divided into logical parts:

  ┌─────────────────────────────────────────┐
  │  Process Memory                         │
  │                                         │
  │  Segment 0: CODE    (instructions) 8KB  │
  │  Segment 1: DATA    (global vars)  2KB  │
  │  Segment 2: STACK   (fn calls)     4KB  │
  │  Segment 3: HEAP    (dynamic)      6KB  │
  └─────────────────────────────────────────┘

  Each segment has a different SIZE (not fixed like pages).
```

### Segment Table

```
  ┌──────────┬────────────┬────────┐
  │ Segment  │ Base Addr  │ Limit  │
  ├──────────┼────────────┼────────┤
  │  0 (code)│  0x1000    │  8KB   │
  │  1 (data)│  0x5000    │  2KB   │
  │  2 (stack)│ 0x8000   │  4KB   │
  │  3 (heap)│  0xA000   │  6KB   │
  └──────────┴────────────┴────────┘

  Base  = where the segment starts in physical memory
  Limit = how big the segment is (prevents out-of-bounds access)
```

### Paging vs Segmentation

```
  ┌────────────────┬──────────────────┬──────────────────────┐
  │                │  PAGING          │  SEGMENTATION        │
  ├────────────────┼──────────────────┼──────────────────────┤
  │  Division size │  Fixed (e.g. 4KB)│  Variable            │
  │  Division by   │  Physical size   │  Logical meaning     │
  │  Ext. fragment │  No ✅           │  Yes ❌              │
  │  Int. fragment │  Yes (last page) │  No ✅               │
  │  Used in       │  Most modern OS  │  Often combined with │
  │                │                  │  paging              │
  └────────────────┴──────────────────┴──────────────────────┘
```
