# TSN (Time-Sensitive Networking) — Complete Streaming Flow

> A step-by-step explainer of how TSN sets up and starts an audio/video stream
> from a **Talker** to a **Listener** across a bridged Ethernet network.

---

## 1. Key Components (Who's Who)

Before diving into the flow, let's understand **what each component does**:

| Component           | Full Name                        | Role                                                                 |
|---------------------|----------------------------------|----------------------------------------------------------------------|
| **Talker NUC**      | Talker (Intel NUC)               | The machine that **sends** the audio/video stream                    |
| **Listener NUC**    | Listener (Intel NUC)             | The machine that **receives** the audio/video stream                 |
| **ENDSTATION**      | End Station                      | Software on Talker/Listener that represents them in the TSN network  |
| **AVTP Client**     | Audio Video Transport Protocol   | The actual app that generates (talker) or consumes (listener) stream |
| **Bridge 1**        | TSN Bridge (Switch)              | First network switch in the path                                     |
| **Bridge 2**        | TSN Bridge (Switch)              | Second network switch in the path                                    |
| **Device Manager**  | Device Manager                   | Agent on each bridge — reports capabilities, applies configurations  |
| **CUC**             | Centralized User Configuration   | Manages end stations — knows who wants to talk/listen                |
| **CNC**             | Centralized Network Controller   | Brain of the network — computes paths, configures QoS (CBS)         |
| **CBS**             | Credit-Based Shaper              | QoS mechanism applied at egress ports to guarantee bandwidth         |
| **LLDP**            | Link Layer Discovery Protocol    | Protocol used by Device Managers to advertise port/bridge info       |

### Think of it like a Phone Call Analogy:

```
  AVTP Client  =  The person who wants to make a phone call
  ENDSTATION   =  The phone itself
  CUC          =  The phone company's customer service (registers users)
  CNC          =  The phone company's network engineer (sets up the line)
  Device Mgr   =  The technician at each cell tower
  CBS          =  A reserved lane on the highway for your call's data
  Bridges      =  Cell towers that relay your call
```

---

## 2. Physical Topology

```
┌───────────────┐       ┌─────────────────────┐       ┌─────────────────────┐       ┌───────────────┐
│  TALKER NUC   │       │      BRIDGE 1       │       │      BRIDGE 2       │       │ LISTENER NUC  │
│               │       │                     │       │                     │       │               │
│ ┌───────────┐ │       │  ┌───────┐          │       │  ┌───────┐          │       │ ┌───────────┐ │
│ │ ENDSTATION│ │       │  │  CUC  │          │       │  │  CNC  │          │       │ │ ENDSTATION│ │
│ └───────────┘ │       │  └───────┘          │       │  └───────┘          │       │ └───────────┘ │
│               │       │  ┌───────────────┐  │       │  ┌───────────────┐  │       │               │
│ ┌───────────┐ │       │  │ Device Manager│  │       │  │ Device Manager│  │       │ ┌───────────┐ │
│ │avtp client│ │       │  └───────────────┘  │       │  └───────────────┘  │       │ │avtp client│ │
│ │  talker   │ │       │                     │       │                     │       │ │ listener  │ │
│ └───────────┘ │       │                     │       │                     │       │ └───────────┘ │
│               │       │                     │       │                     │       │               │
│ [acrn-br0]────┼──────►[enp2s0]──[br0]──[enp1s0]───►[enp0s31f6]──[br0]──[enp1s0]──┼────[acrn-br0] │
│               │       │                     │       │                     │       │               │
└───────────────┘       └─────────────────────┘       └─────────────────────┘       └───────────────┘
                                                                                            │
                                                                                           LAN
```

### Network Interface Mapping:

| Node         | Interface    | Connected To              |
|--------------|-------------|---------------------------|
| Talker NUC   | `acrn-br0`  | Bridge 1 → `enp2s0`      |
| Bridge 1     | `enp2s0`    | Talker NUC → `acrn-br0`  |
| Bridge 1     | `br0`       | Internal bridge           |
| Bridge 1     | `enp1s0`    | Bridge 2 → `enp0s31f6`   |
| Bridge 2     | `enp0s31f6` | Bridge 1 → `enp1s0`      |
| Bridge 2     | `br0`       | Internal bridge           |
| Bridge 2     | `enp1s0`    | Listener NUC → `acrn-br0`|
| Listener NUC | `acrn-br0`  | Bridge 2 → `enp1s0`      |

---

## 3. Step-by-Step Flow (Simple Explainer)

### Step 1 — Physical Topology Setup
> 🔧 *"Build the road before the cars can drive"*

- All 4 nodes are physically connected via Ethernet cables
- Bridge interfaces (`br0`, `acrn-br0`) are configured
- No software services are running yet — just the raw network

---

### Step 2 — Device Managers Start 🟢
> 🚀 *"The technicians arrive at each cell tower first"*

- Device Manager on **Bridge 1** → Starts ✅
- Device Manager on **Bridge 2** → Starts ✅
- They boot up first because everyone else depends on them to know what the bridges can do

```
  Bridge 1                    Bridge 2
  ┌────────────────┐          ┌────────────────┐
  │  [Dev Mgr] 🟢  │          │  [Dev Mgr] 🟢  │
  └────────────────┘          └────────────────┘
       STARTED                     STARTED
```

---

### Step 3 — CNC Announcement 📢
> 📡 *"The network engineer announces: I'm here, report to me!"*

- **CNC** (on Bridge 2) starts and announces its presence
- It contacts **Device Managers** and **CUC**
- Now everyone knows where the "brain" of the network is

```
                    CNC (Bridge 2) 🟢
                   /        |        \
                  ▼         ▼         ▼
            Dev Mgr     Dev Mgr     CUC
           (Bridge 2)  (Bridge 1)  (Bridge 1)
```

---

### Step 4 — Device Managers Send LLDP & Capabilities 📤
> 📋 *"Technicians send their reports: here's what our towers can do"*

- Device Managers use **LLDP** protocol to send to CNC:
  - Port information (which ports exist, their speeds)
  - Bridge capabilities (supported TSN features)
  - Topology information (who is connected to whom)
- **After this step, CNC has a complete map of the entire network!**

```
  Dev Mgr (Bridge 1) ──── LLDP + Capabilities ────► CNC (Bridge 2)
  Dev Mgr (Bridge 2) ──── LLDP + Capabilities ────► CNC (Bridge 2)
```

---

### Step 5 — CUC Starts & Announces 📢
> 📞 *"Customer service is now open for registration!"*

- **CUC** (on Bridge 1) starts and announces its presence
- Both **Endstations** (Talker & Listener) are notified
- Endstations now know WHERE to register their streaming needs

```
                    CUC (Bridge 1) 🟢
                   /                 \
                  ▼                   ▼
          ENDSTATION              ENDSTATION
         (Talker NUC)           (Listener NUC)
```

---

### Step 6 — Endstations Discover Each Other 🔗
> 🤝 *"The caller and receiver find out about each other"*

- Talker ENDSTATION discovers that a Listener exists
- Listener ENDSTATION discovers that a Talker exists
- This discovery happens **through CUC and CNC** (not directly)

```
  ENDSTATION (Talker) ──────► CUC ◄────── ENDSTATION (Listener)
                               │
                          "I know both                
                           of you now!"
```

---

### Step 7 — Endstation Registration to CUC 📝
> ✍️ *"Both parties formally register: I want to send / I want to receive"*

- **Talker ENDSTATION → CUC**: "I am a Talker, I want to send a stream"
- **Listener ENDSTATION → CUC**: "I am a Listener, I want to receive a stream"
- CUC records both registrations

```
  Talker ENDSTATION ───── "Register as TALKER" ─────► CUC
  Listener ENDSTATION ─── "Register as LISTENER" ───► CUC
```

---

### Step 8 — AVTP Client Declares Requirements 🎯
> 📐 *"Here's exactly what kind of stream I need"*

- **AVTP Client Talker** (on Talker NUC) specifies:
  - Stream format (audio/video)
  - Bandwidth needed
  - Frame size & interval
  - Maximum latency tolerance
- **AVTP Client Listener** (on Listener NUC) specifies:
  - What stream formats it can accept
  - Buffer capabilities
- These requirements flow up through ENDSTATION → CUC

```
  ┌──────────────┐                              ┌──────────────┐
  │ avtp client   │                              │ avtp client   │
  │ talker        │                              │ listener      │
  │               │                              │               │
  │ "I need:      │                              │ "I can accept:│
  │  - 10 Mbps    │                              │  - Audio AAF  │
  │  - 125μs      │                              │  - 10 Mbps"   │
  │    interval   │                              │               │
  │  - < 2ms      │                              │               │
  │    latency"   │                              │               │
  └──────┬───────┘                              └──────┬───────┘
         │                                              │
         ▼                                              ▼
    ENDSTATION ──────────► CUC ◄──────────── ENDSTATION
```

---

### Step 9 — Stream Registration & Compute Stream Path 📡
> 🧮 *"CUC tells CNC: these two want to stream, figure out the path!"*

- **CUC → CNC**: Sends stream registration request containing:
  - Talker info + requirements
  - Listener info + requirements
- CNC now has EVERYTHING it needs:
  - ✅ Network topology (from Step 4)
  - ✅ Bridge capabilities (from Step 4)
  - ✅ Stream requirements (from this step)

```
  CUC (Bridge 1) ───── Stream Registration Request ─────► CNC (Bridge 2)
                        │
                        ├── Talker: MAC, requirements
                        ├── Listener: MAC, requirements
                        └── Stream: bandwidth, latency
```

---

### Step 10 — CNC Computes Stream & Configures CBS ⚙️
> 🛤️ *"The network engineer calculates the best route and reserves lanes"*

- CNC computes the **optimal stream path** through the network
- CNC pushes **CBS (Credit-Based Shaper)** configuration to egress ports via Device Managers
- **Egress ports configured** (shown in orange/yellow):
  - Bridge 1: `enp1s0` → CBS configured
  - Bridge 2: `enp1s0` → CBS configured
- CBS guarantees that TSN stream traffic gets **reserved bandwidth** at these ports

```
                        CNC (Bridge 2)
                       /              \
                      ▼                ▼
              Dev Mgr (Bridge 1)   Dev Mgr (Bridge 2)
                      │                │
                      ▼                ▼
              enp1s0 [CBS] 🟡    enp1s0 [CBS] 🟡
             (egress port)      (egress port)
```

---

### Step 11 — CUC Signals: Start Streaming! 🎬
> ▶️ *"Everything is ready — GO!"*

- CUC tells **Talker ENDSTATION**: "Start sending your stream"
- CUC tells **Listener ENDSTATION**: "Start receiving the stream"
- **TSN stream is now LIVE!**

```
  CUC ──── "START SENDING" ────► Talker ENDSTATION
  CUC ──── "START LISTENING" ──► Listener ENDSTATION
```

---

### TSN Stream — Data Flowing! 🔴➡️
> 🌊 *"The stream is live — data flows with guaranteed QoS"*

The actual media data now flows through the CBS-shaped path:

```
  ┌─────────┐    ┌──────────────────────────┐    ┌──────────────────────────┐    ┌──────────┐
  │ TALKER   │    │        BRIDGE 1           │    │        BRIDGE 2           │    │ LISTENER │
  │          │    │                          │    │                          │    │          │
  │ avtp ────┼───►│ enp2s0 ──► br0 ──► enp1s0│───►│enp0s31f6 ──► br0 ──► enp1s0│───►│ avtp     │
  │ client   │    │                    [CBS]🟡│    │                    [CBS]🟡│    │ client   │
  │ talker   │    │                          │    │                          │    │ listener │
  └─────────┘    └──────────────────────────┘    └──────────────────────────┘    └──────────┘
       │                                                                              │
       └──────────── TSN STREAM (QoS Guaranteed, Low Latency) ───────────────────────┘
```

---

## 4. System Interaction Flow (Who Talks to Whom)

This is the **complete sequence** of all communications in order:

```
  TALKER        BRIDGE 1          BRIDGE 2        LISTENER
  (NUC)     (CUC + Dev Mgr)   (CNC + Dev Mgr)     (NUC)
    │              │                 │                │
    │              │                 │                │
    │         ┌────┴────┐      ┌────┴────┐           │
    │         │Dev Mgr  │      │Dev Mgr  │           │
    │         │ START 🟢│      │ START 🟢│           │        ← Step 2
    │         └────┬────┘      └────┬────┘           │
    │              │                 │                │
    │              │           ┌─────┴─────┐         │
    │              │           │CNC START 🟢│         │        ← Step 3
    │              │           └─────┬─────┘         │
    │              │                 │                │
    │              │    announce     │                │
    │              │◄────────────────┤                │        ← Step 3 (CNC → CUC)
    │              │                 │                │
    │              │   LLDP + caps   │                │
    │              ├────────────────►│                │        ← Step 4 (Dev Mgr → CNC)
    │              │                 │                │
    │         ┌────┴────┐            │                │
    │         │CUC START│            │                │        ← Step 5
    │         │   🟢    │            │                │
    │         └────┬────┘            │                │
    │    announce  │          announce│                │
    │◄─────────────┤─────────────────┼───────────────►│        ← Step 5 (CUC → Endstations)
    │              │                 │                │
    │   discover   │                 │     discover   │
    │◄─────────────┼─────────────────┼───────────────►│        ← Step 6 (mutual discovery)
    │              │                 │                │
    │  register    │                 │     register   │
    ├─────────────►│◄────────────────┼────────────────┤        ← Step 7 (Endstations → CUC)
    │              │                 │                │
    │  AVTP reqs   │                 │    AVTP reqs   │
    ├─────────────►│◄────────────────┼────────────────┤        ← Step 8 (requirements)
    │              │                 │                │
    │              │  stream reg req │                │
    │              ├────────────────►│                │        ← Step 9 (CUC → CNC)
    │              │                 │                │
    │              │                 │ compute path   │
    │              │                 │ + configure    │
    │              │   CBS config    │   CBS config   │
    │              │◄────────────────┤───────────┐    │        ← Step 10 (CNC → Dev Mgrs)
    │              │                 │           │    │
    │              │            enp1s0🟡    enp1s0🟡 │        ← CBS set at egress ports
    │              │                 │                │
    │  "START!"    │                 │    "START!"    │
    │◄─────────────┤─────────────────┼───────────────►│        ← Step 11 (CUC → Endstations)
    │              │                 │                │
    │══════════════╪═════════════════╪════════════════│
    │          TSN STREAM (CBS-shaped, QoS)           │        ← STREAMING!
    │══════════════╪═════════════════╪════════════════│
    │              │                 │                │
```

---

## 5. Overall Flow Summary

### Phase 1: Infrastructure Boot-Up (Steps 1–3)

```
 ┌────────────────────────────────────────────────────────────────┐
 │  1. Physical topology is connected                            │
 │  2. Device Managers start on both bridges                     │
 │  3. CNC announces itself → everyone knows the "brain"        │
 └────────────────────────────────────────────────────────────────┘
```

**Purpose**: Get the network infrastructure ready. Device Managers and CNC must be online before anything else can happen.

---

### Phase 2: Network Discovery (Steps 4–6)

```
 ┌────────────────────────────────────────────────────────────────┐
 │  4. Device Managers send LLDP + capabilities → CNC            │
 │  5. CUC starts and announces to Endstations                   │
 │  6. Endstations discover each other (via CUC)                 │
 └────────────────────────────────────────────────────────────────┘
```

**Purpose**: CNC learns the full network topology. CUC comes online. Talker and Listener find each other.

---

### Phase 3: Stream Negotiation (Steps 7–9)

```
 ┌────────────────────────────────────────────────────────────────┐
 │  7. Endstations register with CUC (Talker + Listener)         │
 │  8. AVTP clients declare bandwidth/latency requirements       │
 │  9. CUC sends stream registration request to CNC              │
 └────────────────────────────────────────────────────────────────┘
```

**Purpose**: CUC collects all stream requirements and passes them to CNC. CNC now has everything it needs to compute the stream path.

---

### Phase 4: Stream Configuration & Start (Steps 10–11)

```
 ┌────────────────────────────────────────────────────────────────┐
 │  10. CNC computes path + configures CBS at egress ports       │
 │  11. CUC signals Talker & Listener → STREAM STARTS!          │
 └────────────────────────────────────────────────────────────────┘
```

**Purpose**: CNC reserves bandwidth (CBS) on the computed path. CUC gives the green signal. Stream is live!

---

## 6. TSN Stream Data Path

The actual stream data follows this exact path through the network:

```
  TALKER NUC                                                    LISTENER NUC
      │                                                              ▲
      │ [avtp client talker generates stream]                        │ [avtp client listener receives]
      ▼                                                              │
  ┌─────────┐                                                  ┌─────────┐
  │ acrn-br0 │                                                  │ acrn-br0 │
  └────┬─────┘                                                  └────▲─────┘
       │                                                              │
       ▼                                                              │
  ┌─────────┐    ┌──────┐    ┌─────────┐   ┌───────────┐   ┌──────┐   ┌─────────┐
  │ enp2s0  │───►│  br0 │───►│ enp1s0  │──►│ enp0s31f6 │──►│  br0 │──►│ enp1s0  │
  │(Bridge1)│    │      │    │ [CBS]🟡 │   │ (Bridge2) │   │      │   │ [CBS]🟡  │
  └─────────┘    └──────┘    └─────────┘   └───────────┘   └──────┘   └─────────┘
                  Bridge 1                                  Bridge 2
```

### Key Points:
- **CBS (Credit-Based Shaper)** is configured at **egress ports** (`enp1s0` on both bridges)
- CBS ensures the TSN stream gets **guaranteed bandwidth** — other traffic cannot starve it
- The stream has **bounded latency** — it will arrive within the promised time window
- This is what makes TSN different from regular Ethernet: **deterministic, real-time delivery**

---

## Quick Reference: Acronym Cheat Sheet

| Acronym | Meaning                              |
|---------|--------------------------------------|
| TSN     | Time-Sensitive Networking            |
| CNC     | Centralized Network Controller       |
| CUC     | Centralized User Configuration       |
| CBS     | Credit-Based Shaper (IEEE 802.1Qav)  |
| LLDP    | Link Layer Discovery Protocol        |
| AVTP    | Audio Video Transport Protocol       |
| NUC     | Next Unit of Computing (Intel mini PC)|
| ACRN    | A hypervisor by Intel (hence acrn-br0)|

---

## 7. Deep Dive: How CNC Internally Computes the Best Path 🧠

> This is what happens **inside Step 10** — the "magic" behind CNC's brain.

### 7.1 What CNC Already Knows (Inputs)

By the time CNC needs to compute a path, it has collected **three types of data**:

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    CNC's BRAIN (Inputs)                     │
  │                                                             │
  │  1. TOPOLOGY (from Step 4 - LLDP)                          │
  │     ├── Which bridges exist                                │
  │     ├── Which ports on each bridge                         │
  │     ├── How bridges are connected (link map)               │
  │     └── Link speeds (e.g., 1 Gbps, 100 Mbps)              │
  │                                                             │
  │  2. CAPABILITIES (from Step 4 - Device Managers)           │
  │     ├── Does bridge support CBS? (IEEE 802.1Qav)           │
  │     ├── Does bridge support TAS? (IEEE 802.1Qbv)           │
  │     ├── Number of traffic classes supported                │
  │     ├── Queue depths at each port                          │
  │     └── Max frame size supported                           │
  │                                                             │
  │  3. STREAM REQUIREMENTS (from Step 9 - CUC)               │
  │     ├── Talker MAC address + location                      │
  │     ├── Listener MAC address + location                    │
  │     ├── Required bandwidth (e.g., 10 Mbps)                │
  │     ├── Max latency allowed (e.g., < 2 ms)                │
  │     ├── Frame size (e.g., 256 bytes)                       │
  │     └── Frame interval (e.g., every 125 μs)               │
  └─────────────────────────────────────────────────────────────┘
```

---

### 7.2 Step-by-Step: How CNC Computes the Path

#### Step A — Build the Topology Graph 🗺️

CNC converts the physical network into a **graph** (nodes and edges):

```
  Real Network:                          CNC's Internal Graph:
  
  Talker ── Bridge1 ── Bridge2 ── Listener       T ──── B1 ──── B2 ──── L
                                                    1G      1G      1G
                                                  (link speeds as weights)
```

- **Nodes** = Talker, Bridge 1, Bridge 2, Listener
- **Edges** = Physical links between them
- **Weights** = Link speed, current utilization, hop count

> **Real-world analogy**: Think of it as Google Maps converting roads into a graph to find the shortest route.

---

#### Step B — Collect Constraints 📏

CNC creates a list of **hard rules** the path MUST satisfy:

```
  ┌─────────────────────────────────────────────┐
  │            CONSTRAINTS CHECKLIST             │
  │                                              │
  │  ☐ Bandwidth:  >= 10 Mbps available on      │
  │                every link in the path        │
  │                                              │
  │  ☐ Latency:    Total path delay < 2 ms       │
  │                                              │
  │  ☐ CBS Support: Every bridge on the path     │
  │                 MUST support CBS shaping      │
  │                                              │
  │  ☐ Queue Space: Egress queues must have      │
  │                 room for this stream          │
  │                                              │
  │  ☐ No Conflicts: Path should not create      │
  │                  conflicts with existing      │
  │                  reserved streams             │
  └─────────────────────────────────────────────┘
```

---

#### Step C — Find Shortest Path with Constraints 🔍

CNC uses a **Constrained Shortest Path** algorithm. In simple terms:

```
                        START: Talker NUC
                              │
                              ▼
                     ┌─────────────────┐
                     │ Find ALL possible│
                     │ paths from Talker│
                     │   to Listener    │
                     └────────┬────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
      Path 1:            Path 2:           Path 3:
   T → B1 → B2 → L   T → B1 → B3 → L   T → B4 → B2 → L
   (2 hops, 0.5ms)    (2 hops, 1.2ms)    (2 hops, 3ms)
            │                 │                 │
            ▼                 ▼                 ▼
     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
     │Check bandwidth│  │Check bandwidth│  │Check bandwidth│
     │   >= 10 Mbps? │  │   >= 10 Mbps? │  │   >= 10 Mbps? │
     │    ✅ YES     │  │    ✅ YES     │  │    ❌ NO      │
     └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
            │                 │                 │
            ▼                 ▼                 ▼
     ┌──────────────┐  ┌──────────────┐       REJECTED
     │Check latency  │  │Check latency  │
     │   < 2 ms?     │  │   < 2 ms?     │
     │    ✅ YES     │  │    ✅ YES     │
     └──────┬───────┘  └──────┬───────┘
            │                 │
            ▼                 ▼
     ╔══════════════╗  ┌──────────────┐
     ║   WINNER!    ║  │  Also valid,  │
     ║ Lowest cost  ║  │  but higher   │
     ║  0.5ms path  ║  │  cost (1.2ms) │
     ╚══════════════╝  └──────────────┘
```

**In our 2-bridge setup**, the path is straightforward (only one possible path), but in larger networks with many bridges, CNC has to evaluate multiple routes.

**Algorithm used**: Typically a variant of **Dijkstra's algorithm** or **Bellman-Ford**, modified to check constraints at each hop:

---

#### Step D — Reserve Resources Along the Path 📝

Once the best path is found, CNC **reserves** resources on every link:

```
  BEFORE reservation:
  ─────────────────────────────────────────────────
  
  Bridge 1 (enp1s0):
    Total bandwidth:    1000 Mbps
    Used by streams:     200 Mbps  (existing streams)
    Available:           800 Mbps
  
  Bridge 2 (enp1s0):
    Total bandwidth:    1000 Mbps
    Used by streams:     150 Mbps  (existing streams)
    Available:           850 Mbps
  
  
  AFTER reservation (10 Mbps reserved for new stream):
  ─────────────────────────────────────────────────
  
  Bridge 1 (enp1s0):
    Total bandwidth:    1000 Mbps
    Used by streams:     210 Mbps  ← (+10 Mbps)
    Available:           790 Mbps
  
  Bridge 2 (enp1s0):
    Total bandwidth:    1000 Mbps
    Used by streams:     160 Mbps  ← (+10 Mbps)
    Available:           840 Mbps
```

CNC then sends the **CBS configuration** to each Device Manager via **NETCONF/YANG** protocol.

---

### 7.3 What CNC Sends to Device Managers

The actual configuration pushed to each bridge looks like this:

```
  CNC → Device Manager (Bridge 1):
  ┌─────────────────────────────────────────────┐
  │  Configure PORT: enp1s0                      │
  │                                              │
  │  Traffic Class:  SR Class A (priority 3)     │
  │  Idle Slope:     10 Mbps                     │
  │  Send Slope:     -990 Mbps                   │
  │  Hi Credit:      +1542 bytes                 │
  │  Lo Credit:      -1542 bytes                 │
  │  Queue:          Queue 3 (high priority)     │
  └─────────────────────────────────────────────┘
  
  CNC → Device Manager (Bridge 2):
  ┌─────────────────────────────────────────────┐
  │  Configure PORT: enp1s0                      │
  │                                              │
  │  Traffic Class:  SR Class A (priority 3)     │
  │  Idle Slope:     10 Mbps                     │
  │  Send Slope:     -990 Mbps                   │
  │  Hi Credit:      +1542 bytes                 │
  │  Lo Credit:      -1542 bytes                 │
  │  Queue:          Queue 3 (high priority)     │
  └─────────────────────────────────────────────┘
```

---

## 8. Deep Dive: How CBS (Credit-Based Shaper) Works Internally ⚙️

> CBS is the QoS mechanism defined in **IEEE 802.1Qav** that guarantees bandwidth for TSN streams.

### 8.1 The Problem CBS Solves

Without CBS, all traffic competes equally for the egress port:

```
  WITHOUT CBS (Regular Ethernet):
  ═══════════════════════════════════════════
  
       ┌──────────┐
       │ TSN Audio │──┐
       │ (urgent!) │  │     ┌──────────┐
       └──────────┘  ├────►│ Egress   │───► Wire
       ┌──────────┐  │     │ Port     │
       │ Web/Email │──┘     │ enp1s0  │
       │ (bulk)    │        └──────────┘
       └──────────┘
  
  Problem: Web traffic might BLOCK the audio stream!
  Result:  Audio arrives LATE = glitch/dropout 😩
  
  
  WITH CBS:
  ═══════════════════════════════════════════
  
       ┌──────────┐     Queue 3 (High)
       │ TSN Audio │───► [CBS Shaper] ──┐
       │ (urgent!) │     (guaranteed)   │  ┌──────────┐
       └──────────┘                     ├─►│ Egress   │───► Wire
       ┌──────────┐     Queue 0 (Low)   │  │ Port     │
       │ Web/Email │───► [best effort]──┘  │ enp1s0  │
       │ (bulk)    │                       └──────────┘
       └──────────┘
  
  Result: Audio ALWAYS gets its 10 Mbps! No glitches! ✅
```

---

### 8.2 The Credit System — How It Actually Works

CBS uses a **credit counter** to decide when a stream is allowed to send. Think of it like a **prepaid balance**:

```
  ┌─────────────────────────────────────────────────────┐
  │               CBS CREDIT RULES                       │
  │                                                      │
  │  RULE 1: When the queue has frames WAITING           │
  │          but is NOT sending → credit goes UP         │
  │          (rate = idle slope, e.g., +10 Mbps)         │
  │                                                      │
  │  RULE 2: When the queue IS sending a frame           │
  │          → credit goes DOWN                          │
  │          (rate = send slope, e.g., -990 Mbps)        │
  │                                                      │
  │  RULE 3: Frame can ONLY be sent if credit >= 0       │
  │                                                      │
  │  RULE 4: Credit cannot go above "Hi Credit" limit    │
  │                                                      │
  │  RULE 5: Credit cannot go below "Lo Credit" limit    │
  │                                                      │
  │  RULE 6: If queue is EMPTY → credit resets to 0      │
  └─────────────────────────────────────────────────────┘
```

---

### 8.3 CBS Credit Over Time — Visual Example

Imagine a TSN stream that sends **one 256-byte frame every 125 μs**:

```
  Credit
  (bytes)
    │
  Hi│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    (Hi Credit limit)
    │     /\              /\              /\
    │    /  \            /  \            /  \
    │   /    \          /    \          /    \
   0│──/──────\────────/──────\────────/──────\──────   (zero line)
    │          \      /        \      /        \
    │           \    /          \    /          \
    │            \  /            \  /            \
  Lo│─ ─ ─ ─ ─ ─ \/─ ─ ─ ─ ─ ─ ─\/─ ─ ─ ─ ─ ─ ─    (Lo Credit limit)
    │
    └─────────────────────────────────────────────► Time
         │    │         │    │         │    │
         │    │         │    │         │    │
       idle  send     idle  send     idle  send
       (+)   (-)       (+)   (-)       (+)   (-)
    
    
  WHAT'S HAPPENING:
  ─────────────────────────────────────────────
  
  Phase "idle" (credit going UP ↗):
    - TSN frame is WAITING in the queue
    - Another traffic class is using the wire
    - Credit accumulates at "idle slope" rate (+10 Mbps)
    - "I'm earning the RIGHT to send"
    
  Phase "send" (credit going DOWN ↘):
    - Credit >= 0, so CBS ALLOWS frame transmission
    - Frame is being sent on the wire
    - Credit decreases at "send slope" rate (-990 Mbps)
    - "I'm SPENDING my credit to send"
```
---

### 8.5 CBS vs Regular Ethernet — The Key Difference

```
  ┌───────────────────────────────┬────────────────────────────────┐
  │     REGULAR ETHERNET          │        CBS (TSN)               │
  ├───────────────────────────────┼────────────────────────────────┤
  │ All traffic = best effort     │ TSN traffic = GUARANTEED       │
  │                               │                                │
  │ No bandwidth reservation      │ Bandwidth reserved per stream  │
  │                               │                                │
  │ Latency: unpredictable        │ Latency: bounded & predictable │
  │ (could be 1ms or 100ms)       │ (always < 2ms if configured)   │
  │                               │                                │
  │ Under heavy load:             │ Under heavy load:              │
  │ ALL traffic slows down        │ TSN streams UNAFFECTED         │
  │                               │ (best-effort slows, not TSN)   │
  │                               │                                │
  │ Suitable for:                 │ Suitable for:                  │
  │ Web, email, file transfer     │ Audio, video, industrial       │
  │                               │ control, automotive            │
  └───────────────────────────────┴────────────────────────────────┘
```

---

### 8.6 Real-World Analogy: The Highway Toll Lane 🛣️

```
  ╔═══════════════════════════════════════════════════════════╗
  ║                  THE HIGHWAY ANALOGY                      ║
  ╠═══════════════════════════════════════════════════════════╣
  ║                                                           ║
  ║  Regular Ethernet  =  A normal highway                    ║
  ║                       Everyone shares all lanes.           ║
  ║                       Rush hour? Everyone stuck. 🚗🚗🚗   ║
  ║                                                           ║
  ║  CBS on TSN        =  A highway with a TOLL LANE          ║
  ║                       TSN streams get the toll lane.       ║
  ║                       Rush hour? TSN still zooms! 🏎️💨    ║
  ║                       Regular traffic uses remaining lanes.║
  ║                                                           ║
  ║  idle slope  =  How fast you earn toll credits while      ║
  ║                 waiting at the entrance                    ║
  ║                                                           ║
  ║  send slope  =  How fast you spend credits while          ║
  ║                 driving in the toll lane                   ║
  ║                                                           ║
  ║  hi credit   =  Maximum credits you can save up           ║
  ║                 (can't hoard unlimited passes)             ║
  ║                                                           ║
  ║  lo credit   =  Maximum debt allowed                      ║
  ║                 (can't overdraft too much)                 ║
  ╚═══════════════════════════════════════════════════════════╝
```

---

### 8.7 Complete Picture: CNC + CBS Working Together

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  CNC receives stream request from CUC                           │
  │     │                                                            │
  │     ▼                                                            │
  │  CNC builds topology graph from LLDP data                       │
  │     │                                                            │
  │     ▼                                                            │
  │  CNC finds best path: Talker → Bridge1 → Bridge2 → Listener    │
  │     │                                                            │
  │     ▼                                                            │
  │  CNC calculates CBS parameters:                                  │
  │     │  idle slope = 10 Mbps  (reserved bandwidth)                │
  │     │  send slope = -990 Mbps                                    │
  │     │  hi credit  = +15 bytes                                    │
  │     │  lo credit  = -1527 bytes                                  │
  │     │                                                            │
  │     ▼                                                            │
  │  CNC pushes config via NETCONF/YANG to Device Managers           │
  │     │                         │                                  │
  │     ▼                         ▼                                  │
  │  Device Mgr (Bridge 1)    Device Mgr (Bridge 2)                 │
  │     │                         │                                  │
  │     ▼                         ▼                                  │
  │  Configures CBS on         Configures CBS on                     │
  │  enp1s0 (egress) 🟡        enp1s0 (egress) 🟡                   │
  │     │                         │                                  │
  │     └──────────┬──────────────┘                                  │
  │                ▼                                                  │
  │  CNC reports back to CUC: "Path is ready!"                      │
  │                ▼                                                  │
  │  CUC tells Talker & Listener: "START STREAMING!"                │
  │                ▼                                                  │
  │  ════════ TSN STREAM FLOWS WITH CBS QoS ════════                │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

---

