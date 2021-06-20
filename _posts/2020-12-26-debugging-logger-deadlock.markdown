---
layout: post
title:  "De-bugging a logger deadlock in elixir"
date:   2020-12-26 15:41:54 +0530
excerpt:  "Found a bug in logger in elixir versions 1.9.x and 1.10.x, which caused a process a freeze"
---

Note: This issue was [fixed](https://github.com/elixir-lang/elixir/issues/10420) in elixir [1.11.1](https://github.com/elixir-lang/elixir/blob/v1.11/CHANGELOG.md#v1111-2020-10-16)

At work, I got messaged about some test runs on CI randomly failing. I looked at the job and noticed that most of the tests were frozen. Tests were failing serially one after the other due to timeouts.

The job passed on retry though. I managed to reproduce the issue on local after a few attempts.

This seemed to be the common pattern, in the stack trace of the failing tests.

```
stacktrace:
  (stdlib) gen.erl:169: :gen.do_call/4
  (stdlib) gen_event.erl:239: :gen_event.rpc/2
  (logger) lib/logger.ex:692: Logger.__do_log__/3
``` 

Seemed like [gen_event](http://erlang.org/doc/man/gen_event.html) do_call called by Logger was hanging. Curious. 

I had never seen this issue on production before. The volume of messages logged on test shouldn't be nearly as big as production. 

Also, I vaguely remembered reading about logger being async. Why was it blocking?

Taking a quick peak at logger's documentation, I found a [sync_threshold](https://hexdocs.pm/logger/1.9.0/Logger.html#module-runtime-configuration) config. 
> if the Logger manager has more than :sync_threshold messages in its queue, Logger will change to sync mode, to apply backpressure to the clients.

Our configuration was still at default, which seemed to be 20.

I fired up observer, and kept doing test runs till I came across a failed run.

<img src="/assets/logger_deadlock/logger_processes.png"/>

The message queue length for Logger was going up.

Here's what the message queue looked like.

<img src="/assets/logger_deadlock/logger_stack.png"/>

Messages switched from `notify` to `sync_notify` after ~20 messages. This explained why logger was blocking. Processes calling Logger were waiting for it to respond to these `sync_notify` messages.

But why was logger not processing any of these messages. Even the initial async `notify` messages did not seem to have been read.

<img src="/assets/logger_deadlock/logger_overview.png"/>

The logger process status said 'waiting', which meant the receive block is stuck waiting for some message.

I started looking at it's stack trace.

<img src="/assets/logger_deadlock/logger_stacktrace.png"/>

I saw a `do_terminate` and a `report_terminate`. Perhaps something crashed. But shouldn't the client abort in such cases, why was it still waiting?

Scrolling up, the third line looked strange. It was a recursive call to `Logger.__do_log__/3`. The process was deadlocked trying to call itself.

I looked at gen_event's [code](https://github.com/erlang/otp/blob/526ba5933944a309d5225b801d936d0b41c622f3/lib/stdlib/src/gen_event.erl#L710), to find that if a logger backend's `handle_call` crashed, gen_event tried to report the crash by logging it. From the stack trace, it seemed to be calling elixir's logger. 

I looked at the logger's documentation again, and found the [handle_otp_reports](https://hexdocs.pm/logger/1.9.0/Logger.html#module-error-logger-configuration) config. I turned it off by setting `handle_otp_reports: false`. 

On running the test a few more times, the tests seem to have stopeed freezing, and I saw the actual error.

```log
=ERROR REPORT====
** gen_event handler {'Elixir.Logger.Backends.Console',<0.2833.0>} crashed.
** Was installed in 'Elixir.Logger'
** Last event was: {error,<0.64.0> ...
```

Why was the logger console backend crashing? Looks like it crashes if it [can't serialize metadata](https://github.com/rahuljayaraman/logger_console_backend_crash/blob/main/test/logger_metadata_test.exs). Seems to have been [fixed 1.10.x onwards](https://travis-ci.org/github/rahuljayaraman/logger_console_backend_crash/builds/734918029).

## Summary

Logger calls seem to deadlock if
1. If a logger backend has crashed and
2. If logger is in sync mode and
3. `handle_otp_reports` in turned on 

In such cases,

1. Logger calls gen_event,
2. There's a crash
3. gen_event calls logger to report termination.
4. But logger is in sync mode, deadlock.

Here's a script which causes the issue. It tries to get Logger in sync mode before gen_event can print the crash.

[Seems to cause freezes on 1.9.x, 1.10.x and 1.11](https://travis-ci.org/github/rahuljayaraman/logger_backend_crash)


```
config :logger,
  backends: [
    LoggerBackendCrash
  ],
  handle_otp_reports: true,
  sync_threshold: 10

defmodule LoggerBackendCrash do
  @behaviour :gen_event

  @impl true
  def init(_) do
    {:ok, []}
  end

  @impl true
  def handle_call(_, _) do
    raise "handle_call_error"
    {:ok, []}
  end

  @impl true
  def handle_event(_, _) do
    raise "handle_event_error"
    {:ok, []}
  end

  @impl true
  def handle_info(_, _) do
    {:ok, []}
  end
end

Enum.each(1..20, fn cnt ->
  Logger.info("#{cnt}")
end)
```
