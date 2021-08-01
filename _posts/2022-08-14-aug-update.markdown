---
layout: post
title: "Managarm: August 2022 Update"
excerpt:
---
<span style="font-size: 11pt;">Post by
Arsen Arsenović ([@ArsenArsen](https://github.com/ArsenArsen)),
Dennis Bonke ([@Dennisbonke](https://github.com/Dennisbonke)),
Geert Custers ([@geertiebear](https://github.com/geertiebear)),
Alexander van der Grinten ([@avdgrinten](https://github.com/avdgrinten)),
Matt Taylor ([@64](https://github.com/64))
and Kacper Słomiński ([@qookei](https://github.com/qookei)).
</span>

## Introduction

In this post, we will give an update on the progress that the Managarm
operating system made in the last two and a half years, it has been quite a ride!
For readers who are unfamilar with [Managarm](https://github.com/managarm/managarm):
it is a microkernel-based OS
that supports asynchronicity throughout the entire system while also providing
compatibility with lots of Linux software.

Feel free to **try out Managarm** (e.g., in a virtual machine).
[Our README](https://github.com/managarm/managarm) contains a download link for
a nightly image and instruction for trying out the system.

Since our [2019 status update](https://managarm.org/2019/12/24/end-of-year.html),
Managarm has been visible at various places over the internet.
Most prominently,
Managarm was present at **CppCon 2020** which was held virtually.
Our talk focuses on the use of modern
C++20 within Managarm (including asynchronicity via coroutines);
[a recording can be found on YouTube](https://youtu.be/BzwpdOpNFpQ):

<div style="width: 560px; margin: 0 auto;"><iframe width="560" height="315" src="https://www.youtube.com/embed/BzwpdOpNFpQ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></div>
<br>

Additionally, the YouTube channel
[Systems with JT](https://www.youtube.com/channel/UCrW38UKhlPoApXiuKNghuig) featured
our project. [Check out the video](https://youtu.be/qxrC7yJ7CSA) for a more
hands-on walk through the system:

<div style="width: 560px; margin: 0 auto;"><iframe width="560" height="315" src="https://www.youtube.com/embed/qxrC7yJ7CSA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></div>
<br>

We've also been hosted at **FOSDEM 2022**, where Alexander went over the basics of IPC and the general architecture of the system.
<div style="width: 560px; margin: 0 auto;"><iframe width="560" height="315" src="https://video.fosdem.org/2022/D.microkernel/agrinten.mp4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></div>
<br>

## Major Updates

Major updates since our last post include a 64-bit ARM port, the start of the RISC-V port, support for the [Rust](https://www.rust-lang.org/) programming language in user space, and support for the [xbps package manager](https://github.com/void-linux/xbps).

Another important addition is our
[handbook](https://docs.managarm.org/handbook) that describes parts of the
system in detail (targetting both users and developers). The handbook is still
quite incomplete but regular updates can be expected in the future.

### AArch64 Port

Since July 2020, we have been working on the 64-bit ARM
(= [AArch64](https://en.wikipedia.org/wiki/AArch64)) port of Managarm. In its
current form, the port is able to boot into Weston and kmscon and run some
command-line programs in QEMU. However, a large part of our software repertoire
is still untested, and the port in general is still work in progress. We have
an upcoming blog post that will be going into more detail about the porting
process, implementation details, and the obstacles we faced along the way.

### Rust Support in User Space

Over the last 3 months, [@64](https://github.com/64) has been working to bring Rust to Managarm. So far, this has consisted of:
* Adding information about our `x86_64-unknown-managarm-system` target to [rustc](https://github.com/rust-lang/rust) (so that it can link with our LLVM and such). This initial work was done by [@avdgrinten](https://github.com/avdgrinten).
* Adding bindings for [mlibc](https://github.com/managarm/mlibc) in Rust's [libc](https://github.com/rust-lang/libc) crate ([managarm/bootstrap-managarm#96](https://github.com/managarm/bootstrap-managarm/pull/96/files#diff-3d13f40b9e85b21978241de17ebb802857f04bf1af5b1f0d80f1c76f50d040d2)).
* Patching rust's standard library to support Managarm, and stubbing out functionality that we don't implement yet ([managarm/bootstrap-managarm#96](https://github.com/managarm/bootstrap-managarm/pull/96/files#diff-b086cf11aa261f991d7eb66b8164907ad9ee5bc4165cbc1739d65fcd2607be78)).
* Integrating [cargo](https://github.com/rust-lang/cargo) with our build system so that we can cross-compile crates for Managarm ([managarm/xbstrap#48](https://github.com/managarm/xbstrap/pull/48)).
* Porting [ripgrep](https://github.com/BurntSushi/ripgrep) and [exa](https://github.com/ogham/exa) ([managarm/bootstrap-managarm#102](https://github.com/managarm/bootstrap-managarm/pull/102)).

There is still work to be done before all of Rust's standard library is fully supported within Managarm. Notably, thread creation hits unimplemented code paths within [mlibc](https://github.com/managarm/mlibc), but this is our next priority.

In the long term, we would like to support Rust drivers for Managarm. This will involve adding support for Rust in our IPC protocol codegen tool [bragi](https://github.com/managarm/bragi), and writing a Rust wrapper for our asynchronous syscall API.

### Build Servers and xbps Packages

Two years ago, we deployed [xbbs](https://github.com/managarm/xbbs), a distributed build server specifically crafted for [xbstrap](https://github.com/managarm/xbstrap) and [xbps](https://github.com/void-linux/xbps). It allows us to efficiently and effectively split building a managarm distribution across a handful of servers and to only update parts of the distribution we changed. This has allowed us to come closer to our goal of porting and utilizing xbps for managing system packages in our distribution images. You can use it today to get packages built by us on https://builds.managarm.org/ but you cannot use it in the system itself with xbps just yet, although work is ongoing to fix that.

The goals of this "subproject" include:

- Live building packages as updates are pushed and checking their validity (finding soname changes and similar breaking issues)
- Allowing for spotting errors caused by inconsistent/unclean build environments
- Centralizing the tracking of reproducibility of packages
- Increasing the speed of new packages and updates reaching users and reducing the chance of introducing new errors, by spotting them early and notifying maintainers

## New features in the kernel, POSIX and other servers

**Hardware virtualization (VMX) support.** Shortly after
[our 2019 status update](https://managarm.org/2019/12/24/end-of-year.html),
Managarm received support for hardware virtualization on Intel
CPUs (using Intel's VMX). We plan to extend virtualization support to AMD CPUs
as well (which implement AMD SVM instead of VMX). In the long term, this will
allow us to support the KVM interface, such that we can run hypervisors like
QEMU-KVM natively on Managarm.

**pthreads.** As mentioned in
[our "Porting Software to Managarm" post](https://managarm.org/2020/04/05/porting-software.html),
in 2020 we implemented pthread thread creation and other related functions. mlibc has also
gotten a pthread cancellation implementation, and we have an upcoming blog post
going into detail about it.

**A new IDL Compiler: bragi.** In February 2020, we started work on our own interface description language, bragi. The aim is to replace all of our current protobuf usage with bragi. Although it's not yet fully feature-complete (we still want/need to add features like variants or inheritance), it has been enough to allow us to refactor some of our IPC protocols (namely, the POSIX, hardware, and filesystem protocols) to bragi, and begin implementing new protocols (like the ostrace one) in it.

**New Drivers: AHCI and NVMe and Storage Improvements.** Managarm's block driver stack received significant updates in 2021. We now have drivers for the two most important modern block device controllers AHCI and NVMe. The AHCI driver was written by Matt Taylor ([@64](https://github.com/64)), while the NVMe driver was contributed by Jin Xue ([@Jimx-](https://github.com/Jimx-)). Furthermore, due to a recent PR by Geert Custers ([@geertiebear](https://github.com/geertiebear)), we can now identify partitions by their UUIDs. This feature will make identifying the boot device more robust in the future and is especially important when running Managarm on physical hardware (and not in a virtual machine).

**Networking Improvements.** Our networking stack ("netserver") can now connect to TCP servers over IPv4. It supports basic TCP features; however, the server side of the TCP 3-way handshake and path MTU discovery is not implemented yet. Once these gaps are filled, we will have a mostly complete IPv4 stack.

## New Ports and Port Updates

In the last two years, we received a lot of new and sometimes updated ports, our collection contains over 200 ports now! A lot of the ports are various nice to have things, such as common unix utilities like `grep`, `sed`, `findutils` and `gawk`, development tools like `python`, `make` and `patch` and we got enough of the X11 stack ported that we can run XWayland and several X based apps like `xclock` and `gtklife`. Another noteworthy thing to mention here is the addition of a new bootloader called [limine](https://limine-bootloader.org/), which we now use by default (although `grub` is still supported at this time and there are no plans to remove that support) and the addition of a stripped down `util-linux` port, which includes useful utilities like `mount` and `losetup`. A final mention goes to some Rust ports as mentioned above.
As a blog post without images would be boring, here are some screenshots, first off is Managarm running `python`.
![python](/assets/2022-aug-update/python.png)
After that, we have `xclock`.
![xclock](/assets/2022-aug-update/xclock.png)
And finally we have `exa` running.
![exa](/assets/2022-aug-update/exa.png)

### The road to X11
The road to X11 was quite a bumpy one, with several issues that required digging deep in the X11 codebase to figure out. In the end, the biggest issues were a nasty epoll bug and the usage of abstract unix sockets, that weren't implemented. With that fixed (and a small amount of stubbing of shared memory functions in mlibc) we were able to run the `gtk-demo` demo program succesfully, paving the way for various other X based programs.
![gtk2](/assets/2022-aug-update/gtk2.png)

Outside of XWayland, work is ongoing to also run the classic X.Org server, using its modesetting drivers.

### QEMU
Most of the pieces necessary for QEMU have already been in place, with the exception of `sigaltstack` and partial `munmap`/`mmap`/`mprotect` support. With both of these missing features implemented, we can run QEMU on Managarm, bringing us one step closer to being self-hosting.

![qemu](/assets/2022-aug-update/qemu.png)

### DOOM
Until recently, we didn't have any DOOM port, mainly because we couldn't decide on which source port to use. We eventually decided upon dsda-doom, giving us a modern, yet vanilla DOOM experience, with extra speedrunning features as a bonus.

![doom](/assets/2022-aug-update/doom.png)

## What do we want to achieve in 2022?

**Finish porting the package manager.** In the past months, considerable work went into porting a package manager. While the general infrastructure, both inside Managarm and outside in terms of an repository, are set up, some more work is required to actually get `xbps` to function properly. We aim to implement the missing functionality soon.

**Polish the port collection.** Currently, we have a lot of ports that work at least partially, but some use some pretty ugly hacks to get to that state. We should strive to get the quality of some of those ports up by implementing the proper functionality and not relying on hacks. This mostly means implementing missing functionality and testing for correctness. We also plan to start upstreaming support patches so that we can remove some patches from the collection.

**Complete TTY subsystem.** We currently lack or incorrectly implement many TTY subsystem features (sessions and signals, process groups, et cetera), which are quite necessary for many kinds of programs as well as day-to-day life using the system. This goal is also accompanied by finishing up Unix process credentials and signals. Help is *definitely* wanted with this problem.

Some remaining goals from last time include porting more software, especially to self-host, improving the blockdev stack, making the system generally more stable and improving the netstack, especially the TCP implementation; and, of course, there is always more hardware to improve support for.

## Help Wanted!

We are always looking for new contributors. If you want to get involved in the
project, you can search our
[GitHub issues](https://github.com/managarm/managarm/issues) for interesting tasks.
Aside from contributions to the kernel and/or POSIX layer,
we would be particularly interested in network drivers
and drivers for additional file systems (other than ext2),
since Managarm currently lacks good drivers in these areas.

If you want to get in touch with the team, you can find us on
[Discord](https://discord.gg/7WB6Ur3) or on irc.libera.chat in the #managarm
channel.
Also, if you want to support Managarm, please consider
[donating to the project](https://github.com/sponsors/avdgrinten)!

## Acknowledgements

We thank all contributors to the [Managarm](https://github.com/managarm/managarm),
[mlibc](https://github.com/managarm/mlibc) and related projects.
