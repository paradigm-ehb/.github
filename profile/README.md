## Paradigm — System Overview

Paradigm is a **host-centric management system** built around explicit control, deterministic behavior, and clear trust boundaries. It is composed of three primary parts:

* **Agent** — runs on managed machines
* **Command Center** — operator-side control plane
* **Agent Resources** — low-level system inspection layer

Each component has a **strict responsibility boundary**. There is no “smart middle layer” and no hidden automation.

---

## 1. Paradigm Agent (on the managed host)

### Role

The Agent is a **local authority** over a single machine. It exposes a controlled interface for:

* Observing system state
* Executing explicitly requested actions
* Enforcing security boundaries

It never guesses intent and never self-initiates changes.

### Responsibilities

* Expose a **gRPC API** over TLS / mTLS ( TODO )
* Own the **root of trust** (certificates, enrollment)
* Coordinate access to system facilities:

  * systemd
  * package manager
  * OS-level metrics
* Act as the **only bridge** between the OS and the outside world

### Characteristics

* Long-running daemon
* Deterministic startup and shutdown
* Explicit error surfaces
* No business logic beyond “do or report”

Conceptually:

```
[ OS / Kernel / systemd / pkg mgr ]
              ^
              |
          [ Agent ]
              ^
              |
       secure gRPC boundary
```

---

## 2. Command Center (operator side)

### Role

The Command Center is the **control plane**. It never touches the OS directly and never assumes host state.

Its job is to:

* Talk to many agents
* Present state coherently
* Request actions deliberately

### Responsibilities

* Maintain agent endpoints and identities
* Handle enrollment and certificate exchange
* Issue gRPC calls:

  * Fetch resources
  * Start/stop services
  * Apply updates
* Aggregate and present results (CLI / UI / API)

### Characteristics

* Stateless or lightly stateful
* Replaceable (CLI today, UI tomorrow)
* No privileged access
* No system assumptions

Conceptually:

```
[ Command Center ]
        |
   authenticated gRPC
        |
[ Agent A ]   [ Agent B ]   [ Agent C ]
```

The Command Center **cannot bypass** an agent and **cannot outlive** revoked trust.

---

## 3. Agent Resources (low-level subsystem)

### Role

Agent Resources is a **pure system inspection and querying layer**. It answers the question:

> “What is the current state of this machine, *right now*?”

Nothing more.

### Responsibilities

* Collect snapshots of:

  * CPU
  * memory
  * processes
  * systemd units
  * installed packages
* Abstract distro-specific APIs:

  * `libalpm` (Arch)
  * `libdnf5` (Fedora)
* Provide data in a **memory-safe, CGO-friendly form**

### Design Traits

* Arena-based memory allocation
* Snapshot semantics (no live handles)
* Zero background threads
* No caching across calls
* Clear lifetime boundaries

Conceptually:

```
[ Agent ]
    |
    v
[ Agent Resources ]
    |
    v
[ OS APIs, /proc, D-Bus, libalpm, libdnf5 ]
```

Agent Resources **never talks to the network** and **never mutates state**.

---

## How the Pieces Fit Together

### Read Path (Observation)

```
Command Center
      |
      v
   gRPC call
      |
      v
    Agent
      |
      v
Agent Resources
      |
      v
   OS snapshot
```

* One request → one snapshot
* No shared mutable state
* Results are serialized and returned

### Write Path (Action)

```
Command Center
      |
      v
   gRPC call
      |
      v
    Agent
      |
      v
 systemd / pkg mgr
```

* Agent validates intent
* Executes explicitly
* Reports outcome
* No autonomous retries

---

## What Paradigm Is Optimized For

* Determinism over convenience
* Explicit control over automation
* Debuggability over abstraction
* Long-lived agents in hostile or unknown networks
* System-level correctness

---

## What Paradigm Is Not

* Not a config-management framework
* Not Kubernetes
* Not Ansible
* Not self-healing
* Not declarative infrastructure (yet)

Paradigm is closer to a **secure, inspectable control fabric** than a “manage everything for you” platform.

---

## One-Sentence Summary

**Paradigm is a secure, agent-based system for inspecting and controlling machines through explicit, minimal, and deterministic interfaces—built from the OS up, not from YAML down.**
