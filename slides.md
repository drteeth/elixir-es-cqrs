class: center, middle


# Intro to Event Sourcing and CQRS...
### and some DDD... and stuff...

???

* Intro to event sourcing and cqrs.
* Exploring what ES/CQRS are and how they differ from traditional methods.

---

# Ben Moss
* ben@bitfield.co
* https://github.com/drteeth
* http://bitfield.co
* https://twitter.com/benjamintmoss

???

Hi I’m Ben Moss,
* I work as a consultant doing Ruby and Android work at moment at bitfield.co.
* I have my sites on Elixir for future work.
* I've done a lot of things in my past, including c# where a lot of these ideas originated.

---

# The ushe:
* ACID SQL + 3NF
* Joins as the answer to anything
* Indexing to start
* Decent into dante's caching inferno
* Copy data for offline analysis
* Keep Solr/Elastic index up to date (oh hi foreshadowing)
* Abuse PG because it can do everything. SQL! JSON! GRAPH! GIS

???

Define the traditional methods:	Fully transactional atomic database writes in 3rd normal form, (mega-)joins to answer any question. Indexes and Caching for when things get slow. More recently materialized views. Hint at the hell that is active record callbacks. Copy things out to a reporting database. Find a way to sync an external search engine. Dream about using a graph database because the thought of trying to keep another database in sync sounds hellish. Abuse postgres instead. Postgres: That’s about as far as we can reasonably take things.

---

# Motivations:
Avoid duplication
1 giant model to answer all questions
Bloated models
Mega Joins
Always reading from the source (maybe with some caching)

???

   We always reading, a user posts a video once a week, we re-read 4500 times, why not flip that on it’s head and write once, write the read info out

---

# Domain Driven Design in 2 minutes
* Aggregate:
* Transaction boundary *
* Responsible for maintaining invariants *
* May refer to other aggregates by ID only
* Order + Line Item Example
* Came from OOP in 2003

???

First a crash course in DDD:
   Aggregate: Transactional isolation boundary. Show example of Order + Line Items
           Briefly mention that Customer may or may not be part of the aggregate
           Only reference other aggregates by ID.

---

# What are ES and CQRS?
* Simple ideas with big implications
* Store events instead of state.
* Rebuild state from events
* CQRS: Separate Read and Write logic.

???

What are ES + CQRS?
   It’s a simple idea with echoing implications, I don’t want to oversell or pretend it doesn’t have baggage.

---

# Pithy neck-beard answers:
* It’s a left fold over the history of events!
* Accountants don’t use pencils, they use pens.

???

Pithy neck-beard answer:
It’s a left fold over the history of events!

Greg Young. Reductionist, smart but kind of a dick.
Accountants don’t use pencils, they use pens.
Belies the complexity and pitfalls. I understand that people have committed terrible sins in his name, but let’s not pretend this doesn’t change things.

---

   Classic examples: 2)Bank Accounts, 1) Shopping cart



<!--    Analogies:  -->
<!-- ES: Git, DB Replication log, journalling file systems, Materialized views -->

<!-- The pieces: -->
<!--    Commands -->
<!--    Command Handlers -->
<!-- Validations - Simple and fast only - No DB/ No Blocking -->
<!--        Error or list of events (probably involving the aggregate) -->

<!--    Events -->
<!--        Facts. -->
<!--    Projections -->
<!--        Listen for Facts, combine into useable pieces of state to be read -->
<!--    Event Store -->
<!--        Persist  -->
<!--    Sagas/Process Managers -->

<!-- Command => handler => Events => Store => Projection ⇐ Query => command … etc -->

<!-- Why? -->
<!--    Audits -->
<!--    Read Speed (in mem, straight reads, flat files) -->
<!--    Write Speed (Append only, no contention) -->
<!--    Giant, slow join that involves many larger tables that re-does 99% of it’s work everytime it’s run but still somebody has to wait. -->
<!--    Caching - Not needed? Not *as* needed? No reason you can’t do far future expiration with fingerprinting. You have the source of the events… see what i did there? -->
<!--    Can enable experiments easily -->
<!--        New db => new projection in parallel. Does it match the old? No? Fix it. Yes? Cut traffic over to it. -->
<!--    Scaling -->
<!--        Multiple read stores -->
<!--        Projections *should* be independent -->
<!--    Future needs supported -->
<!--    Microservices become possible. More than a few people saying that “Querying” a microservice is asking for trouble. Instead watch a stream of events that is interesting to you. -->
<!--    Brings order to event systems -> -->
<!--        - Uni-directional flow vs willy-willy flow all over  user -> cmd -> events -> projection -> user -->
<!--    Resilient -->

<!--    Common misconceptions: (Move this to the end of the talk) -->
<!--        Server logs => OMG SO FAST - its not like that -->
<!--        1 per aggregate/transactional boundary: much slower and lower count. -->
<!--        User god object in normal code => User god stream in ES. It’s a bad idea no matter where you are. Would you model everything in 1 table in SQL?  -->
<!--        Does not have to by async: -->
<!--            You can make your existing systems async if you want. Would it be fast? Maybe, would it be harder? Fuck yes. -->
<!--        Duplication. Yes, but maybe that’s ok. -->

<!--    CQRS: Hand in Glove. I conflate the two. -->

<!-- How? -->
<!--    DDD Lite (context and aggregates - can also work in oop) -->
<!--    Elixir => 1 GenServer per process -->
<!-- Commanded (Next time) -->
<!-- Events: -->
<!--        Past tense -->
<!--        Use to figure out aggregate bounds + follow them -->
<!--        Mix coarse and fine grained (TeamCheck + Check for each individual) -->
<!--        Just like REST: Model transaction for transfer between two accounts for ex. -  -->
<!--    ddd/cqrs/es thou shalt not list: -->
<!--        Dependent projections -->
<!--        Read from the write side -->
<!--    You can opt in slowly, you don’t have to go all in. -->
<!--        Fire events > recreate your existing models in a new table, compare. -->
<!--        Don’t have to have your whole system use ES or CQRS. Dial it in. -->

<!--    Transactions: Saga/Proc Manager -->
<!--    This is more about aggregates -->

<!-- Golden hammer? Divine Hammer? -->
<!--    Unique emails pitfall -->
<!--    Eventual consistency pitfall -->
<!--    It’s a whole new ballgame - call everything into question. -->
<!--    The zealots… the zealots… -->


<!-- Ok, so all in? -->
<!--    Seems like good for larger projects.You have to give up transaction anyway, may as well get something for it. -->
<!--    Jury is def out for the smaller (1-box) projects. But on the flip side, you can use synchronous to your advantage here. -->


<!-- Check out @eventideproject's Tweet: https://twitter.com/eventideproject/status/895383797470048256?s=15 -->
