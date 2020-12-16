---
layout: post
title:  Managarm at the End of 2019
excerpt: Managarm is a pragmatic microkernel-based OS with fully asynchronous I/O. We review our achievements of 2019 and set goals for 2020.
---
<span style="font-size: 11pt;">Post by Alexander van der Grinten
([@avdgrinten](https://github.com/avdgrinten))
</span>

## What is Managarm?

Managarm is a pragmatic microkernel-based operating system (OS). In contrast
to most other OSes, all I/O in Managarm is asynchronous (i.e., I/O never blocks
a thread). Managarm also provides a POSIX subsystem that allows many ports to
run on the OS. This includes many tools known from Linux, such as Weston and kmscon.

For more information, check out our [GitHub repository](https://github.com/managarm/managarm).

![Managarm](https://raw.githubusercontent.com/managarm/managarm/assets/screenshots/echo-hello-managarm.png)

## What has happened in 2019?

**Loading drivers via udev.** So far, Managarm lacked a convenient and configurable way to load drivers on demand. This changed: we now load drivers by calling the `runsvr` utility from udev rules that match the supported devices. For example, to load our graphics drivers, we can use the following rules:

	ATTR{vendor}=="0x1234", ATTR{device}=="0x1111", RUN+="/usr/bin/runsvr bind /usr/lib/managarm/server/gfx-bochs.bin", GOTO="managarm_vga_end"
	ATTR{vendor}=="0x1af4", ATTR{device}=="0x1050", RUN+="/usr/bin/runsvr bind /usr/lib/managarm/server/gfx-virtio.bin", GOTO="managarm_vga_end"
	ATTR{vendor}=="0x15ad", ATTR{device}=="0x0405", RUN+="/usr/bin/runsvr bind /usr/lib/managarm/server/gfx-vmware.bin", GOTO="managarm_vga_end"

This change also enabled two further improvements: first, loading drivers is now idempotent: running `runsvr` multiple times does not launch a driver twice. Secondly, `runsvr` is also smart enough to bind new devices to existing driver processes: if you plug in more USB HID devices while the HID driver is already running, no new HID driver process will be launched.

**Kernlets.** Since Managarm is a microkernel, handling shared interrupts (e.g., from PCI devices) with low latencies is challenging. In particular, when a shared interrupt fires, we need to wake up *all* drivers that listen for this interrupt. To avoid multiple context switches, we introduced "kernlets" in Managarm's kernel. Those are small programs that run directly in interrupt contexts and determine which device triggered the interrupt. Kernlets are written in a safe language to ensure that drivers cannot corrupt the kernel (similar to Linux' eBPF). In Managarm, kernlets are verified and compiled in userspace (by a trusted server) and inserted into the kernel as modules.

**llvm-pipe renderer.** Kacper invested a lot of work on the graphics stack this year. We now have support for Mesa's llvm-pipe renderer which is much faster than the previous softpipe renderer. llvm-pipe consistently improves frame rates by more than 2x for basic 2D rendering (and by much more, if shaders are involved).

![Quake](/assets/2019-end-of-year/quake.png)

**XHCI and VMWare SVGA drivers.** Kacper wrote a driver for XHCI, the USB 3 host controller interface. With the addition of this driver, Managarm supports all USB host controllers except for the OHCI USB 1 controller. XHCI also enables much faster boot times when booting from USB 3 pen drives or hard disks. Kacper also wrote a driver for VMWare's SVGA device which provides accelerated buffer swapping and cursor support.

**LAI.** We ditched the ACPICA library in favor of the more lightweight [LAI library](https://github.com/managarm/lai) to execute AML. LAI provides a more direct interface to AML and hides fewer details than ACPICA. Currently, LAI supports all common virtual machines and lots of real hardware. We expect almost all of the remaining bugs on real hardware to be fixed in 2020.

![LAI](/assets/2019-end-of-year/lai.png)

**[Continuous integration](https://ci.managarm.org/).** At the end of 2018, we switched the build orchestration system to [xbstrap](git@github.com:managarm/xbstrap.git). xbstrap operates on top of build systems of individual packages (such as CMake or Meson); its goal is building entire OS distributions. Among other features, xbstrap can handle inter-package dependencies, downloading packages and patching ports. Based on xbstrap, we now provide nightly builds of Managarm. The CI routinely rebuilds the entire distribution, which takes around 4 hours on our CI server. The advantage of this approach is that we can also discover bugs resulting from the interaction between multiple packages. However, as Managarm continues to grow, we will need to switch to incremental rebuilds.

**New ports.** We have various new ports, mostly due to the work of Kacper. kmscon which now provides a lightweight non-graphical console for Managarm. We now also have ports of vim, SDL2 and Quake.

![kmscon](/assets/2019-end-of-year/kmscon.png)

**Core improvements.** In 2019, we made various improvements to Managarm's core. The kernel can now evict cached pages and request user space drivers to perform write-backs of dirty pages as appropriate. This reduces the physical memory footprint of the system. The kernel's *virtual* memory management also saw major refactoring, making copy-on-write robust also in the face of concurrency. A rework of kernel-userspace queues simplified the IPC code both in the kernel and in userspace.

**Move to C++20 coroutines.** In the past we used our own "stackful" emulation for asynchronous coroutines. This is arguably a hack and also consumed much more virtual memory than necessary (due to the allocation of coroutine stacks). We now compile the userspace drivers using clang which enables us to use more (memory) efficient C++20 coroutines instead.

## What do we want to achieve in 2020?

**More robustness in the kernel and POSIX subsystem.** Currently, it is still very easy to crash the POSIX server and (to a lesser extent) the kernel by invoking requests and/or syscalls with illegal arguments. This mostly concerns assertions that crash the process instead of returning an error to the caller. In 2020, we want to fix most of these issues and apply more automated testing and fuzzing to ensure that those interface are robust in the face of malformed input.

**Finalize the networking stack.** In 2019, we refactored and modernized our virtio-net driver. However, the stack is still missing a proper driver that tracks open sockets across the system. Such a driver should be added and integrated with the POSIX subsystem to provide the usual Berkeley socket interface that our POSIX subsystem already provides for UNIX domain and netlink sockets.

**Port a web browser and work towards self hosting.** There are a few things missing to make Managarm more generally usable (e.g., for UNIX programming). Aside from robustness issues mentioned above, we are still lacking a web browser and some tools required to be self hosting (e.g., Git and Python). In the next year, we should work towards fixing those issues and at least port a modern web browser such as Firefox.

**Improve the block driver stack.** Managarm currently lacks arbitration between multiple block drivers. Thus, only a single block driver can present /dev/sdaN devices at a time. The block driver stack should be overhauled to fix these problems; it should also be properly integrated into sysfs to make udev's /dev/by-uuid links work correctly. Regarding drivers, our focus should be on AHCI and NVMe, enabling us to boot on a wide range of hardware.

**Better support for real hardware.** In general, managarm can boot on real hardware just fine. Problems arise mostly due to bugs in USB host controller drivers and sometimes due to unimplemented AML opcodes in LAI. In 2020 we should complete the support of *all* AML opcodes in LAI (there are only a handful left). We should also test on more real machines and fix the remaining USB bugs.

## Trying out Managarm

If you want to try out Managarm, you can download a
[recent build (build 265)](https://pkgs.managarm.org/pinned/2019_12-build265-image.xz), uncompress the image and run it with the following command:

	qemu-system-x86_64 -enable-kvm -m 1024 -device piix3-usb-uhci -device usb-kbd -device usb-tablet -drive id=hdd,file=2019_12-build265-image,format=raw,if=none -device virtio-blk-pci,drive=hdd -vga virtio -debugcon stdio

<span style="font-size: 11pt;">Note that there are still some stability issues
(i.e., race conditions),
especially when booting into kmscon;
we expect to fix those in 2020.</span>

## Help wanted!

We are always looking for new contributors. If you want to get involved in the project,
you can search
[our GitHub issues](https://github.com/managarm/managarm/issues) for interesting tasks.
We would be particularly interested in AHCI and NVMe drivers, work on the networking
stack, porting Rust, or help with better GDB support.

## Acknowledgements

I want to thank all people who contributed to the Managarm project in 2019, including but not limited to:
Kacper Słomiński ([@qookei](https://github.com/qookei)),
Thomas Woertman ([@thomtl](https://github.com/thomtl)),
Matteo Semenzato ([@Matt8898](https://github.com/Matt8898)),
[@mintsuki](https://github.com/mintsuki),
Geert Custers ([@Geertiebear](https://github.com/Geertiebear)),
[@streaksu](https://github.com/streaksu),
toor (py19wjh@leeds.ac.uk),
[@Itay2805](https://github.com/Itay2805),
[@no92](https://github.com/no92)
and [@ArsenArsen](https://github.com/ArsenArsen).
