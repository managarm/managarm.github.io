---
layout: post
title: "Managarm: Status Update and 12th Anniversary"
excerpt: "We review Managarm's progress since our last status update in January."
---
<span style="font-size: 11pt;">Post written by Alexander van der Grinten ([@avdgrinten](https://github.com/avdgrinten)), Dennis Bonke ([@Dennisbonke](https://github.com/Dennisbonke)), Alexander Richards ([@ElectrodeYT](https://github.com/ElectrodeYT)) and Marvin Friedrich ([@marv7000](https://github.com/marv7000)).</span>

## Introduction

In this post, we give an update on the progress of the [Managarm operating system](https://github.com/managarm/managarm) since our [End of 2025 update](https://managarm.org/2026/01/13/end-of-year-update.html) in January. This month also marks Managarm's 12th anniversary, so it's a good time to give a status update.

Some of this year's work was already covered in dedicated posts: we [received an NGI Zero grant](https://managarm.org/2026/06/16/0-nlnet.html) to enhance Managarm's web capabilities, and we wrote about [IOMMU support](https://managarm.org/2026/06/16/1-iommu.html), [swapping of anonymous memory](https://managarm.org/2026/08/19/0-anon-swap.html), [fine-grained capability rights](https://managarm.org/2026/08/22/0-capability-rights.html) and [our port of Epiphany](https://managarm.org/2026/10/13/0-epiphany.html). This post focuses on the remaining changes. Highlights include blocking physical memory allocations, resource accounting via hierarchy capabilities, moving device discovery to userspace, ACPI support on AArch64, a new POSIX device model, IOMMU support in all PCIe drivers, file system `sync()` support and a port of KDE Plasma 6.

## Memory allocator improvements and robustness against OOM

Over the last few months, we reworked how Managarm's kernel allocates physical memory to make the system robust when memory runs low. Previously, physical memory allocations could only succeed or fail immediately. Now, they can sleep until memory becomes available again ([#1702](https://github.com/managarm/managarm/pull/1702)), for example, after pages were written back to disk or swapped out.

Making allocations sleep is non-trivial. Before serving waiters, the kernel needs to be able to determine whether their allocations will succeed. For multi-page allocations, success is difficult to predict without actually trying the allocation because it depends on fragmentation. To work around this issue, multi-page allocations are served from a dedicated contiguous reserve that grows by converting memory that previously only held evictable pages ([#1698](https://github.com/managarm/managarm/pull/1698)). Only single-page allocations may wait. Since they can be satisfied by any free page outside of the contiguous reserve, accounting of free pages is enough to predict whether they will succeed or not.

There are still allocations in the kernel that cannot sleep. To ensure that these can be satisfied, user and regular kernel allocations must keep a certain amount of memory free and sleep instead of violating this floor. The memory that is held back in this way acts as an emergency pool: kernel allocations that cannot sleep fall back to it instead of panicking the kernel.

## Resource accounting via hierarchy capabilities

Managarm now tracks resources in a hierarchy that forms a tree ([#1618](https://github.com/managarm/managarm/pull/1618)). The nodes of the tree are exposed to userspace as capabilities that we call _hierarchy capabilities_. The kernel creates the root node, each server gets its own node and userspace can extend the tree further. For example, our POSIX implementation creates a hierarchy node for each process and file system drivers create nodes for page cache data and metadata. Memory objects are associated with a hierarchy node, and whenever the kernel allocates physical memory for them, it charges that memory to the node. Swap space is accounted for separately ([#1634](https://github.com/managarm/managarm/pull/1634)). The new `lshierarchy` utility displays the memory and swap usage of each node ([#1632](https://github.com/managarm/managarm/pull/1632)).

The resource hierarchy is also the natural place to implement memory limits. As a first step, the kernel now restricts the memory that userspace can allocate to around 90% of physical memory ([#1695](https://github.com/managarm/managarm/pull/1695)). Allocations fail when userspace programs try to exceed this limit. In the future, we plan to extend this mechanism to allow userspace to attach its own memory limits to hierarchy nodes.

## Device discovery in userspace

We have moved the handling of ACPI, Device Tree and PCI devices to userspace with a new server called `sif` ([#1510](https://github.com/managarm/managarm/pull/1510)). It is fully written in Rust and has reached feature parity with the previous kernel mode implementation ([#1696](https://github.com/managarm/managarm/pull/1696)). This required a few kernel changes, including new syscalls for setting up DMA spaces for the IOMMU ([#1639](https://github.com/managarm/managarm/pull/1639)) and configuring IRQs from userspace, like ACPI GSIs. ACPI drivers for the embedded controller and battery were also ported over ([#1650](https://github.com/managarm/managarm/pull/1650)).

There are plenty of advantages over the old scheme: not only does this make the kernel smaller, but bugs in ACPI AML or similar code also cannot take down the kernel anymore. Since `sif` also deals with driver requests in userspace, the attack surface of the kernel is much reduced.

`sif` is now used for all firmware device configuration and discovery ([#1685](https://github.com/managarm/managarm/pull/1685)), and the kernel mode implementation has been removed ([#1697](https://github.com/managarm/managarm/pull/1697)).

## ACPI support on AArch64 and RISC-V

AArch64 devices using ACPI firmware are now supported by Managarm's kernel. GICv2 and GICv3 interrupt controllers, system timers and SMP can now be discovered via ACPI tables ([#1389](https://github.com/managarm/managarm/pull/1389)). This goes hand-in-hand with the move to `sif`, with the new code being written for it from the beginning. We successfully ran Managarm on a Radxa Orion O6 and Radxa Rock 5B+ using ACPI. Managarm also supports ACPI on RISC-V; however, since there are no RISC-V devices with ACPI firmware available at the time of writing, this could only be tested on `qemu-system-riscv64`.

Here's a picture of Managarm running on the Radxa Rock 5B+, printing `dmesg` output:

![Managarm running on the Radxa Rock 5B+](/assets/2026-oct-update/rock5b.jpg)

## posix-devserver: new POSIX device model

POSIX applications on Managarm are not directly owned by the kernel. Instead, the posix-subsystem server is responsible for managing them. posix-subsystem's main purpose is tracking POSIX processes and threads and their resources, including their open file descriptors, mounts, signals and similar. It also provides many special file descriptors such as epoll, timerfd, signalfd, as well as the `/proc` file system. posix-subsystem is generally not responsible for I/O. `read()` and `write()` operations bypass posix-subsystem -- they are directly routed to file system servers or the networking server.

In addition to these process-related tasks, posix-subsystem was also responsible for enumerating devices and for constructing a Linux-compatible `/sys` file system. However, this functionality has now been moved out to a new server called posix-devserver that is written entirely in safe Rust ([#1644](https://github.com/managarm/managarm/pull/1644), [#1656](https://github.com/managarm/managarm/pull/1656)). posix-devserver enumerates devices from Managarm's native mbus discovery bus, resolves dependencies between them, serves `/sys` and emits netlink uevents. The isolation of the device model into a separate server will improve the overall robustness of our system -- if a bug causes posix-devserver to crash or hang, posix-subsystem (and therefore most POSIX processes) will be unaffected. Additionally, the rewrite of this component allowed us to restructure the code in a more maintainable way. Most notably, per-subsystem dependency resolution between devices has been replaced by a unified dependency resolver, and the new implementation is 1500 lines shorter than the old one, which has since been removed ([#1678](https://github.com/managarm/managarm/pull/1678)).

## IOMMU support in PCIe drivers

In one of our [previous blog posts](https://managarm.org/2026/06/16/1-iommu.html), we wrote about Managarm's recently added IOMMU support. The support for IOMMU DMA translation and protection has now been extended to all our PCIe drivers. This includes the USB host controller drivers (UHCI and EHCI in [#1442](https://github.com/managarm/managarm/pull/1442), xHCI had already been converted before), graphics drivers (our Intel GPU driver from lil in [#1437](https://github.com/managarm/managarm/pull/1437), the open-source NVIDIA driver is currently awaiting unrelated fixes), sound (Intel HDA in [#1433](https://github.com/managarm/managarm/pull/1433)) and networking (Intel e1000 in [#1427](https://github.com/managarm/managarm/pull/1427), RTL8168 in [#1444](https://github.com/managarm/managarm/pull/1444)). This series of PRs concludes our IOMMU conversion work -- all relevant drivers now support IOMMUs.

## File system `sync()` support

`sync()` may seem like a pretty minor function, but we are now at a point where we also have to consider reliability when, for example, running Managarm as a server. In such environments, `sync()` becomes a strikingly important function to have, *especially* if the machine is doing things that must be regularly flushed to disk. Additionally, it is also important to have when shutting down.

With that said, `sync()` and its variants are not as simple as they may seem. Sure, the main function does what it says on the tin: it syncs all file systems to disk. However, there are several variations of syncing, ranging from `syncfs(int fd)` (which syncs the file system that backs a particular file) to `fsync(int fd)` (which syncs the specific file) and `fdatasync(int fd)` (which syncs the data, but not the metadata, of the file).

On Managarm, a sync call has to go through several servers and libraries to complete, requiring changes to most elements of the file system stack ([#1629](https://github.com/managarm/managarm/pull/1629)). Additionally, a sync call has to guarantee the data was actually *written*, which involves flushing the write cache on the actual backing device; this was previously only implemented on virtio-blk, but is now implemented on all block devices that Managarm supports (specifically, SATA, NVMe, ATA and USB mass storage devices; [#1687](https://github.com/managarm/managarm/pull/1687)).

## KDE Plasma 6

Managarm has run Weston, the reference Wayland compositor, as its default graphical environment for years. While it is fine for testing GUI applications, real desktop usage asks more from the environment than Weston alone can give us. That is why we spent significant time this year on porting KDE Plasma to Managarm (in a series of PRs that concluded with [bootstrap-managarm#693](https://github.com/managarm/bootstrap-managarm/pull/693)). With the recent improvements in the block stack, KDE Plasma is now usable for daily use. While it is not the default environment on nightly images, it can be installed via the `kde-desktop` metapackage ([bootstrap-managarm#762](https://github.com/managarm/bootstrap-managarm/pull/762)).

Here is a picture of Managarm running KDE Plasma 6 and `fastfetch` in `konsole`:

![KDE Plasma 6 running on Managarm](/assets/2026-oct-update/kde.png)

## Other improvements

Our block stack received significant performance and robustness improvements. The ext2 implementation in libblockfs is now fully thread-safe and distributes its work over multiple threads ([#1560](https://github.com/managarm/managarm/pull/1560)), virtio-block supports multi-queue ([#1592](https://github.com/managarm/managarm/pull/1592)), and ext2 gained caches for metadata ([#1559](https://github.com/managarm/managarm/pull/1559)) as well as lazy block allocation ([#1596](https://github.com/managarm/managarm/pull/1596)).

Following up on our [post on swapping of anonymous memory](https://managarm.org/2026/08/19/0-anon-swap.html), the main part of swapping of copy-on-write memory has been merged ([#1630](https://github.com/managarm/managarm/pull/1630)). On RISC-V, the kernel now supports hardware virtualization via the hypervisor extension ([#1470](https://github.com/managarm/managarm/pull/1470)). Finally, we are currently working on support for USB 3 hubs.

## Supported by NLnet

[This project](https://nlnet.nl/project/Managarm/) was funded through the [NGI0 Commons Fund](https://nlnet.nl/commonsfund/), a fund established by [NLnet](https://nlnet.nl) with financial support from the European Commission’s Next Generation Internet programme, under the aegis of DG Communications Networks, Content and Technology under grant agreement No 101135429. Additional funding is made available by the Swiss State Secretariat for Education, Research and Innovation (SERI).

<img src="/assets/2026-oct-update/nlnet.svg" alt="NLnet logo" width="200" style="display: inline-block; margin-right: 10px; margin-bottom: 10px;" />
<img src="/assets/2026-oct-update/NGI0Commons.svg" alt="NGI0 Commons logo" width="200" style="display: inline-block; margin-bottom: 10px;" />

## Talk to us

We welcome contributions to Managarm! If you want to help out, feel free to look at [our issue tracker](https://github.com/managarm/managarm/issues), or chat with us on [Discord](https://discord.gg/7WB6Ur3) or IRC in `#managarm` on `irc.libera.chat` about ideas.
