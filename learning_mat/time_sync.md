# Timing & Synchronization — PTP 

## What Problem Does PTP Solve?

Imagine you have 100 cameras in a factory, all connected over a network.
You want ALL of them to capture a frame at **exactly** the same instant.
But each camera has its own internal clock, and they all drift differently.

**PTP (Precision Time Protocol)** makes sure every device on the network
agrees on what time it is — accurate down to **nanoseconds**.

Think of it like this:
- NTP (what your laptop uses) = accurate to ~10 milliseconds (good enough for email)
- PTP = accurate to ~10 nanoseconds (needed for 5G, trading, robotics)

---

## Two Real System Examples (Simple)

### Example 1: Factory Camera System

**Problem:** You have 4 cameras watching a production line. You need all
frames to have the EXACT same timestamp so you can compare them later.

```
  THE SETUP:
  ══════════

  🛰️ GPS Antenna (on factory roof)
       |
       | (GPS cable)
       v
  +---------------------+
  |  PTP GRANDMASTER    |    This box receives GPS time.
  |  (a small server    |    It becomes the "Boss Clock"
  |   with GPS card)    |    of the whole factory network.
  +---------------------+
       |
       | Ethernet cable
       v
  +---------------------+
  |   NETWORK SWITCH    |    A PTP-aware switch (Boundary Clock).
  |   (BC mode)         |    It syncs to the Grandmaster above,
  |                     |    then gives time to all cameras below.
  +---------------------+
    |       |       |       |
    v       v       v       v
  +---+   +---+   +---+   +---+
  |CAM|   |CAM|   |CAM|   |CAM|
  | 1 |   | 2 |   | 3 |   | 4 |     Each camera is a PTP Slave.
  +---+   +---+   +---+   +---+     They sync their clock to the switch.
```

**What happens step by step:**

```
  1. GPS antenna gets the real time from satellites
  2. Grandmaster receives it → its clock is now perfect
  3. Grandmaster sends "Sync" message every 1 second to the switch
  4. Switch (BC) locks onto the Grandmaster's time
  5. Switch sends "Sync" to all 4 cameras
  6. Each camera calculates its offset and corrects its clock

  RESULT: All 4 cameras agree on the time within ~100 nanoseconds
          When Camera 1 says "frame captured at 14:30:00.000000100"
          and Camera 3 says "frame captured at 14:30:00.000000120"
          you KNOW they were only 20 ns apart — practically the same instant.
```


## How Does PTP Work? (The Big Picture)

There is one "boss" clock called the **Grandmaster** (usually connected to GPS).
Every other device on the network adjusts its clock to match the Grandmaster.

```
    🛰️ GPS Satellite
        |
        v
  +--------------+
  |  GRANDMASTER  |  <-- "The Boss Clock" (most accurate)
  |  (Master)     |
  +--------------+
        |
   -----+-----  (network)
   |         |
   v         v
+------+  +------+
| Slave|  | Slave|   <-- These adjust their clocks to match the Master
|  #1  |  |  #2  |
+------+  +------+
```

---

## How Does a Slave Sync to the Master? (4 Timestamps)

The whole trick is about figuring out TWO things:
1. **How much delay** is there in the network between Master and Slave?
2. **How much offset** does the Slave's clock have compared to the Master?

PTP uses **4 timestamps** (t1, t2, t3, t4) to figure this out:

```
     MASTER                              SLAVE
        |                                   |
        |-------- Sync Message ------------>|
        |  t1 = time Master sent it         |  t2 = time Slave received it
        |                                   |
        |---- Follow_Up (carries t1) ------>|
        |  "Hey, I sent that Sync at t1"    |  Now Slave knows both t1 and t2
        |                                   |
        |<------- Delay_Req ----------------|
        |                                   |  t3 = time Slave sent this
        |  t4 = time Master received it     |
        |                                   |
        |---- Delay_Resp (carries t4) ----->|
        |  "I got your request at t4"       |  Now Slave knows t3 and t4
        |                                   |
```

**Now the Slave has all 4 timestamps: t1, t2, t3, t4**

### The Math (it's simple)

**Step 1 — Calculate the network delay (one-way):**

```
Delay = [ (t2 - t1) + (t4 - t3) ] / 2
```

Why divide by 2? Because (t2 - t1) includes the delay PLUS the offset,
and (t4 - t3) includes the delay MINUS the offset. Adding them cancels
out the offset, and dividing by 2 gives you the one-way delay.

**Step 2 — Calculate how far off the Slave's clock is:**

```
Offset = (t2 - t1) - Delay
```

**Step 3 — Fix it:**
The Slave subtracts this Offset from its clock. Done! It's now in sync.

### A Concrete Example

```
Say the Master's clock is 100 nanoseconds ahead of the Slave:

  t1 = 1000 ns   (Master sends Sync)
  t2 = 1200 ns   (Slave receives — but Slave's clock is 100ns behind,
                   so it reads 1200 instead of 1100)
  t3 = 2000 ns   (Slave sends Delay_Req)
  t4 = 2100 ns   (Master receives — Master's clock is 100ns ahead)

  Delay  = [(1200 - 1000) + (2100 - 2000)] / 2
         = [200 + 100] / 2
         = 150 ns     ... but actual wire delay is 100 ns?

  Wait — let's redo with symmetric delay of 100 ns:

  t1 = 1000       (Master clock)
  t2 = 1000 + 100(delay) + 100(offset) = 1200   (Slave clock reads high by 100)
  t3 = 2000       (Slave clock)
  t4 = 2000 - 100(offset) + 100(delay) = 2000   (Master clock)

  Delay  = [(1200 - 1000) + (2000 - 2000)] / 2 = [200 + 0] / 2 = 100 ns ✅
  Offset = (1200 - 1000) - 100 = 100 ns ✅

  Slave corrects by -100 ns. Now they match!
```

---

## Who Becomes the Grandmaster? (BMCA)

When the network starts up, every device thinks it might be the boss.
PTP uses the **Best Master Clock Algorithm (BMCA)** to hold an election.

Each device broadcasts its "credentials" in **Announce messages**.
The winner is decided by checking these fields, in order:

```
  Check #1:  priority1       (admin-set, lower = better)
      |
      v  (if tied)
  Check #2:  clockClass      (how good is the clock source? GPS > crystal)
      |
      v  (if tied)
  Check #3:  clockAccuracy   (how accurate is it? in nanoseconds)
      |
      v  (if tied)
  Check #4:  PTP variance    (how stable is it over time?)
      |
      v  (if tied)
  Check #5:  priority2       (tiebreaker, admin-set)
      |
      v  (if STILL tied)
  Check #6:  clockIdentity   (MAC address — lowest wins)
```

> **Simple way to remember:** The device with the best GPS connection and the
> lowest priority number wins. If you plug a GPS clock in, it almost always wins.

---

## Clock Types (What Sits Between Master and Slave)

In a real network, there are switches between the Master and the Slaves.
PTP defines different roles for these switches:

### 1. Ordinary Clock (OC)
A simple device with ONE network port. It's either a Master or a Slave.
Example: A GPS-fed grandmaster, a camera, a sensor.

### 2. Boundary Clock (BC)
A switch that **understands PTP**. It acts as a Slave on one side (syncs to
the Master) and as a Master on the other side (gives time to Slaves).

```
  Grandmaster -----> [ Boundary Clock ] -----> Slave 1
                      (Slave side) (Master side)
                           |
                           +-----> Slave 2
```

**Advantage:** Each Slave gets a clean, fresh PTP signal.
**Think of it as:** A time relay — it receives time, locks on, then re-distributes.

### 3. Transparent Clock (TC)
A switch that **does NOT sync its own clock**. It just passes PTP messages
through, but measures how long the message was stuck inside the switch
(called "residence time") and writes that into the message.

```
  Grandmaster -----> [ Transparent Clock ] -----> Slave
                      "This message was inside
                       me for 350 ns, FYI"
```

**Advantage:** Simpler, no clock to maintain.
**Disadvantage:** Errors can pile up if you have many TCs in a chain.

### Summary: BC vs TC

```
  +-------+       +-------+       +-------+
  |Master | ----> |  BC   | ----> | Slave |
  +-------+       +-------+       +-------+
                  Syncs its own     Gets fresh
                  clock first,      PTP from BC
                  then re-sends

  +-------+       +-------+       +-------+
  |Master | ----> |  TC   | ----> | Slave |
  +-------+       +-------+       +-------+
                  Just passes       Gets original
                  messages thru     PTP message +
                  + adds delay      correction
                  correction
```

---

## One-Step vs Two-Step Mode

There are two ways the Master can tell the Slave what time the Sync was sent:

### Two-Step (Most Common)
1. Master sends a `Sync` message (but doesn't know exact send time yet).
2. After sending, hardware tells Master the precise time → t1.
3. Master sends a `Follow_Up` message saying "t1 was _____".

```
  Master                  Slave
    |--- Sync ------------->|   (Slave notes arrival time = t2)
    |--- Follow_Up (t1) --->|   (Now Slave knows t1 too)
```

### One-Step (Faster, but harder to build)
1. The hardware stamps t1 **directly into the Sync packet** as it leaves.
2. No Follow_Up needed.

```
  Master                  Slave
    |--- Sync (has t1) ---->|   (Slave gets t1 and t2 in one shot)
```
---

## Delay Measurement: E2E vs P2P

### End-to-End (E2E)
- The Slave measures the delay to the **Master directly** (across the whole path).
- Uses `Delay_Req` / `Delay_Resp` messages.
- Simple, works well in small networks.

```
  Master ----[switch]----[switch]---- Slave
  <------------ measures total delay ----------->
```

### Peer-to-Peer (P2P)
- Each **link** measures its own delay independently.
- Uses `Pdelay_Req` / `Pdelay_Resp` messages.
- Much better for large switched networks.

```
  Master ----[switch]----[switch]---- Slave
         <-->        <-->        <-->
       link 1      link 2      link 3
    (each measured separately)
```

The P2P measurement between two neighbors works like this:

```
  Node A                    Node B
    |--- Pdelay_Req ---------->|   A notes send time = t1
    |                          |   B notes arrival time = t2
    |<-- Pdelay_Resp ----------|   B notes send time = t3
    |                          |   A notes arrival time = t4
    |<-- Follow_Up (t2, t3) ---|

  Link Delay = [ (t4 - t1) - (t3 - t2) ] / 2
               (total round trip) - (time B was processing)
               ─────────────────────────────────────────────
                                   2
```

---

## Where Is the Timestamp Captured? (Why Hardware Matters)

The accuracy of PTP depends on WHERE in the system the timestamp is grabbed:

```
  +----------------------------------------------------+
  |  Your App Code         (software timestamp)        |  ~1 ms accuracy 😐
  |  ──────────────────────────────────────────────    |
  |  Operating System / Kernel                         |
  |  ──────────────────────────────────────────────    |
  |  MAC Layer              (hardware timestamp)       |  ~1 μs accuracy 🙂
  |  ──────────────────────────────────────────────    |
  |  PHY Layer              (hardware timestamp)       |  ~1 ns accuracy 🎯
  |  ──────────────────────────────────────────────    |
  |  Wire / Fiber                                      |
  +----------------------------------------------------+
```

> The closer to the wire you timestamp, the less "noise" from the OS
> scheduler, interrupts, and software delays. That's why PTP uses
> **hardware timestamping** at the PHY or MAC layer.

---

## PTP vs NTP — Quick Comparison

```
  NTP                               PTP
  ───                               ───
  Software timestamps               Hardware timestamps
  ~1-50 ms accuracy                 ~1-100 ns accuracy
  Works over any network            Needs PTP-aware switches (ideally)
  Free, runs on anything            Needs special NIC/PHY hardware
  Good for: servers, laptops        Good for: 5G, finance, robotics

  Rule of thumb:
    Need < 1 ms accuracy?  ──> Use PTP
    ~10 ms is fine?        ──> Use NTP (simpler + cheaper)
```

---
