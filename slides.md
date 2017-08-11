class: center, middle

Derrr, "How did i get into this state?"
Show a big bloated model with many flags
It's expired, it's active, it's visible, it's published, it's approved
But I can't imagine how it got into this state


---

# Intro to Event Sourcing and CQRS...
### and some DDD... and stuff...

???

* Intro to event sourcing and cqrs.
* Exploring what ES/CQRS are and how they differ from traditional methods.
---

class: center, middle

# Ben Moss
### Bitfield Consulting

[ben@bitfield.co](mailto:ben@bitfield.co)

[github.com/drteeth](https://github.com/drteeth)

[@benjamintmoss](https://twitter.com/benjamintmoss)

???

Hi I’m Ben Moss,
* I work as a consultant doing Ruby and Android work at moment at bitfield.co.
* I have my sites on Elixir for future work.
* I've done a lot of things in my past, including c# where a lot of these ideas originated.

---

# The status quo
### What most of us are doing most of the time.

---

# SQL with ACID transactions
* Strong consistency
* Easy
* Safe
* Totally reasonable for smaller projects
* Locks and contention
* Doesn't scale that well
* Replication logs...

???

* With SQL we get a solid, sane, general purpose solution.
* Things start to get complicated when it's time to scale.
* Things get complicated after we out-grow a single machine.
* We scale with sharding and replication logs.
* Hang on to that idea.

---
# 3rd normal form
* Provide a canonical source for data and relationships
* Re-combine those into the representations we need
* Designed to be storage efficient
* Often a single monolithic model supports all use cases
# TODO add giant ERD pic

???

We store our data in 3NF to avoid data falling out of sync, and when we need a different view of our data we need to re-constitute it from this 3rd normal form.

We attempt to capture all representations and use cases in this single model and this can be the source of complexity. We also often end up with a lowest common denominator where our model struggles to meet all needs.

---

# Indexing
* Speed up reads by creating alternative indexes behind the scenes
* Optimized for particular read patterns
* Requires domain knowledge
* Sacrifices write speed

???

* SQL is great, but when all our joins are slow because of sequencial scans, we add indexs to get our performance back.
* Indexes are alternative representations of our data, optimized for a particular read pattern.
* They require domain knowledge to implement
* Each index adds cost to write time so that strong consistency is always observed.

Hang on to the idea of multiple, domain-informed representations

---

# Decent into dante's caching inferno
* One of the 2 hard problems in computer science
* Invalidating is complicated
* Adds a layer of complexity to the app
* Orthogonal to business concerns

???

When things get really slow with our apps, we often reach for caching. It often solves the problem but can get really complex in pathological cases and is often used to make up for expensive queries.

---

# Searching
* Offloaded onto Solr/Elastic
* Outside of transaction boundary
* No joins allowed
* Eventually consistent.

???

* We turn to search engines to provide our user with the searching they expect.
* These live outside of the safety of our database transactions (yes, I know postgres can do it), and we often turn to application level callbacks to keep them in sync with our system of record (or database).
* It's also worth mentioning that if we want to join records with the main database, we have to do it at the application level.
* It's important to note that the sync is eventually consistent

---

# Reporting
* Copy & Denormalize
* Eventual constiency
* Possibly stale
* Slow

???

When stakeholders ask for reports, we either join the world in a 15 minute query that brings the server to it's knees or we periodically copy our database into an offline copy that we can work with.
Either way, we often spend 95% of the time and energy spent going over the same ground we covered last time.

---

# Alternative/Specialized data storage
* Graph-friendly structures
* GIS
* Etc

???

Similar to the Searching example, a graph or GIS database might be a better solution, but since the idea of syncing data to yet another database is unpaletteable, we often just force it on our SQL databases.

---

# Recap
* Avoid duplication
* 1 model for all use cases
* Mega Joins
* Bloated models
* Always reading

???

* I think most of us are read-heavy, and yet we spend tons of time and energy re-reading from our big model.
* Can we flip that on it's head?

---

# What are ES and CQRS?

### Event sourcing:
* Store events instead of state.
* Rebuild state from events

### Command Query Responsibility Segregation:
* Separate Read and Write logic.

Simple ideas with big implications...

???

---

# Pithy neck-beard answers:
* It’s a left fold over the history of events!
* Accountants don’t use pencils, they use pens.

???

* Greg Young. Reductionist, smart but kind of a dick.
* Belies the complexity and pitfalls.

---

# Quick example 1: Shopping cart

```elixir
AddedToCart("apples")         # ["apples"]
AddedToCart("pears")          # ["apples", "pears"]
AddedToCart("pretzels")       # ["apples, "pears", "pretzels"]
RemovedFromCart("pretzels")   # ["apples", "pears"]
CartCheckout()                # []
```

* State is derived from apply events
* Current state is lossy
* Events have the full picture

???

* Simplified model
* The state of the cart is a product of apply each event in sequence
* Cart state is derived from the events
* Current state is lossy
* They bought apples and pears, but considered buying the pretzels

---

# Quick example 2: Bank account

```elixir
AccountOpened(123, 100)
AccountOpened(456, 50)
Deposited(123, 20)
Withdrew(123, 80)
Withdrew(456, 20)

123 => 40
456 => 30
```

* Events have already happened
* Events are interleaved

???

---

# Kinda like...
* Git
* Database replication log
* Journaling file systems

???

* Apply deltas
* If you are behind, you can catch up from the last known spot
* Or even from the start

---

# The pieces
* Commands
* Command Handlers
* Events
* EventStore
* Projections
* Process Managers / Sagas

???

Let's talk about the main architectural bits that make up the pattern.

---

# Commands
* Represent some intent
* Named in the imperative
* Kinda like form objects

```elixir
# Open a new account for a user with an initial balance
%OpenAccount {
  account_id: 123,
  initial_balance: 10_000,
  user_id: 54321
}
```

???

---

# Command Handlers
# TODO: elixir example

* Accept or reject a command
* {:error, reason}
* {:ok, [event1, event2]}
* re-hydrate aggregate from store
* Simple, fast validations: no blocking i/o

???

* Validations - Simple and fast only - No DB/ No Blocking
* Error or list of events (probably involving the aggregate)
* Must be idempotent so they can be retried

---
# Events

```elixir
# An account was opened
%AccountOpened {
  account_id: 123,
  initial_balance: 10_000,
  user_id: 54321
}
```

* Facts that have happened
* Past tense
* Often paired with commands
* Will often include metadata like timestamp, user etc

---

# Projections / Event handlers
# TODO: Elixir example

* Listen for events from many streams
* Kind of like a SQL view (materialized)
* Derive new data from aggregate streams
* combine facts like a JOIN does
* Often in service of 1 specific view aka denormalized
* Sync / Async

???

* This is the big win here - You can really run with this idea - more later

---
# Event store:

Durable storage for your events. It's stores events in a append-only way.
* Old school: SQL Table
* Purpose built: EventStore / PumpkinDB
* Awesome by accident: Kafka/Jocko

???

---

# Process Managers / Sagas
# TODO: example
* More on this later/next time but basically:
* Projection + can emit commands

???

---
# How it all fits together
# TODO: diagram of the whole thing

Command => handler => Events => Store => Projection ⇐ Query => command … etc

???

---

# Domain Driven Design in 2 minutes

```elixir
%Order {
  # Reference to another aggregate,
  # Customer maintains it's own transaction isolation
  customer_id: 123,

  # Order's responsibility:
  items: [
    %Item{ item_id: 1, price: 100 },
    %Item{ item_id: 2, price: 200 },
  ]
}
```

* Came from OOP in 2003
* Order and Customer are Aggregate Roots
* Item is not
* May refer to other aggregates by ID only

???

* Detour to talk about DDD
* Origins in OOP in 2003, Eric Evans
* Aggregate => Domain model
* Only reference other aggregates by ID.
* Briefly mention that Customer may or may not be part of the aggregate

---

# Aggregate Root
```elixir
# Maintain invariants over it's children
# Order is the root here, LineItems are children
defmodule Order do
  use GenServer

  def create(id, max_cost) do
    # create an order row in the db
  end

  def add_item(item) do
    # freak out if we'd blow the budget
    # create a line_item row in the db
    # update total cost
  end

  def remove_item(item) do
    # remove a line_item row in the db
    # update total cost
  end
end
```

* Control access to children
* Transaction boundary
* Maintain invariants
* Serialize access with GenServer

???

* Elixir is a really good fit here.

---
# Chaos

```elixir
# Go behind Order's back
defmodule Bad do
  def remove_line_item(line_item_id) do
    # reach into the db and subvert Order's ability to maintain invariants
  end
end

```

???

* Non-linear
* Race conditions

---

# Getting the boundaries right is subjective and tricky

* Customer + Order + LineItem
* Customer || Order + LineItem

???

Talk about how this can be different depending on use case

---
---
# Why would you want to do this?

???

---
# Audit Trail / Logging
* Guarnteed to be correct + complete
* Time-travel debugging
show an example of a bloated model that is in a contradictory state.

???

Talk about the importance of being able to know how a model got into a particular state.

---
# Reads
* Very specific to the task (no joining, it's already done)
* Straight reads from SQL with PK
* NoQL Doc
* Search Engine
* GraphDB
* Flat files
* Binary blobs (proto?)
* Just keep it in memory

???

---

# Writes
* Very fast
* Append-only
* No contention
* Single responsibility
* Just the bare facts

???
---
# Reduce read burden
* Eliminate read-time work
* No joins
* All data is local

???

Imaging a big 10 table join that you read out of everytime you hit a page. - You are constantly re-joining and doing the same work over.
* Project the results as the data changes instead

   Giant, slow join that involves many larger tables that re-does 99% of it’s work everytime it’s run but still somebody has to wait.
---

# Caching

* No longer needed?
* Not as needed?
* Just kidding - it is a cache!
* It's a perfect cache

???

Caching - Not needed? Not *as* needed? No reason you can’t do far future expiration with fingerprinting. You have the source of the events… see what i did there?

---

# Enable experiments
* New projection
* Compare to existing in dev
* Cut over after deploy

???

---
# Scalling
* Read projections are incredibly easy to scale
* Tail the event store => keep your own projections
* Writes is trickier but possible:
* Shard keys
* Split on aggregates
* Micro... something...

???

Multiple read stores

---
# Microservices
* In brief:
* Instead of services querying each other, just tail and emit events.
* Very resilient
* Kafka
* What can you do with ACID here anyway?

???

Microservices become possible. More than a few people saying that “Querying” a microservice is asking for trouble. Instead watch a stream of events that is interesting to you.

---
# Resilient
* So long as event are stored, you can get your state back.
* Easy to back up
* Easy to replicate
???
---
# Knee-jerk concerns and misconceptions:
* Mega stream with 30,000 events/s !@#?
* => Split into streams with much lower event counts (Think of a user profile, 100s of events/yr? God objects => God streams
* Re-loading it will take forever?
* Reloading 1 aggregate from 1000 events is not slow. Snapshot if you must.
* Disk space? => Yep it takes more, deal with it.
* Async everything ?!
* Definately harder, just like everywhere else.

???

Common misconceptions: (Move this to the end of the talk)
Server logs => OMG SO FAST - its not like that
1 per aggregate/transactional boundary: much slower and lower count.
User god object in normal code => User god stream in ES. It’s a bad idea no matter where you are. Would you model everything in 1 table in SQL?
Does not have to by async:
You can make your existing systems async if you want. Would it be fast? Maybe, would it be harder? Fuck yes.
Duplication. Yes, but maybe that’s ok.

---

# How do we do this?
... in Elixir plz.

???

Haven't talked about elixir much so far

---
# Elixir
* GenServer is a great model for an Aggregate.
* 1 per process
* Serialize access to a single stream of events
* In memory cache backed by events
* Timeout => back to sleep
* Likewise projections and process managers are well modelled as genservers
* And command handlers

???

Serialization as transactions.
---
# Commanded
# TODO: url
Leading library for ES/CQRS in Elixir
* Next talk
* Lots to discuss

???

---

Dos:
* Mix of Coarse and fine grain events:
* It's ok to overlap
* A lot like REST, model the missing things. Model a transfer as an aggregate like you might with a REST action. Session etc

???

---

Don'ts:
* Have projections depend on each other - Prefer duplication
* Read from the Write side. Wat... Oh here we go...

???

---

# Implementing
You can go slow:
Fire events from traditional setup
Start building out projections
Replace reads on old model with reads on projected data once it matches
Replace aggregate state with events

???

You can opt in slowly, you don’t have to go all in. Fire events > recreate your existing models in a new table, compare.
Don’t have to have your whole system use ES or CQRS.
Dial it in

---

# Transaction across aggregates

# How?
* Saga or Process Managers
* Long-running process
* Compensating actions from rollback
  * Refund
  * Cancel
  * Gift
  * Apology
  * Flag a human
  * Etc

???

Talk about giving up ACID and 2PC

---
# The Ugly:
> "I'm just looking for one divine hammer / I'd bang it all day "

* Very real complexity
* Did you hear me say Eventual Consistency?
  Yes to all of those alarm bells
* It's a whole new ballgame
  re-learn, throw out all your assumptions right out of the gates, common tasks are made brutal.
  * Send a flippin' email.
  * Validate a unique username

* There are compromises but you will question your sanity.
 Unique emails coliding? => Unlikely!, detect later?!

* The ES/CQRS crowd
- It's really hard to know which way is up
- Thou shalt nots

???

---

# Conclusion
## All in?
* Good for larger projects
* bye bye transactions anyway, may as well get something for it
* Good for smaller/medium? unclear to me still, but you can also scale back your implementation (synchronus, use ACID, etc)
* Next time: Code + my adventures with commanded

???

Ok, so all in?
   Seems like good for larger projects.You have to give up transaction anyway, may as well get something for it.
   Jury is def out for the smaller (1-box) projects. But on the flip side, you can use synchronous to your advantage here.
---
