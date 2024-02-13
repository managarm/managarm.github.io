---
layout: post
title: "Managarm: End of 2023 Update"
excerpt: "Managarm is a pragmatic microkernel-based OS with fully asynchronous I/O. We review our achievements of 2023 and set goals for 2024."
---
<span style="font-size: 11pt;">Post written by Dennis Bonke ([@Dennisbonke](https://github.com/Dennisbonke)), Alexander Richards ([@ElectrodeYT](https://github.com/ElectrodeYT)) and [@no92](https://github.com/no92).
</span>

## Introduction

In this post, we will give an update on the progress that The Managarm Project made since our last update at the end of 2022. For readers who are unfamiliar with [Managarm](https://github.com/managarm/managarm): it is a microkernel-based OS that supports asynchronicity throughout the entire system while also providing compatibility with lots of Linux software. Feel free to try out Managarm (e.g., in a virtual machine). [Our README](https://github.com/managarm/managarm) contains a download link for a nightly image and instruction for trying out the system.

## Headline features
Major updates since our last post include significantly improved `/sys` and `/proc`, netlink support and the addition of WebKitGTK and Sway.

## New features in the kernel, POSIX and other servers

### sysfs support for USB and PCI devices

On Linux, information about devices is exposed under the `/sys/` tree. For instance, on most Intel chipsets, the integrated GPU can be found under `/sys/devices/pci0000:00/0000:00:02.0/`. In there, information is exposed in attribute files and child devices are placed in there, too.

A similar principle is employed to USB devices, except that USB-specific concepts like interfaces are also represented.

Exposing these devices over sysfs allowed us to port Linux' `pciutils` and `usbutils`. This means you can now run `lspci`, `lsusb` and `lsusb.py` and have it work as expected (i.e. just like on Linux).

### Port to x86 architecture

A while back, [@FedorLap2006](https://github.com/FedorLap2006) started porting mlibc to x86 (the 32-bit kind). In order to get this working, we had to implement some architecture-dependent bits and fix some code that was (implicitly) assuming 64-bit.
For starters, implementing the relocations used by the architecture in our dynamic linker (rtdl) lead to a rewrite how they are handled in all architectures. As many relocation types are shared between architectures, we decided to abstract them into generic relocation types that could now share their implementations. This also revealed some diverging implementations that have now been fixed.

Different ELF types (32 vs. 64 bit) are now also abstracted away, that is to say they are generic by using C++ namespace aliases that are configured depending on the target architecture.

In order to test the functionality of the port, we were able to use our pre-existing [test suite](https://github.com/managarm/mlibc/tree/master/tests). So after correctly cross-compiling mlibc and the tests, we can now run the tests using [qemu-user](https://www.qemu.org/docs/master/user/main.html). This allows us to run the suite on GitHub Actions and locally.

The tests helped us identify issues with the x86 port and fix them, as well as to detect and fix a bug in our setjmp implementation on x86_64 and aarch64.

### Support for `NETLINK_ROUTE` sockets

[@geertiebear](https://github.com/geertiebear)'s original netlink work was started due to our browser porting effort. On Linux, browsers request data about IP routing by using netlink sockets. As Managarm strives to also implement Linux APIs where needed, re-using this mechanism was preferred over introducing a Managarm-specific one or reusing one of another OS, like from the *BSDs.

Netlink messages are sent over sockets. Due to how Managarm works, this message is passed through to netserver if the protocol is specified to be `NETLINK_ROUTE`. In netserver, we can now dispatch the messages by reading the `nlmsghdr` and responding with the correct data.

Implementing this also allowed us to port [iproute2](https://wiki.linuxfoundation.org/networking/iproute2). This work allows us to use the `ip` utility to configure routes etc. just like on Linux. Also, DHCP clients like `dhcpcd` can use this to set up routes as determined using DHCP, if we port that in the future.

### New files in `procfs`

On Linux, a lot of information about the system is exposed under `/proc`. Certain software expects that information to be present there in order work correctly. This year we also focused on improving our procfs implementation, exposing various useful bits of information like the uptime of the system (`/proc/uptime`), the current working directory of the process (`/proc/[pid]/cwd`) and process status (`/proc/[pid]/stat`). This allows us to run `htop` to get a process listing. It also allows `neofetch` to display the uptime.

### ABI switch

mlibc has two big ABIs that a port can use, these being the [mlibc ABI](https://github.com/managarm/mlibc/tree/master/abis/mlibc) and the [Linux ABI](https://github.com/managarm/mlibc/tree/master/abis/linux). Managarm has from the very start always used the mlibc ABI, but some software expects the Linux ABI, by hardcoding it or asserting it at compile time. So we decided to move Managarm to the Linux ABI, which involved replacing some hardcoded constants in the kernel with their actual defines and recompiling the entire world against this new ABI. Not necessarily the most shiny work, but useful none the less.

### Support for C11 threads in mlibc

Porting [`foot`](https://codeberg.org/dnkl/foot/) revealed that mlibc didn't support C11 threads yet. But as it turns out, they are really similar to how pthreads work. This made the implementation easy - the code was adapted to be generic to the two interfaces and moved into mlibc internals - now C11 threads and pthreads mostly just call those with the correct parameters.

### Changes in the DRM subsystem

While working on GPU support it became apparent that eir, our pre-kernel, doesn't handle LFB addresses over 4 GiB well, as it is limited to 32-bit addresses. The [workaround](https://github.com/managarm/managarm/commit/9db754ded8d0fd3efffe156c899b9e6868010448) is to ignore the framebuffer in eir if that happens, as thor will pick it up and use it correctly regardless. As we had not previously encountered such a case before, it might be limited to GPU passthrough scenarios.

The virtio-gpu driver received a cleanup this year. As the Mesa support can be enabled at configure time, it would be interesting to implement the OS-side for 3D acceleration - a project for next year, maybe.

## New ports and updates

### Upstreaming our Mesa port

In our quest to reduce the amount of patches needed for packages, we also started to look at patches we could easily upstream. One of the ones that jumped out was Mesa: it is a core package, part of `weston-desktop`, and had patches which (after removal of some no longer needed patches) were extremely simple, and basically amounted to some `#ifdef`s and some minor changes to `meson.build`.

Upstreaming it went smoothly, however it was not done in time to get into version 23.3 of Mesa, meaning that, for the time being, patches are still necessary, however this will soon enough change.

### WebKitGTK

Almost three years ago, we set out on a quest to port a proper browser to Managarm. Our eyes fell upon [`WebKitGTK`](https://webkitgtk.org/) as it "only" required a full GTK stack and isn't as massive as `Firefox` or `Chromium`. While porting `WebKitGTK` itself was relatively painless, getting the included demonstration browser (called `MiniBrowser`) to work was a different beast.

We found out quite quickly that WebKit likes to read its own `mmap()` mappings, exposed under Linux via `/proc/self/maps`. Managarm did not implement this procfs node yet. This was resolved in 2022 due to some good work by [@Geertiebear](https://github.com/Geertiebear). As mentioned above, the next issue we ran into was netlink. With that merged, we ran into the final big issue for a basic browser. Some good old memory corruption somewhere in our C library was the cause of many headaches. A simple define that turned out to be wrong solved that issue and gave us a working browser for the basic items.

![WebKitGTK working](/assets/2023-end-of-year-update/browser-working.png)

However, we didn't want just a browser. We wanted Discord to run! Running Discord would allow us to access our official communications channel from within Managarm, and it would be a nice achievement of course. However, getting an official client to run involves getting electron up and running, which is basically chromium, which is not happening anytime soon, which made the web client the next best thing. Getting to that involved one last big step, the gstreamer stack. After a bunch of WebKit rebuilds to add more logging and looking at the browser console we figured out that we were just missing two plugin packs, namely `gst-plugins-good` and `gst-plugins-bad`. After we had those two ported, we were able to do proper conversations in Discord, finishing a quest we started almost three years ago, getting Discord to run on Managarm.

![Discord](/assets/2023-end-of-year-update/browser-discord.png)

### HexChat

When we started this project, Discord running was still a pipe dream. Luckily that didn't have to be a big issue, as we mirror our Discord channels to IRC, so an IRC client was needed. At the time, we only had GTK2 available so our choice landed on [`HexChat`](https://hexchat.github.io/). While the porting of `HexChat` went quite smoothly, it sat in review hell for well over a year, forgotten by everyone, including me. While cleaning up PRs, it was found, we dusted it off, repaired the bitrot, and got it working again. It was only fair to merge it this time around.

![HexChat](/assets/2023-end-of-year-update/hexchat.png)

### Sway

[Sway](https://swaywm.org/) is a tiling window manager like i3, but for Wayland. After a long period of work, we managed to make it work on Managarm.

![Sway](/assets/2023-end-of-year-update/sway.png)

Getting Managarm up to speed with the features that sway expects and uses took some effort, but are now upstream and will help us in the future. Among those is DRM object sharing via the PRIME ioctls, atomic modesetting and many, many bugfixes.

### htop

With `/proc/self` and `/proc/[pid]` existing in Managarm, having a program to list all processes and do something with that was an obvious addition. So we ported `htop`, which with a few patches was able to list all processes just fine. All in all a fine addition to our collection of ports.

### OpenTTD

While we already have a few games, none of them were very advanced. OpenTTD, in comparison, is a quite advanced simulation game, and is also open source, with open assets, meaning that it was quite a obvious choice to port.

The only thing that does not work with this port is multiplayer over networks, however as our networking subsystem improves we expect that this will begin working.

![OpenTTD](/assets/2023-end-of-year-update/openttd.png)

### gnuconfig

Last year, we upstreamed our targets into the [config.sub](https://lists.gnu.org/archive/html/config-patches/2022-09/msg00001.html) repo. This year, `autoconf v2.72` was released, which includes our targets! Newly generated autotools tarballs with this version (or newer) will require a lot less patching, ideally none at all, now that we're in upstream autoconf.

## What do we want to achieve in the next year?

**Finish porting xbps.** `xbps` works good enough to fetch and install packages, but removing still needs some love. Same goes for the terminal. We're missing what is called cooked mode, leading to some weird input quirks.

**Upstream port patches.** We have already done this with [Mesa](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/25818), however there are plenty of other packages with patches that are (somewhat) easily upstreamable, and doing this would reduce the workload required to maintain and update packages.

**Work towards self hosting.** We made some good steps this year in this direction, with a browser now being available. We're also working on getting Perl to properly run, which is currently hitting some locale issues. We hope to have more improvements to report for this item next year.

**Add more drivers.** Specifically graphics and networking related. Together with a few [lai](https://github.com/managarm/lai) quirks and USB related quirks, this is the biggest issue for running on real hardware, a lack of drivers. Recently, work has been put into writing more network drivers, and some members are looking into adding at least [modesetting drivers for Intel iGPUs](https://github.com/managarm/lil) and older AMD cards. There are some [limited prototypes](https://forum.osdev.org/viewtopic.php?f=1&t=12087&p=347133#p346606), but this is certainly an area we'd like help in, even if it is just testing on real hardware for quirks.

## Help wanted!

We are always looking for new contributors. If you want to get involved in the project, you can search our [GitHub issues](https://github.com/managarm/managarm/issues) for interesting tasks. Aside from contributions to the kernel and/or POSIX layer, we would be particularly interested in drivers for additional file systems (other than ext2), since Managarm currently lacks good drivers in these areas, and better debugging tools, like `ptrace` and related APIs. As always, just testing on real hardware to find quirks or test drivers is also an excellent way to help us!

If you want to get in touch with the team, you can find us on [Discord](https://discord.gg/7WB6Ur3) or on [irc.libera.chat](irc.libera.chat) in the #managarm channel. Also, if you want to support Managarm, please consider [donating to the project](https://github.com/sponsors/avdgrinten)!

## Acknowledgments
We thank all people who contributed to The Managarm Project in 2023, including but not limited to: Alexander van der Grinten ([@avdgrinten](https://github.com/avdgrinten)),  Kacper Słomiński ([@qookei](https://github.com/qookei)), Arsen Arsenović ([@ArsenArsen](https://github.com/ArsenArsen)), Geert Custers ([@Geertiebear](https://github.com/Geertiebear)), Matt Taylor ([@64](https://github.com/64)), Thomas Woertman ([@thomtl](https://github.com/thomtl)), [@mintsuki](https://github.com/mintsuki), [@streaksu](https://github.com/streaksu), [@Qwinci](https://github.com/Qwinci), [@borrrden](https://github.com/borrrden), [@moodyhunter](https://github.com/moodyhunter), [@netbsduser](https://github.com/netbsduser), [@fido2020](https://github.com/fido2020), 
[@Andy-Python-Programmer](https://github.com/Andy-Python-Programmer), [@RaidTheWeb](https://github.com/RaidTheWeb), [@ikbenlike](https://github.com/ikbenlike), [@FedorLap2006](https://github.com/FedorLap2006), [@lukflug](https://github.com/lukflug) and [@cleanbaja](https://github.com/cleanbaja).
