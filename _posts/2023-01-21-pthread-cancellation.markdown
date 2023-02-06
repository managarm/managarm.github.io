---
layout: post
title: "Implementing pthread cancellation in mlibc"
excerpt: "A dive into the implementation details of POSIX pthreads."
---
<span style="font-size: 11pt;">Post written by Geert Custers ([@Geertiebear](https://github.com/Geertiebear)).
</span>

## Introduction

A universal goal for any hobby OS to work towards is self-hosting. This is also
one of the goals Managarm hopes to achieve in the near future. Managarm is a pragmatic microkernel-based OS with fully asynchronous I/O (see our [GitHub repository](https://github.com/managarm/managarm) for more information). The userspace is powered using the highly portable [mlibc](https://github.com/managarm/mlibc) libc.

We would like to be able to compile Managarm natively following the same tools we currently use for cross-compiling. ``git`` is largely used throughout the Managarm compilation and bootstrapping process.  Managarm has a compiling ``git`` port. However, running almost any command hits an unimplemented mlibc function on ``pthread_setcancelstate()``. While this function just sets some flags for a thread, it is a member of a complex pthread feature set - thread cancellation. ``git`` would maybe have worked by making ``pthread_setcancelstate()`` a no-op, however we decided to implement most of the cancellation functionality, for future ports and feature-completeness.
## Requirements

This section summarizes the requirements and terminology for POSIX thread cancellation. More information can be found by reading the documentation for the respective functions in the [POSIX specification](https://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_cancel.html).

To summarize the feature set, cancellation allows threads to be "canceled". This is akin to terminating the thread, but it has different semantics than a plain thread exit. Most importantly, there are different types of cancellation. While a ``pthread_kill()`` or ``pthread_exit()`` is immediate, cancellation is split up into two categories:

* **Asynchronous**: the thread can be interrupted at any time, and, once canceled, the cancellation procedure can start at any time (it is usually executed immediately after a cancellation request).
* **Deferred**: when a thread is canceled using ``pthread_cancel()``, the cancellation procedure is postponed until a "cancellation point" is reached by the canceled thread, or the thread calls ``pthread_testcancel()`` itself.

Cancellation point is an important term to remember. There are several of them in the libc, defined by the [POSIX spec](https://pubs.opengroup.org/onlinepubs/009695399/functions/xsh_chap02_09.html). A thread can set its cancellation type using ``pthread_setcanceltype()``. By default a thread is in deferred cancellation.

On top of these different types of cancellation, cancellation can also be enabled/disabled using ``pthread_setcancelstate()``. Disabled cancellation means that a cancellation request will not be acted on until cancellation is enabled again.

A thread can submit a cancellation request using ``pthread_cancel()``; its signature is as follows.
```c
int pthread_cancel(pthread_t thread);
```

A thread can check if a cancellation request has been queued using ``pthread_testcancel()``. This is semantically equivalent to a custom cancellation point.
## Implementation

### First steps

From just reading the POSIX specification, it can be quite difficult to imagine the exact semantics of implementation. It is often much easier to first study other libc's, which already have the required functionality implemented. Therefore, as a first step, the implementation of [glibc](https://www.gnu.org/software/libc/) and [musl](https://musl.libc.org/) were examined.

This revealed the following key implementation points:
* Asynchronous cancellation is implemented using signals by both musl and glibc. A call to ``pthread_cancel()`` results in SIGCANCEL being sent to the target thread.
* Glibc and musl both use flags in their ``pthread_t`` struct to keep track of cancellation state. Glibc uses a bitfield, while musl uses seperate fields.
* Musl has a curious interaction between the canceling code and their syscall routines, the details of which will be discussed in a following section.

With this knowledge, we can begin implementing pthread cancellation.

### Cancellation state functions
``pthread_secancelstate()`` and ``pthread_setcanceltype()`` are quite easy functions to implement. They boil down to atomically setting some flags. ``pthread_setcancelstate()`` is implemented [here](https://github.com/managarm/mlibc/blob/5e6cfd7e240c9482552bb482653ee2a3806e0add/options/posix/generic/pthread-stubs.cpp#L315), and ``pthread_setcanceltype()`` is implemented [here](https://github.com/managarm/mlibc/blob/5e6cfd7e240c9482552bb482653ee2a3806e0add/options/posix/generic/pthread-stubs.cpp#L280).

There are two notable implementation details. First, these functions can cause a cancellation if the thread has received a cancellation request _and_ asynchronous cancellation is enabled by the call to the function. For example, while the cancellation state of a thread was set to deferred, it received a cancellation request. This doesn't cause anything to happen immediately. However, if the thread now calls ``pthread_setcanceltype()`` to set itself to asynchronous cancellation, it will get cancelled in that call.

### Cancellation semantics

It was at this point in the implementation process that one of our team members shared [this](https://lwn.net/Articles/683118/) LWN article. It is titled "This is why we can't have safe cancellation points", and provides a very good explanation of the problem, a problem that we (the Managarm team) didn't spot when we first started working on cancellation.

The entire problem stems from a couple paragraphs in the POSIX manual, describing the semantics of cancellation, an extract shown below (taken from POSIX 2001).

> Whenever a thread has cancelability enabled and a cancellation request has been made with that thread as the target, and the thread then calls any function that is a cancellation point (such as pthread_testcancel() or read()), the cancellation request shall be acted upon before the function returns. If a thread has cancelability enabled and a cancellation request is made with the thread as a target while the thread is suspended at a cancellation point, the thread shall be awakened and the cancellation request shall be acted upon. However, if the thread is suspended at a cancellation point and the event for which it is waiting occurs before the cancellation request is acted upon, it is unspecified whether the cancellation request is acted upon or whether the cancellation request remains pending and the thread resumes normal execution.

The paragraph describes the behaviour of deferred cancellation. If a thread gets deferred canceled, and it calls into a cancellation point, then the cancellation is acted upon before the function returns.

The situation is more complicated when a thread is *suspended* inside a cancellation point. This can happen when a thread calls into, and is blocked by, ``read()`` or ``accept()`` for example. POSIX specifies that the thread should be woken up, and the cancellation request acted upon. It also specifies that this should happen with the same semantics as if this were a single-threaded program and a syscall was interrupted by a signal and returned EINTR.

As an example, let's investigate ``accept()``. Suppose there is a thread which listens for connections, and is blocking inside an ``accept()`` call. Now it's cancelled. What should happen? If mlibc unconditionally peformed a cancellation request, then this would cause resource leaks. ``accept()`` can allocate a file descriptor. If a file descriptor gets allocated, and the thread is cancelled immediately after returning from ``accept()``, then the file descriptor is lost.

What _should_ happen is mlibc determines whether or not the syscall produced _side-effects_. If the syscall produced side-effects, mlibc should defer the cancellation request, and let the thread clean up its side-effects first. However, if no side-effects were caused, then the thread can be canceled. A mechanism for determining when we are allowed to cancel a thread inside a cancellation point is described in the next section.

### Cancelling

As said before, to cancel a thread ``pthread_cancel`` can be used. The logic for this function is quite simple. The cancel trigger bit in the TCB[^2] is set atomically, and if cancellation is enabled, a SIGCANCEL is delivered to the target thread.

The cancel logic is performed by ``sigcancel_handler()``, the signal handler for SIGCANCEL. A [static constructor](https://github.com/managarm/mlibc/blob/a6cbfbaca35fbe24a56aae720ae67eb45925d8ce/options/posix/generic/pthread-stubs.cpp#L260-L273) is used to install the signal handler at startup. A summarized version of the function is shown below.

```cpp
ucontext_t *uctx = static_cast<ucontext_t*>(ucontext);
// The function could be called from other signals, or from another
// process, in which case we should do nothing.
if (signal != SIGCANCEL || info->si_pid != getpid() ||
            info->si_code != SI_TKILL)
    return;

int old_value = mlib::get_current_tcb()->cancelBits;

if (!(old_value & tcbCancelAsyncBit) &&
            mlibc::sys_before_cancellable_syscall && !mlibc::sys_before_cancellable_syscall(uctx))
        return;

// ...
// Set cancel trigger and canceling bit atomically,
// then call __mlibc_do_cancel()
//
```

There are two notable conditions in this function. First the function checks if the cancellation actually came from the current process, else other programs could theoretically shoot down arbitrary threads.

Secondly, the function checks if deferred cancellation is enabled. This is to handle the issue described in the previous section. First it checks if the ``sys_before_cancellable_syscall`` sysdep [^1] is available. If it is, then it calls that with the thread's ``ucontext``, which is a struct containing the current register state of the thread.

On the linux sysdeps, ``sys_before_cancellable_syscall`` is implemented as follows (code can be found [here](https://github.com/managarm/mlibc/blob/f2c6b8cd7fe67b89d12b23ba0a82e174ba2b7c0a/sysdeps/linux/generic/sysdeps.cpp#L365-L370)).

```cpp
int sys_before_cancellable_syscall(ucontext_t *uct) {
	auto pc = reinterpret_cast<void*>(uct->uc_mcontext.rip);
	if (pc < __mlibc_syscall_begin || pc > __mlibc_syscall_end)
		return 0;
	return 1;
}
```

The function checks if the program counter of the thread is between the symbols ``__mlibc_syscall_begin`` and ``__mlibc_syscall_end``. These symbols are defined in the assembly routine for performing a syscall. On x86 that is as follows (code can be found [here](https://github.com/managarm/mlibc/blob/f2c6b8cd7fe67b89d12b23ba0a82e174ba2b7c0a/sysdeps/linux/x86_64/cp_syscall.S#L16-L23)).

```x86asm
__mlibc_syscall_begin:
	/* tcbCancelEnableBit && tcbCancelTriggerBit */
	and $((1 << 0) | (1 << 2)), %r11
	cmp $((1 << 0) | (1 << 2)), %r11
	je cancel
	syscall
__mlibc_syscall_end:
	ret
```

The assembly routine checks if a thread should be canceled right before it performs the syscall. The labels allow mlibc to detect whether or not the syscall has been completed. If the interruption happens between the labels, then the syscall hasn't happened yet, no side effects have been produced, and the signal handler can continue as normal, which results in the cancellation of the thread.

If however, the program counter is outside those labels, then mlibc cannot determine if the syscall has produced side-effects, so the cancellation is not performed within the signal handler, and the thread continues execution.

Lastly, there is one more way a thread can get canceled, which is right _after_ a syscall has been performed. Below is an extract from mlibc's linux [syscall wrapper](https://github.com/managarm/mlibc/blob/ee4b75216b6b3276f89bdef168402a6c6bde0349/sysdeps/linux/generic/cxx-syscall.hpp#L163-L171).

```cpp
auto result = do_nargs_cp_syscall(sc, sc_cast(args)...);
if (int e = sc_error(result); e) {
    auto tcb = get_current_tcb();
    if (tcb_cancelled(tcb->cancelBits) && e == EINTR) {
        __mlibc_do_cancel();
        __builtin_unreachable();
    }
}
return result;
```

If the syscall returns EINTR, it means that the syscall hasn't produced any side-effects, and so mlibc can cancel the thread if a cancellation is pending. There is one exception, which is for ``close(2)``, where EINTR doesn't necessarily mean that no side-effects haven't been produced. The reason for this particularity is
the following line in the POSIX1.2008 standard:

```
If close() is interrupted by a signal that is to be caught, it shall return -1 with errno set to EINTR and the state of fildes is unspecified.
```
This means that when ``close(2)`` returns EINTR, there is no way to know if it
has produced side effects or not. Actually, it is platform dependent! For
example, on Linux, EINTR means that the file descriptor has been closed.
However, on HP-UX, EINTR means that the file descriptor has *not* been closed.
Currently the special case of ``close(2)`` is not handled in mlibc.

[^1]: sysdeps are mlibc's terminology for describing functions that should be implemented by the target platform. This can be a syscall (e.g. ``read(2)`` ), or functions that rely on specific details of the platform's syscall mechanisms.
[^2]: The Thread Control Block, a structure allocated by the libc to keep track of various variables related to the thread. The TCB in mlibc can be found [here](https://github.com/managarm/mlibc/blob/bf38c0ec9682fa542908b7a659df67eb07f19966/options/internal/include/mlibc/tcb.hpp#L82).

#### On glibc's implementation

The syscall implementation that is presented in this section follows the same one that musl implements. It is their solution to the problem posed by the semantics defined by POSIX. However, glibc took another approach, which is simpler to implement but also leaves it vulnerable to resource leaks.

Instead of always delivering a SIGCANCEL signal on ``pthread_cancel()``, glibc only delivers the signal when asynchronous cancellation is enabled. To support cancelling while blocked in a syscall, glibc does the following on a syscall.

```c
ENABLE_ASYNC_CANCEL();
ret = DO_SYSCALL(...);
RESTORE_OLD_ASYNC_CANCEL();
return ret;
```

The problem with this solution is that there is a window between a syscall completing, and the cancellation type being restored. So, if a thread has deferred cancellation, and a syscall completes that allocated some resources, those resources will be unexpectedly leaked. [This](http://ewontfix.com/2/) EWONTFIX article describes the issue in more detail.
An immediate question one might ask is: if musl (and mlibc too) has a solution
to this problem, why does glibc not use the same approach? Unfortunately, the
solution used by musl and mlibc has a trade-off. To speed up some
commonly used syscalls, the linux kernel maps a dynamic library named ``vdso``
into the address space of every executable. Using this library some syscalls, like
``gettimeofday(2)``, can be executed in userspace, avoiding an expensive context
switch. In case a full syscall does need to be performed, the vDSO can also be
useful. Different CPU platforms have different instructions for executing a
syscall. On x86 Linux one can universally use ``int $80``, but on newer
processors one can also use ``syscall`` or ``sysenter``. Which instruction
performs best depends on the platform. Linux's vDSO dynamically uses the best
interrupt routine available on the platform, saving the C libary the trouble.
Going back to the original problem, in its current form it is impossible to
apply musl's trick to solve the above race condition when using vDSO. This is
because the vDSO does not export the ``__*_syscall_begin`` and
``__*_syscall_end`` markers that musl and mlibc use. A patch was [proposed](https://lkml.org/lkml/2016/3/8/1105) to fix
this issue, but ended up being rejected by the Linux kernel developers. Hence to
fix the issue, performance will be sacrificed. A trade-off the glibc developers
were not willing to make.

## Testing

We added a ``pthread_cancel()`` test, which covers the following cases. 

* Asynchronous cancellation during a ``sleep(1)`` call.
* Testing cancellation points (using ``sleep(1)``) as an example.
* Testing ``pthread_testcancel()``.

More might be added in the future if/when bugs are discovered. The full test can be found [here](https://github.com/managarm/mlibc/blob/master/tests/posix/pthread_cancel.c).

## Conclusion

As it turned out, this particular feature of POSIX pthreads was quite tricky to implement correctly,
and the broader C library community does not have a definitive answer for how to
correctly approach the problem. Fortunately, with support of various resources,
we were able to wrap our heads around the issue, and come to a solution that
fits the needs of mlibc.

Now that this libc feature is finished, it's one more thing that can be checked off the self-hosting TODO list. The Managarm team can always be found for a chat on either [Discord](https://discord.gg/7WB6Ur3) or on IRC on ``#managarm`` on ``irc.libera.chat``.
