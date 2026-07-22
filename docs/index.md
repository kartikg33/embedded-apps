---
hide:
  - navigation
---

# SquashFS-based Apps instead of Containers

<p class="document-subtitle">
A Lightweight SquashFS-based Application Runtime for Resource-Constrained Linux Systems
</p>

<p class="document-meta">
By Kartik Gohil. July 2026
</p>

This paper proposes an application lifecycle architecture for embedded Linux
systems using **squashfs** blobs instead of containers.

Applications are distributed as immutable, self-contained filesystem images (squashfs images) and executed within isolated Linux namespaces. Each application carries its own dynamic loader and runtime libraries to remove dependencies on packages installed on the host Linux kernel.

## Why Not Containers?

Container technologies such as Docker and OCI runtimes have become the dominant approach to application packaging and isolation in cloud and server environments. However, for embedded Linux platforms — particularly broadband and home-networking devices — containers introduce practical constraints that make them unsuitable as the primary deployment model.

### Memory Consumption

Each container image bundles a complete operating system userspace alongside the application itself. On a typical embedded device with 256 MB or 512 MB of RAM, launching even a small number of containers simultaneously exhausts available memory. The resident memory overhead of a minimal container runtime, combined with per-container userspace duplication, leaves little headroom for application logic and concurrent workloads.

### Rising Cost of Memory in 2026

Global memory chip shortages and sustained price increases throughout 2025–2026 have placed significant pressure on device bill-of-materials costs. Manufacturers of broadband gateways, home routers, and set-top boxes are constrained to fixed memory configurations that were specified before memory prices rose. Architectures that minimise per-application memory consumption directly reduce hardware cost requirements and extend the viable lifetime of already-deployed devices.

### Support for Low-Specification Devices

The broadband device market encompasses a wide range of hardware, from high-end Wi-Fi 7 gateways to entry-level ADSL modems. Many deployed devices in the field have constrained CPU, limited flash storage, and small amounts of RAM. A deployment model that scales down to low-specification hardware enables a single architecture to serve the entire device population rather than requiring separate solutions for different hardware tiers.

### Concurrent Application Scaling: A New Broadband Use Case

Broadband platforms are increasingly expected to run multiple independent applications simultaneously — network diagnostics tools, parental controls, security agents, QoS managers, VPN clients, smart home integrations, and operator services. This concurrent multi-application use case is fundamentally different from video playback devices such as Android TV, where the interaction model is sequential: the user switches from one application to another and the platform is designed to hibernate one container before launching the next.

Broadband devices cannot rely on the same hibernation strategy. Network and security applications must remain active continuously to perform their functions. Pausing a firewall or traffic shaping agent to start a diagnostics tool is not operationally acceptable. This means all applications must be resident in memory simultaneously, and the per-application memory cost is directly multiplied by the number of concurrently running applications.

Container runtimes, designed for environments where resources are plentiful or workloads can be queued and hibernated, are not optimised for this pattern. The overhead of running four or five containers simultaneously on a typical broadband device with 256 MB of RAM is currently sufficient to make concurrent multi-application deployment impractical.

### The Result: A Deployment Bottleneck

The combination of high per-container memory overhead, rising memory costs, constrained device hardware, and the requirement for concurrent always-on applications creates a deployment bottleneck. Operators who wish to expand their application ecosystem on existing broadband hardware are throttled not by software capability but by the inability to run more than one or two containers at a time on a typical device.

The architecture proposed in this document addresses this bottleneck directly by eliminating per-application userspace duplication, leveraging kernel demand paging to load only the pages actively required, and removing the resident container runtime daemon entirely.

## Executive Summary

Modern embedded Linux platforms increasingly require independent application deployment, versioning, rollback, and lifecycle management. Container technologies such as OCI provide these capabilities but are primarily designed for cloud-native workloads, where resource availability and operational requirements differ significantly from constrained embedded devices.

Many embedded platforms have limited RAM, flash storage, CPU resources, and network bandwidth. In these environments, packaging every application with an entire userspace root filesystem and managing it through a full container runtime introduces additional storage, operational complexity, and memory overhead that may not be necessary for trusted, managed applications.

This paper proposes an alternative application lifecycle architecture based on **self-contained SquashFS application images** combined with native Linux isolation primitives. The objective is to retain the operational benefits of containerised deployment while significantly reducing implementation complexity and improving efficiency for embedded platforms.

Rather than distributing applications as OCI images, each application is packaged as a compressed, immutable SquashFS filesystem image. The image contains the complete userspace required by the application, including its executable, dynamic loader, C runtime, private shared libraries, configuration defaults, and application assets.

Application images are stored on encrypted persistent flash and mounted directly from storage as read-only filesystems. The application is executed within an isolated Linux namespace environment where the SquashFS image becomes the application's effective root filesystem. As a result, applications do not dynamically link against host libraries and are insulated from changes to the underlying operating system userspace.

The Linux kernel remains the only intentionally shared runtime component.

This approach creates a clear separation between the immutable operating system and independently managed applications while leveraging existing kernel capabilities such as demand paging, compressed read-only filesystems, and the page cache. Only application code and data that are actively accessed are loaded into memory, allowing efficient utilisation of available RAM without unpacking application images.

## Proposed Architecture

The proposed platform consists of four primary components:

* **Application Bundle** – A signed, compressed, read-only SquashFS image containing a complete application userspace.
* **Application Supervisor** – A trusted system service responsible for verification, installation, mounting, execution, monitoring, updates, and rollback.
* **Sandbox Runtime** – A lightweight execution environment constructed from Linux namespaces, cgroups, capability management, seccomp, and Linux Security Module policies.
* **Persistent Bundle Store** – An encrypted repository for immutable application images with support for versioning and atomic activation.

Unlike traditional container runtimes, this architecture does not require layered OCI images, writable overlay filesystems, or a resident container daemon. Instead, the operating system constructs a minimal execution environment specifically for each application.

```text
Signed SquashFS Bundle
          │
          ▼
  Application Supervisor
          │
   Verify + Mount
          │
          ▼
 ┌────────────────────────────┐
 │ Mount Namespace            │
 │ PID Namespace              │
 │ IPC Namespace              │
 │ Network Namespace          │
 │ Cgroup                     │
 │ Seccomp                    │
 │ Capabilities               │
 │ LSM Policy                 │
 └────────────────────────────┘
          │
          ▼
 Self-contained Application
 (Own glibc, loader, libs)
          │
          ▼
 Shared Linux Kernel
          │
          ▼
Encrypted Persistent Flash
```

## Key Design Principles

The architecture is founded on several core principles:

### Immutable Applications

Applications are never modified after installation. Every application image is mounted read-only, ensuring software integrity and enabling atomic version switching and rollback.

### Self-Contained Userspace

Applications carry their own runtime libraries and dependencies. They do not consume shared userspace libraries from the host operating system, eliminating dependency conflicts and reducing ABI coupling.

### Shared Kernel

Applications continue to share the host Linux kernel, providing efficient execution while avoiding the overhead of virtual machines.

### Native Kernel Isolation

Application isolation is achieved through existing Linux kernel mechanisms rather than a container runtime abstraction. The supervisor creates dedicated namespaces, applies capability restrictions, configures cgroups, and enforces security policy before launching each application.

### Explicit Resource Access

Applications receive access only to explicitly authorised devices, filesystems, sockets, and platform services. Host resources are never implicitly exposed.

### Lifecycle Independence

Applications can be installed, upgraded, rolled back, started, stopped, and removed independently of the base operating system.

## Expected Benefits

The proposed architecture offers several advantages for embedded Linux platforms.

### Reduced Storage Requirements

Packaging only the application userspace eliminates unnecessary duplication of operating system components while still maintaining application independence.

### Efficient Memory Utilisation

Application images remain compressed on persistent storage. Kernel demand paging loads executable pages and data only when required, reducing peak memory utilisation compared to extracting applications onto writable filesystems.

### Deterministic Runtime Environment

Each application executes against its own validated runtime libraries, preventing unexpected behaviour resulting from operating system library updates.

### Simplified Updates

Applications can be upgraded independently from the operating system through side-by-side installation and atomic activation.

### Stronger Platform Integrity

The base operating system remains immutable and isolated from application software, reducing the risk of accidental modification or corruption.

### Smaller Runtime Footprint

Removing the need for a general-purpose container engine reduces background services and overall platform complexity.

## Security Model

Security is provided through multiple complementary layers rather than a single isolation mechanism.

Each application is cryptographically verified prior to installation and execution. Verified application images are mounted read-only and executed within isolated Linux namespaces. Capabilities are removed by default, Linux Security Module policies restrict resource access, seccomp filters constrain available system calls, and cgroups enforce CPU, memory, and process limits.

Applications receive access only to explicitly authorised resources.

The architecture assumes a trusted kernel and trusted application supervisor. The operating system root filesystem remains protected from application modification throughout the application lifecycle.

## Comparison with Conventional Containers

Although the proposed architecture shares several Linux kernel technologies with container implementations, it differs fundamentally in packaging philosophy.

Instead of distributing a complete container image containing an application and duplicated operating system environment, applications are distributed as immutable, self-contained userspace bundles designed specifically for managed embedded platforms.

The goal is not to replace container technology generally, but to provide a deployment model better suited to environments where applications are trusted, platform hardware is fixed, and operational efficiency is prioritised over maximum workload portability.

## Technical Challenges

Several areas require further investigation during the proof-of-concept phase.

These include:

* Selection of the optimal SquashFS compression algorithm and block size.
* Performance characteristics during cold application startup.
* Memory utilisation compared with equivalent OCI deployments.
* Cryptographic signing and trust-chain implementation.
* Management of persistent application state across software upgrades.
* Kernel compatibility requirements.
* Namespace configuration for constrained embedded platforms.
* Benchmarking against existing container runtimes.

These investigations will determine whether the proposed architecture provides measurable improvements over existing deployment mechanisms.

## Proof of Concept

The proposed proof of concept will implement a minimal application supervisor capable of:

* Installing signed SquashFS application images.
* Verifying application authenticity and integrity.
* Mounting images directly from encrypted persistent storage.
* Constructing isolated application namespaces.
* Executing applications using only bundled runtime libraries.
* Applying security policy through Linux namespaces, cgroups, seccomp, capabilities, and Linux Security Modules.
* Monitoring application health.
* Supporting atomic updates and rollback.

Performance measurements will compare this architecture against equivalent OCI-based deployments, evaluating storage consumption, memory utilisation, startup latency, runtime overhead, and implementation complexity.

## Conclusion

The Self-Contained SquashFS Application Runtime proposes a lightweight alternative to conventional container deployment for resource-constrained Linux devices.

By combining immutable SquashFS application images with Linux's native isolation mechanisms, the architecture seeks to deliver independent application lifecycle management, deterministic runtime environments, strong platform integrity, and efficient resource utilisation without the operational overhead of traditional container runtimes.

The proposal intentionally leverages proven Linux kernel capabilities rather than introducing new kernel functionality. Its success depends not on replacing containers, but on identifying a deployment model that is better aligned with the requirements of embedded systems where resources are constrained, applications are managed, and long-term maintainability is paramount.

The next phase of work is the implementation of a proof of concept to validate the architectural assumptions, quantify performance characteristics, and determine whether the proposed model offers meaningful advantages over existing application deployment technologies.
