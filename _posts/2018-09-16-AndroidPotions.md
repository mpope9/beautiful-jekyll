---
layout: post
title: Elixir On Android Part 1
maintitle: Using elixir on android
tags: [android, java, elixir, erlang, september, 2018]
---

My eyes snap open.  The room is dark and the wind, who has sauntered into my room finding a new home, is flapping the blackout curtains wildly.  A cold chill is running down my spine.  I had a dream, an Android that learned alchemy.  I must teach it how to `mix` potions.  How to share its cookies with Elixir.  There is work to be done.

To get started, we'll build an addition calculator that only accepts `x + y`.  A simple GenServer<sup>[[1]](https://hexdocs.pm/elixir/GenServer.html)</sup> will work wonderfully for our example.  Plugging this into a `mix` project and a `Supervisor` to register its name locally as `:calculator` is quite easy<sup>[[2]](https://hexdocs.pm/elixir/Supervisor.html#content)</sup>.

```elixir
defmodule JavaTest.Calculator do
   @moduledoc """
   A simple, simple calculator.
   """

   use GenServer

   def start_link(opts) do
      {} = GenServer.start_link(__MODULE__, :ok, opts)
      {:ok, pid}
   end

   def init(:ok) do
      {:ok, %{}}
   end

   def handle_call({:add, lvalue, rvalue}, _from, state) do
      IO.puts "#{lvalue + rvalue}"
      {:reply, lvalue + rvalue, state}
   end
end
```

To breath life into it, sit cross-legged with index finger tips touching pinky finger tips.  Repeat after me
```bash
iex --sname liu-kang --erl "-setcookie test-cookie" -S mix run
```
Some explination of the incantation:
*  `--sname`: is the name of our node.
*  `--erl`: setting things up in the way of our ancestors.
*  `-setcookie test-cookie`: insecure, but each of our code needs have this anchor pont.

Next we must brew some Java to `mix` with our Elixr before we assemble our Android.

Our two languages are distant, their ways of life are incompatble.  The `JInterface`<sup>[[3]](http://erlang.org/doc/apps/jinterface/jinterface_users_guide.html)</sup>, provided in our erlang installation is the hermit who'll play translator.  Mine resided quitetly and full of exceitement, waiting to be utilized in `/usr/lib/erlang/lib/jinterface-1.9/priv`.  Yours is waiting too, just as quietly but as equally excited.

```java
import com.ericsson.opt.erlang.*
```
Is how we'll summon our little hermit.


There are two known paths that we can take.  But only one is correct going forward.
We can attempt to use the `epdm` and `rpc` calls as the channel of communication, but I am unsure if we can bring this into the strange, distant lands we are going with Android.
So we must use the mail boxes to do our communication.

What we first need to do is create an erlang node, set its cookie, then create a mailbox for it.

```java
public class JavaElixirSender {
    private final String ELIXIR_NAME = "liu-kang@warriorShrine";
	private final String NODE_NAME = "goro";
	private final String MBOX_NAME = "goro-mailbox";
	private final String COOKIE = "test-cookie";
	private final String PROCESS_NAME = "calculator";

	static void start() throws IOException, 
				   OtpAuthException, 
				   InterruptedException {
		OtpNode node = new OtpNode(NODE_NAME);
		node.setCookie(COOKIE);
		OtpMbox mailBox = node.createMbox(MBOX_NAME);
	}

	public static void main (String [] args) throws 
			IOException, 
			OtpAuthException, 
			InterruptedException {
		start();
	}
}
```

Here is our desired message format to send to our GenServer.  While in Elixir, we can simply call this from the remote console, when registering the process name as `:calculator`:
```elixir
GenServer.call(:calculator, {:add, 1, 2})
```

We must imagine what this looks like, in its most primordial state:

```erlang
{'$gen_call', {self(), make_ref()}, {:add, 1, 2}}
```

We must not be afraid to craft this in Java.  We have the tools and the materials<sup>[[4]](http://erlang.org/pipermail/erlang-questions/2010-March/050245.html)</sup>
```java
OtpErlangObject[] tag = new OtpErlangObject[2];
tag[0] = mailBox.self();
tag[1] = node.createRef();

OtpErlangObject[] data = new OtpErlangObject[3];
data[0] = new OtpErlangAtom("add");
data[1] = new OtpErlangInt(1);
data[2] = new OtpErlangInt(2);

OtpErlangObject[] body = new OtpErlangObject[3];
message[0] = new OtpErlangAtom("$gen_call");
message[1] = new OtpErlangTuple(tag);
message[2] = new OtpErlangTuple(data);

OtpErlangTuple message = new OtpErlangTuple(body);
```

We have provided all that is needed.  Now, lets see if these two deamons would like to speak:
```java
mailBox.send(PROC_NAME, ELIXIR_NODE_NAME, message);
OtpErlangObject response = mailBox.receive(1000);

if(response != null) {
	OtpErlangTuple responseTuple = (OtpErlangTuple) response;
	System.out.println(responseTuple.elementAt(1).toString());
}
```

What I hear is harmony between them.  The addition is returned.  All is well.

Look out for part 2, where I'll find out if its even possible to continue forward :)


Keep it real,

Matthew
