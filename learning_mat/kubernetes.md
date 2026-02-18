# Docker, Kubernetes & Cloud Native

---

## 1. The Problem — "It Works on My Machine"

```
  Developer:   "The app works on MY laptop!"
  Ops/Server:  "It crashes on the server!"

  WHY?
  - Different OS version
  - Different library versions
  - Different environment variables
  - Missing dependencies

  ┌──────────────────┐         ┌──────────────────┐
  │  Dev's Laptop    │         │  Production      │
  │                  │         │  Server           │
  │  Python 3.10     │         │  Python 3.8  ❌  │
  │  libssl 1.1      │         │  libssl 3.0  ❌  │
  │  Ubuntu 22       │         │  CentOS 7    ❌  │
  │                  │         │                  │
  │  App: ✅ works   │         │  App: 💥 crashes │
  └──────────────────┘         └──────────────────┘

  SOLUTION: Package the app WITH its ENTIRE environment.
  That's what CONTAINERS do.
```

---

## 2. What Is a Container?

A container is a **lightweight, isolated package** that has your app
AND everything it needs to run — OS libraries, dependencies, config.

```
  A CONTAINER is like a LUNCHBOX 🍱

  You don't just carry rice. You carry:
    - Rice (your app)
    - Curry (dependencies)
    - Spoon (runtime)
    - Napkin (config files)

  Everything is SELF-CONTAINED.
  Doesn't matter if you eat in office, park, or train.
  The lunchbox works EVERYWHERE.

  Same with containers:
    Doesn't matter if you run on your laptop, a server,
    or AWS/Azure/GCP. The container works EVERYWHERE. ✅
```

### Container vs Virtual Machine

```
  VIRTUAL MACHINE (VM):
  ┌──────────────────────────────────────────────┐
  │  HOST OS (Windows / Linux)                    │
  │  ┌──────────────────────────────────────────┐│
  │  │  HYPERVISOR (VMware / VirtualBox)        ││
  │  │  ┌────────────┐  ┌────────────┐          ││
  │  │  │  VM 1       │  │  VM 2       │         ││
  │  │  │  ┌────────┐│  │  ┌────────┐│          ││
  │  │  │  │ App A  ││  │  │ App B  ││          ││
  │  │  │  │ Libs   ││  │  │ Libs   ││          ││
  │  │  │  │ FULL OS││  │  │ FULL OS││  ← HEAVY!││
  │  │  │  │ (2 GB) ││  │  │ (2 GB) ││          ││
  │  │  │  └────────┘│  │  └────────┘│          ││
  │  │  └────────────┘  └────────────┘          ││
  │  └──────────────────────────────────────────┘│
  └──────────────────────────────────────────────┘
  Each VM has its OWN full OS. Heavy (GBs), slow to start.


  CONTAINER:
  ┌──────────────────────────────────────────────┐
  │  HOST OS (Linux kernel)                       │
  │  ┌──────────────────────────────────────────┐│
  │  │  CONTAINER ENGINE (Docker)               ││
  │  │  ┌────────────┐  ┌────────────┐          ││
  │  │  │ Container 1│  │ Container 2│          ││
  │  │  │  ┌────────┐│  │  ┌────────┐│          ││
  │  │  │  │ App A  ││  │  │ App B  ││          ││
  │  │  │  │ Libs   ││  │  │ Libs   ││          ││
  │  │  │  │ NO OS! ││  │  │ NO OS! ││  ← LIGHT!│
  │  │  │  │ (50 MB)││  │  │ (50 MB)││          ││
  │  │  │  └────────┘│  │  └────────┘│          ││
  │  │  └────────────┘  └────────────┘          ││
  │  └──────────────────────────────────────────┘│
  └──────────────────────────────────────────────┘
  Containers SHARE the host OS kernel. Light (MBs), instant start.


  ┌──────────────────┬────────────────┬────────────────┐
  │                  │  VM            │  CONTAINER     │
  ├──────────────────┼────────────────┼────────────────┤
  │  Size            │  GBs           │  MBs           │
  │  Startup         │  Minutes       │  Seconds       │
  │  OS              │  Full OS each  │  Shares host   │
  │  Isolation       │  Strong        │  Good enough   │
  │  Performance     │  ~5% overhead  │  Near native   │
  │  Use case        │  Different OS  │  Same OS, many │
  │                  │  on same HW    │  apps isolated │
  └──────────────────┴────────────────┴────────────────┘
```

### How Do Containers Work? (Linux Magic)

```
  Containers use TWO Linux kernel features:

  1. NAMESPACES — gives each container its OWN view of the system
     ┌────────────────────────────────────────────────────────┐
     │  Container A sees:          Container B sees:          │
     │  PID 1: my_app              PID 1: my_other_app       │
     │  Hostname: "web-1"          Hostname: "db-1"          │
     │  Network: 172.17.0.2        Network: 172.17.0.3       │
     │                                                        │
     │  They think they're the ONLY process on the machine!  │
     │  But they're actually sharing the same Linux kernel.   │
     └────────────────────────────────────────────────────────┘

  2. CGROUPS — limits HOW MUCH resources a container can use
     ┌────────────────────────────────────────────────────────┐
     │  Container A: max 512MB RAM, max 1 CPU core            │
     │  Container B: max 1GB RAM, max 2 CPU cores             │
     │                                                        │
     │  Even if Container A has a memory leak, it can only    │
     │  eat 512MB. It WON'T crash Container B or the host.   │
     └────────────────────────────────────────────────────────┘
```

---

## 3. Docker — The Container Tool

Docker is the most popular tool to **build, ship, and run** containers.

### Key Concepts

```
  ┌───────────────────────────────────────────────────────────────┐
  │                                                               │
  │  DOCKERFILE        → Recipe (instructions to build image)    │
  │       │                                                       │
  │       ▼ docker build                                         │
  │  DOCKER IMAGE      → Snapshot (frozen, read-only template)   │
  │       │                                                       │
  │       ▼ docker run                                           │
  │  DOCKER CONTAINER  → Running instance (live, writable)       │
  │                                                               │
  └───────────────────────────────────────────────────────────────┘

  ANALOGY:
    Dockerfile  = Recipe for a cake
    Image       = The frozen cake (ready to eat, can make many copies)
    Container   = A cake on the table being eaten (running instance)

  From ONE image, you can run MANY containers.
  Just like from ONE recipe, you can bake MANY cakes.
```

### Dockerfile — The Recipe

```dockerfile
  # A real Dockerfile example:

  FROM ubuntu:22.04              # Start with Ubuntu base image
  
  RUN apt-get update && \        # Install system dependencies
      apt-get install -y python3 python3-pip

  WORKDIR /app                   # Set working directory

  COPY requirements.txt .        # Copy dependency file
  RUN pip3 install -r requirements.txt   # Install Python deps

  COPY . .                       # Copy your app code

  EXPOSE 8080                    # Document which port the app uses

  CMD ["python3", "app.py"]      # Command to run when container starts
```

**What each instruction does:**
```
  ┌──────────┬──────────────────────────────────────────────────┐
  │  Command │  What it does                                    │
  ├──────────┼──────────────────────────────────────────────────┤
  │  FROM    │  Base image to start from (like Ubuntu, Alpine) │
  │  RUN     │  Execute a command during BUILD time            │
  │  COPY    │  Copy files from your machine INTO the image    │
  │  WORKDIR │  Set the current directory inside the image     │
  │  EXPOSE  │  Document which port the app listens on         │
  │  ENV     │  Set environment variables                      │
  │  CMD     │  Default command when container STARTS          │
  │ENTRYPOINT│ Like CMD but harder to override                  │
  └──────────┴──────────────────────────────────────────────────┘
```

### Docker Image Layers

```
  Each instruction in a Dockerfile creates a LAYER.
  Layers are STACKED and CACHED.

  ┌────────────────────────────┐
  │  Layer 5: CMD python3 app  │  ← your app runs
  ├────────────────────────────┤
  │  Layer 4: COPY . .         │  ← your code
  ├────────────────────────────┤
  │  Layer 3: RUN pip install  │  ← python packages
  ├────────────────────────────┤
  │  Layer 2: RUN apt install  │  ← system packages
  ├────────────────────────────┤
  │  Layer 1: FROM ubuntu:22   │  ← base OS
  └────────────────────────────┘

  WHY LAYERS?
  If you change ONLY your app code (Layer 4),
  Docker reuses Layers 1-3 from cache. Build is FAST!

  If you change requirements.txt, only Layers 3-5 rebuild.
  Layer 1-2 still cached. Still pretty fast.
```

### Essential Docker Commands

```
  # BUILD an image from a Dockerfile
  docker build -t my-app:v1 .
         └─ tag/name ──┘  └─ look for Dockerfile in current dir

  # RUN a container from an image
  docker run -d -p 8080:80 --name web my-app:v1
         └─ detached  └─ host:container port  └─ container name

  # LIST running containers
  docker ps

  # STOP a container
  docker stop web

  # VIEW logs
  docker logs web

  # GO INSIDE a running container
  docker exec -it web /bin/bash

  # LIST images
  docker images

  # REMOVE a container
  docker rm web

  # REMOVE an image
  docker rmi my-app:v1

  # PUSH image to Docker Hub (share with others)
  docker push myusername/my-app:v1

  # PULL image from Docker Hub
  docker pull nginx:latest
```

### Docker Volumes — Persistent Data

```
  Problem: when a container is deleted, ALL data inside is LOST.
  Solution: VOLUMES — store data OUTSIDE the container.

  ┌──────────────────────────┐
  │  Container (temporary)   │
  │                          │
  │  /app/data ──────────────│────► VOLUME on host
  │  (writes go to volume)   │     /var/lib/docker/volumes/mydata
  │                          │
  └──────────────────────────┘

  Even if the container dies, the volume SURVIVES.

  docker run -v mydata:/app/data my-app
                └─ volume name  └─ path inside container
```

### Docker Networking

```
  Containers can talk to each other through Docker networks.

  ┌──────────────────────────────────────────────┐
  │  Docker Network: "my-network"                 │
  │                                               │
  │  ┌────────────┐     ┌────────────┐           │
  │  │  web-app   │ ──► │  database  │           │
  │  │ 172.18.0.2 │     │ 172.18.0.3 │           │
  │  └────────────┘     └────────────┘           │
  │                                               │
  │  web-app can reach database by NAME:          │
  │  mysql://database:3306   (not IP, just name!) │
  └──────────────────────────────────────────────┘

  docker network create my-network
  docker run --network my-network --name database mysql
  docker run --network my-network --name web-app my-app
```

### Docker Compose — Multi-Container Apps

When your app needs MULTIPLE containers (web + db + cache),
use **Docker Compose** to define them all in ONE file.

```yaml
  # docker-compose.yml

  version: "3.8"
  services:
    web:                          # Container 1: your app
      build: .
      ports:
        - "8080:80"
      depends_on:
        - db
        - redis
      environment:
        - DB_HOST=db
        - REDIS_HOST=redis

    db:                           # Container 2: database
      image: postgres:15
      volumes:
        - db-data:/var/lib/postgresql/data
      environment:
        - POSTGRES_PASSWORD=secret

    redis:                        # Container 3: cache
      image: redis:7

  volumes:
    db-data:                      # Named volume for DB persistence
```

```
  # Start ALL containers
  docker compose up -d

  # Stop ALL containers
  docker compose down

  # View logs of all containers
  docker compose logs

  That single file replaces 3 long docker run commands!
```

```
  What Docker Compose creates:

  ┌──────────────────────────────────────────────────────┐
  │  Docker Compose Network (auto-created)                │
  │                                                       │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
  │  │   web    │  │    db    │  │  redis   │           │
  │  │ :8080    │──│ :5432    │  │ :6379    │           │
  │  │          │  │          │  │          │           │
  │  └──────────┘  └──────────┘  └──────────┘           │
  │                     │                                 │
  │                     ▼                                 │
  │              ┌────────────┐                           │
  │              │  db-data   │  (volume, persists)       │
  │              │  volume    │                           │
  │              └────────────┘                           │
  └──────────────────────────────────────────────────────┘
```

---

## 4. Docker Registry — Sharing Images

```
  A REGISTRY is a storage/warehouse for Docker images.
  Like GitHub but for Docker images instead of code.

  ┌──────────────┐   docker push   ┌─────────────────────┐
  │  Your Machine │ ─────────────► │  REGISTRY            │
  │  (built image)│                │  (Docker Hub / ECR / │
  └──────────────┘                │   GCR / ACR)         │
                                   └─────────────────────┘
                                            │
                                   docker pull
                                            │
                                   ┌────────▼──────────┐
                                   │  Server / Cloud    │
                                   │  (runs the image)  │
                                   └───────────────────┘

  Public:  Docker Hub (hub.docker.com) — free for public images
  Private: AWS ECR, Google GCR, Azure ACR — your company's images
```

---

## 5. Cloud Native — The Big Picture

**Cloud Native** = building apps that are DESIGNED to run in the cloud,
taking full advantage of cloud features (scaling, resilience, automation).

### Cloud Native Principles

```
  ┌─────────────────────────────────────────────────────────────┐
  │  CLOUD NATIVE PRINCIPLES                                     │
  │                                                             │
  │  1. CONTAINERIZED                                            │
  │     Pack your app in containers (Docker)                    │
  │     Runs the same everywhere                                │
  │                                                             │
  │  2. MICROSERVICES                                            │
  │     Break your app into small, independent services         │
  │     Each service does ONE thing well                        │
  │                                                             │
  │  3. DYNAMICALLY ORCHESTRATED                                │
  │     Use Kubernetes to manage containers automatically       │
  │     Scale up/down based on demand                           │
  │                                                             │
  │  4. DEVOPS / CI/CD                                          │
  │     Automate build, test, deploy                            │
  │     Push code → auto-deploys to production                  │
  └─────────────────────────────────────────────────────────────┘
```

### Monolith vs Microservices

```
  MONOLITH (Traditional):
  ┌──────────────────────────────────────┐
  │  ONE BIG APPLICATION                  │
  │                                      │
  │  ┌──────┐ ┌──────┐ ┌──────┐        │
  │  │ Auth │ │Orders│ │ Pay  │        │
  │  │      │ │      │ │ment  │        │
  │  └──────┘ └──────┘ └──────┘        │
  │  ┌──────┐ ┌──────┐                  │
  │  │Search│ │Notify│   All tightly    │
  │  │      │ │      │   coupled!       │
  │  └──────┘ └──────┘                  │
  │                                      │
  │  ONE codebase, ONE deployment        │
  │  If payment breaks → EVERYTHING down │
  └──────────────────────────────────────┘


  MICROSERVICES (Cloud Native):
  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │ Auth │  │Orders│  │ Pay  │  │Search│  │Notify│
  │      │  │      │  │ment  │  │      │  │      │
  │ :3001│  │ :3002│  │ :3003│  │ :3004│  │ :3005│
  └──┬───┘  └──┬───┘  └──┬───┘  └──────┘  └──────┘
     │         │         │
     └─────────┴─────────┘
           API calls between services

  Each service:
  ✅ Has its OWN codebase
  ✅ Has its OWN database
  ✅ Can be deployed INDEPENDENTLY
  ✅ Can be written in DIFFERENT languages
  ✅ Can SCALE independently (need more payment? add more copies)
  ✅ If payment crashes → only payment is down, rest keeps working
```
---

## 6. Why Do We Need Kubernetes?

Docker can run containers. But what happens when you have **hundreds** of containers?

```
  SMALL SCALE (Docker is fine):
  ┌──────────────┐
  │  1 Server    │
  │  3 containers│    ← you can manage this by hand
  └──────────────┘

  LARGE SCALE (you need help):
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ Server 1 │ │ Server 2 │ │ Server 3 │ │ Server 4 │
  │ 25 cont. │ │ 30 cont. │ │ 20 cont. │ │ 35 cont. │
  └──────────┘ └──────────┘ └──────────┘ └──────────┘

  Questions you can't answer manually:
  - Container crashed → who restarts it?
  - Server 2 is overloaded → who moves containers?
  - Need 50 copies of my app → who creates them?
  - New version deployed → how to do it without downtime?
  - Container needs 2GB RAM → which server has space?

  KUBERNETES answers ALL of these. Automatically. ✅
```

---

## 7. Kubernetes (K8s) — The Container Orchestrator

**Kubernetes** = a system that **manages containers across many machines**,
automatically handling deployment, scaling, and healing.

```
  Think of Kubernetes as a MANAGER for your containers:
  YOU say:   "I want 5 copies of my web app, always running."
  K8s says:  "Done. And if one crashes, I'll replace it automatically."
```

### Kubernetes Architecture

```
  A Kubernetes CLUSTER has two types of machines:

  ┌─────────────────────────────────────────────────────────────────┐
  │                    KUBERNETES CLUSTER                           │
  │                                                                 │
  │  ┌────────────────────────────────────┐                         │
  │  │         CONTROL PLANE              │   The "BRAIN"           │
  │  │         (Master Node)              │                         │
  │  │                                    │                         │
  │  │  ┌──────────┐  ┌──────────────┐    │                         │
  │  │  │ API      │  │ Scheduler    │    │                         │
  │  │  │ Server   │  │ (decides     │    │                         │
  │  │  │ (front   │  │  where pods  │    │                         │
  │  │  │  door)   │  │  run)        │    │                         │
  │  │  └──────────┘  └──────────────┘    │                         │
  │  │  ┌──────────┐  ┌──────────────┐    │                         │
  │  │  │ etcd     │  │ Controller   │    │                         │
  │  │  │ (database│  │ Manager      │    │                         │
  │  │  │  of ALL  │  │ (watches &   │    │                         │
  │  │  │  state)  │  │  fixes)      │    │                         │
  │  │  └──────────┘  └──────────────┘    │                         │
  │  └────────────────────────────────────┘                         │
  │                       │                                         │
  │          ┌────────────┼────────────┐                            │
  │          ▼            ▼            ▼                            │
  │  ┌─────────────┐ ┌────────────┐ ┌────────────┐                  │
  │  │ WORKER      │ │ WORKER     │ │ WORKER     │  The "MUSCLE"    │
  │  │ NODE 1      │ │ NODE 2     │ │ NODE 3     │                  │
  │  │             │ │            │ │            │                  │
  │  │ ┌────┐┌───┐ │ │ ┌────┐     │ │ ┌────┐┌───┐│                  │
  │  │ │Pod ││Pod│ │ │ │Pod │     │ │ │Pod ││Pod││                  │
  │  │ │ A  ││ B │ │ │ │ C  │     │ │ │ D  ││ E ││                  │
  │  │ └────┘└───┘ │ │ └────┘     │ │ └────┘└───┘│                  │
  │  │             │ │            │ │            │                  │
  │  │ kubelet     │ │ kubelet    │ │ kubelet    │                  │
  │  │ kube-proxy  │ │ kube-proxy │ │ kube-proxy │                  │
  │  └─────────────┘ └────────────┘ └────────────┘                  │
  └─────────────────────────────────────────────────────────────────┘
```

### Control Plane Components (The Brain)

```
  ┌────────────────────┬──────────────────────────────────────────┐
  │  Component         │  What it does                            │
  ├────────────────────┼──────────────────────────────────────────┤
  │  API Server        │  The FRONT DOOR. All commands go through │
  │  (kube-apiserver)  │  it. kubectl talks to this.              │
  │                    │  Like a receptionist at a hospital.      │
  ├────────────────────┼──────────────────────────────────────────┤
  │  etcd              │  The DATABASE. Stores ALL cluster state. │
  │                    │  "How many pods? Where? What config?"    │
  │                    │  Like the hospital's patient records.    │
  ├────────────────────┼──────────────────────────────────────────┤
  │  Scheduler         │  Decides WHICH NODE a new pod goes on.   │ 
  │                    │  Looks at CPU/RAM available on each node.│
  │                    │  Like a nurse assigning patients to rooms│
  ├────────────────────┼──────────────────────────────────────────┤
  │  Controller Manager│  WATCHES the cluster state and FIXES it. │
  │                    │  "You wanted 3 pods but only 2 running?  │
  │                    │   I'll create one more."                 │
  │                    │  Like a hospital supervisor checking     │
  │                    │  everything is OK.                       │
  └────────────────────┴──────────────────────────────────────────┘
```

### Worker Node Components (The Muscle)

```
  ┌────────────────────┬──────────────────────────────────────────┐
  │  Component         │  What it does                            │
  ├────────────────────┼──────────────────────────────────────────┤
  │  kubelet           │  Agent on EACH node. Takes orders from  │
  │                    │  the control plane. Makes sure pods are │
  │                    │  running. Like a nurse on each floor.   │
  ├────────────────────┼──────────────────────────────────────────┤
  │  kube-proxy        │  Handles NETWORKING. Routes traffic to  │
  │                    │  the right pod. Like a switchboard      │
  │                    │  operator routing phone calls.          │
  ├────────────────────┼──────────────────────────────────────────┤
  │  Container Runtime │  Actually RUNS containers (Docker,      │
  │                    │  containerd, CRI-O). The engine.       │
  └────────────────────┴──────────────────────────────────────────┘
```

---

## 8. Kubernetes Core Concepts

### Pod — The Smallest Unit

```
  A POD is the smallest thing Kubernetes manages.
  It's a wrapper around ONE or more containers.

  Usually: 1 pod = 1 container (your app)

  ┌──────────────────────────────────┐
  │  POD                             │
  │                                  │
  │  ┌────────────────────────────┐  │
  │  │ Container: my-web-app     │  │
  │  │ Image: my-app:v1          │  │
  │  │ Port: 8080                │  │
  │  └────────────────────────────┘  │
  │                                  │
  │  IP: 10.244.1.5                  │
  │  (pod gets its OWN IP address)   │
  └──────────────────────────────────┘

  Sometimes: 1 pod = 2+ containers (sidecar pattern)
  ┌──────────────────────────────────┐
  │  POD                             │
  │  ┌──────────────┐ ┌───────────┐  │
  │  │  App         │ │  Log      │  │
  │  │  Container   │ │  Sidecar  │  │
  │  │  (main app)  │ │  (ships   │  │
  │  │              │ │   logs)   │  │
  │  └──────────────┘ └───────────┘  │
  │  Containers in the SAME pod      │
  │  share network & storage.        │
  └──────────────────────────────────┘

  KEY RULE: Pods are EPHEMERAL (temporary).
  They can be killed and recreated at any time.
  NEVER store important data inside a pod.
```

### Service — Stable Networking for Pods

```
  Problem: Pods have IPs, but pods are TEMPORARY.
           When a pod dies and is replaced, it gets a NEW IP!
           How do other services find it?

  Solution: A SERVICE — a stable endpoint that routes to pods.

  ┌───────────────────────────────────────────────────────────┐
  │                                                           │
  │   Other apps call: http://my-web-service:80               │
  │                         │                                 │
  │                         ▼                                 │
  │              ┌──────────────────┐                         │
  │              │    SERVICE       │  Stable name & IP       │
  │              │  "my-web-service"│  Never changes!         │
  │              │  IP: 10.96.0.10  │                         │
  │              └────────┬─────────┘                         │
  │                       │ load balances traffic             │
  │            ┌──────────┼──────────┐                        │
  │            ▼          ▼          ▼                        │
  │       ┌────────┐ ┌────────┐ ┌────────┐                   │
  │       │ Pod 1  │ │ Pod 2  │ │ Pod 3  │  Pods come & go  │
  │       │10.244..│ │10.244..│ │10.244..│  IPs change      │
  │       └────────┘ └────────┘ └────────┘  Service hides it!│
  └───────────────────────────────────────────────────────────┘

  Service Types:
  ┌──────────────┬───────────────────────────────────────────┐
  │  Type        │  What it does                             │
  ├──────────────┼───────────────────────────────────────────┤
  │  ClusterIP   │  Internal only. Other pods can reach it. │
  │  (default)   │  Can't access from outside the cluster.  │
  ├──────────────┼───────────────────────────────────────────┤
  │  NodePort    │  Opens a port on EVERY node.             │
  │              │  Access: <NodeIP>:30080                   │
  ├──────────────┼───────────────────────────────────────────┤
  │  LoadBalancer│  Creates a cloud load balancer (AWS/GCP) │
  │              │  Gets a PUBLIC IP. Production use.       │
  └──────────────┴───────────────────────────────────────────┘
```

### Namespace — Organize Your Cluster

```
  Namespaces = folders to organize your resources.

  ┌─────────────────────────────────────────────┐
  │  Kubernetes Cluster                          │
  │                                              │
  │  ┌───────────────────┐  ┌─────────────────┐ │
  │  │  namespace: dev    │  │ namespace: prod  │ │
  │  │                   │  │                 │ │
  │  │  web-app (2 pods) │  │ web-app (10pods)│ │
  │  │  db (1 pod)       │  │ db (3 pods)    │ │
  │  │                   │  │                 │ │
  │  └───────────────────┘  └─────────────────┘ │
  │                                              │
  │  Same names, but in different namespaces!    │
  │  They don't interfere with each other.       │
  └─────────────────────────────────────────────┘
```

### ConfigMap & Secret — Externalize Config

```
  DON'T hardcode config in your app!

  ConfigMap = key-value pairs for NON-SENSITIVE config
  Secret    = key-value pairs for SENSITIVE data (passwords, keys)

  ┌──────────────────────────────────────────────────┐
  │  ConfigMap: app-config                            │
  │    DB_HOST: "postgres-service"                    │
  │    LOG_LEVEL: "info"                              │
  │    MAX_CONNECTIONS: "100"                          │
  │                                                    │
  │  Secret: app-secrets                              │
  │    DB_PASSWORD: "c2VjcmV0" (base64 encoded)       │
  │    API_KEY: "YWJjZGVm"                            │
  └──────────────────────────────────────────────────┘

  Your pod reads these as environment variables or files.
  Change config → pods pick it up. No rebuild needed!
```

### Persistent Volume — Storage That Survives

```
  Pods are temporary. When they die, data dies too.
  PersistentVolume (PV) = actual storage (disk on cloud/host)
  PersistentVolumeClaim (PVC) = a REQUEST for storage by a pod

  ┌──────────────────────────────────────────────┐
  │  Pod                                          │
  │  ┌──────────────┐                             │
  │  │  my-database │                             │
  │  │  container   │                             │
  │  │              │                             │
  │  │  /var/data ──│──► PVC ──► PV (10 GB disk) │
  │  └──────────────┘         (actual cloud disk) │
  │                                                │
  │  Pod dies → PV survives → new pod reattaches! │
  └──────────────────────────────────────────────┘
```

---

## 9. YAML — How You Talk to Kubernetes

Everything in K8s is defined in **YAML files** (manifests).

```yaml
  # deployment.yaml — tells K8s what to run

  apiVersion: apps/v1
  kind: Deployment              # what type of resource
  metadata:
    name: my-web-app            # name of this deployment
  spec:
    replicas: 3                 # run 3 copies
    selector:
      matchLabels:
        app: web                # find pods with label "app: web"
    template:                   # pod template
      metadata:
        labels:
          app: web              # label for the pods
      spec:
        containers:
        - name: web
          image: my-app:v1      # Docker image to use
          ports:
          - containerPort: 8080
          resources:
            limits:
              memory: "256Mi"
              cpu: "500m"       # 0.5 CPU cores
```

```
  # Apply it:
  kubectl apply -f deployment.yaml

  # K8s reads the YAML and makes it happen:
  "OK, I'll create 3 pods running my-app:v1 on port 8080,
   each limited to 256MB RAM and 0.5 CPU cores."
```
---

## 10. How It All Fits Together — Deployment Flow

```
  DEVELOPER writes code
       │
       ▼
  Dockerfile → docker build → Docker IMAGE
       │
       ▼
  docker push → IMAGE goes to REGISTRY (Docker Hub / ECR)
       │
       ▼
  Write Kubernetes YAML (deployment.yaml, service.yaml)
       │
       ▼
  kubectl apply → K8s CONTROL PLANE receives the request
       │
       ▼
  SCHEDULER picks which NODES have space
       │
       ▼
  KUBELET on each node pulls the IMAGE from registry
       │
       ▼
  CONTAINERS start running inside PODS
       │
       ▼
  SERVICE provides stable endpoint for traffic
       │
       ▼
  Users access your app! 🎉


  Full picture:
  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐
  │Developer│───►│ Registry │───►│   K8s    │───►│ Users access │
  │ builds  │    │ (stores  │    │ (deploys │    │ the app via  │
  │ & pushes│    │  images) │    │  to pods)│    │ Service/LB   │
  └─────────┘    └──────────┘    └──────────┘    └──────────────┘
```

---

## Quick Reference — Key Terms

```
  ┌────────────────────┬──────────────────────────────────────────┐
  │  Term              │  One-liner                               │
  ├────────────────────┼──────────────────────────────────────────┤
  │  Container         │  Lightweight isolated package for apps  │
  │  Docker            │  Tool to build & run containers         │
  │  Image             │  Frozen snapshot of app + dependencies  │
  │  Dockerfile        │  Recipe to build an image               │
  │  Docker Compose    │  Run multi-container apps locally       │
  │  Registry          │  Storage for Docker images              │
  │  Kubernetes (K8s)  │  Orchestrator — manages containers     │
  │  Cluster           │  A set of machines running K8s         │
  │  Node              │  One machine in the cluster            │
  │  Pod               │  Smallest unit — wraps container(s)   │
  │  Deployment        │  Manages replicas + rolling updates    │
  │  Service           │  Stable network endpoint for pods      │
  │  Namespace         │  Folder to organize resources          │
  │  ConfigMap/Secret  │  External config for pods              │
  │  PV / PVC          │  Persistent storage for pods           │
  │  kubectl           │  CLI tool to control K8s               │
  │  Cloud Native      │  Apps designed for cloud (containers + │
  │                    │  microservices + orchestration)        │
  └────────────────────┴──────────────────────────────────────────┘
```
