---
title: "Missed OS X Clock Guide"
date: 2017-11-26T08:03:19+03:00
draft: false
comments: true
---

**Table of contents**

   * [Preface](#preface)
   * [Motivation](#motivation)
   * [High resolution time management in Linux](#linux)
   * [Mach Kernel](#mach-kernel)
   * [Mach Clock API](#mach-clock-api)
   * [Deprecated Mach Clock API](#deprecated-mach-clock-api)
   * [Links](#links)

## Preface

This is a story of a little fight with an obscure OS X clock API.

When it comes to high-resolution time management in OS X wouldn't it be
a dreamy to have the following routines:

   - Receive current monotonically incrementing time value.
   - Sleep for a specified interval.
   - Sleep until a specified time.

## Motivation

This fight was begun when I tried to make time-related operations portable in
[Roc](https://github.com/roc-project/roc).

**Thanks**

I would like to thank my friends and colleagues:

   - [Victor Gaydov](https://gavv.github.io/about/) and [his project](https://github.com/roc-project/roc),
     that was a reason why this article has appeared and for dozen helpful
     advices how to improve this article.

## Linux

Linux provides the following routines for high resolution clock management:

   - [clock_nanosleep](http://man7.org/linux/man-pages/man2/clock_nanosleep.2.html)
   - [nanosleep](http://man7.org/linux/man-pages/man2/nanosleep.2.html)

You probably shouldn't be afraid if `clock_nanosleep` isn't implemented on
your system, you can use [gettimeofday](http://man7.org/linux/man-pages/man2/gettimeofday.2.html)
and `nanosleep`, but you'll lose a precision and the monotony because of
`gettimeofday`. You should test it first and maybe it will be acceptable in
your case.

As you remember this article isn't about Linux and to figure out how OS X clock
API actually works, we need to dig into OS X architecture a little.

## Mach Kernel

[XNU](https://en.wikipedia.org/wiki/XNU) (X is not UNIX) is OS X kernel.
It is constructed from two distinct parts: BSD interface used to build
POSIX API; Mach interface to handle processes, threads, virtual memory, and
scheduling. XNU is a [hybrid kernel](https://en.wikipedia.org/wiki/Hybrid_kernel).

Mach kernel was heavily reworked in release 3 (1993) and became a pure kernel,
moving the BSD part into the user space. Excluding UNIX-specific code from the
kernel allows to easily replace BSD with another operating system or even
running several ~~Linux~~ operating system interfaces simultaneously on the top
of the Mach kernel.

Mach design is all about IPC (inter-process communication mechanism), it is
called Mach Ports. Whenever you want to use some kernel or user space
facilities, you need to send a message to some port. Mach ensures security
by requiring that message senders and receivers have rights. A right consists
of a port name and a capability to send or receive on that port, and is much
like a capability in object-oriented systems. Only one task may receive
messages on a corresponding port, but many tasks may send messages to this port.
Mach port is very similar to a Linux file descriptor.

## Mach Clock API

There are 3 ways to work with time in OSX:

   - API based on `mach_timespec_t`
   - API based on `AbsoluteTime`
   - `clock_gettime`, that was introduced in OS X Siera, but there aren't any
     news about `clock_nanosleep` introduction. It won't be dicsussed in this
     article.

**API based on mach_timespec_t**

By exploring [man pages in XNU repository](https://github.com/apple/darwin-xnu/tree/master/osfmk/man), we can figure out, that Mach API has something called clock_service.


Let's explore what we can do with it:

```c
    #include <mach/mach.h>      /* host_get_clock_service */
    #include <mach/clock.h>     /* clock_get_time */
    #include <mach/mach_error.h> /* error handling */

    kern_return_t err = KERN_SUCCESS;
    clock_serv_t host_clock;

    // mach/clock_types.h
    //
    // kern_return_t
    // host_get_clock_service(
    //     host_t			host,
    //     clock_id_t		clock_id,
    //     clock_t			*clock)
    //
    // Reserved clock id values for default clocks.
    //
    // #define SYSTEM_CLOCK	    	0
    // #define CALENDAR_CLOCK		1
    // #define REALTIME_CLOCK		0
    //
    // Where `clock_id` is the identification of the desired kernel clock with
    // possible values:
    //
    //    - REALTIME_CLOCK stands for a time since a boot time. It has the same
    //      semantic as CLOCK_MONOTONIC.
    //
    //    - BATTERY_CLOCK - (typically) low resolution clock that survives
    //      power failures or service outages.
    //
    //    - HIGHRES_CLOCK - a high resolution clock.
    //
    // This function returns a send right to a kernel clock's service port.
    err = host_get_clock_service(mach_host_self(), SYSTEM_CLOCK, &host_clock);
    if (err != KERN_SUCCESS) {
        mach_error("host_get_clock_service: ", err);
        exit(EXIT_FAILURE);
    }

    // struct mach_timespec {
    //    unsigned int  tv_sec;         /* seconds */
    //    clock_res_t   tv_nsec;    /* nanoseconds */
    // };
    // typedef struct mach_timespec mach_timespec_t;
    //
    // As you can see, mach_timespec_t is the same as timespec in Linux.
    mach_timespec_t now;

    err = clock_get_time(host_clock, &now);
    if (err != KERN_SUCCESS) {
        mach_error("clock_get_time: ", err);
        exit(EXIT_FAILURE);
    }

    // We should deallocate the port returned by host_get_clock_service,
    // because we no longer need it.
    //
    // You should be careful with mach ports allocation and deallocation,
    // because it is usually can cause the ports leak problem.
    err = mach_port_deallocate(mach_task_self(), host_clock);
    if (err != KERN_SUCCESS) {
        mach_error("mach_port_deallocate: ", err);
        exit(EXIT_FAILURE);
    }
```

Mach API also provides a system call similar to `clock_nanosleep`. It is called
[clock_sleep](https://github.com/apple/darwin-xnu/blob/master/osfmk/kern/clock_oldops.c#L493).

For more details about Mach API I recommend to use the following resources:

   - OSF Mach Kernel Principles
   - OSF Mach Kernel Interfaces
   - [XNU man pages](https://github.com/apple/darwin-xnu/tree/master/osfmk/man)
   - [XNU man pages from MIT](http://web.mit.edu/darwin/src/modules/xnu/osfmk/man/)

It's worth to mention, that API based on `mach_timespec_t` is
[deprecated](https://developer.apple.com/library/content/documentation/Darwin/Conceptual/KernelProgramming/Mach/Mach.html#//apple_ref/doc/uid/TP30000905-CH209-TPXREF111).  The
possible reasons are discussed in [Deprecated Mach clock API](#deprecated-mach-clock-api)
section.

**API based on AbsoluteTime**

Apple explains hight resolution clock management in the following articles:

   - [High Precision Timers in iOS / OS X](https://developer.apple.com/library/content/technotes/tn2169/_index.html)
   - [Mach Absolute Time Units](https://developer.apple.com/library/content/qa/qa1398/_index.html)

We are recommended to use `mach_absolute_time` function.

`mach_absolute_time` returns a Mach Time unit - clock ticks. The length of a
tick is a CPU dependent. On most Intel CPUs it probably will be 1 nanoseconds
per tick, but we shouldn't rely on this fact. The kernel provides a
transformation factor that can be used to convert abstract Mach time units to
nanoseconds.

Example of receiving current time in nanoseconds using `mach_absolute_time`:

```c
    #include <mach/mach_time.h> /* mach_absolute_time */

    mach_timebase_info_data_t info;

    kern_return_t ret = mach_timebase_info(&info);
    if (ret != KERN_SUCCESS) {
        // Some kind of disaster...
    }

    // You should cache this value to avoid calculating it each time.
    double steady_factor = (double) info.numer / info.denom;

    uint64_t now_ns = mach_absolute_time() * steady_factor;
```

**Mach APIs under the hood**

Let's try to receive current time using both APIs:

   - `mach_timespec_t`: 556680752418861 ns
   - `mach_absolute_time`: 55668752378055 ns

Both values belongs to the same time domain.

* API based on `mach_absolute_time` uses [RDTSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter) underneath.
  You can review [source](https://opensource.apple.com/source/Libc/Libc-320.1.3/i386/mach/mach_absolute_time.c)
  for more details.

* API based on `mach_timespec_t` uses RDTSC through Mach ports.

## Deprecated Mach Clock API

There are problems with time API based on `mach_timespec_t`. To receive the
current time you need 3 system calls:

   - host_get_clock_service
   - clock_get_time
   - mach_port_deallocate

`host_get_clock_service` and `mach_port_deallocate` usually will be called only
once. We need to value the cost of `clock_get_time`.
Using [Benchmarks for time-related operations for OS X and Linux](https://github.com/dshil/tb) we can see the difference.

`mach_absolute_time`:

| function name      | number of calls | avg (ns/call) |
|--------------------|-----------------|---------------|
| mach_absolute_time | 1000            | 35            |
| mach_absolute_time | 1000000         | 36            |
| mach_absolute_time | 5000000         | 37            |

`clock_get_time`:

| function name  | number of calls | avg (ns/call) |
|----------------|-----------------|---------------|
| clock_get_time | 1000            | 839           |
| clock_get_time | 1000000         | 832           |
| clock_get_time | 5000000         | 813           |

The cost of system call is important, but it's not a key problem here.

First, let's look at the diagram below to figure out what is performed
underneath. As you remember XNU is all about IPC.

![host_get_clock_service](/blog/missed-os-x-clock-guide/host_get_clock_service.png)

To deal with a clock service you need:

   - Put a port (task_port), where to send the result, in a message.
   - Send the message to the specified port (clock_service).
   - The message will be put in the port queue.
   - Receive clock_port.

Kernel allocates a clock_port for us, using which we'll obtain a required
current time.

![clock_get_time](/blog/missed-os-x-clock-guide/clock_get_time.png)

   - Put a port (clock_port), where to send the result, in a message.
   - Send the message to the specified port (clock_service).
   - The message will be put in the port queue.
   - Receive current time.

![mach_port_deallocate](/blog/missed-os-x-clock-guide/mach_port_deallocate.png)

When we don't need this port we should deallocate it to avoid a [port leak
problem](https://robert.sesek.com/2012/1/debugging_mach_ports.htm://robert.sesek.com/2012/1/debugging_mach_ports.html).

It seems, that we don't have any problems here, but the devil in the details.
A port is a protected bounded queue. If your system is under a high load, the
queue can be full and we'll need to wait until our message will proceed. We
can meet this problem in each previously mentioned functions. As a result of
our "high precision time management" will lose a precision.

## Links

   - [Monotonic time in Mac OS X](http://web.archive.org/web/20100517095152/http://www.wand.net.nz/~smr26/wordpress/2009/01/19/monotonic-time-in-mac-os-x/comment-page-1/)
   - [libuv fight with OS X clock API](https://github.com/joyent/libuv/pull/1325)
   - [clock_gettime for OS X](http://web.archive.org/web/20100501115556/http://le-depotoir.googlecode.com:80/svn/trunk/misc/clock_gettime_stub.c)
   - [Mach overview](http://web.eecs.utk.edu/~qcao1/cs560/papers/mach.pdf)
   - [OS X kernel guide lines](https://developer.apple.com/library/content/documentation/Darwin/Conceptual/KernelProgramming/About/About.html)
   - [Mac OS X Internals book](http://www.osxbook.com)
   - [RDTSC vs HPET](https://aufather.wordpress.com/2010/09/08/high-performance-time-measuremen-in-linux)
   - [Linux clock_gettime internals](http://linuxmogeb.blogspot.ru/2013/10/how-does-clockgettime-work.html)
   - [Timers internals](http://aakinshin.net/blog/post/stopwatch/#linux)
