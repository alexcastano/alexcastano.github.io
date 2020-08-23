---
title: "Why and how a GenServer stops?"
description: >
  The different possibilities to stop a GenServer and
  how to handle the termination process.
tags: [Elixir]
header:
  overlay_filter: 0.5
last_modified_at: 23/08/2020

---

The last day, I was creating a GenServer and I realized that I didn't know the proper way to stop it.
I had a vague idea about the process, but I needed fine-grained details.
This post is the result of my little research about it.

I think this is a very important matter, however,
I sometimes see a lot of people that don't pay enough attention to it.
If you want to create a resilient Elixir application, the startup and shutdown process must be properly implemented.
The application will behave much better even in the worst situations.

Thanks to the Erlang supervision tree,
if you have a bug or something unexpected happens,
we are giving a chance to the application to heal itself.
An extraordinary feature in my opinion.

It is also important because we are living in the Kubernetes era,
where servers are continuously created and destroyed.
For this reason, the shutdown process must be under control at all levels.
Today we focus on how a GenServer stops.

## The terminate/2 callback

Before we look at the reasons why GenServer stops, let's see the `terminate/2` callback.
It is not a mandatory one, we can implement only when we need it,
but it can help us if we want to do something before stopping.
Besides, this is our last chance.
After the execution of this function, we will lose the state of the GenServer forever.

It receives two arguments, the first one is the reason for shutting down,
and the second is the last state of the process.
We will see more about them in the following examples.

## How a GenServer can stop itself

We have two contexts from which to stop GenServer, from the GenServer process itself, and from any other process. Let's start with the first one.

### A bug in the code

I don't know about you, but the most common way I see a GenServer stop is a bug in my code. This is how it behaves when this happens:

```elixir
defmodule Buggy do
  use GenServer

  @impl true
  def init(state) do
    {:ok, state}
  end

  @impl true
  def handle_info({:div, num}, state) do
    state = state + 1
    IO.inspect(":div received, updating state to: #{state}")
    IO.inspect("Divission = 100 / #{num} = #{100 / num}")
    {:noreply, state}
  end

  @impl true
  def terminate(reason, state) do
    IO.inspect("terminate/2 callback")
    IO.inspect({:reason, reason})
    IO.inspect({:state, state})
  end
end
```

```elixir
iex> {:ok, pid} = GenServer.start(Buggy, 0)
{:ok, #PID<0.139.0>}

iex> send(pid, {:div, 10})
{:div, 10}
":div received, updating state to: 1"
"Divission = 100 / 10 = 10.0"

iex> send(pid, {:div, 0})
{:div, 0}
":div received, updating state to: 2"
"terminate/2 callback"
{:reason,
 {:badarith,
  [
    {Buggy, :handle_info, 2, [file: 'buggy.ex', line: 13]},
    {:gen_server, :try_dispatch, 4, [file: 'gen_server.erl', line: 680]},
    {:gen_server, :handle_msg, 6, [file: 'gen_server.erl', line: 756]},
    {:proc_lib, :init_p_do_apply, 3, [file: 'proc_lib.erl', line: 226]}
  ]}}
{:state, 1}

## Error log
"""
18:00:28.809 [error] GenServer #PID<0.139.0> terminating
** (ArithmeticError) bad argument in arithmetic expression
    buggy.ex:13: Buggy.handle_info/2
    (stdlib 3.13) gen_server.erl:680: :gen_server.try_dispatch/4
    (stdlib 3.13) gen_server.erl:756: :gen_server.handle_msg/6
    (stdlib 3.13) proc_lib.erl:226: :proc_lib.init_p_do_apply/3
Last message: {:div, 0}
State: 1
"""
```

It is interesting to see that the state received by the `terminate/2` callback is the last one before the last `handle_call/2` callback was called.
We didn't have the chance to return the updated state,
so this is the best the GenServer behavior can do.
We will see this pattern more times,
so keep it in mind when developing the `terminate/2` callback.

We can also see that we received a lot of details about the crash in the reason parameter.
It can be interesting to report an error if it stops for an unexpected reason.

### Raise an exception

Something very similar will happen if we manually raise an exception:

```elixir
defmodule Raising do
  use GenServer

  @impl true
  def init(state) do
    {:ok, state}
  end

  @impl true
  def handle_info(:raising, state) do
    state = state + 1
    IO.inspect(":raising received, updating state to: #{state}")
    raise "Raising!"
    # Never return
    # {:noreply, state}
  end

  @impl true
  def terminate(reason, state) do
    IO.inspect("terminate/2 callback")
    IO.inspect({:reason, reason})
    IO.inspect({:state, state})
  end
end
```

```elixir
iex> {:ok, pid} = GenServer.start(Raising, 0)
{:ok, #PID<0.150.0>}

iex> send(pid, :raising)
":raising received, updating state to: 1"
:raising
"terminate/2 callback"
{:reason,
 { %RuntimeError{message: "Raising!"},
  [
    {Raising, :handle_info, 2, [file: 'raising.ex', line: 13]},
    {:gen_server, :try_dispatch, 4, [file: 'gen_server.erl', line: 680]},
    {:gen_server, :handle_msg, 6, [file: 'gen_server.erl', line: 756]},
    {:proc_lib, :init_p_do_apply, 3, [file: 'proc_lib.erl', line: 226]}
  ]}}
{:state, 0}
## Error log
"""
18:08:25.685 [error] GenServer #PID<0.150.0> terminating
** (RuntimeError) Raising!
    raising.ex:13: Raising.handle_info/2
    (stdlib 3.13) gen_server.erl:680: :gen_server.try_dispatch/4
    (stdlib 3.13) gen_server.erl:756: :gen_server.handle_msg/6
    (stdlib 3.13) proc_lib.erl:226: :proc_lib.init_p_do_apply/3
Last message: :raising
State: 0
"""
```

### Return :stop in a callback


However, the most appropriate way to stop it, in most cases,
is to return the tuple `{:stop, reason, state}` in almost any callback:

```elixir
defmodule Stop do
  use GenServer

  @impl true
  def init(state) do
    {:ok, state}
  end

  @impl true
  def handle_info({:quit, reason}, state) do
    state = state + 1
    IO.inspect(":quit received, updating state to: #{state}")
    {:stop, reason, state}
  end

  @impl true
  def terminate(reason, state) do
    IO.inspect("terminate/2 callback")
    IO.inspect({:reason, reason})
    IO.inspect({:state, state})
  end
end
```

```elixir
iex> {:ok, pid} = GenServer.start(Stop, 0)
{:ok, #PID<0.160.0>}

iex> send(pid, {:quit, :normal})
{:quit, :normal}
":quit received, updating state to: 1"
"terminate/2 callback"
{:reason, :normal}
{:state, 1}
```

As we can see, the state can be updated this time before calling the callback.

### Kernel.exit/1

It has very similar behaviour to the previous one.
The function sends an exit signal to the current process, in our case, to the GenServer:

```elixir
defmodule Exit do
  use GenServer

  @impl true
  def init(state) do
    {:ok, state}
  end

  @impl true
  def handle_info({:exit, reason}, _state) do
    IO.inspect(":exit received")
    exit(reason)
  end

  @impl true
  def terminate(reason, state) do
    IO.inspect("terminate/2 callback")
    IO.inspect({:reason, reason})
    IO.inspect({:state, state})
  end
end
```

```elixir
iex> {:ok, pid} = GenServer.start(Exit, 0)
{:ok, #PID<0.192.0>}

iex> send(pid, {:exit, :normal})
{:exit, :normal}
":exit received"
"terminate/2 callback"
{:reason, :normal}
{:state, 0}
```

## Important note about the "reason"

In our two last example we could manipulate the "reason" to stop a GenServer.
This reason has an extra meaning for the OTP.
There are three values considered "normal" reason values:

* `:normal`
* `:shutdown`
* `{:shutdown, term}`

What happens when the reason is different from these three values? From the documentation:

> Exiting with any other reason is considered abnormal and treated like a crash. This means the default supervisor behaviour kicks in, error reports are emitted, and so forth.

For example, we can see that an error log is emitted with custom exit signal:

```elixir
iex> {:ok, pid} = GenServer.start(Exit, 0)
{:ok, #PID<0.204.0>}

iex> send(pid, {:exit, :my_own_reason})
{:exit, :my_own_reason}
":exit received"
"terminate/2 callback"
{:reason, :my_own_reason}
{:state, 0}
## Error log
"""
18:34:29.527 [error] GenServer #PID<0.204.0> terminating
** (stop) :my_own_reason
    exit.ex:12: Exit.handle_info/2
    (stdlib 3.13) gen_server.erl:680: :gen_server.try_dispatch/4
    (stdlib 3.13) gen_server.erl:756: :gen_server.handle_msg/6
    (stdlib 3.13) proc_lib.erl:226: :proc_lib.init_p_do_apply/3
Last message: {:exit, :my_own_reason}
State: 0
"""
```

Similar when we use `{:stop, :my_own_reason, state}`:

```elixir
iex> {:ok, pid} = GenServer.start(Stop, 0)
{:ok, #PID<0.196.0>}

iex> send(pid, {:quit, :my_own_reason})
{:quit, :my_own_reason}
":quit received, updating state to: 1"
"terminate/2 callback"
{:reason, :my_own_reason}
{:state, 1}
## Error log
"""
18:31:02.909 [error] GenServer #PID<0.196.0> terminating
** (stop) :my_own_reason
Last message: {:quit, :my_own_reason}
State: 1
"""
```

We can use the `{:shutdown, term}` to add a custom value to the reason without considering it a crash:

```elixir
iex> {:ok, pid} = GenServer.start(Stop, 0)
{:ok, #PID<0.201.0>}

iex> send(pid, {:quit, {:shutdown, :my_own_reason}})
{:quit, {:shutdown, :my_own_reason}}
":quit received, updating state to: 1"
"terminate/2 callback"
{:reason, {:shutdown, :my_own_reason}}
{:state, 1}
## No error log
```

## How a GenServer can be stopped externally

We've already seen how a GenServer can stop itself.
Now, we will see how other processes can do the same thing.

### GenServer.stop/3

This function receives the PID---or any value that can be considered a GenServer name---the reason to stop it and a timeout.
As you can imagine due to the timeout parameter, this function is synchronous,
it won't continue until ensuring that the GenServer completely stops:

```elixir
defmodule GenServerStop do
  use GenServer

  @impl true
  def init(state) do
    {:ok, state}
  end

  @impl true
  def terminate(reason, state) do
    IO.inspect("terminate/2 callback")
    IO.inspect({:reason, reason})
    IO.inspect({:state, state})
    IO.inspect("sleep before terminating")
    Process.sleep(1_000)
    IO.inspect("really terminating")
  end
end
```

```elixir
iex> {:ok, pid} = GenServer.start(GenServerStop, 0)
{:ok, #PID<0.127.0>}
iex> GenServer.stop(pid)
"terminate/2 callback"
{:reason, :normal}
{:state, 0}
"sleep before terminating"
"really terminating"
:ok
```

The default value for `reason` is `:normal` and for `timeout` is `5_000`.
Again, if the reason is a custom one, it will be logged:


```elixir
iex> {:ok, pid} = GenServer.start(GenServerStop, 0)
{:ok, #PID<0.131.0>}
iex> GenServer.stop(pid, :abnormal)
"terminate/2 callback"
{:reason, :abnormal}
{:state, 0}
"sleep before terminating"
"really terminating"

# Error log
"""
09:35:49.180 [error] GenServer #PID<0.131.0> terminating
** (stop) :abnormal
Last message: []
State: 0
"""
:ok
```

If we don't give enough time to terminate, the `GenServer.stop/3` function won't return,
it will crash before returning. However, the `terminate/2` will continue finishing:

```elixir
iex> {:ok, pid} = GenServer.start(GenServerStop, 0)
{:ok, #PID<0.135.0>}
iex> GenServer.stop(pid, :small_timeout, 100)
"terminate/2 callback"
{:reason, :small_timeout}
{:state, 0}
"sleep before terminating"
# Our current iex process exit because timeout, Error log:
"""
** (exit) exited in: GenServer.stop(#PID<0.135.0>, :small_timeout, 100)
    ** (EXIT) time out
    (elixir 1.10.3) lib/gen_server.ex:981: GenServer.stop/3
"""
# The GenServer process finishes anyway some milliseconds later
"really terminating"
# Error log because is not a normal reason:
"""
09:38:39.020 [error] GenServer #PID<0.135.0> terminating
** (stop) :small_timeout
Last message: []
State: 0
"""
```

### Link to another process

When two processes are linked, if one dies the other dies too.
But how exactly does this process work?
Let's start with a simple example:

```elixir
defmodule Link do
  use GenServer

  @impl true
  def init(state) do
    IO.inspect({:init, self()})
    {:ok, state}
  end

  @impl true
  def handle_info({:link, link}, state) do
    Process.link(link)
    {:noreply, state}
  end

  @impl true
  def terminate(reason, state) do
    IO.inspect("terminate/2 callback in pid #{inspect(self())}")
    IO.inspect({:reason, reason})
    IO.inspect({:state, state})
  end
end
```

```elixir
iex> {:ok, pid_1} = GenServer.start(Link, 0)
{:init, #PID<0.120.0>}
{:ok, #PID<0.120.0>}
iex> {:ok, pid_2} = GenServer.start(Link, 0)
{:init, #PID<0.122.0>}
{:ok, #PID<0.122.0>}
iex> send(pid_1, {:link, pid_2})
{:link, #PID<0.122.0>}
iex> GenServer.stop(pid_2, :my_own_reason)
terminate/2 callback in pid #PID<0.122.0>
{:reason, :my_own_reason}
{:state, 0}

# Error log from pid 2:
"""
18:57:10.499 [error] GenServer #PID<0.122.0> terminating
** (stop) :my_own_reason
Last message: []
State: 0
"""
:ok
iex> Process.alive?(pid_1)
false
```

We stopped the `pid_2` and we see it executes the `terminate/2` callback.
The same is not true for the `pid_1`.
The GenServer running with the `pid_1` died because it was linked to `pid_2`.
Interestingly, the `terminate/2` callback has not been called at all for the `pid_1`.

#### Trapping exit signals

One way we can modify this default behaviour is trapping exit signals.
When a process died, the linked processes receives a exit signal.
There are two possibilities then:

* The process doesn't trap the exit signal, so it dies with the same reason.
* The process traps the exit signal, so it is converted in a regular message.

An example of the last case would be:

```elixir
defmodule TrapLink do
  use GenServer

  @impl true
  def init(state) do
    # Trap exits
    Process.flag(:trap_exit, true)
    {:ok, state}
  end

  @impl true
  def handle_info({:link, link}, state) do
    Process.link(link)
    {:noreply, state}
  end

  def handle_info(msg, state) do
    IO.inspect({:handle_info, msg, state})
    {:noreply, state}
  end

  @impl true
  def terminate(reason, state) do
    IO.inspect("terminate/2 callback in pid #{inspect(self())}")
    IO.inspect({:reason, reason})
    IO.inspect({:state, state})
  end
end
```

```elixir
iex> {:ok, pid_1} = GenServer.start(TrapLink, 0)
{:ok, #PID<0.187.0>}
iex> {:ok, pid_2} = GenServer.start(TrapLink, 0)
{:ok, #PID<0.189.0>}
iex> send(pid_1, {:link, pid_2})
{:link, #PID<0.189.0>}
iex> GenServer.stop(pid_2, :normal)
"terminate/2 callback in pid #PID<0.189.0>"
{:reason, :normal}
{:state, 0}
# This message is received by the pid_1
{:handle_info, {:EXIT, #PID<0.189.0>, :normal}, 0}
:ok
iex> Process.alive?(pid_1)
true
```

So the GenServer running in the `pid_1` survived this time.
Because it didn't die so the `terminate/2` callback it wasn't called either.
Instead, it received the `{:EXIT, #PID<0.189.0>, :normal}` message in the `handle_info/2` callback.
The same behaviour happens with not "normal" reasons:

```elixir
iex> {:ok, pid_1} = GenServer.start(TrapLink, 0)
{:ok, #PID<0.198.0>}
iex> {:ok, pid_2} = GenServer.start(TrapLink, 0)
{:ok, #PID<0.200.0>}
iex> send(pid_1, {:link, pid_2})
{:link, #PID<0.200.0>}
iex> GenServer.stop(pid_2, :abnormal)
"terminate/2 callback in pid #PID<0.200.0>"
{:reason, :abnormal}
{:state, 0}

# Error log from pid_2:
"""
19:28:14.982 [error] GenServer #PID<0.200.0> terminating
** (stop) :abnormal
Last message: []
State: 0
"""
# This message is received by the pid_1
{:handle_info, {:EXIT, #PID<0.200.0>, :abnormal}, 0}
:ok
```

Of course, in the `handle_info/2` callback,
we could have decided to stop the GenServer using one of the different methods we previously saw.
The main point, however, it's that **the GenServer doesn't die directly if it traps exit signals when a linked process dies**.
We should do this manually.


### Process.exit/2

We learned about the exit signals.
Any process can send an exit signal to any other process.
To do it we can use the `Process.exit/2` function.
As in the previous example,
the consequences of calling this function will depend on whether or not the signals are trapped:

```elixir
defmodule ExternalExit do
  use GenServer

  @impl true
  def init(state) do
    {:ok, state}
  end

  @impl true
  def handle_info(:trap, state) do
    Process.flag(:trap_exit, true)
    {:noreply, state}
  end

  def handle_info(msg, state) do
    IO.inspect({:info, msg, state})
    {:noreply, state}
  end

  @impl true
  def terminate(reason, state) do
    IO.inspect("terminate/2 callback")
    IO.inspect({:reason, reason})
    IO.inspect({:state, state})
  end
end

```

```elixir
iex> {:ok, pid} = GenServer.start(ExternalExit, 0)
{:ok, #PID<0.156.0>}
iex> Process.exit(pid, :abnormal)
true
iex> Process.alive?(pid)
false
iex> {:ok, pid} = GenServer.start(ExternalExit, 0)
{:ok, #PID<0.160.0>}
iex> Process.exit(pid, :shutdown)
true
iex> Process.alive?(pid)
false
```

And again, the **GenServer doesn't execute the `terminate/2` callback with an exit signal**.
It dies as soon as the signal is received.

On the other side, when the signals are trapped,
it doesn't die unless we manually want it:

```elixir
iex> {:ok, pid} = GenServer.start(ExternalExit, 0)
{:ok, #PID<0.164.0>}
iex> send(pid, :trap)
:trap
iex> Process.exit(pid, :abnormal)
{:info, {:EXIT, #PID<0.111.0>, :abnormal}, 0}
true
iex> Process.alive?(pid)
true
```

When using the function `Process.exit/2`, there are two exceptions to the previous behaviour.
First, when we use `:normal` reason.
For this case, the documentation says:

> If reason is the atom :normal, pid will not exit (unless pid is the calling process, in which case it will exit with the reason :normal). If it is trapping exits, the exit signal is transformed into a message {:EXIT, from, :normal} and delivered to its message queue.

```elixir
iex> {:ok, pid} = GenServer.start(ExternalExit, 0)
{:ok, #PID<0.169.0>}
iex> Process.exit(pid, :normal)
true
iex> Process.alive?(pid)
true
iex> send(pid, :trap)
:trap
iex> Process.exit(pid, :normal)
{:info, {:EXIT, #PID<0.111.0>, :normal}, 0}
true
iex> Process.alive?(pid)
true
```

We can see the process didn't die, even when it wasn't trapping the exit signals!

The other exception is when we use with the `:kill` reason:

> If reason is the atom :kill, that is if Process.exit(pid, :kill) is called, an untrappable exit signal is sent to pid which will unconditionally exit with reason :killed.

```elixir
iex> {:ok, pid} = GenServer.start(ExternalExit, 0)
{:ok, #PID<0.185.0>}
iex> send(pid, :trap)
:trap
iex> Process.exit(pid, :kill)
true
iex> Process.alive?(pid)
false
```

We couldn't trap the `:kill` signal, we sent the message: "You must die now!" :)

### Supervisor stops it

In a regular application, a GenServer is usually supervised by a Supervisor.
The Supervisor is in charge of stopping its GenServer children in the shutdown process.
When the Supervisor wants to stop a child, it sends an exit signal too,
in a very similar way to the previous example did.

We will use `Supervisor.stop/3`, analogous to `GenServer.stop/3` but for Supervisors:

```elixir
defmodule ParentStop do
  use GenServer

  def start_link(_) do
    GenServer.start(__MODULE__, 0)
  end

  @impl true
  def init(state) do
    {:ok, state}
  end

  @impl true
  def handle_info(:trap, state) do
    Process.flag(:trap_exit, true)
    {:noreply, state}
  end

  def handle_info(msg, state) do
    IO.inspect({:info, msg, state})
    {:noreply, state}
  end

  @impl true
  def terminate(reason, state) do
    IO.inspect("terminate/2 callback")
    IO.inspect({:reason, reason})
    IO.inspect({:state, state})
  end
end
```

```elixir
iex> {:ok, pid} = Supervisor.start_link([ParentStop], strategy: :one_for_one)
{:ok, #PID<0.120.0>}
iex> Supervisor.stop(pid, :normal, 5_000)
:ok
```

In this case, the `terminate/2` callback is not called.
Let's try with another reason:

```elixir
iex> {:ok, pid} = Supervisor.start_link([ParentStop], strategy: :one_for_one)
{:ok, #PID<0.131.0>}
iex> Supervisor.stop(pid, :abnormal, 5_000)

# Error log from the GenServer:
"""
19:57:36.026 [error] GenServer #PID<0.131.0> terminating
** (stop) :abnormal
Last message: []
State: {:state, {#PID<0.131.0>, Supervisor.Default}, :one_for_one, {[ParentStop],
%{ParentStop => {:child, #PID<0.132.0>, ParentStop,
{ParentStop, :start_link, [[]]}, :permanent, 5000, :worker, [ParentStop]}}},
:undefined, 3, 5, [], 0, Supervisor.Default, {:ok, { %{intensity: 3, period: 5, strategy: :one_for_one},
[%{id: ParentStop, start: {ParentStop, :start_link, [[]]}}]}}}
"""

# The iex process dies too because it was linked to the Supervisor which dies with :abnormal reason:
"""
** (EXIT from #PID<0.129.0>) shell process exited with reason: :abnormal
"""
```

Same behaviour! Definitely, **when an exit signal is received on a GenServer,
it never runs `terminate/2`**.

In the next example we will trap exit signals to change the behaviour:

```elixir
iex> {:ok, pid} = Supervisor.start_link([TrapParentStop], strategy: :one_for_one)
{:ok, #PID<0.149.0>}
iex> Supervisor.stop(pid, :normal, 5_000)
{:info, {:EXIT, #PID<0.149.0>, :shutdown}, 0}

# Error log after 5 seconds:
"""
** (exit) exited in: GenServer.stop(#PID<0.149.0>, :normal, 5000)
    ** (EXIT) time out
    (elixir 1.10.3) lib/gen_server.ex:981: GenServer.stop/3
"""
```

Well, this time we received the exit signal. We had the chance to react in the `handle_info/2` callback.
Although, we decided that the GenServer should continue returning `{:noreply, state}`.
After the five seconds of timeout, the Supervisor killed our GenServer because it didn't kill itself.

## Conclusion

A GenServer has 100 different ways to die.
We didn't explore all of them, but the most common ones.
Try to use the one that best suits your situation.

As the last recommendation, don't blindly trust the `terminate/2` callback, as it doesn't always run.
If you really need to execute code when a GenServer dies, it is best to create another linked process that captures the exit signals.
The code would be very similar to our "Link process" example.

Happy coding!
