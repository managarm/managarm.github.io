---
layout: post
title: "Managarm: End of 2022 Update"
excerpt: "Managarm is a pragmatic microkernel-based OS with fully asynchronous I/O. We review our achievements of 2022 and set goals for 2023."
---
<span style="font-size: 11pt;">Post written by Dennis Bonke ([@Dennisbonke](https://github.com/Dennisbonke)) and [@no92](https://github.com/no92).
</span>

## Introduction

In this post, we will give an update on the progress that The Managarm Project made since our last update in August of 2022. For readers who are unfamilar with [Managarm](https://github.com/managarm/managarm): it is a microkernel-based OS that supports asynchronicity throughout the entire system while also providing compatibility with lots of Linux software. Feel free to try out Managarm (e.g., in a virtual machine). Our [README](https://github.com/managarm/managarm) contains a download link for a nightly image and instruction for trying out the system.

## Headline features
Major updates since our last post include support for memfd, support for PRIME, work into getting [mlibc](https://github.com/managarm/mlibc) working with Linux and the addition of Qt6 and GTK+ 3.

## New features in the kernel, POSIX and other servers

### PRIME ([PR](https://github.com/managarm/managarm/pull/419))
_PRed and written by [@no92](https://github.com/no92)_

Some of my work this year revolved around `drm`, [our implementation](https://github.com/managarm/managarm/tree/master/core/drm) of Linux's [Direct Rendering Manager](https://en.wikipedia.org/wiki/Direct_Rendering_Manager). This implementation works by supplying a common interface for drivers to plug into (via C++ classes, for example) and for userspace (via [DRM ioctls](https://dri.freedesktop.org/docs/drm/gpu/drm-uapi.html) or [libdrm](https://gitlab.freedesktop.org/mesa/drm)), which is linked into our graphics drivers as a shared library. Currently, support is limited to modesetting-related operations; our driver list is comprised of `bochs`, `plainfb`, `virtio`, `vmware` (and `intel`, which is old and doesn't currently use DRM).

However, some programs do make use of some features that are commonly available on Linux, but were not implemented in managarm yet. In particular, usage of the `DRM_IOCTL_PRIME_FD_TO_HANDLE` and `DRM_IOCTL_PRIME_HANDLE_TO_FD` ioctls was missing, but is a requirement for sway and an optional feature in mesa and weston.

This feature is necessary, as DRM handles for objects are only valid for the `open()`ed `/dev/dri/cardX` and thus cannot be shared across processes. To facilitate this, the above ioctls exist - they convert DRM handles to fds, which can then be passed across UNIX sockets - and can then be converted back to a DRM handle in a different process, to use it with DRM. For an example on how this can be used (albeit involving EGL), see [this blog post](https://blaztinn.gitlab.io/post/dmabuf-texture-sharing/).

As of now, no specific support within the graphics drivers was necessary. However, all the drivers either use VESA or (U)EFI (plainfb) or virtualized hardware (bochs, virtio, vmware), so on real hardware some driver support might become necessary to allow for this buffer sharing API to work with accelerated graphics.

In the end, this PR allowed for weston to be updated to version 10, and might come handy in the future if sway were to get ported, for example. Since the PR got merged, mesa also enables support for dmabuf - which in the end also relies on these ioctls.

### memfd ([PR](https://github.com/managarm/managarm/pull/418))
_PRed and written by [@no92](https://github.com/no92)_

`memfd_create` is a function on [Linux](https://manpages.ubuntu.com/manpages/bionic/man2/memfd_create.2.html) (and some [other OSes](https://www.freebsd.org/cgi/man.cgi?query=memfd_create&apropos=0&sektion=3&manpath=FreeBSD+13.1-RELEASE+and+Ports&arch=default&format=html)), that creates an anonymous memory-backed file and returns a file descriptor. This allows for using the default libc file manipulation functions with a memfd object, including passing it through UNIX sockets.

This year, we implemented this functionality in managarm and immediately noticed programs making use of it, including weston/wayland. Porting this functionality also allowed us to continue our ongoing quest of porting software to managarm.

## New ports and updates

### Qt6 ([PR](https://github.com/managarm/bootstrap-managarm/pull/175))
_PRed and written by [@Dennisbonke](https://github.com/Dennisbonke)_

After the success of GTK+ 2, we wanted to try tackle Qt5. Qt5 turned out to be a mess for cross-compilation, but with enough effort a hacky port was produced that was also working. It was, however, in no state to upstream. This was largely because of qmake, the build system in use by Qt5.

When Qt6 came around, they switched to CMake, which is a lot nicer for cross-compilation, and without much effort, a basic Qt6 port came along. While commandline applications are nice and all, what we really wanted was a fun game based on Qt6, so the choice was made to port [LibreMines](https://bollos00.github.io/LibreMines/), a minesweeper based on Qt6. This was done successfully as shown below.

![Libremines](/assets/2022-end-of-year-update/qt6-libremines.png)

### GTK+ 3 ([PR](https://github.com/managarm/bootstrap-managarm/pull/169))
_PRed and written by [@Dennisbonke](https://github.com/Dennisbonke)_

As mentioned in the [August 2022](https://managarm.org/2022/08/14/aug-update.html) update, we had GTK+ 2 working already. However, one major browser that was attempted required GTK+ 3, namely WebKitGTK. So, off we went on a challenge to get the required dependencies in. The big hassles proved to be GTK+ 3 itself, mainly due to runtime files missing, like GDK cache files, IM loader cache files, and the MIME database, which was fixed by adding post-install scripts and letting xbps configure all packages during boot, and DBus, which we still do not have fully working. The solution to DBus was quite simple, after we found out that DBus is only used for accessibility. We patched out the initialization calls for accessibility and after some locking fixes in mlibc we were able to run the `gtk3-demo` program successfully.

![gtk3-demo](/assets/2022-end-of-year-update/gtk3-demo.png)

## Other projects updates

### mlibc on Linux ([repo](https://github.com/no92/linux-mlibc))
While working towards running a proper browser on Managarm, we struggled to determine whether mlibc was at fault, or whether the Managarm kernel or its servers were wrong. To eliminate the first possibility and granting easy access to lots of nice debug tools, we started the [mlibc LFS](https://github.com/no92/linux-mlibc) project, named after Linux From Scratch. It is essentially Linux From Scratch, but uses mlibc instead of glibc.

We quickly got a small userspace running in a chroot, but it eventually grew out to something that we can boot up in a VM, a neccesity when testing out graphical programs. The major things that came from this still ongoing project are the fixes and additions to the mlibc Linux ABI, general usability under Linux, and an easier way to debug applications in the future. While we did get to a point where we could run WebKitGTK reasonably, this has since regressed on a nasty deadlock in pthreads, which we're working on fixing.

## What do we want to achieve in the next year?

**Finish porting xbps.** Unfortunately, despite improvements to the network stack, we did not finish porting `xbps` this year. We aim to have `xbps` in a state where an end user can just update the package repository and install packages by the end of next year, on supported network hardware.

**Upstream port patches.** We made a good initial effort this year to start upstreaming patches, and we managed to [land our targets](https://lists.gnu.org/archive/html/config-patches/2022-09/msg00001.html) in the GNU config repo, used by autotools. In the coming year, we would like to start upstreaming our targets to the respective toolchain packages (binutils, GCC, LLVM, libtool and rustc), and if successful, start upstreaming patches to other packages.

**Port a web browser and work towards self hosting.** While we made great improvements in this area, including the addition of the `links` browser mentioned in the August update, we aim to have one of the common browsers (Chrome, Firefox, WebKitGTK) running by the end of 2023. We also would like to get more common development programs in, like `git`. While we also had this goal at the end of 2019, it turned out that the given timeframe was too ambitious. However, due to the progress achieved in the past three years, like porting the X stack and GTK 3, and the addition of someone to the team who likes to work on porting software, we feel that we can achieve this goal in 2023.

**Add more drivers.** Specifically graphics and networking related. Together with a few [LAI](https://github.com/managarm/lai) and USB related quirks, a lack of drivers is the biggest issue for running on real hardware. Recently, work has been put into writing more network drivers, and some contributors are looking into adding modesetting drivers for Intel iGPUs. This is certainly an area where we would benefit from additional contributions, even just further testing on real hardware.

## Help wanted!

We are always looking for new contributors. If you want to get involved in the project, you can search our [GitHub issues](https://github.com/managarm/managarm/issues) for interesting tasks. Aside from contributions to the kernel and/or POSIX layer, we would be particularly interested in drivers for additional file systems (other than ext2), since Managarm currently lacks good drivers in these areas, and better debugging tools, like `ptrace` and related APIs. As always, just testing on real hardware to find quirks or test drivers is also an excellent way to help us!

If you want to get in touch with the team, you can find us on [Discord](https://discord.gg/7WB6Ur3) or on [irc.libera.chat](irc.libera.chat) in the #managarm channel. Also, if you want to support Managarm, please consider [donating to the project](https://github.com/sponsors/avdgrinten)!

## Acknowledgements
We thank all people who contributed to The Managarm Project in 2022, including but not limited to: Alexander van der Grinten ([@avdgrinten](https://github.com/avdgrinten)),  Kacper Słomiński ([@qookei](https://github.com/qookei)), Arsen Arsenović ([@ArsenArsen](https://github.com/ArsenArsen)), Geert Custers ([@Geertiebear](https://github.com/Geertiebear)), Matt Taylor ([@64](https://github.com/64)), Thomas Woertman ([@thomtl](https://github.com/thomtl)), [@mintsuki](https://github.com/mintsuki), [@streaksu](https://github.com/streaksu), [@czapek1337](https://github.com/czapek1337), [@ElectrodeYT](https://github.com/ElectrodeYT), [@cleanbaja](https://github.com/cleanbaja), [@borrrden](https://github.com/borrrden), [@jimx-](https://github.com/jimx-), [@ikbenlike](https://github.com/ikbenlike), [@Kyota-exe](https://github.com/Kyota-exe), [@wxwisiasdf](https://github.com/wxwisiasdf), [@AtieP](https://github.com/AtieP), [@piotrrak](https://github.com/piotrrak), [@ilobilo](https://github.com/ilobilo), [@davidtranhq](https://github.com/davidtranhq) and [@LittleCodingFox](https://github.com/LittleCodingFox).
