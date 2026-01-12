---
layout: post
title: "Managarm: End of 2025 Update"
excerpt: "Managarm is a pragmatic microkernel-based OS with fully asynchronous I/O. We review our achievements of 2024 and 2025 and set goals for 2026."
---
<span style="font-size: 11pt;">Post written by Dennis Bonke ([@Dennisbonke](https://github.com/Dennisbonke)), Alexander van der Grinten ([@avdgrinten](https://github.com/avdgrinten)), [@no92](https://github.com/no92), Arsen Arsenović ([@ArsenArsen](https://github.com/ArsenArsen)), [@48cf](https://github.com/48cf) and Marvin Friedrich ([@marv7000](https://github.com/marv7000)).
</span>

## Introduction

It has been two years since our last status update on the [Managarm operating system](https://github.com/managarm/managarm); hence, we have quite a lot of new features, updates and improvements to share.

For readers unfamiliar with the project, Managarm is a fully asynchronous microkernel-based operating system with source-level compatibility with Linux software. Check out [our README](https://github.com/managarm/managarm) that also explains how to try out the system.

Two years ago, Managarm turned 10 years old. We are proud to see that the project has attracted many contributors and that the system has gained a lot of capabilities since its inception. Some of the newest additions are presented below.

## Major feature additions

### RISC-V

One of the major additions that we merged in 2024 is support for the 64-bit variant of the RISC-V architecture ([PR #752](https://github.com/managarm/managarm/pull/752)). Support was introduced both in our kernel and user-space such that we can now boot to our Weston desktop also on RISC-V. In the kernel, this involved the addition of RISC-V to the boot protocol, architecture-specific code for RISC-V memory management and thread switching, and drivers for RISC-V interrupt controllers (i.e., the PLIC, APLIC and IMSIC controllers), among other things. On the user-space side, we could build on pre-existing support for the `riscv64-linux-mlibc` target of [our C library mlibc](https://github.com/managarm/mlibc) that was easy to adapt from Linux to Managarm.

Booting to Weston on RISC-V currently works on the qemu `virt` target. Managarm works on real RISC-V hardware to varying extent, with most testing happening on the Banana Pi BPI-F3. However, support for the desktop environment on real hardware will require the addition of some new drivers, most notably to support USB.

### NVIDIA Open GPU Driver

Ever [since NVIDIA open-sourced](https://www.phoronix.com/review/nvidia-open-kernel) their kernel-side Linux driver, the possibility for other OSes to port it has existed, even if it always looked quite complex. The only successful effort to port this driver to another OS has been [Haiku](https://discuss.haiku-os.org/t/haiku-nvidia-porting-nvidia-driver-for-turing-gpus/16520/), but this has not been upstreamed yet. The BSD family has partial support by virtue of sharing the DRM implemention with Linux, something that is not really workable for us. But with a bit of research it became clear that the [nvidia-open driver](https://github.com/NVIDIA/open-gpu-kernel-modules/) is written quite portably, as they seem to share this codebase across all the officially supported operating systems.

The nvidia-open driver has a set of sysdeps that implement OS-specific functionality while exposing it to the driver in a portable way. Such functionality includes getting system information, getting time, memory management, PCI access, many types of locking, and logging to name but a few. For a driver of this size and significance, the [implementation](https://github.com/managarm/managarm/blob/a79e117aa16136eb84f10433fae2c2632711323f/drivers/gfx/nvidia-open/src/sysdeps/os-interface.cpp) we needed is surprisingly brief.

The driver is split up into multiple modules on Linux, and this structure translates over into our code, albeit in adapted form:
- `nvidia.ko`: On Linux, this kernel modules provides basic primitives for interacting with the GPU. These primitives are implemented as ioctls, which are used for things like allocating memory, submitting commands or power management. This module alone is not enough to get video output, though.
- `nvidia-modeset.ko`: This module implements higher-level functions on top of `nvidia.ko` that can be used to do things like modesetting.
- `nvidia-drm.ko`: This module connects the primitives exposed by `nvidia-modeset.so` to DRM, exposing the driver to userspace via the normal DRM UAPI.

For Managarm, we ended up building the equivalent of `nvidia.ko` and `nvidia-modeset.ko` as static libraries that we link into our GPU driver server. Using the primitives they provide, we provide our own wiring to our DRM implementation. This is necessary as our DRM implementation has a completely different driver-side API as Linux's DRM, despite providing the same DRM UAPI interface. This is the reason we cannot use `nvidia-drm.ko` for Managarm, as that relies on the driver-side bindings.

This approach ended up working quite nicely, in large part due to the functions being provided by nvidia-modeset being a good match to our DRM driver-side interface. The result is a working modeset on Managarm ([PR #951](https://github.com/managarm/managarm/pull/951)):

![NVIDIA modeset on real hardware](/assets/2025-end-of-year-update/nvidia-real-hardware.png)

Please do note that this is different than having 3D accelerated graphics - this requires more work on separate issues. The obvious problem is the fact that the userland libraries that NVIDIA ships with their drivers, even the open one, are still proprietary closed-source blobs. We cannot just use them on Managarm, as they are linked against glibc (including glibc-specific and versioned symbols) and may use direct syscalls, something that does not translate well to Managarm. Instead, there have been efforts to make mesa's NVK work with nvidia-open as a backend, as an alternative to the nouveau backend that it usually uses. Such an approach would also have the nice upside of being able to provide a fully open-source NVIDIA graphics stack on Linux using nvidia-open, and would also be portable to other operating systems that choose to port the nvidia-open driver while providing a DRM UAPI.

### Support for `systemd` init
In 2025 we embarked on the quest to switch from a hand-rolled init system to something more suited for the job. Considering we aim for Linux source compatibility and the fact that we already used udev, we logically went with systemd ([PR #447](https://github.com/managarm/bootstrap-managarm/pull/447)). After a few deep debugging sessions we managed to boot with systemd, which has now been the default for several months already.

![systemd booting](/assets/2025-end-of-year-update/systemd-booting.png)

## New features in kernel, POSIX and drivers

### Kernel

#### Load Balancing

While Managarm's kernel always had SMP (= multi-core) support, it did not yet have an algorithm to move threads between CPUs to achieve a uniform load distribution among all CPUs. The kernel's new load balancer ([PR #819](https://github.com/managarm/managarm/pull/819)) tackles this issue. It works by estimating the load of every CPU in regular intervals (using an exponentially decaying average of the CPU time of all threads on each CPU). If the load is not evenly distributed, the algoritm tries to re-distribute it by letting under-utilized CPUs pull tasks from over-utilized CPUs.

#### UEFI Boot
The ability to boot directly on UEFI systems was also an important addition ([PR #730](https://github.com/managarm/managarm/pull/730)). This allows users to load Managarm without relying on external UEFI-based bootloaders and also supports U-Boot's EFI implementation.

#### Limine Boot
Managarm has been using the Limine bootloader for a while, alongside with offering support for booting with GRUB2. This relied on us having support for the multiboot2 or Linux protocols.

As Limine also provides its own boot protocol, we added support for this ([PR #744](https://github.com/managarm/managarm/pull/744)), enabling us to share this boot protocol with very little changes across our supported architectures.

### POSIX implementation

#### coredump

To improve our debugging experience, we added basic support for producing core dumps ([PR #987](https://github.com/managarm/managarm/pull/987)). Although this is limited to x86_64 for now, it has helped us to debug issues that would otherwise have been quite difficult to do.

Implementing this was surprisingly straight-forward: a coredump is an ELF file with some notes to relay information about the process and its state, and a dump of the memory regions that are not file-backed. This is enough to load it into gdb and to be able to inspect the state at crash time. Coredumps have been particularly useful for situations where multiple processes crash, as that is a scenario where our gdbserver implementation has the shortcoming of only running for the first abnormally terminated process. Instead, coredumps are written out for as many processes as you wish.

As our support is currently pretty basic, we would like to extend this further. Among other things, supporting it on aarch64 and riscv64 would be nice, as well as implementing some of the interfaces around it. For instance, we do not have a way to change the core pattern (`/proc/sys/kernel/core_pattern` on Linux), and neither do we support configuration about what should be dumped - Linux allows users to select between multiple policies of what memory regions to dump. Wiring it up to `coredumpctl` would also be a logical next step.

#### Thread Groups

Managarm's POSIX implementation initially treated threads as separate processes with a shared address space. That violated the POSIX specification in several ways; among other things, all threads returned different values from `getpid()`. We recently rewrote this logic to support Linux-like _thread groups_, i.e., processes that share some of their state (such as PIDs, signal handlers and child notifications) ([PR #1014](https://github.com/managarm/managarm/pull/1014)). The new implementation makes several glib tests pass that previously failed and it fixes some potential hangs that application ran into in the past.

### Networking

#### Cellular networking

As some laptops come with cellular modems these days, we have considered adding support for some of them. Due to the diverse set of modems available and their different interfaces used (some are PCI-based, others are exposed as USB devices for instance), supporting all of them is infeasible without having access to the hardware and significant development time. However, we did end up having a ThinkPad with a Sierra Wireless EM7455 in it, which supports 4G connectivity and exposes that over the standard MBIM interface via USB. After some wiring, we were able to use ModemManager to connect to mobile network and browse the web with it ([PR #705](https://github.com/managarm/managarm/pull/705)). Even using some auxiliary functions like sending and receiving SMS was possible, although some other features (like GPS) are unsupported for now.

#### IP stack improvements

While Managarm had client side TCP support for some time, we recently also gained support for TCP listening sockets ([PR #1112](https://github.com/managarm/managarm/pull/1112)). This allows running HTTP servers and other server software on Managarm. Other feature additions in the IP stack include an IPv4 loopback device ([PR #1115](https://github.com/managarm/managarm/pull/1115)) and support for ICMP responses. The latter now allows Managarm devices to be pinged from the network.

#### NVMe-oF and PXE

Instead of booting Managarm from a disk image, we wanted to offer a way to quickly iterate by employing network booting to offer a fully diskless Managarm boot. Usually, network boot is offered by way of PXE - this has the network announcing via DHCP where clients can download a file to boot from. PXE allows us to deploy a bootloader like Limine, which we can use to load further stages. While we use TFTP for these early stages of boot, this does not scale to deploying an image. NVMe over Fabrics was a good fit for us, as this allows us to mount entire disks over the network, while also plugging into our existing NVMe driver infrastructure ([PR #805](https://github.com/managarm/managarm/pull/805)). In the end, we were able to do entirely diskless boots of Managarm on test machines. With some additional scripting and firmware configurations this can even be entirely automated with Wake-On-LAN.

#### K1-emac driver and GENETv5

The SpacemiT K1 is a RISC-V SoC used on SBCs like the Banana Pi BPI-F3. We implemented a driver for the k1-emac ([PR #1019](https://github.com/managarm/managarm/pull/1019)) Ethernet controller and a PHY abstraction in netserver. PHY is the hardware component responsible for transmitting the Ethernet packets over the physical wires and the abstraction has also paved the way for inclusion of the Broadcom GENETv5, which is the NIC present on the Raspberry Pi 4B ([PR #1022](https://github.com/managarm/managarm/pull/1022)).

## New ports and updates
The past two years saw a lot of changes and additions in our ports tree. As of January 2026, we have 454 packages indexed in our repository for x86_64! A few noticable ones and highlights follow.

### GNOME stack
This year we also ported several parts of the GNOME stack. The most challenging part here is their heavy use of GObject Introspection, which brings a lot of issues with cross compilations along with it. This was hacked around with the use of a host-mlibc targeting Linux and liberal use of patchelf. Even with these workarounds, it proved to be a persistent problem and therefore this project was one of many starts and stops ([PR #296](https://github.com/managarm/bootstrap-managarm/pull/296)).

![GNOME Calculator](/assets/2025-end-of-year-update/gnome-calculator.png)

### KDE Stack
While the GNOME stack was sometimes stuck, we also worked on the KDE stack this year. Here we also went for the calculator, but we also went for a new IRC client that is actually maintained, replacing the now-archived hexchat ([PR #363](https://github.com/managarm/bootstrap-managarm/pull/363)). The plan is to continue this work in 2026.

![Konversation IRC client](/assets/2025-end-of-year-update/konversation.png)

### Python packaging
As our [main build system](https://github.com/managarm/xbstrap) is written in Python, it makes sense to think about packaging python modules. After some looking around to see how others do it, we opted to follow Gentoo and use gpep517 ([PR #584](https://github.com/managarm/bootstrap-managarm/pull/584)). The python packages ported are now used to power our `ci-boot` infrastructure script.

### Java
Lastly, we worked on porting Java this year. This involved some in-depth debugging after an initial port by [Qwinci](https://github.com/Qwinci) and eventually traced back to an oversight in our CoW implementation with split mappings. After that was done we could run basic Java applications ([PR #605](https://github.com/managarm/bootstrap-managarm/pull/605)).

![Java hello world](/assets/2025-end-of-year-update/java.png)

#### Minecraft
But of course we wanted more. We teamed up with [Mathewnd](https://github.com/Mathewnd) from [Astral](https://astral-os.org/) to get Minecraft working on our OS. During this debugging session we did have a small detour and ran a very old Minecraft classic in the browser.

![Minecraft classic running in the browser](/assets/2025-end-of-year-update/minecraft-in-browser.png)

After that we went on and had _another_ detour with running a Minecraft server and connecting to it from a Linux host, which also worked fine after implementing the neccesary server-side parts of the network stack.

![Managarm hosting a Minecraft server](/assets/2025-end-of-year-update/minecraft-server.png)

Then we finally went back to the main prize, actual Minecraft. This, too, worked out fine with some debugging help as previously mentioned. We only tried an early alpha version due to it needing fewer native libraries but we're pretty sure we can run up to somewhat recent versions with LWJGL2 and when we port LWJGL3 up to 1.20.3.

![Minecraft Alpha running on Managarm](/assets/2025-end-of-year-update/minecraft.png)

We even have some epic gameplay to show here!

[![Minecraft Alpha running on Managarm](https://img.youtube.com/vi/a2eEsHpFYsw/0.jpg)](https://www.youtube.com/watch?v=a2eEsHpFYsw)

### os-test

We ported [os-test](https://sortix.org/os-test/), a cross-OS testsuite, to Managarm this year and regularly publish [current results to its homepage](https://sortix.org/os-test/#results) ([PR #597](https://github.com/managarm/bootstrap-managarm/pull/597)). We recently published a [blog post](https://managarm.org/2025/11/03/os-test.html) about the effort, and have since worked on fixing bugs, adding missing features and contributing tests to os-test itself.

## Infrastructure improvements

### The `xbbs` distribution build server recreated from ground-up
In the last year, [`xbbs`](https://github.com/managarm/xbbs), the build server for our distribution, was entirely rewritten.

The new version of `xbbs` features several improvements (in no particular order):

- **Better log view**: In presence of JavaScript, the `xbbs` web view now uses `xterm.js` to interpret and display colored build logs. Also, the log viewer will follow the output from running builds, meaning one can track builds as they run. It is also now easier to debug coordinator failures, as `xbbs` will store a coordinator log for each job, with details of what was happening in that build.
- **Dark theme**: This change is certainly the most important! You will now be able to look at `xbbs` at night without straining your eyes, as long as your system is using a dark theme.
- **Cleaner, more modular implementation**: The `xbbs` implementation itself was significantly improved; whereas the old `xbbs` was a slapdash hack meant to distribute our build process as soon as possible, the new `xbbs` has a significantly cleaner codebase, that is thoroughly documented, and more modular. The design is such that any build system could be integrated into it, so long as its jobs can be expressed as a directed acyclic graph.
- **Easier administration**: The new `xbbs` foregoes assumptions on the layout of the filesystem on workers and the coordinator, making it far easier to deploy. It now also performs all coordinator to worker communication over a single port, making implementing a firewall easier. The build server is also far more resistant to workers failing mid-execution, and will retry jobs that fail in such abnormal ways.

If you want to see the new `xbbs` web UI, our official instance is at [builds.managarm.org](https://builds.managarm.org/), though you might get bored. A good build server is one that you don't think about.

### Hardware CI

To improve our testing capabilites on real hardware, we implemented infrastructure for automated booting and testing of Managarm on real hardware. The hardware CI software stack consists of two servers: a central relay server that is responsible for authentication and for scheduling CI runs and multiple decentralized target servers that are connected via UART and ethernet connections to the hardware devices that we want to test. We are using network boot over TFTP or HTTP to boot the devices, UARTs to connect to their consoles and NVMe-over-TCP to provide the disk image to the devices.

So far, we have connected four devices to hardware CI, namely Raspberry Pi 4 (AArch64), Orion O6 (AArch64 with ACPI), Banana Pi BPI-F3 (RISC-V) and an Intel N100 board (x86_64).

### GitHub CI improvements

To enable easier automated testing of Managarm, we implemented a `ci-boot` script that boots the system in Qemu, launches an arbitrary shell script, redirects the script's output to the host, and uploads artifact files from the Managarm guest to the host ([PR #1078](https://github.com/managarm/managarm/pull/1078)).

For x86_64, we adjusted our GitHub Actions CI to run various boot configurations through `ci-boot`, including the `os-test` testsuite. We also improved our CI pipeline to run boot tests on all architectures and we also test boot with KASAN enabled. Finally, on the linting side, we now also run `clang-tidy`.

## What's next?

There are several things that we want to focus on in the next few months. One aspect is performance and stability improvements, as well as improved POSIX compliance and testing. On the ports side, we want to focus on a richer desktop experience (beyond the currently supported Weston and Sway desktops) and some server side applications. Furthermore, we always want to improve our driver support on real hardware. Last but not least, we will hopefully see more upstreaming of Managarm patches in the future. Next steps in this area may be Rust `std` support (as the Managarm target for Rust is already upstream) and our GCC patches.

## Help wanted!
We are always looking for new contributors. If you want to get involved in the project, you can search our [GitHub issues](https://github.com/managarm/managarm/issues) for interesting tasks. As mentioned before, focus points for us are improved performance and stability, additional drivers on all fronts and of course fixing bugs that may occur somewhere. A good way to help with that, other than contributing code, is of course running our [nightly images](https://builds.managarm.org/projects/managarm/success/repo/files/x86_64/image.xz) either on real hardware or in QEMU, play around with it and report anything that is broken.

If you want to get in touch with the team, you can find us on [Discord](https://discord.gg/7WB6Ur3) or on [irc.libera.chat](irc.libera.chat) in the #managarm channel. Also, if you want to support Managarm, please consider [donating to the project](https://github.com/sponsors/avdgrinten)!

## Acknowledgments
We thank all people who contributed to Managarm in 2024 and 2025, including but not limited to: Kacper Słomiński ([@qookei](https://github.com/qookei)), Geert Custers ([@Geertiebear](https://github.com/Geertiebear)), Matt Taylor ([@64](https://github.com/64)), Matheus ([@Mathewnd](https://github.com/Mathewnd)), Alexander Richards ([@ElectrodeYT](https://github.com/ElectrodeYT)), [@mintsuki](https://github.com/mintsuki), [@streaksu](https://github.com/streaksu), [@Qwinci](https://github.com/Qwinci), [@d-tatianin](https://github.com/d-tatianin), [@johnsonjh](https://github.com/johnsonjh), [@KSPAtlas](https://github.com/KSPAtlas), [@apache-hb](https://github.com/apache-hb), [@lzcunt](https://github.com/lzcunt), [@Andy-Python-Programmer](https://github.com/Andy-Python-Programmer), [@FedorLap2006](https://github.com/FedorLap2006), [@netbsduser](https://github.com/netbsduser), [@rayanmargham](https://github.com/rayanmargham), [@sasdallas](https://github.com/sasdallas), [@awewsomegamer](https://github.com/awewsomegamer), [@Alessandro-Salerno](https://github.com/Alessandro-Salerno), [@da4089](https://github.com/da4089), [@Ekaums](https://github.com/Ekaums), [@oberrow](https://github.com/oberrow), [@Valeryum999](https://github.com/Valeryum999), [@monkuous](https://github.com/monkuous), [@tunis4](https://github.com/tunis4), [@9xbt](https://github.com/9xbt), [@pitust](https://github.com/pitust), [@clonidine](https://github.com/clonidine), [@ishan-karmakar](https://github.com/ishan-karmakar), [@offlinemark](https://github.com/offlinemark) and [@DeanoBurrito](https://github.com/DeanoBurrito).