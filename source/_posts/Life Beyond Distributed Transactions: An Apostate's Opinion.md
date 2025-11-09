---
title: "Note on Life Beyond Distributed Transactions"
date: 2021-06-27 10:00:00
tags: [distributed system]
categories: [tech]
---

## Problem Statement

Real-world application data grows fast. Those data will have to be repartitioned over time. Global serializability then becomes a problem. This introduces challenges in maintaining global serializability. However, user applications often struggle to achieve good performance using distributed transaction algorithms like 2PC (as noted by the author in 2007).

As a result, user applications typically implement custom solutions to coordinate distributed procedures tailored to their systems. This paper discusses a model and guidelines for users to handle data scaling challenges and for infrastructure developers to design new frameworks.

## Assumption

- Application data grows rapidly, but the types of data are limited.
- Application developers write scale-agnostic code.
- Infrastructure provides multiple disjoint scopes of transaction serializability (e.g., single-machine atomicity).
- For messaging, most applications prefer (or infrastructure typically provides) at-least-once delivery but at-most-once acceptance.
- Exactly-once semantics, akin to the TCP protocol, work well for temporary, non-persistent messages in short-lived processes. However, achieving exactly-once semantics for durable, long-lived messages is significantly more complex. Refer to [RIFL](https://web.stanford.edu/~ouster/cgi-bin/papers/rifl.pdf) to see an implementation example).

## Pattern

Given these assumptions, the author proposes a design pattern for a scale-agnostic application layer.

**Entity**

An entity is an addressable user data type where atomicity is maintained. Each entity can be identified by a unique key. The author emphasizes that data referenced by secondary and primary indices typically belong to different entities due to performance considerations (e.g., coordination costs).

**Message**

The message is used to communicate across entities. When the update of entity A and the update of entity B are related (e.g. causality), there will be messages between A to B.
Messaging from entity A to entity B should be outside the transaction of either entity A or B. (For example, a user submits an order. The application updates the order table row A.
If that succeeds, then it needs to modify the balance table row B to charge the user. That may require a message that relates A to B. The messaging procedure should be outside transaction of A to prevent B receives the message before the transaction of A is committed).

**Activity**

The effect of a Message must be idempotent under the assumption of at-least-once-delivery. So the downstream entity needs to maintain some **state** to track those messages. That state is a part of an activity. An application can define more relationships (i.e. activities) between the two entities.
The author thinks if there is messaging between two entities, there should be an activity tracking these two entities. Under the assumption of limited user data types, the number of activities is also limited. So with this pattern, it doesn't mean application developers have to write a lot of code.
Another important state of the activity is the **uncertainty**. For example, we are going to charge the bank account, but before we do that we don't know how much money is in the account. That is uncertainty. The activity needs to save the state of uncertainty. The goal is to resolve the uncertainty. Considering if we are implementing a distributed transaction, normally the uncertainty is resolved along with the "locking", either pessimistic or optimistic. At the user application level, we track them as tentative operations, confirmations (like transaction committed), and cancellations (like transaction aborted).

**Workflow**

Application business logic will be composited by different messages and related activities (which are like building blocks).

### My Thoughts

In 2007, when NoSQL databases were gaining traction, this paper provided a design pattern for implementing complex business logic atop NoSQL databases with single-row atomicity. However, implementing such logic remains more challenging compared to using distributed RDBMSs. Applications must handle activities and their states, build state machines to respond to activity outcomes, and address distributed failures—an inherently labor-intensive process.

Many assumptions in this paper no longer hold in 2021. Today, we have robust solutions like exactly-once semantic messaging and global transactions, which simplify application development. Nevertheless, this pattern remains relevant in certain scenarios.

But in certain areas, this pattern will still be useful.

The first use case I can think of is when the workflow is accessing multiple services/databases. And only accessing a single service supports the atomicity. In some real-world business logics, a human can be an entity as well and can be involved in the workflow (The human interaction will usually take a very long time to process). So you have to break a global transaction into smaller transactions. I came across [7 Use Cases in 7 Minutes Each] (https://www.youtube.com/watch?v=sXGlQruUrWE), which talks about the real use cases by using AWS SWF. AWS SWF has many similar ideas and abstractions to help application developers focus on the business logic, without worrying about workload scaling (not data scaling) and distributed failures. SWF has the concept of activity worker and workflow worker, where the activity is like a simple step of business logic, the workflow is like a decision-maker/coordinator of different activities' execution. Application developers just need to break the business logic into activities and workflows. All other messaging, state durability, and distributed failures are handled by SWF.
