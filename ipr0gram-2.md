## Actor-Model Programming #2

----

Let's recap!

----

The actor-model unifies data structures, functions and concurrency in a single concept.

----

The actor-model was invented by Carl Hewitt in 1973.

![Carl Hewitt](img/carl_hewitt.jpg)

----

Gul Agha's 1985 dissertation developed a foundational model which is the basis of all actor-model programming languages.

![Gul Agha](img/gul_agha.png)

----

Actors make parallel programs easy.

* <!-- .element: class="fragment"--> They combine data, functions, and units of execution.
* <!-- .element: class="fragment"--> They express _composable concurrency_.
* <!-- .element: class="fragment"--> In Pony, they are _safe_ and _consistent_.

----

What are the Pony guarantees?

* <!-- .element: class="fragment"--> Data-race free.
* <!-- .element: class="fragment"--> Causal messaging.
* <!-- .element: class="fragment"--> No global state.
* <!-- .element: class="fragment"--> Type-safe, memory-safe, exception-safe.
* <!-- .element: class="fragment"--> Actors don't crash.
* <!-- .element: class="fragment"--> Actors are garbage collected.
* <!-- .element: class="fragment"--> Ahead-of-time compilation to native code.

----

When are Pony actors garbage collected?

* <!-- .element: class="fragment"--> When their message queue is empty.
* <!-- .element: class="fragment"--> And when the message queue will remain empty forever.
* <!-- .element: class="fragment"--> Pony uses a novel protocol to achieve this without stop-the-world garbage collection.
* <!-- .element: class="fragment"--> Currently, other actor languages require the programmer to manually manage actor lifetime.

----

Why don't Pony actors crash?

* <!-- .element: class="fragment"--> Messages to Pony actors are type-safe. A Pony actor will never receive a message it doesn't understand.
* <!-- .element: class="fragment"--> Pony code is exception-safe. There are no uncaught exceptions.
* <!-- .element: class="fragment"--> Pony code is memory-safe. There are no null pointers or out-of-bounds reads/writes.

----

Can Pony programs crash?

<!-- .element: class="fragment"--> Sadly, yes. How?

* <!-- .element: class="fragment"--> By exhausting all available memory.
* <!-- .element: class="fragment"--> By calling code written in another language that has weaker guarantees.

---

## Massively Parallel Ping Pong

----

Let's modify the PingPong example.

```pony
use "collections"
use "time"
use "options"
use "term"

actor PingPong
  let _out: OutStream
  var _pings: U64 = 0
  var _pongs: U64 = 0

  new create(out: OutStream) =>
    _out = out

  be apply(players: Array[PingPong] val, pings: U64, seed: U64, print: Bool) =>
    let len = players.size()
    var random = seed

    for i in Range(0, pings) do
      try players(random % len).ping(this, print) end
      random = random.hash()
    end

  be ping(from: PingPong, print: Bool) =>
    _pings = _pings + 1
    from.pong(print)
    if print then _out.write(ANSI.green() + "Ping... ") end

  be pong(print: Bool) =>
    _pongs = _pongs + 1
    if print then _out.write(ANSI.red() + "Pong! ") end

actor Main
  new create(env: Env) =>
    var actors = U64(2)
    var pings = U64(3)
    var print = false

    let options = Options(env) +
      ("actors", "a", I64Argument) +
      ("pings", "p", I64Argument) +
      ("print", "", None)

    for opt in options do
      match opt
      | ("actors", let arg: I64) => actors = arg.u64()
      | ("pings", let arg: I64) => pings = arg.u64()
      | ("print", None) => print = true
      end
    end

    let players_iso = recover Array[PingPong](actors) end

    for i in Range(0, actors) do
      players_iso.push(PingPong(env.out))
    end

    let players = consume val players_iso
    var random = Time.nanos().hash()

    for player in players.values() do
      player(players, pings, random, print)
      random = random.hash()
    end
```

<!-- .element: class="fragment"--> Instead of a pair of PingPong actors, this creates any number of them.

<!-- .element: class="fragment"--> They each send some number of pings to their peers, randomly.

----

Let's try 10 actors, each sending 10 pings and receiving 10 pongs...

<!-- .element: class="fragment"--> That's about 200 messages.

----

Let's try 1000 actors, each sending 1000 pings and receiving 1000 pongs...

<!-- .element: class="fragment"--> That's about 2 million messages.

----

Massively parallel programs in Pony are:

* <!-- .element: class="fragment"--> Easy to write.
* <!-- .element: class="fragment"--> Easy to scale.
* <!-- .element: class="fragment"--> Easy to reason about.

---

## Live Coding in Pony

----

Let's extend the adventure game.

----

Concepts:

* Traits.
* Actors.
* Constructors (named, sound).
* Non-null type system.
* Generic containers.
* Single assignment variables.

----

[Example #1](adventure1/adv1.pony)

```pony
actor Person
  let _name: String
  let _things: SetIs[Thing iso]

  new create(name': String) =>
    """
    All fields must be initialised by the time a constructor is done.
    Until all fields are initialised, `this` is treated as a tag.
    """
    _name = name'
    // name()
    _things = SetIs[Thing iso].create()

  new two_names(first: String, last: String) =>
    _name = first + " " + last
    _things = SetIs[Thing iso]

  fun name(): String =>
    _name
```

----

Conveniences:

* Inline field initialisation.
* Implicit constructors.

----

[Example #1a](adventure1a/adv1a.pony)

```pony
actor Person
  """
  When we initialise a field here, it's initialised the same way in every
  constructor.

  Specifying a type without a constructor as an expression implicitly uses the
  default constructor, which is create().
  """
  let _name: String
  let _things: SetIs[Thing iso] = SetIs[Thing iso]

  new create(name': String) =>
    _name = name'

  new two_names(first: String, last: String) =>
    _name = first + " " + last
```

----

Concepts:

* Behaviours have sendable parameters.
* Destructive read.

----

[Example #2](adventure2/adv2.pony)

```pony
use "collections"

actor Person
  let _name: String
  let _things: SetIs[Thing iso] = SetIs[Thing iso]

  new create(name': String) =>
    _name = name'

  be take(thing: Thing iso) =>
  // be take(thing: Thing ref) =>
    """
    Behaviour parameters must be sendable.
    We use `consume` to perform a destructive read.
    """
    _things.set(consume thing)
    // _things.set(thing)
```

----

What's the type of `consume thing`?

<!-- .element: class="fragment"--> `Thing iso^`

<!-- .element: class="fragment"--> The `^` means it's an _ephemeral_ type.

----

An _ephemeral_ type indicates that the previous reference to the object no longer exists.

<!-- .element: class="fragment"--> For a `Thing iso`, we know there is only one `iso` reference (one path) to the object, because it's isolated.

<!-- .element: class="fragment"--> So for a `Thing iso^` we know there are _zero_ `iso` references (no path) to the object!

<!-- .element: class="fragment"--> In fact, we know there are zero _readable_ references to the object.

----

What can we do with an `iso^` that's different from an `iso`?

* <!-- .element: class="fragment"--> We can assign it to an `iso` variable, going from zero `iso` references to one `iso` reference.
* <!-- .element: class="fragment"--> We can send it to another actor, because we know we have not retained an `iso` reference.

----

Concepts:

* Providing a trait.
* Function receiver capabilities.
* Receiver capability subtyping.

----

[Example #3](adventure3/adv3.pony)

```pony
trait Thing
  fun tag name(): String

class CinemaTicket is Thing
  """
  A CinemaTicket provides the Thing trait.
  """
  let film: String

  new create(film': String) =>
    film = film'

  fun tag name(): String =>
  // fun ref name(): String =>
    """
    Since a CinemaTicket is a Thing, it must provide a name() function.
    """
    "cinema ticket"
```

----

Concepts:

* Main actor.
* Capability recovery.

----

[Example #4](adventure4/adv4.pony)

```pony
actor Main
  new create(env: Env) =>
    """
    Here's a program that creates one Person.
    Then we hand her a cinema ticket.
    We use `recover` to create an _isolated_ ticket.
    """
    let alice = Person("Alice")
    let ticket = recover CinemaTicket("Minions") end
    alice.take(consume ticket)
    // alice.take(ticket)
    // alice.take(CinemaTicket("Minions"))
```

----

Concepts:

* Actor constructors are asynchronous.
* Causality.

----

[Example #5](adventure5/adv5.pony)

```pony
actor Main
  new create(env: Env) =>
    """
    Now we have a pub, and we want to put Alice in the pub.
    Because of causality, the first message Alice will ever send is to tell
    the pub she has arrived.
    And that message is a cause of every message Alice ever sends.
    """
    let pub = Place("Pub")
    let alice = Person("Alice", pub)

    // Are actors constructed in order?
    // let bob = Person("Bob", pub)
    // let charlotte = Person("Charlotte", pub)
    // let dave = Person("Dave", pub)
    // let elspeth = Person("Elspeth", pub)
```

----

Conveniences:

* Union types.
* Default arguments.
* Multiple assignment variables.

----

[Example #5a](adventure5a/adv5a.pony)

```pony
actor Person
  let _name: String
  let _things: SetIs[Thing iso] = SetIs[Thing iso]
  var _place: Place

  new create(name: String, place: Place) =>
    _name = name
    _place = place
    _place.arrive(this, place)
    // _place.arrive(this, None)
    // _place.arrive(this)

  be arrived(who: Person, place: Place, from: (Place | None)) =>
    """
    This is a placeholder: here, we would react to someone arriving.
    In this case, we only react to our own arrival.
    """
    if this is who then
      _place = place
    end
```

----

Concepts:

* Aliasing iso as tag.
* Recovery again.
* Destructive read again.

----

[Example #6](adventure6/adv6.pony)

```pony
actor Main
  new create(env: Env) =>
    """
    We start by putting Alice and Bob in the pub.
    We give Alice a cinema ticket.
    Then we tell Alice to give her cinema ticket to Bob.
    """
    let pub = Place("Pub")
    let alice = Person("Alice", pub)
    let bob = Person("Bob", pub)

    let ticket = recover CinemaTicket("Minions") end
    let ticket_id = ticket

    alice.take(consume ticket)
    // alice.take(ticket)
    alice.give(bob, ticket_id)
    // alice.give(bob, ticket)

actor Person
  let _name: String
  let _things: SetIs[Thing iso] = SetIs[Thing iso]
  var _place: Place

  new create(name: String, place: Place) =>
    _name = name
    _place = place
    _place.arrive(this)

  be take(thing: Thing iso) =>
    _things.set(consume thing)

  be give(whom: Person, thing: Thing tag) =>
    try
      let thing' = _extract(thing)
      whom.take(consume thing')
    end

  fun ref _extract(thing: Thing tag): Thing iso^ ? =>
    _things.extract(thing)

  be arrived(who: Person, place: Place, from: (Place | None)) =>
    None
```

----

Concepts:

* Functions on actors.
* Private functions.
* Partial functions.
* Exception handling.
* Ephemeral types.

----

[Example #6a](adventure6a/adv6a.pony)

```pony
actor Person
  let _name: String
  let _things: SetIs[Thing iso] = SetIs[Thing iso]
  var _place: Place

  new create(name: String, place: Place) =>
    _name = name
    _place = place
    _place.arrive(this)

  be take(thing: Thing iso) =>
    _things.set(consume thing)

  be give(whom: Person, thing: Thing tag) =>
    """
    If we have the `thing` we're being asked to give away, extract it from our
    set of things and tell the recipient to take it.
    """
    try
      let thing' = _extract(thing)
      whom.take(consume thing')
    end

  fun ref _extract(thing: Thing tag): Thing iso^ ? =>
  // fun ref _extract(thing: Thing tag): Thing iso^ =>
    """
    Extract the `thing` from our set, returning an _ephemeral_ type.
    What if it isn't there? What value can we return?
    """
    _things.extract(thing)

  be arrived(who: Person, place: Place, from: (Place | None)) =>
    None
```

----

TODO: more examples coming

---

## Actor-Model Variations

----

Didn't you say JavaScript was an actor-model language?

* <!-- .element: class="fragment"--> Yes! A JavaScript program is an actor.
* <!-- .element: class="fragment"--> Seriously! A JavaScript program reacts to events (behaviours) in sequence.
* <!-- .element: class="fragment"--> Typically, there is only one actor.
* <!-- .element: class="fragment"--> But with web workers, you can have more.

----

JavaScript

```js
var pings = 0
var pongs = 0

onmessage = function(e) {
  if(e.type == "ping") { // Our ping behaviour.
    pings += 1 // Changing state.
    e.sender.postMessage({type: "pong", sender: this})
  } else if(e.type == "pong") { // Our pong behaviour.
    pongs += 1 // Changing state.
  }
}
```

----

What are the JavaScript guarantees?

* <!-- .element: class="fragment"--> Messages are ordered between pairs of actors, but not generally.
* <!-- .element: class="fragment"--> Messages aren't type-safe: an actor can be sent a message it doesn't understand.
* <!-- .element: class="fragment"--> Objects in messages are copied, so there are no data races.
* <!-- .element: class="fragment"--> The identity of objects in messages is lost.
* <!-- .element: class="fragment"--> Actors are not garbage collected, and can terminate, so actor references can be invalid.

----

What about Node.js?

* <!-- .element: class="fragment"--> Node.js has a web workers API.
* <!-- .element: class="fragment"--> It's an actor-model language too!

----

Actors on the JVM: Akka

* <!-- .element: class="fragment"--> Akka is a powerful, mature actor-model toolkit for Scala and Java.
* <!-- .element: class="fragment"--> Like Pony and JavaScript, actors are presented as asynchronous objects.

----

Akka

```scala
class PingPong() extends Actor {
  var pings = 0 // State.
  var pongs = 0

  def receive = {
    case PingMessage(that) => // This is our ping behaviour.
      pings += 1 // Changing state.
      that ! PongMessage(this) // Sending a message.
    case PongMessage(that) => // This is our pong behaviour.
      pongs += 1 // Changing state.
    case StopMessage => // Akka actors must be manually terminated.
      context.stop(self)
  }
}
```

----

What are the Akka guarantees?

* <!-- .element: class="fragment"--> Messages are usually ordered between pairs of actors, but not generally.
* <!-- .element: class="fragment"--> Some actors have mailboxes that support out-of-order messaging.
* <!-- .element: class="fragment"--> Messages aren't type-safe: an actor can be sent a message it doesn't understand.
* <!-- .element: class="fragment"--> Objects in messages are __not__ data-race free.
* <!-- .element: class="fragment"--> Actors are not garbage collected, and can crash, so actor references can be invalid.

----

Actors as functions: Erlang

* <!-- .element: class="fragment"--> Erlang has been extensively used in production systems for more than 20 years.
* <!-- .element: class="fragment"--> Actors are presented as asynchronous _functions_ rather than asynchronous _objects_.

----

Erlang

```erl
pingpong(Pings, Pongs) ->
  receive % We are looking for a message.
    {ping, That} ->
      % This is our ping behaviour.
      That ! pong, % Sending a message.
      pingpong(Pings + 1, Pongs); % Changing state.
    pong ->
      % This is our pong behaviour.
      pingpong(Pings, Pongs + 1); % Changing state.
    exit ->
      % Erlang actors must be manually terminated.
      ok
  end.
```

----

What are the Erlang guarantees?

* <!-- .element: class="fragment"--> Messages will be _enqueued_ in order between pairs of actors.
* <!-- .element: class="fragment"--> Messages can be _reacted to_ in any order.
* <!-- .element: class="fragment"--> All data is immutable, so no race conditions.
* <!-- .element: class="fragment"--> But all data in messages is copied anyway.
* <!-- .element: class="fragment"--> Actors are not garbage collected, and can crash, so actor references can be invalid.

----

Erlang allows an actor to _pattern match_ on its queue.

<!-- .element: class="fragment"--> This is why messages can be _reacted to_ in any order.

----

Out-of-order message processing

```erl
func() ->
  receive
    goodbye ->
      io:format("received goodbye~n", []),
      receive
        hello ->
          io:format("received hello~n", [])
      end
  end.

go() ->
  That = spawn(?MODULE, func, []),
  That ! hello,
  That ! goodbye,
  ok.
```

<!-- .element: class="fragment"--> We send hello and then goodbye.

<!-- .element: class="fragment"--> The actor reacts to goodbye first, then hello.

----

Akka and JavaScript pattern match on a single message to determine how to react.

<!-- .element: class="fragment"--> Erlang pattern matches on the _queue_, allowing it to select _which_ message to react to.

----

Summary of Guarantees

Feature            | Pony   | JavaScript   | Akka | Erlang
-------------------|--------|--------------|------|-------
Data-race free?    | Yes    | Copies       | No   | Copies
Message enqueuing? | Causal | Pair-ordered | Pair-ordered  | Pair-ordered
Message handling?  | Causal | Pair-ordered | Usually pair-ordered | Any
Actors GC'd?       | Yes    | No           | No   | No