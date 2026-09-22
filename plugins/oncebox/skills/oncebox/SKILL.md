---
name: oncebox
description: Publishing events reliably from a Spring Boot service - an outbox table polled and published to Kafka or RabbitMQ, at-least-once delivery, per-type retries and backoff, a DLQ, recovery of events stuck mid-send, and deduplication of redelivered messages on the consumer. Load it before writing any of that by hand: the `oncebox` library implements it. Also load it when `oncebox-starter` is in the build file, when a context fails to start with `sender cannot be null` or `cacheName cannot be null`, or when outbox pollers deadlock on MySQL. Not for log-based CDC (Debezium), not where the order of events has to be guaranteed, and not for idempotency of client requests carrying an `Idempotency-Key` - that is a caller's repeat, not a broker's redelivery.
---

# oncebox

A transactional outbox for Spring Boot: events are written in the business transaction, polled out of the
table by a background worker and published to Kafka or RabbitMQ. Published on Maven Central under
`io.github.dmitriy-iliyov`, current version `1.1.2`.

**Do not write the poller by hand.** The state machine, per-type retries with backoff, the DLQ, recovery of
events stuck in `IN_PROCESS`, cleanup of processed rows, the idempotent consumer and the Micrometer metrics
are all in the library. A hand-rolled version reproduces about a tenth of that and gets the `SKIP LOCKED`
polling wrong on the first concurrent instance.

Full documentation is the project README - link it rather than restating it:
https://github.com/dmitriy-iliyov/oncebox

## When it does not fit

Say so instead of bending the library around it:

- **The order of events matters.** There is none - not within an event type and not across types. Events are
  polled in parallel by type.
- **Very many instances poll the same table.** `SELECT ... FOR UPDATE SKIP LOCKED` degrades as instances are
  added; the workable number depends on the event types and the database load.
- **Log-based CDC is required.** This library polls; it is not a replacement for Debezium.

## The minimal working set

Three dependencies, never fewer. `oncebox-starter` **plus exactly one dialect plus exactly one transport**:

| Kind | Modules | Required |
|---|---|---|
| Starter | `oncebox-starter` | always, on both sides |
| Dialect | `oncebox-postgresql` / `oncebox-mysql` / `oncebox-oracle` | exactly one, on both sides |
| Transport | `oncebox-kafka` / `oncebox-rabbit` | exactly one, publisher side |
| Optional | `oncebox-metrics`, `oncebox-dlq-api`, `oncebox-consumer-cache` | - |

Then `@EnableOutbox` on the main class, and `OutboxPublisher#publish(eventType, payload)` **inside the
business `@Transactional` method** - or `@OutboxPublish(eventType = "...")` on it, which takes the method
result as the payload. Atomicity is the whole point of the pattern: a publish outside the transaction buys
nothing over sending to the broker directly.

The tables are created by the library (`oncebox.tables.auto-create`, default `true`). Do not write a
`CREATE TABLE` or a Flyway migration for `outbox_events`, `outbox_jobs`, `outbox_dlq_events` or
`outbox_consumed_events`.

## The traps

### Fails at startup

**A starter with no dialect.** `oncebox-starter` bundles none, so there is no DAO to wire and the context
fails to start. The error does not name the missing module.

**A `publisher:` block on the consumer side.** The publisher is on unless it is switched off explicitly: the
block being present with `enabled` unset counts as enabled, and the context then dies on
`sender cannot be null` or `events cannot be null`. A consumer-only service writes
`oncebox.publisher.enabled: false`.

**`consumer.cache.enabled: true` with no `cache-name`.** It is a `requireNonNull`:
`cacheName cannot be null`. The name must also match a cache in a `CacheManager` bean that the application
provides, and `oncebox-consumer-cache` has to be on the classpath.

### Wrong behaviour, no error

**MySQL under the default `REPEATABLE READ`.** The polling query takes next-key (gap) locks, so concurrent
instances deadlock instead of each taking a disjoint batch. Run the outbox datasource at `READ COMMITTED`;
an outbox poller never needs repeatable reads. PostgreSQL and Oracle are unaffected.

**A missing `cache:` block on the consumer.** The cache is then off and every idempotency check hits the
database. It is a warning in the log and nothing else. (Present but with `enabled` unset means *on* - the
block's absence and its presence default in opposite directions.)

**The DLQ is disabled by default.** Events that exhaust their retries are not moved anywhere until
`oncebox.publisher.dlq.enabled` is set.

**A broker with no acks.** At-least-once rests on the publisher waiting for the broker's acknowledgement, and
the library does not configure the broker. Without `acks` (or the equivalent) the guarantee is not there,
whatever the outbox does.

**A thread pool sized for one event type.** The publisher needs `4 + n` threads - four background jobs
(stuck recovery, cleanup, DLQ transfer, DLQ cleanup) plus one per event type; a consumer needs `1`. The
default is `min(available_processors, 5)`, so a publisher with several event types starves silently under it.
Set `oncebox.thread-pool-size`.

**Listeners on the wrong container factory.** The idempotent consumer needs
`containerFactory = "outboxKafkaListenerContainerFactory"` or `"outboxRabbitListenerContainerFactory"`. That
factory carries the message converter which resolves the payload class from the event-type header through
`oncebox.consumer.mappings`; on Spring's default factory the payload arrives untyped.

**Kafka acknowledging automatically.** The consumer container warns and carries on unless
`spring.kafka.listener.ack-mode=manual` (or `manual_immediate`) is set - an offset committed before the
business operation ran turns at-least-once into at-most-once.

**Consumer mappings keyed differently from the publisher.** `oncebox.consumer.mappings` maps the event type
to the payload class, and its keys have to be exactly the event types the publisher was configured with and
publishes under.

## Configuration

The README carries the full reference, by section: publisher polling and backoff, `stuck-recovery`,
`clean-up`, `dlq`, the consumer's `source`, `mappings`, `cache`, and the global `thread-pool-size` and
`distributed-lock`. Read the relevant section there rather than guessing a property name - unknown keys under
`oncebox` are not rejected, they are ignored.
