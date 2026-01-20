+++
title = "Axon Adventures Part 1 - Event Sourcing"
date = "2026-01-16T00:00:00Z"
draft = false
summary = "We start our pragmatic journey through Axon Framework with Event Sourcing, including real-world use cases, trade-offs, and when it makes sense."
categories = ["posts"]
author = "Vincenzo Santonastaso"
+++


I've been playing with Event Sourcing for a while now, and I'll be honest: the first time I read about it, I thought everything was overcomplicated.
"_Just save the state, why do you need to store every single thing that happened_?" 
But before understanding what it is, I think it's fundamental to understand what problems it tries to solve.

## What is Event Sourcing (really)?

Think about your bank account. When you open your banking app, you see a balance. That's state. But how did you get there? You had deposits, withdrawals, transfers, fees. Those are events.

Most systems only store the current balance. Done. Event Sourcing flips that: it stores every event that ever happened, and derives the current state by replaying them. 
Your balance isn't stored directly but it's calculated from every transaction since you opened the account.

Why would anyone do this? Because in some domains, **the events are the truth**. The history matters. Auditing, compliance, debugging production issues, understanding user behavior, all of that becomes trivial when you have the full event log.

Of course, this sounds insane to most developers at first. "_You're telling me I have to replay thousands of events just to know if a user has enough credit?_" More or less but there are snapshots and optimizations for that.

## When should you use Event Sourcing?

Let's be real: most CRUD apps don't need this. If you're building a basic task manager or a blog platform, just use Spring Data JPA.

Event Sourcing makes sense when audit trails are critical. Let's think about financial systems, healthcare, legal stuff.
Or when the process matters as much as the outcome, like order fulfillment or approval workflows. 
When you need to replay or reprocess past decisions for analytics or ML training data. When business rules change and you need to reinterpret tax calculations, pricing models, and things like that.

When does it make "less" sense? Simple CRUD where the current state is all that matters. High throughput write scenarios with no historical value. Teams not ready for the mental overhead of eventual consistency. Tight deadlines with no time to learn the model.

Trade-offs? You're dealing with more complex queries, more moving parts, more storage. Every event is persisted forever. There's a learning curve you have to deal with.

## Axon Framework in 2 minutes

Axon is a Java framework that makes Event Sourcing and CQRS (Command Query Responsibility Segregation) less painful. Without it, you have to build your own event store, handle command routing, manage projections, and deal with eventual consistency. It's a lot of stuff.

Axon gives you an event store out of the box, aggregates that manage their own consistency, automatic command and query handling, sagas for long-running business processes, and snapshot support so you don't replay all events every time.

It doesn't make Event Sourcing easy, but it makes it survivable. You still have to think differently about your domain model. You still have to deal with eventual consistency. But at least the plumbing is handled.


## Let's see a real use case: Product lifecycle



Let's say you're building a system that manages product orders. A product goes through multiple stages: _reserved_, _confirmed_, _delivered_. Different teams interact with it. Support needs to know why a product got stuck. Finance needs to audit when things were confirmed and payment results.

With traditional JPA entities, you'd probably have something like:

```kotlin
@Entity
class Product(
    @Id val id: String,
    var status: ProductStatus,
    var customerId: String?,
    var amount: BigDecimal?,
    var reservedAt: Instant?,
    var confirmedAt: Instant?,
    var deliveredAt: Instant?,
    var canceledAt: Instant?,
    var cancelReason: String?
)
```

This works until it doesn't. You start adding more fields: `canceledAt`, `canceledReason`, `confirmedBy`, `reservationExpiredAt`. Your entity becomes a bit heavy. Queries get messy. And when someone asks "why did this product get delivered without confirmation?" you have no idea because you only stored the final state.

## Solving the problem with Event Sourcing

Quick terminology crash course, because you'll see these everywhere:

**Aggregate** - The single source of truth for a specific entity. It enforces business rules and decides whether a command is valid. Think of it as a goalkeeper that protects consistency.

**Command** - An instruction to do something ("ReserveProduct", "ConfirmOrder"). It can be rejected if it violates business rules.

**Event** - A fact that already happened ("ProductReserved", "OrderConfirmed"). Events can't be rejected because they're in the past.

**Projection** - A read model built by listening to events. It's optimized for queries, not for business logic. You can have multiple projections for different purposes.

**Saga** - A long-running process that coordinates multiple aggregates and represents a workflow. It listens to events and sends commands to orchestrate complex workflows (like "when product is reserved, charge the credit card, then confirm the order").

With Axon, you model the lifecycle as a series of events. The aggregate replays them to reconstruct its state.

```kotlin
@Aggregate
class ProductAggregate() {
    @AggregateIdentifier
    private lateinit var productId: String
    private var state: ProductState? = null
    private lateinit var customerId: String
    private lateinit var amount: BigDecimal

    @CommandHandler
    constructor(command: ReserveProductCommand) : this() {
        if (state != null) {
            throw IllegalStateException("Product cannot be reserved twice.")
        }
        apply(ProductReservedEvent(command.productId, command.customerId, command.amount))
    }

    @EventSourcingHandler
    fun on(event: ProductReservedEvent) {
        productId = event.productId
        customerId = event.customerId
        amount = event.amount
        state = ProductState.RESERVED
    }

    @CommandHandler
    fun handle(command: ConfirmProductCommand) {
        if (state != ProductState.RESERVED) {
            throw IllegalStateException("Only a RESERVED product can be confirmed.")
        }
        apply(ProductConfirmedEvent(productId))
    }

    @EventSourcingHandler
    fun on(event: ProductConfirmedEvent) {
        state = ProductState.CONFIRMED
    }

    enum class ProductState {
        RESERVED, CONFIRMED, DELIVERED, CANCELED
    }
}
```

See what's happening? Commands express intent. Events express facts. The aggregate enforces rules.
Basically the aggregate receives commands, validates them, and emits events (using `apply(...)`). 

When you load a `ProductAggregate` from the event store, Axon replays events rebuilding the state (using`@EventSourcingHandler` methods). You get full traceability for free (or at least that's the perception).

For queries, you build projections:

```kotlin
@Component
class ProductProjection(private val jdbcTemplate: JdbcTemplate) {

    @EventHandler
    fun on(event: ProductReservedEvent) {
        jdbcTemplate.update(
            "INSERT INTO product_view (product_id, status, customer_id, amount, reserved_at) VALUES (?, ?, ?, ?, ?)",
            event.productId, "RESERVED", event.customerId, event.amount, Instant.now()
        )
    }

    @EventHandler
    fun on(event: ProductConfirmedEvent) {
        jdbcTemplate.update(
            "UPDATE product_view SET status = ?, confirmed_at = ? WHERE product_id = ?",
            "CONFIRMED", Instant.now(), event.productId
        )
    }

    @QueryHandler
    fun handle(query: GetProductQuery): ProductView? {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM product_view WHERE product_id = ?",
            { rs, _ ->
                ProductView(
                    productId = rs.getString("product_id"),
                    status = rs.getString("status"),
                    customerId = rs.getString("customer_id"),
                    amount = rs.getBigDecimal("amount"),
                    reservedAt = rs.getTimestamp("reserved_at")?.toInstant(),
                    confirmedAt = rs.getTimestamp("confirmed_at")?.toInstant()
                )
            },
            query.productId
        )
    }
}
```

The projection listens to events and builds a read model optimized for queries. You can have multiple projections for different use cases. One for the API. One for reporting. One for machine learning.
The read model can be persisted on a table, a NoSQL database, exported to a data warehouse, whatever fits your needs.

Now, what about sagas? Let's say your product reservation needs to trigger a payment, and only after payment succeeds should the product be confirmed. That's cross aggregate coordination. 
A saga listens to `ProductReservedEvent`, sends a `ChargePaymentCommand`, waits for `PaymentSucceededEvent`, then sends `ConfirmProductCommand`. If payment fails, it sends `CancelReservationCommand`. You're orchestrating a multi-step process without coupling your aggregates together. It's messy in traditional architectures-sagas make it explicit and testable.

Here's what the saga looks like:

```kotlin
@Saga
class ProductLifecycleSaga {

    private lateinit var productId: String
    private lateinit var paymentId: String

    @StartSaga
    @SagaEventHandler(associationProperty = "productId")
    fun on(event: ProductReservedEvent, commandGateway: CommandGateway) {
        productId = event.productId
        paymentId = UUID.randomUUID().toString()

        commandGateway.send<Any>(ChargePaymentCommand(
            paymentId = paymentId,
            productId = event.productId,
            customerId = event.customerId,
            amount = event.amount
        ))
    }

    @SagaEventHandler(associationProperty = "productId")
    fun on(event: PaymentSucceededEvent, commandGateway: CommandGateway) {
        commandGateway.send<Any>(ConfirmProductCommand(event.productId))
    }

    @SagaEventHandler(associationProperty = "productId")
    fun on(event: PaymentFailedEvent, commandGateway: CommandGateway) {
        commandGateway.send<Any>(CancelReservationCommand(
            productId = event.productId,
            reason = "Payment failed: ${event.reason}"
        ))
    }

    @SagaEventHandler(associationProperty = "productId")
    fun on(event: ProductConfirmedEvent, commandGateway: CommandGateway) {
        commandGateway.send<Any>(DeliverProductCommand(productId))
    }

    @EndSaga
    @SagaEventHandler(associationProperty = "productId")
    fun on(event: ProductDeliveredEvent) {
    }

    @EndSaga
    @SagaEventHandler(associationProperty = "productId")
    fun on(event: ProductReservationCanceledEvent) {
    }
}
```

The saga starts when a product is reserved, sends a payment charge command, waits for payment success or failure, then either confirms the product (on success) or cancels the reservation (on failure). After confirmation, it triggers delivery. Each step is explicit. Each transition is traceable. If something fails, you know exactly where and why.

And here's the upside: if business rules change, you can rebuild projections from scratch by replaying all events. Try doing that with a relational database where you've been mutating rows for three years.

## Final thoughts

Event Sourcing isn't a silver bullet. It won't make your codebase simpler. It won't make your queries faster. But it will make certain problems like auditing, compliance and complex workflows much more manageable.

Axon Framework handles the boring infrastructure stuff so you can focus on modeling your domain properly. It's opinionated, which is good when you're trying to learn a new paradigm.

Is it overkill for most projects? Absolutely. Should you use it everywhere? Absolutely no. But when you're staring at a tangled mess of status fields and audit data, and someone asks "how did we get here?" that's when Event Sourcing starts to make sense.

I'm still learning this stuff, honestly. Some days I love it. Some days I miss the simplicity of `UPDATE products SET status = 'CONFIRMED'`. But for the right problems, it's an efficient way to model the domain.

One thing I haven't covered here: the whole "separate your reads from your writes" thing. That's CQRS, and it deserves its own journey. 

_Spoiler: it works really well with Event Sourcing, but you don't need one to use the other._


## Want to go deeper?

If you're curious and want to learn more about this stuff, here are some resources that actually helped me:

**Axon Framework Documentation**  
[https://docs.axoniq.io/](https://docs.axoniq.io/)  
The official docs (surprisingly readable).

**Building Event-Driven Microservices**  
by Adam Bellemare (O'Reilly Media)  
This book isn't specifically about Axon, but it's one of the better explanations of event-driven architecture I've read. It covers event sourcing, stream processing, and how to actually structure these systems in production. Less theory, more "here's how you actually build this."

**Domain-Driven Design**  
by Eric Evans (Addison-Wesley)  
The blue book. If you're going to do Event Sourcing seriously, you need to understand DDD.

---

**Code Repository**  
All the code snippets shown in this article are available in a working demo project:  
[https://github.com/vsantona/axon-demo](https://github.com/vsantona/axon-demo)

---
