# KubeVirt PoC

A proof of concept exploring **KubeVirt and the integration of virtual machines with Kubernetes**.

The project covers the concepts behind KubeVirt, its architecture and components, along with the practical setup and operation of virtual machines alongside regular Kubernetes workloads.

## Overview

[KubeVirt](https://kubevirt.io/) extends Kubernetes with virtualization capabilities, allowing virtual machines to be managed within a Kubernetes environment.

This PoC explores KubeVirt from both a **conceptual** and **practical** perspective — understanding how the platform works internally and then applying those concepts in a Kubernetes environment.

## Repository Structure

```text
.
├── README.md
└── kubevirt-concepts/
    ├── README.md
    ├── 01-introduction
    ├── 02-architecture
    ├── 03-components
    ├── 04-virtual-machines
    ├── 05-security-rbac
    └── 06-reference-glossary
```

### `kubevirt-concepts/`

This directory is the **conceptual reference for KubeVirt**.
It covers the fundamentals needed to understand how KubeVirt works, including:

- KubeVirt architecture
- Core components
- Virtual machines and VM instances
- VM lifecycle
- Kubernetes and KubeVirt resources
- Networking
- Storage
- Supporting components and concepts    

It is intended to provide the background needed to understand the practical work in the PoC.

### Practical Work

The practical portion of the PoC focuses on applying these concepts in a Kubernetes environment, including installing KubeVirt and running virtual machines alongside regular Kubernetes workloads.

## What This PoC Covers

- KubeVirt architecture and components
- KubeVirt installation and configuration
- Creating and running virtual machines
- VM lifecycle management
- Running VMs alongside Kubernetes Pods
- VM networking and storage    
- Accessing and interacting with VMs
- Practical experimentation and troubleshooting

## Key Idea

The central idea explored in this PoC is the coexistence of traditional Kubernetes workloads and virtual machines within the same cluster:

```text
                 Kubernetes Cluster
                        │
              ┌─────────┴─────────┐
              │                   │
        Kubernetes Pods       KubeVirt VMs
              │                   │
       Container workloads    Virtual workloads
```

KubeVirt provides the virtualization layer that allows VMs to become part of the Kubernetes environment and be managed alongside containerized workloads.

## References

- [KubeVirt](https://kubevirt.io/) 
- [KubeVirt Documentation](https://kubevirt.io/user-guide/)
- [Kubernetes](https://kubernetes.io/)