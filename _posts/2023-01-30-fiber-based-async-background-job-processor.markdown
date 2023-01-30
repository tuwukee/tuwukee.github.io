---
layout: post
title:  "Fiber based async background job processor"
date:   2023-01-30 21:37:00 +0100
categories: ruby fibers
---

*This is part 2 of the series about Fibers. The first part is [here]({% post_url 2023-01-29-fibers-and-ruby %}).*

Usually, the systems manage all I/O waiting by allocating a separate OS thread for every request or worker. This approach is effective to some extent, but OS thread is relatively expensive to create and the context switches performed by the OS also don’t come for free. This leads to a lot of overhead. With fibers, it is possible to utilize non-blocking I/O to minimize this overhead.

When dealing with CPU-heavy tasks, breaking down the workload into smaller parts through the use of fibers will not bring benefits if the system lacks the resources to handle the workload. However, many Ruby processes spend a significant amount of time waiting for I/O operations, such as awaiting responses from API or database calls, or writing data to a file. This is the perfect usecase for fibers.

### Socketry overview

As of now, [Socketry](https://github.com/socketry) is the most popular collection of the asynchronous libraries in the Ruby world.

[Async](https://github.com/socketry/async) is the framework providing handy interfaces to make fiber-driven development even simpler. It has a built-in Fiber.scheduler with epoll/kqueue and io_uring support. It encapsulates all scheduler-related calls, so the users could easily spin up asynchronous tasks, see the example below:

{% highlight ruby %}

Async do
  resources.each do |resource|
    Async do
      result = api_client.get(resource)
      DB.connection.save(result)
    rescue => err
      logger.error(err)
    end
  end
end

{% endhighlight %}

Given, that `socketry/async` provides all the required APIs to build a job-processor, then such projects simply had to start appearing.

### Background job processor implementation

Here's a sample implementation [Jiggler](https://github.com/tuwukee/jiggler). `jiggler` is inspired by `sidekiq` and althrough it doesn’t support as many features, it still implements a very similar paradigm, so it’s fair to compare these 2 in terms of performance, to clearly see all the benefits fibers and `socketry/async` can provide when compared with the pure thread-based approach.

Conceptually `jiggler` consists of two parts: the client and the server.
The client is responsible for pushing jobs into Redis and allows to read statistics, while the server reads jobs from Redis, processes them, and writes statistics.

The server consists of 3 parts: `Manager`, `Poller`, `Monitor`.

- `Manager` spins up and handles workers.
- `Poller` periodically fetches data for retries and scheduled jobs.
- `Monitor` periodically loads stats data into redis.

### Pre-emptive fiber scheduling

One important catch here, is that we want `Poller` and `Monitor` to be guaranteed to work in their time, even if the workers perform some CPU-heavy tasks. We want to have up-to-date stats and stable polling. With threads we don’t care, as OS periodically switches threads without developer’s interventions, while the `Fiber.scheduler` waits for a command to switch context.

That’s called co-operative scheduling. Once started, a task within a co-operative scheduling system will continue to run until it relinquishes control. This is usually at its synchronisation point.

In a pre-emptive model tasks can be forcibly suspended.

Wander Hillen wrote [a great article](https://www.wjwh.eu/posts/2021-02-07-ruby-preemptive-fiber.html) explaining the problematics in the context of Ruby, highly recommend to read it.

`Jiggler` solves this problem by forcing the scheduler to switch between fibers in case their execution takes more time than a given threshold value. It introduces a dedicated thread, which encapsulates the scheduler management. The dedicated thread adds some overhead, yet it’s compensated with the achieved time execution control.

{% highlight ruby %}
CONTEXT_SWITCHER_THRESHOLD = 0.5

def patch_scheduler
  @switcher = Thread.new(Fiber.scheduler) do |scheduler|
    loop do
      sleep(CONTEXT_SWITCHER_THRESHOLD)
      switch = scheduler.context_switch
      next if switch.nil?
      next if Process.clock_gettime(Process::CLOCK_MONOTONIC) - switch < CONTEXT_SWITCHER_THRESHOLD

      Process.kill('URG', Process.pid)
    end
  end

  Signal.trap('URG') do
    next Fiber.scheduler.context_switch!(nil) unless Async::Task.current?
    Async::Task.current.yield
  end

  Fiber.scheduler.instance_eval do
    def context_switch
      @context_switch
    end

    def context_switch!(value = Process.clock_gettime(Process::CLOCK_MONOTONIC))
      @context_switch = value
    end

    def block(...)
      context_switch!(nil)
      super
    end

    def kernel_sleep(...)
      context_switch!(nil)
      super
    end

    def resume(fiber, *args)
      context_switch!
      super
    end
  end
end

{% endhighlight %}

### Benchmarks

The latest benchmarks are available on the [jiggler README page](https://github.com/tuwukee/jiggler/blob/main/README.md).

It depends on the payload, but on the given samples it usually saves 10–20% of memory with the same concurrency settings. For the speed — it shows rather good results with File IO, but the difference is not so impressive with `net/http` or `PG` requests.

But keep in mind, that when using fibers — the concurrency can be set to higher values. Technically the overhead of spawning workers is expected to be low. So it’s possible to test it with 50, or 100, or even more workers against a specific payload, in case there is indeed a lot of I/O going on. The limitation is rather in the connection pool (when the workers are doing some DB queries) or in the external services accepting the requests (in case of network requests).

### Limitations

Developers should be careful when using C-extentions within fibers. `socketry/async` doesn’t support `JRuby` as of now. No support for Windows users as well.

When running the benchmarks with OSX (M1 processor) — they weren’t as good both natively and in Docker. Why? I don’t know for sure, and the investigations from my side is still ongoing (more like it stands still, but I hope to find out one day anyway).

I tried to test directly socketry/async against native Ruby threads, and against [Polyphony](https://github.com/digital-fabric/polyphony) — another interesting framework for fibers), and `Polyphony` actually works great on OSX M1. So the problem might be in `kqueue()` support implementation within `socketry/async` Fiber scheduler, but I didn’t get anywhere further than that, and it might be a false lead.

*The article was originaly posted at [medium](https://medium.com/@alieckaja/unleashing-the-power-of-fibers-for-background-jobs-8a22e3a38cd1)* but now it's splitted in parts and migrated to this blog.
