---
layout: post
title:  "Concurrent mocks in elixir using seq_trace"
date:   2020-12-30 15:41:54 +0530
---

Let's say you want to test some function, by mocking out one of it's dependencies. If your test mocks a module using a library like [meck](https://github.com/eproxus/meck), it creates a new version of that module. Other concurrent tests running at that time might also get this new version of the module that they weren't expecting.

Even if you use `Application.put_env` to swap implementations, it changes env for the entire application and the new implementation becomes available to all concurrent tests.

To solve this, tests with such mocks run with `async: false`.


## Narrowing down on the problem

Let's look at the problem closer, by attempting to solve the concurrency issue. Consider a module `Module`, that has a function `io`. Let's try mocking out `io` for a test.

```elixir
defmodule Module do
  def exec() do
    io().do_something(_arg1, _arg2)
  end
  
  defp io() do
    {module, fun, arity} = {Module, :io, 0}
    
    mock = get_mock(module, fun, arity)
    if mock do
      apply(mock, [])
    else  
      # default to original implementation
    end
  end
end

defmodule ModuleTest1 do
   test "mock 1" do
     mock(Module, :io, fn -> :mocked_io_1 end)
   end
end
```

We changed the implementation of the `io` function to 
1. Try and find the mocked implementation, setup by the test, at runtime.
2. If it can't find one, it defaults to the original implementation.

But what happens if multiple tests attempt to mock the`io` function concurrently. How does `get_mock` choose between the different mocked implementations. `get_mock` needs some way to figure out which test is calling it.

```elixir
defmodule ModuleTest1 do
   test "mock 1" do
     mock(Module, :io, fn -> :mocked_io_1 end)
   end
end

defmodule ModuleTest2 do
   test "mock 2" do
     mock(Module, :io, fn -> :mocked_io_2 end)
   end
end
```

## Passing context

One way we can to pass context within a process is to use the [process dictionary](https://hexdocs.pm/elixir/Process.html#get/0). We use `Process.put(:apply_this_mock, mocked_fn)` in the test. In the module we use `Process.get(:apply_this_mock)` to get the mock implementation setup by the test.

But this won't work across process boundaries. ie. if the mocked code lives on a different process than where `Process.put` was called.

```
[ModuleTest] --> [process 1] --> [process 2] --> [Module]

[ModuleTest2] --> [process 5] --> [process 7] --> [Module]
```

`Module` may be called via any sequence of calls between processes. `Module` needs to know which mock implementation was set at the start of these sequences by the test.

## $callers

Another way to solve the problem is to use [$callers](https://hexdocs.pm/elixir/1.10.0/Task.html#module-ancestor-and-caller-tracking). It allows Module to know processes in a call sequence, but only if processes are [Tasks](https://hexdocs.pm/elixir/Task.html) or managed by [supervised tasks](https://hexdocs.pm/elixir/Task.html#module-supervised-tasks)

[Mox](https://github.com/dashbitco/mox) is another mocking library that uses this [approach](https://github.com/dashbitco/mox/blob/fac3fb0cc87bdbf8fb25ba675611b4c4055add4f/lib/mox.ex#L716) to solve the concurrency problem.

It won't work if any of the intermediate processes are started by other means
like Supervisors or if they're manually started. 

Mox might expect the test to manually start all involved processes using task.

## Sequential tracing

Another way could be using erlang's [sequential tracing](https://erlang.org/doc/man/seq_trace.html#whatis). 
From it's docs

> Sequential tracing makes it possible to trace information flows between processes resulting from one initial transfer of information.

We can use `seq_trace.set_token(:label, label)` to set a label for a sequential trace along with the test. This label becomes available to all the intermediate processes in the call sequence, including the one in which `Module` is.

```
[ModuleTest1] --> [process 1] --> [process 2] --> [Module]
    |--------------------- label1 ---------------------|
    
    ^ set_token(:label, "label1")
```

Let's set a label along with the test.

```elixir
defmodule ModuleTest1 do
   test "mock 1" do
     label = "label1"
     SequentialTrace.set_label(label)
     mock(label, Module, :io, fn -> :mocked_io_1 end)
   end
end

defmodule ModuleTest2 do
   test "mock 2" do
     token = "label2"
     SequentialTrace.set_label(label)
     mock(label, Module, :io, fn -> :mocked_io_2 end)
   end
end

defmodule SequentialTrace do
  defp set_label() do
  # Do not overwrite labels setup by parent processes
    case get_label() do
      nil ->
        label = UUID.uuid4()
        :seq_trace.set_token(:label, label)
        label

      label ->
        label
    end
  end

  defp get_label() do
    case :seq_trace.get_token(:label) do
      {:label, label} ->
        label

      [] ->
        nil
    end
  end
end
```

On the `Module` side, we look for the label set by the calling process, and choose a mocked implementation.

```elixir
defmodule Module do
  def exec() do
    io().do_something(_arg1, _arg2)
  end
  
  defp io() do
    token = SequentialTrace.get_label()
    
    {module, fun, arity} = {Module, :io, 0}
    # Get mock for specific token
    mock = get_mock(token, module, fun, arity)
    if mock do
      apply(mock, [])
    else  
      Application.get_env(:your_app, :io)
    end
  end
end
```

## Caveats for seq_trace in production

In our case, we used the `seq_trace label` as a reference `ID` for a sequence. We used this `ID` to fetch context stored elsewhere. Setting this `ID` to some unique key woks for us.

`:seq_trace` allows using any data structure as a label. Meaning instead of using it as a `ID`, processes might also keep state on it.
eg: a process might use `:seq_trace.set_token(:label, %{process_a: "foo"})`. 

The problem with this approach, is that if multiple processes rely on label for state, these processes need to agree on a data structure before hand so they don't overwrite each other's state. eg: if process B modifies the same state, it must preserve process A's state and add it's changes to to different key on the map.`:set_trace.set_token(:label ,%{process_a: "foo", process_b: "bar"})`.

This agreement of coming up with a composable data structure might not happen, unless the API becomes more restrictive. 

Because of this, my guess is that it's still not feasible to use in production to propogate state, even if you plan to use it as an id, because some other process might change it. 

Also, worth noting the [performance considerations](https://erlang.org/doc/man/seq_trace.html#performance-considerations) when using `seq_trace` in production.
> The performance degradation for a system that is enabled for sequential tracing is negligible as long as no tracing is activated. 
> When tracing is activated, there is an extra cost for each traced message, but all other messages are unaffected.

Might be ok to ignore these issues for the test environment though. Other libraries which use `seq_trace` might be mostly tracing libraries, and can probably be disabled while testing.

## Full implementation with `seq_trace`

Sample implementation can be found [here](https://github.com/rahuljayaraman/cswap). It

- Supports concurrent mocking across process boundaries using [:seq_trace](https://erlang.org/doc/man/seq_trace.html)
- Supports a decorator to swap code for test builds, using [arjan/decorator](https://github.com/arjan/decorator)

Note: [Bug](https://bugs.erlang.org/browse/ERL-602) in seq_trace 
