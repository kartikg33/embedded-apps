# Embedded Apps

This repository hosts a proposal for an application lifecycle architecture for embedded Linux systems.

Applications are distributed as immutable, self-contained filesystem images (squashfs images) and executed within isolated Linux namespaces. Each application carries its own dynamic loader and runtime libraries to remove dependencies on packages installed on the host Linux kernel.

## Documentation

Read the published documentation:

https://kartikg33.github.io/embedded-apps/