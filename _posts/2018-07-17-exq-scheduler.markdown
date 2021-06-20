---
layout: post
title: "Introducing exq-scheduler, a distributed scheduler for Sidekiq"
date:   2018-07-17 15:41:54 +0530
excerpt: To solve some synchronization issues in sidekiq-scheduler, we wrote exq-scheduler
---

One of our customers wanted to setup a time based [job scheduling](https://en.wikipedia.org/wiki/Job_scheduler) system, similar to [cron](https://en.wikipedia.org/wiki/Cron), to reliably schedule critical tasks. They were already using [sidekiq](https://github.com/mperham/sidekiq), to run some tasks in the background.

To improve fault tolerance, we tried running [sidekiq-schedular](https://github.com/moove-it/sidekiq-scheduler) (a popular scheduling library) in a distributed setup. But we found synchronization issues[^first] that could lead to jobs being scheduled multiple times under some scenarios.

In our effort to fix these issues, we built [exq-scheduler](https://github.com/activesphere/exq-scheduler).

We had to understand some finer aspects of distributed systems, to get a better understanding of how processes synchronize using shared memory. We plan to write about some of these understandings in subsequent notes.

## About Sidekiq and Exq

[Sidekiq](https://github.com/mperham/sidekiq) is a job processing library. It has two components, a `client` and `worker`. It uses a redis LIST as storage. A job instruction is a json object complying with a schema. A sidekiq `client` creates a job instruction in the specified schema & pushes it to a queue (redis LIST). A sidekiq `worker` listening to the queue receives this instruction and performs a related task.

`client` and `worker` can be written in any language, as long as they work with the same schema, as implemented by sidekiq. [Exq](https://github.com/akira/exq) is an elixir library, which complies with the same storage schema. In other words, you could use sidekiq client with exq worker, or visa versa, or even use exq for both `client` and `worker`.

*[exq-scheduler](https://github.com/activesphere/exq-scheduler) is a sidekiq `client` that enqueues job instructions as per a time schedule.*

These time schedules can be configured using [cron](https://en.wikipedia.org/wiki/Cron) syntax as follows.

```elixir
config :exq_scheduler, :schedules,
  signup_report: %{
    cron: "0 * * * *",
    class: "SignUpReportWorker",
    args: ["arg1", "arg2"],
    queue: "default"
  }
```

## Features

Here are some of [exq-scheduler's](https://github.com/activesphere/exq-scheduler) salient features. We might expand on implementation details in future posts.

- Run multiple schedulers while avoiding duplicate job instructions.<br>
We do this by building a key deterministically using execution time and pushing to the queue in a transaction.

- [Redis sentinel](https://redis.io/topics/sentinel) support.<br>
This allows improving availability of shared redis instance.

- Schedule missed jobs.<br>
If scheduler experiences down time for some reason (node restarted etc.), jobs missed in the last 1 hour are scheduled by default. The interval can be configured.

- Define schedules for a timezone.

```elixir
config :exq_scheduler,
  missed_jobs_window: 60 * 60 * 1000,
  time_zone: "Asia/Kolkata"
```

- API consistent with sidekiq and sidekiq-scheduler.<br>
This enables any compatible job runner to pick up instructions.
Also allows use of [sidekiq](https://github.com/mperham/sidekiq/wiki/Monitoring#web-ui) and [sidekiq-scheduler](https://github.com/moove-it/sidekiq-scheduler#sidekiq-web-integration)'s web UI.

- Deploy with existing Exq workers without major changes in deployment setup.

## Stability

[exq-scheduler](https://github.com/activesphere/exq-scheduler) is currently being used in production with one of our customers. The library is backed by [tests](https://github.com/activesphere/exq-scheduler/tree/master/test) which introduces various faults and verifies that invariants are always maintained.

[^first]: Issues were caused by lack of determinism when building a unique key for scheduled tasks and not handling edge cases, which account for failure of scheduler, after acquiring a lock.  https://github.com/moove-it/sidekiq-scheduler/issues/156, https://github.com/moove-it/sidekiq-scheduler/issues/181
