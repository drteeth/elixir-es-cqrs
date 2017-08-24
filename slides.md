class: center, middle

# Intro to Event Sourcing and CQRS
### (And a little DDD)

---
class: center, middle

# Ben Moss
### Bitfield Consulting

[ben@bitfield.co](mailto:ben@bitfield.co)

[github.com/drteeth](https://github.com/drteeth)

[@benjamintmoss](https://twitter.com/benjamintmoss)

???

Hi I’m Ben Moss,
* I currently work as a consultant doing Ruby and Android work at my Shop Bitfield.co.
* I have my sights on Elixir for future work.
* Yell "mumbling" at me if I mumble.

---
class: center, middle

# Intro to Event Sourcing and CQRS
### (And a little DDD)

???

# We're going to be exploring what ES/CQRS are and how they differ from traditional methods.

# With a quick show of hands, Who has looked into either of these patterns?

---
class: center, middle, inverse

# Context: where are we now?

???

First, a bit of context so we have something to constrast against when we define Event Sourcing and CQRS.

---
class:

# SQL & ACID transactions

Pros:
* Well known
* Easy
* Safe
* Strong consistency

Cons:
* Locks and contention
* Doesn't scale that easily

???

# For most backend development, SQL is a solid bet, but it can start to fall down at scale

* Especially when we out-grow a single machine

---
class:

# 3rd normal form
* Provide a canonical source for data and relationships
* Re-combine those into the representations we need
* Often a single monolithic model supports all use cases

<!-- ![Giant ERD](images/big-erd.png) -->

???

We usually keep our data in 3NF as it gives us a nice duplicate-free model.

When we need a different view on our data, we need to build it from this form. Group, Join, etc help us to do this, usually at read time.

---
class:

# Indexing
* Speed up reads by creating alternative indexes behind the scenes
* Alternative structure written to disk that is optimized for particular read patterns
* Requires domain knowledge to write them
* Sacrifices write speed for read speed

???

# When reads get slow, we can add indexes to improve response times.

# Indexing speeds up reads by keeping alternative lookups for specific read patterns.

# This comes at the cost of write speed.

* UI design informs which indexes are important
* Each index adds cost to write time so that strong consistency is always observed.
* Hang on to the idea of multiple, domain-informed representations

---
class:

# Descent into Dante's caching inferno
* One of the 2 hard problems in computer science
* Invalidating is complicated
* Adds a layer of complexity to the app
* Orthogonal to business concerns

???

# Caching is hard and is often used to make up for slow queries.

---
class:

# Searching
* Offloaded onto Solr/Elastic
* Outside of transaction boundary
* No joins!
* Eventually consistent

???

# Searching is another example where we need an alternative representation of our data, but this time it's eventually consistent and can't participate in transactions.

* We often turn to application level callbacks to keep them in sync with our system of record (or database).
* It's also worth mentioning that if we want to join records with the main database, we have to do it at the application level.
* It's important to note that the sync is eventually consistent

---
class:

# Reporting & Analytics
* Batch model
* Often denormalized
* Eventual constiency

???

# Reporting & Analytics are further examples of alternative representations.

---
class: middle, center, inverse
# What are ES and CQRS then?

???

With that in mind, What are Event Sourcing and CQRS?

---
class:

# Event sourcing

* Store the list of changes to a model
* Build that model's current state from those changes

???

# Instead of mutating the current state, keep a journal of changes from the initial state

---
class:

# Command-Query Responsibility Segregation

* Separate Reads from Writes
* Queries which return the current state
* Commands which change that state
* Don't mix the two
* May involve 2 different datastores.

???

# CQRS is about spliting our read and write paths. At the code level and maybe even at the data store level

---
class:

# What is Event sourcing?
> "It’s just a left fold over the history of events!" -- Greg Young

???

# Now that we've introduced the terms, let's dig into Event Sourcing a bit more.

It's creator, Greg Young reduces it to this quote:

---
class:

# What is Event sourcing?
> "It’s just a left fold over the history of events!" -- Greg Young

```elixir
current_state = List.foldl(events, initial_state, fn (e, state) ->
  apply_event(e, to: state)
end)
```

???

# What does that look like? Hmmm...

---
class:

# What is Event sourcing?
> "It’s just a left fold over the history of events!" -- Greg Young

```elixir
current_state = List.foldl(events, initial_state, fn (e, state) ->
  apply_event(e, to: state)
end)
```

Thanks for nothing.

???

# While true, it ignores the effect this has on our system. It's a bit too pithy.

---
class:

# What is Event sourcing?

> "Accountants don’t use pencils, they use pens." -- Greg Young

???

# Next slide

---
class:

# What is Event sourcing?

> "Accountants don’t use pencils, they use pens." -- Greg Young

Events are immutable

---
class:

# What is Event sourcing?

> "Accountants don’t use pencils, they use pens." -- Greg Young

Events are immutable

Generate compensating ones to correct mistakes

???

# Simlar to accounting ledgers, events never change.
# New facts are added that supercede older ones.

---
class:

# Quick example 1: Shopping cart

```elixir
%AddedToCart{ item: "apples" }        # ["apples"]
%AddedToCart{ item: "pears" }         # ["apples", "pears"]
%AddedToCart{ item: "pretzels" }      # ["apples, "pears", "pretzels"]
%RemovedFromCart{ item: "pretzels" }  # ["apples", "pears"]
%CartCheckout{}                       # []
```

* State is derived from applying events
* Keeps the interim steps

???

# Here's a quick example to get a feel for how this works
# They bought apples and pears, but considered buying the pretzels

---
class:

# Quick example 2: Bank account

```elixir
# Events
%AccountOpened{ id: 1, balance: 100 }
%Deposited{ id: 1, amount: 20 }
%AccountOpened{ id: 2, amount: 50 }
%Withdrew{ id: 1, amount: 80 }
%Withdrew{ id: 2, amount: 20 }

# Current state
%Account{ id: 1, balance: 40 }
%Account{ id: 2, balance: 30 }
```

* Events have already happened
* Events are interleaved in time

???

---
class:

# Kinda like...
* Git
* Database replication log
* Journaling file systems

???

# Applying deltas
* If you are behind, you can catch up from the last known spot
* Or even from the start

---
class: middle, center, inverse

# How can we build a system like this?

---
class:

# The pieces
* Commands
* Events
* EventStore
* Command Handlers
* Projections
* Process Managers / Sagas

???

# These are building blocks of the pattern

---
class:

# How it all fits together

![Event Sourcing Diagram](images/event_sourcing_overview.svg)

# Start at command, we'll look at this again in a minute

---
class:

# Commands
* Represents some intent
* Named in the imperative

```elixir
# Open a new account for a user with an initial balance
%OpenAccount {
  account_id: 123,
  user_id: 54321
  initial_balance: 10_000,
}
```

???

Typically IDs are passed in
UUIDs are a popular choice for this reason

---
class:

# Events

* Represents a fact that has happened
* Named in past tense
* Often paired with commands

```elixir
# An account was opened for a user with an initial balance
%AccountOpened {
  account_id: 123,
  initial_balance: 10_000,
  user_id: 54321,
  date: ~N[2017-08-24 18:00:00],
}
```

---
class:

# Event store:

Durable storage for your events. It's stores events in an append-only way.

* Easy & familiar: A SQL table
* Purpose built: EventStore
* Awesome by accident: Kafka/Jocko
* Optimistic concurrency

???

# The Event store, while central is faily simple

Version numbers are often used to guard against concurrency problems

---
class:

# Command Handlers

* Accepts or rejects commands
* Returns a list of events or an error

```elixir
defmodule AccountHandler do
  def handle(&Account{} = account, %OpenAccount{} = command) do
    if account.open do
      {:error, :account_already_open}
    else
      event = %AccountOpened {
        account_id: command.account_id,
        client_id: command.client_id,
      }
      {:ok, [event]}
    end
  end
  def handle(&Account{} = account, %Deposit{} = command) do
    ...
  end
end
```

???

# Command handlers accept commands and turn them into new events

* Validations - Simple and fast only - No DB/ No Blocking
* Error or list of events (probably involving the aggregate)

---
class:

# Projections

* Listen for events from many streams
* Derive new data
* Combine facts sort of like a SQL Join would
* Kind of like a SQL view, but materialized
* Often in service of a specific view
* Denormalized
* Can be Sync or Async
* Eventualy consistent

???

In the same way that tracking the events for 1 model allows us to build up it's current state, we can also combine the events for several models to derive new information.

We call these projections as they project streams of events into new states.


---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_1.svg)

???

* On the left, the read side, we have our event store, which is empty
* We have 2 projections that will tail the event log
* PostIndex will be an index of posts with the comment count
* PostDetail will be a detail view with post and comment bodies
  * PostDetail will focus on a single entry for clarity

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_2a.svg)

???

A post is submitted

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_2b.svg)

???

It's current state looks like this

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_2c.svg)

???

The index projection picks up the event and updates it's cache

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_2d.svg)

???

Followed by the detail projection

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_3a.svg)

???

When a comment is added...

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_3b.svg)

???

Comment count is updated in the index projection,

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_3c.svg)

???

And the body is added to the detail projection

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_4a.svg)

???

Another comment

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_4b.svg)

???

Update the count

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_4c.svg)

???

Add the body

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_5a.svg)

???

A second post is made

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_5b.svg)

???

It's state looks like this

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_5c.svg)

???

A new element is added to the index

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_6a.svg)

???

A comment is made on the 2nd post

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_6b.svg)

???

It's comment count is updated

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_7a.svg)

???

When the 2nd post is deleted...

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_7b.svg)

???

It's status is updated

---
class:

# A Projection

![Event Sourcing Diagram](images/es_cqrs_projection_7c.svg)

???

It's removed from the index, and we're done.

---
class:

# Another Projection


```elixir
%CustomerRegistered { customer_id: 1, name: "Lola the Dog" }
%AccountOpened { account_id: 1, customer_id: 1, initial_balance: 100 }
%Deposited { account_id: 1, amount: 100 }

%CustomerRegistered { customer_id: 2, name: "Mattia Gheda" }
%AccountOpened { account_id: 2, customer_id: 2, initial_balance: 0 }
%Deposited { account_id: 2, amount: 100 }
%Withdrew { account_id: 2, amount: 100 }
```

### Customer roll-up
id | customer_name | balance | status
-- | ------------- | ------- | --------
1 | Lola the Dog | $200.00 | Is a Good dog
2 | Mattia Gheda | $0.00 | Broke

???

# Here the events are projected into a SQL table

* Inserting and updating a SQL table in this example
* Denormalize data for specific use cases
* Note that here you can infer data such as the status column

---
class:

```elixir
defmodule AccountBalanceProjection do

  def handle(&CustomerRegistered{} = e) do
   db.insert(e.customer_id, name: e.name, balance: 0)
  end

  def handle(%AccountOpened{} = e) do
   db.update(e.customer_id, balance: e.initial_balance)
  end

  def handle(%Deposited{} = e) do
    row = db.find(e.customer_id)
    new_balance = row.balance + e.amount
    db.update(e.customer_id, balance: new_balance,
      status: judge(new_balance))
  end

  def handle(&Withdrew{} = e) do
    row = db.find(e.customer_id)
    new_balance = row.balance - e.amount
    db.update(e.customer_id, balance: new_balance,
      status: judge(new_balance))
  end

  def handle(_), do: :ok

  defp judge(balance) do
    # Judge Mattia harshly...
    ...
  end

end
```

???

* This database is logically or physically separate from the command store.
* Single threaded - you are free to do normal read/write SQL stuff here.
* This is the big win here - You can really run with this idea - more later

Questions about projections?

---
class:

# Process Managers / Sagas
* Used to handle business processes
  * Sending email
  * Long-running processes
* Projection + can emit commands
* Eventually consistent
* Lots more to talk about here.

???

# Process Managers are your tool for coordinating aggregates and reacting to events

---
class:

# A Process manager

```elixir
defmodule Welcomer do

  def handle(%CustomerRegistered{} = e) do
    case db.find_account_by(email: e.email) do
      account -> merge(e, account)
      nil -> send_welcome_email(e)
    end
  end

  defp merge(e, account) do
    # return a command
    [%MergeAccount {
      existing_account_id: account.id,
      new_account_id: e.account_id,
      ...
    }]
  end

  defp send_welcome_email(e) do
    db.insert(account) # remember the account for next time
    mailer.send_welcome_email(e.email)
    [] # no commands
  end
end
```

???

* The events have already happened
* Emit new commands to compensate
* Perform side effects

---
class:

# How it all fits together

![Event Sourcing Diagram](images/event_sourcing_overview.svg)

???

* A Command is sent
* The command handler accepts it an generates one or more events
* These events are stored in the store
* The events are published
* A Projection subscribes to one or more events and maintains a projection
* Queries read from projections
* Process managers also subscribe to events
* And they may feed new commands back into the system

#### Blue: Write-side
#### Green: Read-side

---
class:

# Process Managers & Transactions

* Coordinates models
* Can hold state
* Can crash and be restarted
* Can involve other systems unlike SQL transactions

???

# Sometimes the system needs to watch over a series of events and commands and react to what it sees.

---
class:

# Process Managers & Transactions

```elixir
# listen for:
%Transfer.Requested{ tx_id: 1, from: 123, to: 456, amount: 100 }

# emit:
%Account.Withdraw{ account_id: 123, amount: 100, tx_id: 1 }

# listen for:
%Account.Withdrew{ account_id: 123, amount: 100, tx_id: 1 }

# emit:
%Account.Deposit{ account_id: 456, amount: 100, tx_id:1 }

# listen for:
%Account.Deposited{ account_id: 456, amount: 100, tx_id: 1 }

# emit:
%Transfer.Complete{ tx_id: 1, status: :ok }
```

???

# Here we see an example of coordinating 2 aggregates

---
class: middle, center, inverse

# Interlude: Domain Driven Design in 2 minutes

???

# We're going to take a super quick detour here to talk about DDD as many of the things it has to say apply really well to ES/CQRS

---
class:

# Aggregate

Basically a domain model, but with a few restrictions.

---
class:

# Aggregate

```elixir

%Customer {
  id: 123,
  name: "Ben Moss",
}

%Order {
  id: 456,
  customer_id: 123, # Reference to another aggregate,
  items: [
    %Item{ item_id: 1, price: 100 },
    %Item{ item_id: 2, price: 200 },
  ]
}

```

* Order and Customer are Aggregates (in this example)
* Item is not a root, Order owns it.
* May refer to other aggregates by ID only
* You may not hold references to other aggregates

???

---
class:

# Aggregate

* Control access to children
* Provide a transaction boundary
* Maintain invariants
* Serialize access with GenServer

---
class:

# Aggregate

```elixir
defmodule Order do
  use GenServer

  defstruct id: nil, max_cost: 0, total: 0, items: []

  def handle_cast({:create, id, max_cost}, _from, _) do
    {:noreply, %Order{id: id, max_cost: max_cost}}
  end

  def handle_cast({:add_item, item}, _from, order) do
    new_total = order.total + item.cost
    if new_total > order.max_cost do
      raise "freak out"
    else
      new_items = [ item | order.items ]
      {:noreply, %{order | total: new_total, items: new_items}}
    end
  end

  def remove_item(item) do
    # update total, remove the item...
  end
end
```

???

# Order aggregate maintains a consistent state
* Elixir is a really good fit here.

---
class:

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

# This is the alternative

* Non-linear
* Race conditions

---
class: middle, center, inverse

# Why would you want to use ES/CQRS/DDD?

???

---
class:

# Audit Trail / Logging
* How did my model get into this state?
* Guaranteed to be correct & complete

```elixir
# How did we get here??
%GiantModel {
  ...
  expired: true,
  active: true,
  visible: false,
  published: true,
  approved: false
  ...
}
```

???

# Ever look at a record in the database and wonder how it got into the state it's in?

---
class:

# Time-travel debugging


```elixir
date = ~D[2013-08-14]

unspace = Office.hydrate(until: date)
unspace.status => :lounge_mode_in_effect

unspace = unspace |> Office.apply(%PinballRelatedIncidentHappened{})
unspace.status => :literally_on_fire
```

???

# It's easy to see the state of the system at a particular point in history.

# Reports!

---
class:

# Reading from projectsion is fast and simple
* Denormalized
* Joining and Grouping already done
* Tailored to the each use case
* Optimized for reading
* So simple, you don't need an ORM

```sql
-- index page
select * from post_index limit 100;

-- detail page
select * from posts_with_comments_and_authors where post_id = 123;

```

???

# Reading from projections is fast and easy.

---
class:

# Caching

* No longer needed?

---
class:

# Caching

* Not as needed?

---
class:

# Caching

* Just kidding - it is a cache!

---
class:

# Caching

* Just kidding - it is a cache!
* It's a perfect cache

---
class:

# Caching

* Just kidding - it is a cache!
* It's a perfect cache
* That knows exactly when and how to invalidate itself

---
class:

# Feed auxilary services

Projections can be used to keep auxilary services in sync:
* Analytics/Metrics
* Accounting
* Calendars
* Salesforce ><

???

# More possibilities

---
class:

# Read and write sides can be different

### Read side
* Denormalized SQL tables
* NoSQL Documents
* Search Engine queries
* GraphDB queries
* Generate flat text files and serve them statically
* Generate markdown and publish those via jekyll/hugo
* Binary blobs (generated images, serialized protobuf, etc)
* Just keep it in memory - Who needs a disk?

### Write side
* A SQL table

???

# Projecting datas in this way opens up new possibilies

---
class:

# Fast writes
* Append-only
* No contention
* Just the facts

???

# Writes are fast and uncomplicated

---
class:

# Scaling

### Read projections are easy to scale:
* They depend on a stream of immutable events
* Just bring up new instances of the projections
* Replay the events through them to get them up to speed


---
class:

# Scaling

### Writes are harder to scale, but still possible
* Shard on aggregate type
* Shard on aggregate id
* Kafka

???

* Read and write can scale independantly
* Replicating read stores
* Replicating write stores

---
class:

# Features you didn't anticipate

### Un-delete example:
* Delete as usual: %DeleteWidget { id: 123 }
* Create a projection which includes deleted widgets
* Emit %UndeleteWidget { id: 123 }

???

# By storing intent, we don't lose info that we can act on later.

---
class:

# Features you didn't anticipate

### Similarly:
* Undo.
* Versioning
* Publishing

???

---
class:

# Microservices in brief
* Pretty good fit here.
* Instead of services querying each other, just tail and emit events.
* Resilient to upstream outages (they have their own cache)
* Kafka is popular in the space as a distributed log
* Eventually consistent anyway...

???

# Without dragging in the kitchen sink...

---
class:

# Resilient
* So long as event are stored, you can get your state back.
* Easy to back up
* Easy to replicate

???

# Safe and future proof

---
class: middle, center, inverse

# Knee-jerk concerns and misconceptions:

---
class:

### Won't there be like a bajillion events per second?
* People always think about their webserver logs. It's slower than that

---
class:

### Won't there be like a bajillion events per second?
* People always think about their webserver logs. It's slower than that
* But it can still be a lot.

---
class:

### Isn't re-loading all of the events from the beginning of time slow?

---
class:

### Isn't re-loading all of the events from the beginning of time slow?
* Just the events for 1 aggregate

---
class:

### Isn't re-loading all of the events from the beginning of time slow?
* Just the events for 1 aggregate
* Tends to be in the < 1000 range

---
class:

### Isn't re-loading all of the events from the beginning of time slow?
* Just the events for 1 aggregate
* Tends to be in the < 1000 range
* Snapshots are an option

---
class:

### Isn't re-loading all of the events from the beginning of time slow?
* Just the events for 1 aggregate
* Tends to be in the < 1000 range
* Snapshots are an option
* Maybe you have the wrong aggregate boundary

---
class:

### Isn't re-loading all of the events from the beginning of time slow?
* Just the events for 1 aggregate
* Tends to be in the < 1000 range
* Snapshots are an option
* Maybe you have the wrong aggregate boundary
* Avoid God streams (Don't hang everything off of user)

---
class:

### Doesn't this take more disk space?

---
class:

### Doesn't this take more disk space?
* Yep. Deal with it, disk is cheap.

---
class:

### Doesn't async make everything hard?

---
class:

### Doesn't async make everything hard?
* It suuuure can.

---
class:

### Doesn't async make everything hard?
* It suuuure can.
* Not a requirement.

---
class:

### Doesn't async make everything hard?
* It suuuure can.
* Not a requirement.
* Totally reasonable to mix sync and async

---
class: middle, center, inverse

# How do we do it?

---
class:

# Elixir
* GenServer is a great model for an Aggregate.
  * 1 per process
  * Serialize access to a single stream of events
  * In memory cache backed by events
  * Timeout => Stop
* Projections too
* And Process Managers
* And Command Handlers

Great tools for managing concurrency and serialization

???

# More on this next time, but the takeaway is that Elixir is a great fit.

Even though this pattern came from the OOP world, a functional approach really works well and Elixir's flavour works particularly well.

---
class:

# Commanded
### [github.com/slashdotdash/commanded](https://github.com/slashdotdash/commanded)

* Leading library for ES/CQRS in Elixir
* Next talk
* Lots to discuss

???

# We'll look at an implementation with this library next time

---
class:

# Implementing
* You can go slow
* Not an all-or-nothing proposition
  * Only some models
* Fire events from traditional setup
* Start building out projections
* Replace reads on old model with reads on projected data once it matches

???

# Implementing ES/CQRS can be daunting but you can go slowly.

---
class:

## All in?
* Good for larger projects
  * Bye-bye transactions anyway, may as well get something for it
* Good for smaller/medium?
  * Unclear to me still
* Next time:
  * Code + my adventures with Commanded

???


---
class: middle, center, inverse

# Questions?

ben@bitfield.co

Slides: https://drteeth.github.io/elixir-es-cqrs

???

# That's it for me, I hope that gives you a good taste of what ES/CQRS & DDD are about
