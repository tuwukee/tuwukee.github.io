---
layout: post
title:  "Fibers and Ruby"
date:   2023-01-29 18:43:25 +0100
categories: ruby fibers
---
Ruby 3 has introduced a game-changing feature for concurrent programming with the release of `Fiber::SchedulerInterface`. This powerful tool allows developers to manage fibers, making it easier to handle context switching in I/O-bound tasks.

### What is Fiber?

Fibers are primitives for implementing lightweight cooperative concurrency. Fibers exist within a thread and only one fiber per thread can run at a time. They use very little memory, so it is possible to create thousands of fibers without a huge memory footprint. Another important aspect is that threads are managed by the operating system’s scheduler, and fibers are scheduled and managed by developers.

First introduced in Ruby 1.9 (December 2007), it’s not a new concept. Despite being lightweight and providing more control, fibers didn’t gain wide community popularity, staying a rather rarely used feature. Perhaps, due to the fact that Ruby didn’t provide a simple out-of-the-box scheduling interface, or maybe because the community is heavily Rails-centric, thus it’s just easier for people to stick to threads as a more commonly used construct.

However, there are still several fiber-using libraries, which used to be pioneers and gained some popularity:

- [Celluloid](https://github.com/celluloid/celluloid) — not maintained since 2016, but back at the times it was one of the main Ruby gems providing handy API to write concurrent Ruby code.
- [Em-synchrony](https://github.com/igrigorik/em-synchrony) — a fibered implementation of EventMachine, latest commits happenned about 5 years ago.

When using fibers in older Ruby versions (from 1.9 until 3), developers had to keep track of all the fibers and manually manage them by calling the `Fiber::yield`, `Fiber#resume` and `Fiber#transfermethods`. This process can be complex, too verbose and error-prone.

Nowadays, with Ruby 3 the fibers management has become much simpler.

### What is Fiber Scheduler?

`Fiber::SchedulerInterface` provides a set of hooks invoked when a blocking operation begins/ends.

Basically, Ruby standard I/O methods such as `IO#wait_readable`, `IO#wait_writable`, `IO#read`, `IO#write`, `Kernel.sleep`, etc., have been patched to yield to the scheduler if it‘s defined in the context of the current thread. And the scheduler, in turn, passes execution control to the other ready fibers.

This automatically makes all standard Ruby calls scheduler-friendly. However, there are still a lot of Ruby gems using C-extentions, which could perform I/O on their own bypassing Ruby’s native methods. For example, different kinds of DB-adapters. Fortunately, it’s relatively easy to implement a call to the scheduler from a C-extention, so the fiber-support across such gems gradually grows. It’s up to developers to check if a given gem with a built-in C-extention supports the scheduler and thus if it makes sense to use with fibers.

### Different kinds of event selectors

An event selector is a Linux kernel feature that allows a program to monitor multiple file descriptor sources for events, such as input or output operations. In other words, that is a mechanism that among other things can notify a program whenever a specific non-blocking operation is complete.

For example, if the fiber was blocked because no data was available on the socket it reads from, it should be resumed once the data arrives. The scheduler should be notified that the non-blocking operation is complete by using one of the next strategies underneath:

- `poll()/epoll()` is the default mechanism for watching sockets to see if new data is available on almost all Linux systems.
- `io_uring` is a Linux kernel library that provides a high-performance interface for asynchronous I/O operations. It was introduced in Linux kernel version 5.1 and aims to address some of the limitations and scalability issues of the existing asynchronous I/O interface.
- `kqueue()` is used in OSX/FreeBSD.
- `IO Completion Ports` for Windows.

Most production systems would most likely use `epoll()` as it’s the most common approach for Linux systems, however it might worth giving a try to io_uring if it’s supported by the specific Fiber.scheduler and is available in the OS.

It worth mentioning that before the arrival of Fiber scheduler, the main library which took care of I/O events start/end detections was [Nio4r](https://github.com/socketry/nio4r). And it is still used in Action Cable, Puma, Async v1 and a lot of other libraries dealing with I/O.

*[The next article]({% post_url 2023-01-30-fiber-based-async-background-job-processor %}) of the series.*

*The article was originaly posted at [medium](https://medium.com/@alieckaja/unleashing-the-power-of-fibers-for-background-jobs-8a22e3a38cd1)* but now it's splitted in parts and migrated to this blog.
