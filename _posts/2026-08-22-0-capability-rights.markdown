---
layout: post
title: "Fine-grained kernel-level capability rights for Managarm"
---
<span style="font-size: 11pt;">Post written by Alexander van der Grinten ([@avdgrinten](https://github.com/avdgrinten)).</span>

Managarm recently gained support for kernel-level capability rights. In this blog post, we explore the motivation for this change, how the new design works and what the state of access control is in Managarm.

## Motivation

Managarm exposes kernel objects to userspace through the use of capabilities. In particular, a thread can only access a kernel object if it owns a capability that grants access to the object. The capabilities that a thread owns are held in a `Universe` data structure and addressed by a 64-bit handle. This is similar to how resource access works in other systems as well: for example, UNIX file descriptors (FDs) are handles to open files and owning an FD lets a program access the corresponding open file. However, note that FDs are not true capabilities since UNIX-like operating systems often perform ambient access control checks on top -- for example, file operations such as `fchown()` require root even after the file has already been opened.

Perhaps surprisingly, Managarm's capabilities carried no permission bits so far. Owning a capability was binary -- a program was either allowed to execute _all_ possible operations on an object or it had no access at all. This was fine for POSIX applications as POSIX applications on Managarm never create objects that they share with trusted servers. Instead, they are granted all IPC objects that they need to access by their corresponding POSIX server -- and these IPC objects never expose any sensitive operations. However, for server-to-server communication, the existing model unnecessarily required trusting remote servers. For example, a server sending a memory object to another server had to trust the receiver to not truncate the object's size to zero. Likewise, when sharing an IPC stream with multiple servers, each server was able to close the shared stream.

In a series of recent PRs (in Managarm's repository: [#1503](https://github.com/managarm/managarm/pull/1503), [#1506](https://github.com/managarm/managarm/pull/1506), [#1516](https://github.com/managarm/managarm/pull/1516)), we added rights (i.e., permission bits) to capabilities to remove the need for servers to trust their peers.

## Design

### Rights bits on capabilities

In Managarm's new rights design, a set (i.e., bitmask) of rights is attached to each capability. We currently define 14 such rights (out of 32 bits that are available in the implementation):

- `Grant` / `Take` control whether capabilities can be attached to or taken from universes.
- `Read` / `Write` / `Execute` control access to memory objects and thread register states.
- `Invoke` controls the ability to do IPC.
- `Assign` controls the ability to construct new objects.
- `Derive` / `Provision` / `Pin` / `Fence` control specific operations on memory objects.
- `Signal` / `Wait` gate event-related syscalls.
- `Manage` controls the ability to perform potentially destructive operations (such as truncating memory objects or shutting down IPC channels).

Note that the rights are per-capability and not per-object (i.e., two threads can have access to the same memory object but different rights). Rights are also monotonically decreasing by design: a program can drop rights that it has but it cannot gain new ones (without being handed new capabilities by external servers).

Our design of capability rights is similar to the design in other microkernels. For example, seL4 defines four rights bits that are overloaded depending on the capability type (for example, a read right on an seL4 notification gates the ability to wait on the notification). Zircon's vocabulary of rights is overall quite similar to ours; however, the exact kernel objects that Zircon has are different and Zircon gates capability duplication and transfer behind rights. Managarm intentionally does not gate transfer and duplication; we may add an explicit mechanism for capability revocation in the future (potentially restricted to the cases that would benefit our servers).

### Rights transfer through IPC

Managarm has two pre-existing ways to copy capabilities:

1. A `helTransferDescriptor()` syscall allows transferring capabilities between the calling thread's universe and other universes (provided that the thread has a capability that allows it to address a remote universe).
2. The `PushDescriptor` (on the sender side) and `PullDescriptor` (on the receiver side) message actions allow the transfer of capabilities as part of IPC.

We enhanced both mechanisms to integrate with rights:
- `PushDescriptor` now takes a bitmask `exposedRights` of rights that it wants to send. If we denote the rights of the capability on the sender side by `originalRights`, then the rights that the receiver gets are `newRights = originalRights & exposedRights`. Hence, it is possible to send all rights that the caller has by specifying an all-ones bitmask for `PushDescriptor`.
- `PullDescriptor` now takes a mask `requiredRights` of rights that the receiver expects. The transfer fails if `(newRights & requiredRights) != requiredRights`. This enables programs to fail fast in case a server does not send the rights that they expect. A program that does not immediately care about the rights that it receives can specify zero for `requiredRights`.
- `helTransferDescriptor()` now takes both `exposedRights` and `requiredRights` with the same semantics as for `PushDescriptor` and `PullDescriptor`.

## Status and Future Work

As of today, the entire rights system has been implemented. Inside the kernel, capability accessor functions that do not check rights have been replaced by ones that check rights. The old accessors are deleted, ensuring that all code paths perform checks. In userspace, servers have been updated to only share a minimal set of rights with their peers.

Overall, the transition has been smooth. We encountered a few places where server-to-server shared memory that was intended to be read-only was in fact mapped as read-write (one of these cases was a shared memory page storing the boot time to real time translation that is published by our `clocktracker` server). With the new rights design, this turned into a hard error and it was fixed.

The new design will also come in handy for related tasks that we want to tackle in the near future, such as adding access control to our object discovery bus and the removal of all remaining ambient authority in the system.

## Supported by NLnet

[This project](https://nlnet.nl/project/Managarm/) was funded through the [NGI0 Commons Fund](https://nlnet.nl/commonsfund/), a fund established by [NLnet](https://nlnet.nl) with financial support from the European Commission’s Next Generation Internet programme, under the aegis of DG Communications Networks, Content and Technology under grant agreement No 101135429. Additional funding is made available by the Swiss State Secretariat for Education, Research and Innovation (SERI).

<img src="/assets/2026-anon-swap/nlnet.svg" alt="NLnet logo" width="200" style="display: inline-block; margin-right: 10px; margin-bottom: 10px;" />
<img src="/assets/2026-anon-swap/NGI0Commons.svg" alt="NGI0 Commons logo" width="200" style="display: inline-block; margin-bottom: 10px;" />

## Talk to us

We welcome contributions to Managarm! If you want to help out, feel free to look at [our issue tracker](https://github.com/managarm/managarm/issues), or chat with us on [Discord](https://discord.gg/7WB6Ur3) or IRC in `#managarm` on `irc.libera.chat` about ideas.
