---
layout: post
title:  "Granting at least once delivery"
date:   2023-02-14 08:00:00 +0100
categories: ruby fibers
thumbnail: /assets/images/granting-thumb-diagram.png
description: Exactly-once is the message delivery guarantee the most developers want to achieve in their systems. In the most cases, the implementation of the set of rules backing it up should include the application layer. When it comes to messaging systems, exactly-once delivery at an app level is much easier to be built on top of at-least-once, since at-most-once sometimes loses messages.
---

*This is part 3 of the series. The previous part is [here]({% post_url 2023-01-30-fiber-based-async-background-job-processor %}).*

`Exactly-once` is the message delivery guarantee the most developers want to achieve in their systems. In the most cases, the implementation of the set of rules backing it up should include the application layer. The idea is explained in details in [this great article by Szymon Pobiega](https://exactly-once.github.io/posts/exactly-once-delivery/).

Since `at-most-once` sometimes loses messages, `exactly-once` is much easier to be built on top of an `at-least-once` background job processor (or any messaging system).

The non pro version of the most popular Ruby's background job processor [Sidekiq](https://github.com/mperham/sidekiq) implements `at-most-once` delivery. The pro version comes with `at-least-once` guarantee.

> Just remember that Sidekiq will execute your job at least once, not exactly once. Even a job which has completed can be re-run. Redis can go down between the point where your job finished but before Sidekiq has acknowledged it in Redis. Sidekiq makes no exactly-once guarantee at all.

from [sidekiq's best practices](https://github.com/mperham/sidekiq/wiki/Best-Practices#2-make-your-job-idempotent-and-transactional)

Let's dig a bit into the problematics of it. Where and when the messages are being lost and why it's hard to migrate from `at-most-once` to `at-least-once`?

See the simplified diagram:
![Diagram 1](/assets/images/granting-at-least-once-delivery-diagram-1.svg){:class="post-image max-height-400"}

A sample message loss scenario: a job was pulled from `Redis`, and the worker started to perform the task, but the job processor received a `kill` signal, as a result the worker could not complete the task in time — so the pulled message was lost. The processor has timeouts and workers have some time to finish the processing before the `shutdown`/`restart`, but timeouts cannot grant a complete reliability.

One of possible ways to achieve reliability is to use `BRPOPLPUSH` instead of `BRPOP`. As of Redis version 6.2.0, `BRPOPLPUSH` is deprecated, and can be replaced by `BLMOVE`. This (or similar) approach is being used in `sidekiq pro` version. It's an atomic operation, so every time it pops something from the queue, it pushes the same message into a backup queue (let's name it `in_progress` queue). Once the job is done, the relevant data can be safely removed from the `in_progress` queue. This way, the messages won't be lost, thus it grants `at-least-once` execution. However, `BRPOPLPUSH` (same for `BLMOVE`) has a downside: it doesn't support multiple sources. It has to be run per queue, and it in the way it's being used the performance degrades. There's a lot of discussion about this problem in the community (f.e. [request to add MBRPOPLPUSH](https://github.com/redis/redis/issues/1785)), however, it hasn't been resolved yet.

> Because of Redis limits, super_fetch has to poll the queues in Redis for jobs, rather than blocking. If you have M queues being processed by N processes, you will get M * N rpoplpush Redis calls per second which can lead to a lot of Redis traffic and CPU burn.
> The solution is to reduce the number of queues or specialize your Sidekiq processes: have each process only handle 3-4 queues.

from [sidekiq's reliability](https://github.com/mperham/sidekiq/wiki/Reliability#limitations)

One more issue of this architecture: the number of `Redis` connections equals to the specified concurrency.

Moving queue reading to a separate entity and making it independent of workers and concurrency settings would reduce the load on Redis. In the other words, the idea is to read the jobs from `Redis` using `BLMOVE` from each queue with a dedicated connection, while the workers are going to process the pulled jobs. The concurrency can be set to a rather high value, while the number of `Redis` connections will depend on the number of specified queues.

To verify this idea, let's compare the performance of parallel `BRPOP` reads with `BLMOVE` using Ruby's `redis-client` and `async`.

The test setup: create 3 different queues in `Redis`, push 100_000 messages into each, and measure how much time it takes to read the messages:

- 3 connections with `BRPOP` listening on the full list of queues (`BRPOP queue0 queue1 queue2`)
- 3 connections with `BLMOVE` (one per queue) with an additional backup `in_progress` queue as the destination (`BLMOVE queue0 queue0_in_progress LEFT RIGHT`)

```ruby
require 'async'
require 'async/pool'
require 'redis-client'

COUNT = 100_000

# async pool requires a specific interface
class CustomClient < RedisClient
  def concurrency
   1
  end
 
  def viable?
   connected?
  end
 
  def closed?
   @raw_connection.nil?
  end
 
  def reusable?
   !@raw_connection.nil?
  end
end

config = CustomClient.config(timeout: nil)
pool = Async::Pool::Controller.wrap(limit: 3) do
 config.new_client
end

# upload 100_000 messages
3.times do |i|
  Sync do
    pool.acquire do |cl|
      cl.call('DEL', "testlist#{i}")
      cl.call('DEL', "testlist_in_process#{i}")
      COUNT.times do
        cl.call('LPUSH', "testlist#{i}", '100')
      end
    end
  end
end
```

The tests are being run on Apple M1 MacOS 13.2.
The code running `BLMOVE`:

```ruby
def run_blmove(pool)
  Async do
    3.times do |i|
      key = "testlist#{i}"
      reserved_key = "testlist_in_process#{i}"
      values = []
      Async do
        loop do
          value = pool.acquire do |cl|
            cl.blocking_call(false, 'BLMOVE', key, reserved_key, 'LEFT', 'RIGHT', 2)
          end
          values << value if value
          break if values.count == COUNT
        end
      end
    end
  end
end
```

The resulting time is `7.69s`.

Then, the code running `BRPOP` is:

```ruby
def run_brpop(pool)
  keys = ['testlist0', 'testlist1', 'testlist2']
  values = []

  Async do
    3.times do |i|
      Async do
        loop do
          break if values.count == COUNT * 3
          value = pool.acquire do |cl|
            cl.blocking_call(false, 'BRPOP', *keys, 2)
          end
          values << value if value
        end
      end
    end
  end
end
```

The resulting time is `9.87s`.

Increase the number of concurrent reads, re-run the `BRPOP` with 3 queues, but 12 pool concurrency and 12 nested async tasks: the result is `9.54s`.

30 pool concurrency and 30 nested async tasks: the result is `9.63s`. The higher concurrency numbers do not provide a better reading speed, and it's still much slower compared to 3 `BLMOVE` connections.

A big advantage of fibers is that they are spawned at much cheaper cost than threads. Now, that we're sure that `BRPOP` can be replaced with `BLMOVE` and the process of reading from `Redis` can be safely separated from workers without a performance drop, we can create a dedicated `Fetcher` for each queue and run it with fibers. Each `Fetcher` reads jobs, writes the data into `in_progress` queue, pushes the job into `Priority Queue` (`PQueue`), and the workers pool reads and process the data. On complete, the job is being removed from `in_progress` queue with `Acknowledger`. 

Workers pop the jobs from a `Priority Queue`, so the ones with the higher prio are going to be processed first. The fetchers are smart enough, so they won't read anything from Redis to push it into `PQueue` unless there are free workers available.

See the simplified diagram:
![Diagram 2](/assets/images/granting-at-least-once-delivery-diagram-2.svg){:class="post-image"}

This hieararchy guarantees `at-least-once` jobs execution for regular jobs. There's more to overview, in the upcoming articles I'll explain how `in_progress` queues are being handled in case a worker wasn't able to complete a job. Also there's a big topic how to make scheduled at a specific time jobs reliable, and the same for ones which are failed and going to be retried. It's not covered by this diagram and is going to be addressed in the future versions. Stay tuned!

This approach, in particular additional `LREM` call for each job on completion, adds some overhead, thus `at-least-once` works slower compared to `at-most-once`, depending on the payload it takes some additional memory and CPU. However, in cases when reliability is important, this could be a fine compromise.

Jiggler implements the given approach in 0.1.0 version. In fact, it provides a setting for users to specify `at-least-once` or `at-most-once` delivery strategy. The complete performance results can be found on [README page](https://github.com/tuwukee/jiggler/blob/main/README.md).
