# Portworx Learning Path — Spec-Driven Course

## Goal

Build production-grade understanding of Portworx from first principles — not command memorization.

Most engineers get stuck during incidents because they don't understand the full stack:

```
Application
 ↓
Pod
 ↓
Volume Mount
 ↓
PVC
 ↓
PV
 ↓
StorageClass
 ↓
CSI
 ↓
Storage Backend
 ↓
Disk
```

This course reverses that — you build upward from disk to application, so every incident makes sense.

---

## How to Use This Course

Each module is a **spec file** — it defines:

- What to learn (learning objectives)
- How to verify you learned it (success criteria)
- What to build (hands-on labs)
- What to answer (interview questions)
- What to watch for in production (production takeaways)

Work through each module **in order**. Do not skip phases.

---

## Course Structure

### Phase 1 — Storage Fundamentals

> Before touching Kubernetes. Build the mental model that makes everything else debuggable.

| Module | Title | File |
|--------|-------|------|
| 01 | Storage From First Principles | [phase-1-storage-fundamentals/module-01-storage-first-principles.md](./phase-1-storage-fundamentals/module-01-storage-first-principles.md) |
| 02 | Linux Storage | [phase-1-storage-fundamentals/module-02-linux-storage.md](./phase-1-storage-fundamentals/module-02-linux-storage.md) |

### Phase 2 — Kubernetes Storage

> Understand how Kubernetes abstracts storage — and where those abstractions leak.

| Module | Title | File |
|--------|-------|------|
| 03 | Why Containers Need Persistent Storage | [phase-2-kubernetes-storage/module-03-containers-and-persistent-storage.md](./phase-2-kubernetes-storage/module-03-containers-and-persistent-storage.md) |
| 04 | Persistent Volumes Deep Dive | [phase-2-kubernetes-storage/module-04-persistent-volumes.md](./phase-2-kubernetes-storage/module-04-persistent-volumes.md) |
| 05 | CSI Architecture | [phase-2-kubernetes-storage/module-05-csi-architecture.md](./phase-2-kubernetes-storage/module-05-csi-architecture.md) |
| 06 | Stateful Applications | [phase-2-kubernetes-storage/module-06-stateful-applications.md](./phase-2-kubernetes-storage/module-06-stateful-applications.md) |

### Phase 3 — Portworx

> Now every component has context. Portworx is a CSI driver + distributed storage system + operator.

| Module | Title | File |
|--------|-------|------|
| 07 | Portworx Architecture | [phase-3-portworx/module-07-portworx-architecture.md](./phase-3-portworx/module-07-portworx-architecture.md) |
| 08 | Portworx Installation | [phase-3-portworx/module-08-portworx-installation.md](./phase-3-portworx/module-08-portworx-installation.md) |
| 09 | Portworx Data Plane | [phase-3-portworx/module-09-portworx-data-plane.md](./phase-3-portworx/module-09-portworx-data-plane.md) |
| 10 | Portworx Operations | [phase-3-portworx/module-10-portworx-operations.md](./phase-3-portworx/module-10-portworx-operations.md) |
| 11 | Portworx Enterprise Features | [phase-3-portworx/module-11-portworx-enterprise-features.md](./phase-3-portworx/module-11-portworx-enterprise-features.md) |
| 12 | Portworx Production Debugging | [phase-3-portworx/module-12-portworx-production-debugging.md](./phase-3-portworx/module-12-portworx-production-debugging.md) |

---

## Expected Outcome

After all 12 modules:

```
Disk
 ↓
Filesystem
 ↓
Linux Storage
 ↓
Kubernetes Storage
 ↓
PV / PVC
 ↓
CSI
 ↓
Portworx Control Plane
 ↓
Portworx Data Plane
 ↓
Replication
 ↓
Disaster Recovery
 ↓
Production Operations
```

You will be able to trace any production incident from symptom to root cause across this entire stack.

---

## Spec-Driven Learning Model

Each spec file follows this structure:

```
Status          → draft | in-progress | complete
Prerequisites   → what you must know before starting
Objectives      → what you will be able to do after
Concepts        → the spec of what to study
Labs            → hands-on exercises to build
Exercises       → scenario-based practice
Interview Qs    → questions that prove understanding
Production Qs   → production takeaways and war stories
Success Criteria→ how to know you are done
```

A module is **complete** when you can answer every interview question cold and finish every lab without looking at the answer.
