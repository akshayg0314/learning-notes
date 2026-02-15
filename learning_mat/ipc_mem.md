# IPC & Memory Management — Explained Simply

## What Problem Are We Solving?

In Linux, every program runs in its own little "bubble" (called a process).
One process CANNOT just peek into another process's memory — the OS blocks it
for safety. But sometimes two programs NEED to share data.

**Example:** A PTP daemon figures out the correct time. A camera app needs
that time to stamp frames. They are TWO separate processes. How do they talk?

That's what **IPC (Inter-Process Communication)** is for.

```
  +------------------+                    +------------------+
  | PTP Daemon       |   How do they      | Camera App       |
  | (Process A)      |   share data?      | (Process B)      |
  |                  |   ──────────>      |                  |
  | "time is 14:30"  |       ???          | "I need the time"|
  +------------------+                    +------------------+
        |                                        |
        |  Each process has its OWN memory       |
        |  They CAN'T see each other's data      |
        |  The OS keeps them separated            |
        +────────────────────────────────────────+
```

**IPC is the set of tools Linux gives you to let processes talk to each other.**

---

## The IPC Methods (From Simplest to Most Powerful)

### 1. Pipes — The Simplest Way

A pipe is like a one-way water hose between two processes.
One process writes data in, the other reads it out.

```
  Process A ───────── PIPE ───────── Process B
  (writer)          [======>]         (reader)

  Data flows ONE WAY only (left to right).
  If B wants to reply, you need a SECOND pipe going the other way.
```

**How it works:**
```
  Process A does:   write(pipe_fd, "hello", 5)
  Process B does:   read(pipe_fd, buffer, 5)    → gets "hello"
```

**Two types of pipes:**

```
  1. UNNAMED PIPE  (created with pipe())
     - Only works between parent and child processes
     - Lives in memory, no file on disk
     - Disappears when processes die

     Example:  ls | grep ".txt"
               ^^   ^^
           writer   reader    (the shell creates a pipe between them)

  2. NAMED PIPE / FIFO  (created with mkfifo())
     - Has a name on the filesystem (like /tmp/my_pipe)
     - ANY two processes can use it (don't need to be related)
     - Still one-way

     $ mkfifo /tmp/my_pipe
     Process A:  echo "hello" > /tmp/my_pipe
     Process B:  cat /tmp/my_pipe    → prints "hello"
```

**When to use pipes:**
- Simple, one-way data flow
- Parent → Child communication
- Small amounts of data

**Limitations:**
- One-way only (need 2 pipes for two-way)
- No random access (can't go back and re-read)
- Data disappears after reading (no persistence)
- Slow for large data (has to copy between kernel and user space)

---

### 2. Message Queues — Like a Mailbox with Priorities

A message queue is like a mailbox where processes drop "letters" (messages).
Unlike a pipe, each message is a **separate packet** with a **priority**.

```
  Process A                MESSAGE QUEUE              Process B
  (sender)                [  mailbox  ]               (receiver)
                          ┌──────────┐
     msg "urgent" ──────> │ URGENT!  │ ──> read by B first (high priority)
     msg "normal" ──────> │ normal   │ ──> read by B second
     msg "low"    ──────> │ low      │ ──> read by B last
                          └──────────┘
```

**How it's different from a pipe:**
```
  PIPE:              |h|e|l|l|o|w|o|r|l|d|    ← just a stream of bytes
                     You have to figure out where one message ends

  MESSAGE QUEUE:     | "hello" | "world" |    ← separate packets
                     Each message is already separated for you
                     AND each can have a priority number
```

**Two flavors in Linux:**

```
  1. System V Message Queues  (older, uses msgget / msgsnd / msgrcv)
  2. POSIX Message Queues     (newer, uses mq_open / mq_send / mq_receive)

  POSIX is the modern one — use this for new code.
```

**Simple code example (POSIX):**
```
  SENDER:
    mqd_t mq = mq_open("/my_queue", O_WRONLY | O_CREAT, 0644, &attr);
    mq_send(mq, "sensor alert!", 13, priority=9);   // high priority

  RECEIVER:
    mqd_t mq = mq_open("/my_queue", O_RDONLY);
    mq_receive(mq, buffer, max_size, &priority);     // gets highest priority first
    // buffer now contains "sensor alert!"
```

**When to use message queues:**
- You need separate, discrete messages (not a byte stream)
- Some messages are more urgent than others (priority)
- Multiple senders, one receiver (or vice versa)

**Real example:**
```
  PTP daemon detects a clock jump  →  sends HIGH priority msg to all apps
  PTP daemon does routine update   →  sends LOW priority msg
  Camera app reads the queue       →  gets the clock jump alert FIRST
```

---

### 3. Shared Memory — The Fastest Way (No Copying!)

This is the big one. Shared memory lets two processes read/write the
**exact same chunk of RAM**. No copying, no kernel involvement after setup.

```
  WITHOUT Shared Memory (using pipes):
  ╔══════════════╗          ╔══════════════╗          ╔══════════════╗
  ║  Process A   ║  copy→   ║   KERNEL     ║  copy→   ║  Process B   ║
  ║  (user space)║ ======>  ║  (pipe buf)  ║ ======>  ║  (user space)║
  ╚══════════════╝          ╚══════════════╝          ╚══════════════╝
                   2 copies needed! Slow for big data.

  WITH Shared Memory:
  ╔══════════════╗                                    ╔══════════════╗
  ║  Process A   ║──────────┐          ┌──────────────║  Process B   ║
  ║  (user space)║          │          │              ║  (user space)║
  ╚══════════════╝          ▼          ▼              ╚══════════════╝
                  ┌──────────────────────────┐
                  │   SHARED MEMORY (RAM)     │
                  │   Both read/write here    │
                  │   ZERO copies needed!     │
                  └──────────────────────────┘
```

**How to create shared memory (POSIX):**
```
  Step 1:  Create a named shared memory object
           int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0644);

  Step 2:  Set its size
           ftruncate(fd, 4096);    // 4 KB

  Step 3:  Map it into your process's address space
           void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                            MAP_SHARED, fd, 0);

  Step 4:  Use it like normal memory!
           memcpy(ptr, "hello from A", 12);

  In Process B, do Steps 1 and 3 (skip Step 2 — it already exists).
  Now Process B can read: printf("%s", ptr);  → prints "hello from A"
```

**Think of it like a whiteboard in a room:**
```
  Process A and Process B are two people.
  Normally they each have their own notebook (private memory).
  Shared memory = a whiteboard on the wall that BOTH can read and write.

  A writes "temperature = 25°C" on the whiteboard.
  B walks over and reads it. No copying, no messenger, instant.
```

**BUT there's a danger — race conditions!**
```
  What if A is writing "temperature = 25" and B reads it
  RIGHT IN THE MIDDLE of the write?

  A writes "temper"... B reads "temper" → GARBAGE! 💥

  This is called a RACE CONDITION.
  Solution: use a Semaphore (explained next) to take turns.
```

---

### 4. Semaphores — Traffic Lights for Shared Resources

A semaphore is a counter that controls access to a shared resource.
Think of it as a traffic light or a bathroom key.

```
  THE BATHROOM KEY ANALOGY:

  There's 1 bathroom key hanging on the wall.
  (This is a semaphore with value = 1)

  Person A:
    1. Takes the key (semaphore goes from 1 → 0)
    2. Uses the bathroom (accesses shared memory)
    3. Returns the key (semaphore goes from 0 → 1)

  Person B:
    1. Tries to take the key
    2. Key is gone (semaphore = 0) → WAITS
    3. A returns the key (semaphore = 1)
    4. B takes the key → goes in

  Result: Only ONE person in the bathroom at a time. No collisions.
```

**How it works with shared memory:**
```
  Process A                                   Process B
     |                                           |
     |  sem_wait(&sem)  ← take the lock          |
     |  (semaphore: 1 → 0)                       |
     |                                           |
     |  ... writes to shared memory ...          |  sem_wait(&sem) ← BLOCKED!
     |  ptr->temperature = 25;                   |  (semaphore is 0, must wait)
     |                                           |
     |  sem_post(&sem)  ← release the lock       |
     |  (semaphore: 0 → 1)                       |
     |                                           |  sem_wait succeeds!
     |                                           |  (semaphore: 1 → 0)
     |                                           |
     |                                           |  reads ptr->temperature → 25 ✅
     |                                           |
     |                                           |  sem_post(&sem) ← release
```

**Two types of semaphores:**
```
  1. BINARY SEMAPHORE (value is only 0 or 1)
     = like a mutex / lock
     = "only ONE process can enter at a time"

  2. COUNTING SEMAPHORE (value can be > 1)
     = like a parking lot with N spaces
     = "up to N processes can access at a time"

     Example: you have 3 database connections
              Semaphore starts at 3
              Each process that takes a connection does sem_wait (3→2→1→0)
              4th process must wait until someone releases
```

---

## Memory Management — Explained Simply

### What Is Memory Mapping? (mmap)

Normally, to read a file, you do:
```
  1. open() the file
  2. read() some bytes into a buffer    ← data is COPIED from disk to kernel
  3. kernel copies from kernel buffer to your buffer   ← ANOTHER copy
  4. you use the buffer
  Total: 2 copies of data
```

With **mmap()**, you skip the copies:
```
  1. open() the file
  2. mmap() it → the OS says "this part of your memory IS the file"
  3. you just read/write memory directly
  Total: 0 copies — the OS handles it with page tables
```

**Picture:**
```
  NORMAL read():
  ┌────────────┐     copy #1     ┌────────────┐     copy #2     ┌────────────┐
  │  DISK/FILE │  ──────────>    │   KERNEL    │  ──────────>    │  YOUR APP  │
  │            │                 │   BUFFER    │                 │   BUFFER   │
  └────────────┘                 └────────────┘                 └────────────┘

  WITH mmap():
  ┌────────────┐                                                ┌────────────┐
  │  DISK/FILE │  ← ─ ─ ─ ─ same physical pages ─ ─ ─ ─ ─ ─>  │  YOUR APP  │
  │            │     (OS maps them directly, no copying)        │   MEMORY   │
  └────────────┘                                                └────────────┘
```

### How mmap() Actually Works (Under the Hood)

The magic is in something called the page table — a translation table that the CPU hardware uses to convert virtual addresses (what your program sees) to physical addresses (actual RAM locations).

Step-by-step, what happens when you call mmap():

```
  YOUR PROGRAM calls:
    ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

  Here's what the OS does behind the scenes:

  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 1: Reserve virtual address space                          │
  │                                                                  │
  │  OS picks a free range in your process's virtual memory.         │
  │  Let's say it picks 0x7FFF1000.                                  │
  │                                                                  │
  │  Your process's virtual memory:                                  │
  │  ┌──────────────────────────────┐                                │
  │  │ 0x00400000  [your code]     │                                │
  │  │ 0x00600000  [your data]     │                                │
  │  │ ...                         │                                │
  │  │ 0x7FFF1000  [RESERVED]  ◄── new! but NOTHING is loaded yet  │
  │  └──────────────────────────────┘                                │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 2: Update the PAGE TABLE (but DON'T load any data yet!)   │
  │                                                                  │
  │  The OS writes an entry in the page table that says:             │
  │                                                                  │
  │    "virtual address 0x7FFF1000 → backed by file X, offset 0"    │
  │                                                                  │
  │  PAGE TABLE (one per process, maintained by OS):                 │
  │  ┌──────────────────┬──────────────────────────────────┐        │
  │  │  Virtual Addr    │  Maps To                         │        │
  │  ├──────────────────┼──────────────────────────────────┤        │
  │  │  0x00400000      │  Physical RAM page 0x1A000       │        │
  │  │  0x00600000      │  Physical RAM page 0x2B000       │        │
  │  │  0x7FFF1000      │  FILE "data.txt", offset 0       │  ← NEW│
  │  │                  │  (not in RAM yet! just a note)   │        │
  │  └──────────────────┴──────────────────────────────────┘        │
  │                                                                  │
  │  KEY POINT: No data has been read from disk yet!                │
  │  The OS is lazy. It just made a PROMISE.                        │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 3: Your program reads *ptr (touches the memory)           │
  │                                                                  │
  │    char c = ptr[0];    // read the first byte                    │
  │                                                                  │
  │  CPU tries to access virtual address 0x7FFF1000.                │
  │  CPU checks page table → finds "backed by file, NOT in RAM"    │
  │                                                                  │
  │  💥 PAGE FAULT! (this is NOT a crash — it's a normal signal)   │
  │                                                                  │
  │  The CPU says to the OS:                                         │
  │  "Hey, this address has no physical RAM behind it yet!"         │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 4: OS handles the page fault                              │
  │                                                                  │
  │  OS goes: "Ah, 0x7FFF1000 is mapped to file 'data.txt'!"       │
  │                                                                  │
  │  OS does:                                                        │
  │    1. Grabs a free physical RAM page (say 0x5C000)              │
  │    2. Reads 4096 bytes from file into that RAM page             │
  │    3. Updates the page table:                                    │
  │                                                                  │
  │  PAGE TABLE (updated):                                           │
  │  ┌──────────────────┬──────────────────────────────────┐        │
  │  │  Virtual Addr    │  Maps To                         │        │
  │  ├──────────────────┼──────────────────────────────────┤        │
  │  │  0x7FFF1000      │  Physical RAM page 0x5C000  ✅   │        │
  │  └──────────────────┴──────────────────────────────────┘        │
  │                                                                  │
  │    4. Restarts your program's instruction                       │
  │       → CPU retries the read → SUCCESS this time               │
  │                                                                  │
  │  Your program has NO IDEA this happened. It just sees data.     │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 5: All future reads/writes — NO more disk I/O             │
  │                                                                  │
  │  Now that the file's data IS in RAM:                             │
  │                                                                  │
  │    ptr[0] = 'H';     // writes directly to RAM page 0x5C000    │
  │    char x = ptr[10]; // reads directly from RAM page 0x5C000   │
  │                                                                  │
  │  The CPU translates 0x7FFF1000 → 0x5C000 in HARDWARE.          │
  │  No system call. No kernel involved. Just a table lookup.       │
  │  This is why it's so fast!                                      │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 6: Writing back to disk (for file-backed mmap)            │
  │                                                                  │
  │  When you modify the RAM page, the OS marks it "dirty."         │
  │  At some point (or when you call msync() / munmap()):           │
  │                                                                  │
  │    OS writes the dirty RAM page BACK to the file on disk.       │
  │                                                                  │
  │  So: ptr[0] = 'H'  →  eventually  →  file on disk changes too │
  │                                                                  │
  │  You never called write(). You never called fwrite().           │
  │  You just wrote to memory, and the OS synced it to disk.        │
  └──────────────────────────────────────────────────────────────────┘

```

### Why mmap Matters for Hardware (PTP, NICs, etc.)

Hardware devices have **registers** — tiny memory locations that control them.
For example, a PTP network card has a register at a physical address like `0xFE000000`
that holds the current hardware timestamp.

```
  WITHOUT mmap:
  App calls ioctl() → goes into kernel → kernel reads register → copies to app
  Slow! Every read requires a system call (context switch).

  WITH mmap:
  App does mmap() once → register appears at address 0x7FFF1000 in app memory
  App just reads *ptr → gets the hardware timestamp DIRECTLY
  Fast! No system call needed after the initial mmap.

  ┌──────────────────────────────────────────────────────────┐
  │  YOUR APP's VIRTUAL MEMORY                               │
  │                                                          │
  │  0x00400000  [code segment]                              │
  │  0x00600000  [data segment]                              │
  │  ...                                                     │
  │  0x7FFF1000  [mmap'd hardware register] ←── reads this   │
  │               │                                          │
  └───────────────│──────────────────────────────────────────┘
                  │
                  │  (OS page table maps this to...)
                  ▼
  ┌──────────────────────────────────────────────────────────┐
  │  PHYSICAL HARDWARE (PTP NIC)                             │
  │  0xFE000000  [timestamp register] = 1707849230.123456789 │
  └──────────────────────────────────────────────────────────┘
```

---

### User Space vs Kernel Space — What's the Difference?

```
  ┌────────────────────────────────────────────────────────┐
  │                    YOUR COMPUTER'S RAM                  │
  │                                                        │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │              USER SPACE  (top half)               │  │
  │  │                                                  │  │
  │  │  Where YOUR programs run:                        │  │
  │  │  - Camera app                                    │  │
  │  │  - PTP client                                    │  │
  │  │  - Web browser                                   │  │
  │  │                                                  │  │
  │  │  Rules:                                          │  │
  │  │  - Can't touch hardware directly                 │  │
  │  │  - Can't see other process's memory              │  │
  │  │  - Must ask kernel for everything (system calls) │  │
  │  ├──────────────────────────────────────────────────┤  │
  │  │              KERNEL SPACE (bottom half)           │  │
  │  │                                                  │  │
  │  │  Where the OS lives:                             │  │
  │  │  - Device drivers (NIC, disk, etc.)              │  │
  │  │  - Memory manager                                │  │
  │  │  - Scheduler                                     │  │
  │  │                                                  │  │
  │  │  Rules:                                          │  │
  │  │  - Can touch ALL hardware                        │  │
  │  │  - Can see ALL memory                            │  │
  │  │  - Full power, full danger                       │  │
  │  └──────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────┘
```

**Why does this split exist?**
```
  If your camera app had a bug and could touch hardware directly,
  it could accidentally overwrite the disk controller's registers
  and corrupt your entire hard drive.

  The kernel protects everything by acting as a gatekeeper.
  Your app says: "Hey kernel, please read this file for me"
  Kernel says:   "OK, I'll do it safely and give you the data"

  This is a SYSTEM CALL (like read(), write(), open(), mmap()).
```

**The problem for real-time / PTP:**
```
  System calls are SLOW (~1-10 microseconds each)
  For PTP, you need to read a timestamp in ~10 nanoseconds
  You can't afford a system call every time!

  Solution: mmap() the hardware register ONCE at startup,
            then read it directly from user space forever.
            No more system calls. ✅
```

---

## How They All Fit Together (A Real Example)

Let's say you're building a PTP-synced camera system. Here's how all the
IPC and memory concepts work together:

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                      LINUX SYSTEM                                   │
  │                                                                     │
  │  ┌─────────────────┐           ┌──────────────────┐                │
  │  │  PTP DAEMON      │           │  CAMERA APP      │                │
  │  │  (Process A)     │           │  (Process B)     │                │
  │  │                  │           │                  │                │
  │  │  1. mmap() the   │           │  4. mmap() the   │                │
  │  │     NIC register │           │     shared mem   │                │
  │  │     to read HW   │           │                  │                │
  │  │     timestamps   │           │  5. sem_wait()   │                │
  │  │                  │           │     to lock       │                │
  │  │  2. Calculates   │           │                  │                │
  │  │     PTP offset   │           │  6. Reads the    │                │
  │  │                  │           │     current time  │                │
  │  │  3. sem_wait()   │           │     from shared  │                │
  │  │     then writes  │           │     memory       │                │
  │  │     corrected    │           │                  │                │
  │  │     time to      │           │  7. sem_post()   │                │
  │  │     SHARED MEM   │           │     to unlock    │                │
  │  │     sem_post()   │           │                  │                │
  │  │                  │           │  8. Stamps the   │                │
  │  └────────┬─────────┘           │     video frame  │                │
  │           │                     └────────┬─────────┘                │
  │           │                              │                          │
  │           ▼                              ▼                          │
  │     ┌────────────────────────────────────────┐                     │
  │     │       SHARED MEMORY (/dev/shm/ptp)     │                     │
  │     │                                        │                     │
  │     │   { current_time: 14:30:00.000000050,  │                     │
  │     │     offset:       -23 ns,              │                     │
  │     │     status:       "LOCKED" }           │                     │
  │     │                                        │                     │
  │     └────────────────────────────────────────┘                     │
  │                                                                     │
  │     ┌────────────────────────────────────────┐                     │
  │     │       SEMAPHORE (guards shared mem)     │                     │
  │     │       value: 1 (unlocked)               │                     │
  │     └────────────────────────────────────────┘                     │
  │                                                                     │
  │     ┌────────────────────────────────────────┐                     │
  │     │       mmap'd NIC REGISTER              │                     │
  │     │       (PTP hardware timestamp)          │                     │
  │     │       mapped at 0x7FFF1000 in PTP app   │                     │
  │     └────────────────────────────────────────┘                     │
  └─────────────────────────────────────────────────────────────────────┘
```

**The flow:**
```
  1. PTP daemon mmap()s the NIC's hardware timestamp register (once at startup)
  2. PTP daemon reads hardware timestamps directly (no system call — FAST)
  3. PTP daemon does the t1/t2/t3/t4 math → calculates offset
  4. PTP daemon sem_wait() → writes corrected time to shared memory → sem_post()
  5. Camera app sem_wait() → reads time from shared memory → sem_post()
  6. Camera app stamps the frame with that time
  7. This repeats thousands of times per second

  No pipes, no message queues — just raw shared memory
  protected by a semaphore. Maximum speed. ✅
```

---

## Quick Reference — When to Use What

```
  ┌──────────────────┬──────────────────────────────────────────────────┐
  │  METHOD          │  USE WHEN                                        │
  ├──────────────────┼──────────────────────────────────────────────────┤
  │  Pipe            │  Simple parent→child, small data, one-way       │
  │  Named Pipe      │  Two unrelated processes, still one-way         │
  │  Message Queue   │  Need separate messages with priorities         │
  │  Shared Memory   │  Need MAX SPEED, large data, frequent access    │
  │  Semaphore       │  Need to protect shared memory from races       │
  │  mmap (file)     │  Need fast file I/O without copying             │
  │  mmap (hardware) │  Need direct access to device registers         │
  └──────────────────┴──────────────────────────────────────────────────┘

  Speed ranking (fastest to slowest):
  ──────────────────────────────────
  1. Shared Memory + mmap    ← ZERO copies, direct RAM access
  2. Message Queues          ← one copy, but kernel manages it
  3. Pipes                   ← two copies (user→kernel→user)
  4. Sockets                 ← two copies + protocol overhead
  5. Files                   ← two copies + disk I/O
```