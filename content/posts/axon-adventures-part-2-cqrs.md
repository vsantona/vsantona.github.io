+++
title = "Axon Adventures Part 2 - Command Query Responsibility Segregation (CQRS)"
date = "2026-01-16T00:00:00Z"
draft = false
summary = "Our journey continues through CQRS with Axon Framework, how and when to separate reads and writes without losing your mind."
categories = ["posts"]
author = "Vincenzo Santonastaso"
+++

The first time someone brought up CQRS in a design discussion, my reaction was pretty much the same I had with Event Sourcing: *this feels like an over-engineered solution to a problem we probably don’t have*.

After actually using it in a few real systems, my opinion changed. Not because CQRS is magic, but because, when applied with intent, it’s often just a way to stop your persistence model from collapsing under the weight of read requirements.

CQRS has a reputation problem. Mention it in a meeting and half the room assumes you’re proposing six months of refactoring. The other half mentally checks out. In reality, you’ve probably already implemented something CQRS stuff without realizing it.

I talked about Event Sourcing recently, and a few people asked about CQRS. They’re often mentioned together, but they solve different problems.

## What is CQRS?

CQRS stands for *Command Query Responsibility Segregation*. Big name, small idea:  
**don’t use the same model to change state and to read data.**

You send **commands** to a write model that enforces business rules.  
You run **queries** against read models designed for fast, convenient access.

Those models don’t have to look the same. They don’t even have to share a database.

A couple of important clarifications, because this is where my confusion usually starts:

**CQRS is not Event Sourcing.**  
You can use CQRS with a traditional database. You can use Event Sourcing without CQRS. They work well together, but one doesn’t require the other.

**CQRS is not microservices.**  
You can apply CQRS inside a monolith. One codebase, one deployment. This is about separation of concerns, not architecture diagrams.

Most applications use a single model for everything. You load an entity, mutate it, and query it back. That’s fine, until the write model starts accumulating fields that only exist for reporting, dashboards, or API responses. 
At that point, the model stops representing the domain and starts representing query’s needs.

## When does CQRS make sense?

You shouldn't use CQRS everywhere. It's really useful in a few specific situations.

If your system has far more reads than writes, separating the two lets you optimize independently. **The write side focuses on consistency and validation. The read side focuses on speed and usability.**

If your write operations involve state transitions, invariants, and rules that *really matter*, keeping them isolated from query concerns makes the code easier.

It also helps when different parts of your system need different views of the same data. Your API may need basic info, your dashboard needs detailed history, analytics needs summaries. They all work with the same domain, just shaped differently.

When should you skip CQRS? If you're building simple CRUD apps with small datasets, it's probably overkill. If your team is already stretched, the extra complexity might not be worth it. And if your current read and write models work fine together, CQRS won't make things better.

Don't use CQRS just because it sounds impressive. Use it when it solves a real problem you're already dealing with.

## Axon Framework and CQRS

Axon Framework is built with CQRS in mind. At first, it might feel like the framework is forcing you into a specific pattern. Over time, you'll realize it's actually helping you avoid common mistakes.

On the **command side**, you have aggregates. They receive commands, enforce invariants, and emit events. This is your write model. Its job is correctness, not convenience.

On the **query side**, you have projections. They listen to events and build read models tailored to specific use cases. These models are free to be denormalized, duplicated, cached, or however you want.

The result is a clean separation:
- business rules live in one place
- query complexity lives in another place
- changing how data is queried doesn't require touching business logic

This doesn't make your system simpler. It separates two different kinds of complexity so you can handle them independently.


## Real use case: Product lifecycle

Let’s reuse the same product lifecycle example from the Event Sourcing article. A product moves through states: reserved, confirmed, delivered. Different people need different views of that lifecycle.

Support wants to know *what happened and when*.  
The public API just needs the current status.  
Analytics wants counts and trends.

With a single model, you end up making compromises. You add fields just for reports. Your queries get more complicated. You add more indexes. And your write operations slow down because they're dealing with stuff that only matters for reading.

This is the core problem CQRS addresses: the write model and read models want different things.

## Solving the problem with CQRS

With CQRS, you stop forcing them to agree.

**Command model** (write side):

(This will look familiar if you read the Event Sourcing article. Same aggregate, different focus.)

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

This model doesn’t care about dashboards or APIs. Its only responsibility is enforcing business rules.

**Read model for API**:

```kotlin
data class ProductView(
    val productId: String,
    val status: String,
    val customerId: String?,
    val amount: BigDecimal?
)

@Component
class ProductApiProjection(private val jdbcTemplate: JdbcTemplate) {

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
                    amount = rs.getBigDecimal("amount")
                )
            },
            query.productId
        )
    }
}
```

This is exactly what the API needs. Nothing more.

**Read model for support dashboard**:

```kotlin
data class ProductDetailView(
    val productId: String,
    val status: String,
    val customerId: String?,
    val amount: BigDecimal?,
    val reservedAt: Instant?,
    val confirmedAt: Instant?,
    val deliveredAt: Instant?,
    val canceledAt: Instant?,
    val cancelReason: String?
)

@Component
class ProductDetailProjection(private val jdbcTemplate: JdbcTemplate) {

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

    @EventHandler
    fun on(event: ProductDeliveredEvent) {
        jdbcTemplate.update(
            "UPDATE product_view SET status = ?, delivered_at = ? WHERE product_id = ?",
            "DELIVERED", Instant.now(), event.productId
        )
    }

    @QueryHandler
    fun handle(query: GetProductDetailQuery): ProductDetailView? {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM product_view WHERE product_id = ?",
            { rs, _ ->
                ProductDetailView(
                    productId = rs.getString("product_id"),
                    status = rs.getString("status"),
                    customerId = rs.getString("customer_id"),
                    amount = rs.getBigDecimal("amount"),
                    reservedAt = rs.getTimestamp("reserved_at")?.toInstant(),
                    confirmedAt = rs.getTimestamp("confirmed_at")?.toInstant(),
                    deliveredAt = rs.getTimestamp("delivered_at")?.toInstant(),
                    canceledAt = rs.getTimestamp("canceled_at")?.toInstant(),
                    cancelReason = rs.getString("cancel_reason")
                )
            },
            query.productId
        )
    }
}
```

**Projection for external system export**:
This projection doesn't build a query model. It writes events to daily batch files that external systems can pick up. Maybe you need to sync product status with a legacy system that processes files overnight, send updates to a data warehouse via SFTP, or generate daily exports for analytics teams.


```kotlin
data class ProductExportRecord(
    val productId: String,
    val eventType: String,
    val timestamp: Instant,
    val payload: String
)

@Component
class ProductExportProjection(
    @Value("\${export.file.path:/var/exports/products}")
    private val exportPath: String
) {

    @EventHandler
    fun on(event: ProductReservedEvent) {
        val record = ProductExportRecord(
            productId = event.productId,
            eventType = "PRODUCT_RESERVED",
            timestamp = Instant.now(),
            payload = """{"customerId":"${event.customerId}","amount":${event.amount}}"""
        )
        
        writeToExportFile(record)
    }

    @EventHandler
    fun on(event: ProductConfirmedEvent) {
        val record = ProductExportRecord(
            productId = event.productId,
            eventType = "PRODUCT_CONFIRMED",
            timestamp = Instant.now(),
            payload = """{"productId":"${event.productId}"}"""
        )
        
        writeToExportFile(record)
    }

    @EventHandler
    fun on(event: ProductDeliveredEvent) {
        val record = ProductExportRecord(
            productId = event.productId,
            eventType = "PRODUCT_DELIVERED",
            timestamp = Instant.now(),
            payload = """{"productId":"${event.productId}"}"""
        )
        
        writeToExportFile(record)
    }
    
    private fun writeToExportFile(record: ProductExportRecord) {
        val fileName = "product_export_${LocalDate.now()}.jsonl"
        val file = File(exportPath, fileName)
        
        // Append record as JSON Lines format (one JSON per line)
        file.appendText("${toJson(record)}\n")
        
        // In a real system:
        // - A batch job picks up completed daily files
        // - Files get transferred to SFTP/S3
        // - External systems process the files at their own pace
        // - You might compress files older than X days
    }
    
    private fun toJson(record: ProductExportRecord): String {
        return """{"productId":"${record.productId}","eventType":"${record.eventType}","timestamp":"${record.timestamp}","payload":${record.payload}}"""
    }
}
```


Same events. Same domain. Completely different read needs.

Each projection evolves independently. You can add a new one without touching the aggregate. You can delete one without breaking the system. That’s the real value of CQRS.

## Final thoughts

CQRS won’t automatically make your system faster or cleaner. Used blindly, it just adds more moving parts.

Used intentionally, it gives you a clear boundary between *deciding what is allowed* and *showing information to users*. That separation tends to age well.

Start small. One aggregate. One projection. See if the separation actually helps. If it doesn’t, don’t force it.

Yes, you’ll end up with more classes and more files. But each piece has a single responsibility instead of one model trying to please everyone.

CQRS and Event Sourcing work well together, but they’re independent choices. Axon makes both approachable, which is why they often show up in the same system.

---

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
