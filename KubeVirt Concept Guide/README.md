# KubeVirt Notes

> **One-line idea:** Kubernetes manages the platform (scheduling, storage, networking, RBAC). KubeVirt adds virtual machines as native Kubernetes objects, running on top of that same platform via QEMU/KVM.

## How this is organized

| #   | Doc                                                      | What it covers                                                     | Read this if...                                             |
| --- | -------------------------------------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------------- |
| 1   | [`01-introduction.md`](./01-introduction.md)             | What KubeVirt is, why it exists, the core mental model             | you're new to KubeVirt or explaining it to someone else     |
| 2   | [`02-architecture.md`](./02-architecture.md)             | Control plane, node agents, request flow, how a VM actually starts | you want to know what's happening under the hood            |
| 3   | [`03-components.md`](./03-components.md)                 | Every moving part in detail: storage, CDI, networking, services    | you're integrating or debugging a specific subsystem        |
| 4   | [`04-virtual-machines.md`](./04-virtual-machines.md)     | VM vs VMI, lifecycle, creating/managing VMs, migration             | you're actually running VMs day to day                      |
| 5   | [`05-security-rbac.md`](./05-security-rbac.md)           | Namespaces, RBAC, isolation, what actually secures a VM            | you're setting up multi-tenant or production access control |
| 6   | [`06-reference-glossary.md`](./06-reference-glossary.md) | Cheat sheet, glossary, the "five things to remember"               | you already know the material and need a fast lookup        |

## Reading paths

- **First pass (understand the shape):** 1 → 2 → 6
- **Building a PoC:** 1 → 2 → 3 → 4, then 5 before you show it to anyone else
- **Just need one thing:** jump straight to the doc — each one repeats the couple of terms it depends on, so you're not forced to backtrack

## Conventions used across every doc

- **KubeVirt vs Kubernetes** is the recurring axis: every doc calls out explicitly which layer (Kubernetes primitive vs KubeVirt addition) a concept belongs to.
- **VM** = the persistent `VirtualMachine` object (desired state). **VMI** = `VirtualMachineInstance`, the actual running thing. This distinction is introduced in doc 1 and used everywhere after.
- Diagrams are plain text blocks — no tooling required to read them.
- "Kubernetes-native" means: modeled as a CRD, managed with `kubectl`/`virtctl`, governed by RBAC and namespaces like any other cluster object — not a separate system bolted on the side.

## Scope note

This is a conceptual + operational reference, not the official docs. For exact API fields, current default values, or version-specific behavior, cross-check [kubevirt.io/user-guide](https://kubevirt.io/user-guide/) — KubeVirt moves fast and some defaults (especially around networking bindings and live migration) change between releases.
