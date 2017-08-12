class: center, middle

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
* I have my sights on Elixir for future work.
* I've done a lot of things in my past, including c# where a lot of these ideas originated.

---
class: center, middle, inverse

# Motivation: where are we now?

---
class: middle

# SQL with ACID transactions
* Strong consistency
* Easy
* Safe
* Totally reasonable for smaller projects
* Locks and contention
* Doesn't scale that well

Replication logs... hmmm...

???

* With SQL we get a solid, sane, general purpose solution.
* Strong consistency, it's easy, it's safe and totally reasonable for smaller projects
* Things start to get complicated when it's time to scale.
* Especially when we out-grow a single machine
* We scale by sharding
* We scale by using leader/follower replication
* Hang on to that idea.

---
class: middle

# 3rd normal form
* Provide a canonical source for data and relationships
* Re-combine those into the representations we need
* Designed to be storage efficient
* Often a single monolithic model supports all use cases

<!-- ![Giant ERD](images/big-erd.png) -->

???

We store our data in 3NF to avoid data falling out of sync, and when we need a different view of our data we need to re-constitute it from this 3rd normal form.

We attempt to capture all representations and use cases in this single model and this can be the source of complexity. We also often end up with a lowest common denominator where our model struggles to meet all needs.

---
class: middle

# Indexing
* Speed up reads by creating alternative indexes behind the scenes
* Alternative structure written to disk that is optimized for particular read patterns
* Requires domain knowledge to write them
* Sacrifices write speed for read speed

???

# Indexing

Sequentially reading from source tables is slow, so we use indexing

* Speed up reads by creating alternative indexes behind the scenes
* Alternative structure
  * Separate map of the documents based on some field
* Optimized for particular read patterns
* Requires domain knowledge
  * UI design informs which indexes are important
* Sacrifices write speed
  * Each index adds cost to write time so that strong consistency is always observed.

Hang on to the idea of multiple, domain-informed representations

---
class: middle

# Descent into Dante's caching inferno
* One of the 2 hard problems in computer science
* Invalidating is complicated
* Adds a layer of complexity to the app
* Orthogonal to business concerns

???

* When things get really slow with our apps, we often reach for caching.
* It often solves the problem but can get really complex in pathological cases and is often used to make up for expensive queries.

---
class: middle

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
class: middle

# Reporting & Analyics
* Batch model
* Often denormalized
* Eventual constiency
* Possibly stale
* Slow

???

When stakeholders ask for reports, we either join the world in a 15 minute query that brings the server to it's knees or we periodically copy our database into an offline copy that we can work with.
Either way, we often spend 95% of the time and energy spent going over the same ground we covered last time.

---
class: middle

# Alternative/Specialized data storage
* Graph-friendly structures
* GIS
* Often an all-or-nothing proposition

???

Similar to the Searching example, a graph or GIS database might be a better solution, but since the idea of syncing data to yet another database is unpaletteable, we often just force it on our SQL databases.

---
class: middle, center, inverse
# What are ES and CQRS then?

---
class: middle, center

# Event sourcing
##### Store events instead of state.
##### Rebuild state from events

---
class: middle, center

# Command-Query Responsibility Segregation

##### Separate Reads from Writes

???

---
class: middle

# What is Event sourcing?
> "It’s just a left fold over the history of events!" -- Neckbeard McGee

---
class: middle

# What is Event sourcing?

```elixir
List.foldl(events, initial_state, fn (e, state) ->
  apply_event(e, to: state)
end)
```

???

* Greg Young. Reductionist, smart but kind of a dick.
* Belies the complexity and pitfalls.

---
class: middle

# What is Event sourcing?

Thanks for nothing.

---
class: middle

# What is Event sourcing?

> "Accountants don’t use pencils, they use pens." -- Neckbeard McGee

---

class: middle

# What is Event sourcing?

Events are immutable

Generate compensating ones to correct mistakes

???

Simlar to accounting ledgers, events never change. new facts are added the replace older ones.

---
class: middle

# Quick example 1: Shopping cart

```elixir
%AddedToCart{ item: "apples" }        # ["apples"]
%AddedToCart{ item: "pears" }         # ["apples", "pears"]
%AddedToCart{ item: "pretzels" }      # ["apples, "pears", "pretzels"]
%RemovedFromCart{ item: "pretzels" }  # ["apples", "pears"]
%CartCheckout{}                       # []
```

* State is derived from applying events
* Current state is lossy
* Events have the full picture

???

* Simplified model of a shopping cart
* Add apples, pears and pretzels, then realized that pretzels are nasty and took them out
* The state of the cart is the product of apply each event in sequence
* Cart state is derived from the events
* Current state is lossy
* They bought apples and pears, but considered buying the pretzels

---
class: middle

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
class: middle

# Kinda like...
* Git
* Database replication log
* Journaling file systems

???

* Apply deltas
* If you are behind, you can catch up from the last known spot
* Or even from the start

---
class: middle, center, inverse

# How does it work?

---
class: middle

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
class: middle

# How it all fits together

![Event Sourcing Diagram](images/event_sourcing_overview.svg)

---
class: middle

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
class: middle

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
class: middle

# Command Handlers

* Hydrate ~~aggregate~~ model from store
* Accept or reject the command
* Based on simple validations
* Must be idempotent

```elixir
defmodule AccountHandler do
  def handle(%OpenAccount{} = command) do
    # Load the events for this account from the store
    stream = "account_#{command.account_id}"
    events = event_store.load_events(stream)

    # replay the events (re-hydrate the account)
    account = List.foldl(events, %Account{}, &Account.apply/2)

    # now use it to validate the incoming command
    if account.open do
      {:error, :account_already_open}
    else
      # return 0, 1, or many events
      event = %AccountOpened { account_id: 123 }
      {:ok, [event]}
    end
  end
  def handle(%Deposit{} = command), do: []
end
```

???

Command handlers accept commands and turn them into events

* Validations - Simple and fast only - No DB/ No Blocking
* Error or list of events (probably involving the aggregate)
* Must be idempotent so they can be retried

---
class: middle

# Projections / Event handlers

* Listen for events from many streams
* Derive new data
* Combine facts sort of like a SQL Join would
* Kind of like a SQL view, but materialized
* Often in service of a specific view
* Denormalized
* Can be Sync or Async
* Eventualy consistent.

---
class: middle

# Combine streams to derive new states


```elixir
%CustomerRegistered { customer_id: 1, name: "Lola Gheda" }
%AccountOpened { account_id: 1, customer_id: 1 }
%Deposited { account_id: 1, amount: 200 }

%CustomerRegistered { customer_id: 2, name: "Mattia Gheda" }
%AccountOpened { account_id: 2, customer_id: 2 }
%Deposited { account_id: 2, amount: 100 }
%Withdrew { account_id: 2, amount: 100 }
```

### Customer roll-up
id | customer_name | balance | status
-- | ------------- | ------- | --------
1 | Lola Gheda | $200.00 | Good dog
2 | Mattia Gheda | $0.00 | Broke

???

* Inserting and updating a SQL table in this example
* Denormalize data for specific use cases
* Note that here you can infer data such as the status column

---
class: middle

```elixir
defmodule AccountBalanceProjection do

  def handle(&CustomerRegistered{} = e) do
   db.insert(e.customer_id, name: e.name, balance: 0)
  end

  def handle(%AccountOpened{} = e) do
   db.update(e.customer_id, balance: 0)
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

---
class: middle

# Event store:

Durable storage for your events. It's stores events in an append-only way.

* Easy & familiar: A SQL table
* Purpose built: EventStore
* Awesome by accident: Kafka/Jocko

???

---
class: middle

# Process Managers / Sagas
* Used to handle business processes
  * Sending email
  * Long-running processes
* Projection + can emit commands
* Guard side effects against replays
* Eventually consistent
* Lots more to talk about here.


---
class: middle

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
    mailer.send_welcome_email(e.email) # Don't do this on replay!
    [] # no commands
  end
end
```

???

* The events have already happened
* Emit new commands to compensate
* Perform side effects
* Beware of replays and side effects

---
class: middle

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
class: middle, center, inverse

# Interlude: Domain Drive Design in 2 minutes

---
class: middle

# DDD Crash Course

```elixir

%Customer {
  id: 123,
  name: "Ben Moss",
}

%Order {
  customer_id: 123, # Reference to another aggregate,
  items: [
    %Item{ item_id: 1, price: 100 },
    %Item{ item_id: 2, price: 200 },
  ]
}

```

* Came from OOP in 2003
* Order and Customer are Aggregate Roots (in this example)
* Item is not a root, Order owns it.
* May refer to other aggregates by ID only
* You may not hold references to other aggregates

???

* Detour to talk about DDD
* Origins in OOP in 2003, Eric Evans
* Aggregate => Domain model
* Only reference other aggregates by ID.
* Briefly mention that Customer may or may not be part of the aggregate

---
class: middle

# Aggregate Root

* Control access to children
* Transaction boundary
* Maintain invariants
* Serialize access with GenServer

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

???

* Elixir is a really good fit here.

---
class: middle

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
class: middle, center, inverse

# Why would you want to use ES/CQRS/DDD?

???

---
class: middle

# Audit Trail / Logging
* How did my model get into this state?
* Guaranteed to be correct & complete
* Breadcrumbs
* Bug in the process manager? Fix + replay

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

Talk about the importance of being able to know how a model got into a particular state.

---
class: middle

# Time-travel debugging


```elixir
date = ~D[2013-08-14]

unspace = Office.hydrate(until: date)
unspace.status => :lounge_mode_in_effect
unspace.people_inside => [:eric]

unspace = unspace |> Office.apply(%PinballRelatedIncidentHappened{})
unspace.status => :literally_on_fire
unspace.people_inside => [:eric]

unspace = unspace |> Office.apply(%BossModeEngaged{hero: :eric})
unspace.status => :even_more_on_fire
unspace.people_inside => []
```

???

It's easy to see the state of the system at a particular point in history.

---
class: middle

# Fast and simple reads
* Tailored to the each use case
* Denormalized
* Optimized for reading
* Like ViewModels
* No need for an ORM

```sql
-- index page
select * from post_index limit 100;

-- detail page
select * from posts_with_comments_and_authors where post_id = 123;

````

---
class: middle

# Read side and write side can be different DBs:
* Denormalized SQL tables
* NoSQL Documents
* Search Engine queries
* GraphDB queries
* Generate flat text files and serve them statically
* Generate markdown and publish those via jekyll/hugo
* Binary blobs (generated images, serialized protobuf, etc)
* Just keep it in memory - Who needs a disk?

???

Imaging a big 10 table join that you read out of everytime you hit a page. - You are constantly re-joining and doing the same work over.
* Project the results as the data changes instead

Giant, slow join that involves many larger tables that re-does 99% of it’s work everytime it’s run but still somebody has to wait.

---
class: middle

# Feed auxilary services

Projections can be used to keep auxilary services in sync:
* Analytics/Metrics
* Accounting
* Calendars
* Salesforce ><

---
class: middle

# Fast writes
* Append-only
* No contention
* Just the facts
* Defer expensive work until we've accepted the write
  * Allows the system to move on to the next write
  * Allows the caller to move on if they want
???

---
class: middle

# Caching

* No longer needed?

---
class: middle

# Caching

* Not as needed?

---
class: middle

# Caching

* Just kidding - it is a cache!

---
class: middle

# Caching

* Just kidding - it is a cache!
* It's a perfect cache

---
class: middle

# Caching

* Just kidding - it is a cache!
* It's a perfect cache
* That knows exactly when and how to invalidate itself

---
class: middle

# Scaling

### Read projections are easy to scale:
* They depend on a stream of immutable events
* Just bring up new instances of the projections
* Replay the events through them to get them up to speed

---
class: middle

# Scaling

### Writes are harder to scale, but still possible
* Shard on aggregate type
* Shard on aggregate id

???

* Read and write can scale independantly
* Replicating read stores
* Replicating write stores

---
class: middle

# Microservices in brief
* Pretty good fit here.
* Instead of services querying each other, just tail and emit events.
* Very resilient
* Events can be shared with an event bus (Kafka is popular in the space)
* You are giving up consistency anyway
* DDD: Bounded Context

???

---
class: middle

# Resilient
* So long as event are stored, you can get your state back.
* Easy to back up
* Easy to replicate

???

---
class: middle, center, inverse

# Knee-jerk concerns and misconceptions:

---
class: middle

### Won't there be like a bajillion events per second?
* People always think about their webserver logs. It's slower than that
* But it can still be a lot.

---
class: middle

### Isn't re-loading all of the events from the beginning of time slow?
* Just the events for 1 aggregate
* Tends to be in the < 1000 range
* Snapshots are an option
* Maybe you have the wrong aggregate boundary
* Avoid God streams (Don't hang everything off of user)

---
class: middle

### Doesn't this take more disk space?
* Yep. Deal with it, disk is cheap.

---
class: middle

### Doesn't async make everything hard?
* It suuuure can. Not a requirement.
* Totally reasonable to mix sync and async

---
class: middle, center, inverse

# How do we do it?

---
class: middle

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

Serialization as transactions.

---
class: middle

# Commanded
### [github.com/slashdotdash/commanded](https://github.com/slashdotdash/commanded)

* Leading library for ES/CQRS in Elixir
* Next talk
* Lots to discuss

???


---
class: middle

# Implementing
* You can go slow:
* Fire events from traditional setup
* Start building out projections
* Replace reads on old model with reads on projected data once it matches
* Replace aggregate state with events

???

You can opt in slowly, you don’t have to go all in. Fire events > recreate your existing models in a new table, compare.
Don’t have to have your whole system use ES or CQRS.
Dial it in

---
class: middle

# Coordinating multiple aggregates

### Q: Serialization is not guaranteed across aggregates, so how do we handle 2 aggregates?

### A: Model the interaction between the two as it's own aggregate
That gets us our linear access to the events involved and we can issue commands against the other aggregates

```elixir
%FundTransferRequested{ transaction_id: 1, from: 123, to: 456, amount: 100 }
%Withdrew{ account_id: 123, transaction_id: 1, amount: 100 }
%Deposited{ account_id: 456, transaction_id: 1, amount: 100 }
%FundTransferCompleted { transaction_id: 1 }
```

---
class: middle

# Failure

If something goes wrong, you have the history of what happened and can correct by issue compensating
events. (Yes that's much easier said than done)

* Refund
* Cancel
* Gift
* Apologize
* Flag a human
* Etc

???

Talk about giving up ACID and 2PC

---
class: middle

# The Ugly:

* Very real complexity
* Did you hear me say Eventual Consistency?
  * Yes to all of those alarm bells

* Throw out all your assumptions, common tasks are made brutal.
  * Send a flippin' email.
  * Validate a unique username

* The ES/CQRS crowd
* It's really hard to know which way is up
* Thou shalt nots

???

---
class: middle

## All in?
* Good for larger projects
  * Bye-bye transactions anyway, may as well get something for it
* Good for smaller/medium?
  * Unclear to me still
  * Not all-or-nothing (Sync, only some models, etc)
* Next time: Code + my adventures with Commanded

???

Ok, so all in?

Seems like good for larger projects.You have to give up transaction anyway, may as well get something for it.

Jury is def out for the smaller (1-box) projects. But on the flip side, you can use synchronous to your advantage here.

---
class: middle, center, inverse

# Questions?

ben@bitfield.co

Slides: https://drteeth.github.io/elixir-es-cqrs
