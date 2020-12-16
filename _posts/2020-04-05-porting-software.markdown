---
layout: post
title: Porting Software to Managarm
excerpt: A general overview of porting Mednafen and other software to Managarm.
---
<span style="font-size: 11pt;">Post by Kacper Słomiński
([@qookei](https://github.com/qookei)) and Alexander van der Grinten
([@avdgrinten](https://github.com/avdgrinten))
</span>

## Introduction
Porting existing software to new operating systems is not always an easy task. Yet, the general procedure of porting different software packages is often surprisingly similar. In this blog post, we walk through the process of porting the multi-system emulator [Mednafen](https://mednafen.github.io/) to the Managarm operating system. Managarm is a pragmatic microkernel-based OS with fully asynchronous I/O (see our [GitHub repository](https://github.com/managarm/managarm) for more information). While we discuss the specific example of Mednafen, this post can be seen as a general recipe for porting software to Managarm.

![Mednafen](/assets/2020-porting-software/mednafen-1.png)

**Overview.** Porting a package commonly consists of the following steps:
* Setting up the package's build system for (cross-)compilation to the new OS.
* Building the package and fixing any problems arising in the package's `./configure` script or during compilation.
* Implementing new OS (and libc) functionality to make the package work.

## Preparations and build system
The first step when adding any package is locating the source code. This is usually not too hard. In the case of Mednafen, there is no public Git repository; hence, we will build from a source tarball. We decide to use the latest version at the time of writing, namely `1.24.1`. Now, the next step is to figure out how to compile the package. Mednafen uses the (dated) GNU Autotools build system. Compiling it amounts to:
```
$ ./autogen.sh
$ ./configure --prefix=/usr
$ make
$ DESTDIR=... make install
```

`autogen.sh` is a script to regenerate `configure` (via `autoreconf`); this step is not necessary for most modern build systems such as CMake or Meson. As usual, `configure` resolves dependencies and sets up the package's build directory while `make` and `make install` compile and install the package, respectively.

**`--prefix` and `DESTDIR`.** When cross compiling, we need to ensure that packages are installed to the correct directories. Since we want to install the package system-wide on the target OS, we set the `configure` option `--prefix=/usr`. However, care needs to be taken during `make install`: without any further parameters, `make install` would install the package to the `/usr` directory *on our host machine*. Luckily, the build system allows to override this behavior via the `DESTDIR` environment variable. Intuitively, `DESTDIR` replaces the root directory `/` when performing `make install`: when `DESTDIR` is set, all files are installed to `${DESTDIR}/usr` instead of `/usr` (assuming a `--prefix` of `/usr`). Hence, `DESTDIR` allows us to correctly install the package into the system root *of our target OS*.

### Integration into Managarm's build system
Since Managarm uses [xbstrap](https://github.com/managarm/xbstrap) to orchestrate the build process of all packages, adding a new package requires adding an entry in xbstrap's `bootstrap.yml` file. The easiest way to add a new entry is simply copying some already existing entry that is close enough to what we need. Of course, it is necessary to modify parts of it, like the source URL or package dependencies. It is also necessary to customize the regeneration step (that invokes `autogen.sh`), the configure step, and the build step. The resulting YAML fragment looks as follows:

```yml
  - name: mednafen
    source:
      subdir: 'ports'
      url: 'https://mednafen.github.io/releases/files/mednafen-1.24.1.tar.xz'
      format: 'tar.xz'
      extract_path: 'mednafen'
      tools_required: [host-autoconf-v2.69, host-automake-v1.15,
                       host-libtool, host-pkg-config]
      regenerate:
        - args: ['./autogen.sh']
    tools_required: [system-gcc, host-pkg-config]
    pkgs_required: [sdl2]
    configure:
      - args:
        - '@THIS_SOURCE_DIR@/configure'
        - '--host=x86_64-managarm'
        - '--prefix=/usr'
        - '--with-sysroot=@SYSROOT_DIR@' # Tell libtool about our system root.
        - '--without-libsndfile' # We do not have a libstdfile port yet.
        - '--disable-debugger' # The debugger requires a working iconv.
    build:
      - args: ['make', '-j@PARALLELISM@']
      - args: ['make', 'install']
        environ:
            DESTDIR: '@THIS_COLLECT_DIR@'
        quiet: true
```

## Building the package

After adding the new package, the remaining steps are compiling and testing it, and fixing any problems that arise. Compiling packages for the first time consists of running the configure step (i.e `./configure`) to generate the build files, and running the build step to actually compile the sources. We invoke these build steps through xbstrap using the `xbstrap install --rebuild mednafen` command.

### Configuration
For Mednafen, the initial `configure` run fails with the following message:
```
checking for pthread_create... yes
checking for sem_init... no
configure: error: *** pthreads not found!
```
Upon closer inspection of the `config.log` file, it becomes clear that the configure script checks for the presence of `sem_init` and reports an error when this function is not found (below, non-relevant lines are omitted):
```
configure:18170: checking for sem_init
configure:18170: x86_64-managarm-g++ -std=gnu++11 -fsigned-char -o conftest -g -O2   conftest.cpp  >&5
.../x86_64-managarm/bin/ld: /tmp/ccn2dBNk.o: in function `main':
.../mednafen/conftest.cpp:194: undefined reference to `sem_init'
collect2: error: ld returned 1 exit status
configure:18170: $? = 1
configure: failed program was:
| int
| main ()
| {
| return sem_init ();
|   ;
|   return 0;
| }
configure:18170: result: no
configure:18177: error: *** pthreads not found!
```
This compilation failure is caused by the fact that [mlibc](https://github.com/managarm/mlibc) (the C library that Managarm uses) misses the corresponding function declarations. For Mednafen to work, this issue needs to be fixed.

**Fixing `sem_*`.** Adding `sem_*` functions to the C library requires (i) adding the headers and declarations, (ii) adding function definitions and (iii) actually implementing those functions. At this stage, we are primarily interested in making `configure` happy; hence, it is enough to add only stubs to mlibc (and not full implementations of `sem_*`). Looking at the [POSIX standard](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/semaphore.h.html), the required declarations look roughly like this:

```c
typedef struct __mlibc_sem_t {
	unsigned int __mlibc_count;
} sem_t;

int sem_init(sem_t *sem, int pshared, unsigned int initial_count);
int sem_destroy(sem_t *sem);
int sem_wait(sem_t *sem);
int sem_timedwait(sem_t *sem, const struct timespec *abstime);
int sem_post(sem_t *sem);
```
Note the `__mlibc` prefix for internal implementation details that are not specified by POSIX. 
After adding these declarations to mlibc (i.e., to a new `semaphore.h` header), together with appropriate stubs, the configure step succeeds and the actual build can start.¹

¹ <span style="font-size: smaller;">The new function declarations are in [`options/posix/include/semaphore.h`](https://github.com/managarm/mlibc/blob/9c994bc925885a4db8f51abd199f09f6874bdb95/options/posix/include/semaphore.h) with stubs in [`options/posix/generic/semaphore-stubs.cpp`](https://github.com/managarm/mlibc/blob/9c994bc925885a4db8f51abd199f09f6874bdb95/options/posix/generic/semaphore-stubs.cpp). After adding them, the C library needs to be reinstalled using `xbstrap install --rebuild mlibc{,-headers}`.</span>

### Compilation
Compilation issues are fixed in similarly to configuration issues.
The initial compilation hits a missing `typedef` in mlibc, followed
by a more severe gap:

***Missing iconv.*** At this point, the build fails due to missing `iconv_*` declarations. These are part of the C library (or, alternatively, can be obtained from GNU's libiconv library). Thus, they can be fixed in the same way as the `sem_*` issues mentioned above, by adding new declarations and stubs to mlibc.²

² <span style="font-size: smaller;">These declarations can be found in [`options/posix/include/iconv.h`](https://github.com/managarm/mlibc/blob/9c994bc925885a4db8f51abd199f09f6874bdb95/options/posix/include/iconv.h); the stubs are in [`options/posix/generic/iconv-stubs.cpp`](https://github.com/managarm/mlibc/blob/9c994bc925885a4db8f51abd199f09f6874bdb95/options/posix/generic/iconv-stubs.cpp).</span>

## Implementation work

After compilation, several issues remain that prevent Mednafen from working correctly. Often, programs need some minor corrections, for example in C library functions. In the case of Mednafen, however, some more work is required. We quickly discuss missing features that have been implemented to finally make Mednafen work correctly.

**pthreads.** While mlibc already has stubs for all of the `pthread_*` functions, thread creation via `pthread_create` was not implemented yet. This is required since Mednafen uses a second thread for audio handling. Implementing `pthread_create` consists of two parts:
 - implementing this function in the C library,
 - and handling threads in Managarm's POSIX server.

To this end, a new request (`sys_clone`) to create a new thread is added to the POSIX server. In mlibc, the `pthread_create` function is modified to invoke the newly added `sys_clone` request. Before calling into the POSIX server, mlibc prepares the stack of the new thread and sets up the so-called *thread control block* (TCB) which stores all thread-local variables. Afterwards, control is handed over to the user provided function.

(While we are still missing an implementation for `pthread_join` and possibly other thread related functions, this is enough to run the emulator. It does become a problem when ones tries to close Mednafen and results in a crash.)

**POSIX semaphores.** Like pthread mutexes, POSIX semaphores are used to enforce mutual exclusion of concurrently executing threads. Since mlibc already supports the `pthread_mutex_*` functions, adding support for `sem_*` is comparatively simple. Indeed, both the pthread functions and POSIX semaphores are implemented on top of futexes.³

**`mkdir()` on ext2.** Even though Managarm already supports the ext2 file system, support for directory creation was still lacking. Adding this support involves the POSIX server (to send a request to the file system driver) and the file system code. Extending the latter turns out to be straightforward since all necessary ingredients (i.e., allocating a new inode and adding `.` and `..` directory entries to it) were already in place.

³ <span style="font-size: smaller;">The implementation can be found in [`options/posix/generic/semaphore-stubs.cpp`](https://github.com/managarm/mlibc/blob/9c994bc925885a4db8f51abd199f09f6874bdb95/options/posix/generic/semaphore-stubs.cpp)`.</span>

## Conclusions
In this blog post, we walked through the process of porting the multi-platform emulator Madnafen to Managarm. Most steps of this process are prototypical for other ports: in addition to the port's integration into the build system, changes to the C library are usually required when porting new software to Managarm. Additionally, Mednafen required some additions to drivers and POSIX code. Luckily, for many other ports, this last step is not needed.

![Mednafen](/assets/2020-porting-software/mednafen-2.png)
