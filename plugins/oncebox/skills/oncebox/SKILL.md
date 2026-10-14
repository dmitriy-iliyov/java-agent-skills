---
name: oncebox
description: Publishing events reliably from a Spring Boot service - an outbox table polled and published to Kafka or RabbitMQ, at-least-once delivery, per-type retries and backoff, a DLQ, recovery of events stuck mid-send, and deduplication of redelivered messages through an inbox on the receiving side. Load it before writing any of that by hand - the `oncebox` library implements it. Also load it when `oncebox-starter` is in the build file, when a context fails to start with `'sender' cannot be null`, `'cache-name' cannot be null`, `'sender.emergency-timeout' must be less than 'stuck-recovery.max-batch-processing-time'` or a `NoUniqueBeanDefinitionException` for `IdempotentConsumer`, when a publish fails with `IllegalTransactionStateException` (no existing transaction found for propagation mandatory), when outbox pollers deadlock on MySQL, or when a service silently skips events that another service or another listener consumed. Not for log-based CDC (Debezium), not where the order of events has to be guaranteed, and not for idempotency of client requests carrying an `Idempotency-Key` - that is a caller's repeat, not a broker's redelivery.
---

# oncebox

On Maven Central under `io.github.dmitriy-iliyov`, current version `2.0.0`. A service gets two mechanisms, used
together or apart: the **outbox** writes events in the business transaction and a background worker polls them out
of the table to the broker; the **inbox** records the ids of consumed events, so a redelivered event is skipped.

**Do not write the poller or the deduplication by hand.** The state machine, per-type retries with backoff, the
DLQ, recovery of events stuck in `IN_PROCESS`, cleanup of processed rows, the idempotent consumer and the
Micrometer metrics are all in the library. A hand-rolled version reproduces about a tenth of that and gets the
`SKIP LOCKED` polling wrong on the first concurrent instance.

## When it does not fit

Say so instead of bending the library around it:

- **The order of events matters.** There is none, within an event type or across types - types are polled in
  parallel.
- **Very many instances poll the same table.** `SELECT ... FOR UPDATE SKIP LOCKED` degrades as instances are
  added; the workable number depends on the event types and the database load.
- **Log-based CDC is required.** The library polls; it does not replace Debezium.

## The minimal working set

`oncebox-starter` **plus exactly one dialect**, and on the outbox side **plus exactly one transport**:

| Kind | Modules | Required |
|---|---|---|
| Starter | `oncebox-starter` | always, outbox and inbox |
| Dialect | `oncebox-postgresql` / `oncebox-mysql` / `oncebox-oracle` | exactly one, outbox and inbox |
| Transport | `oncebox-kafka` / `oncebox-rabbit` | exactly one, outbox side |
| Optional | `oncebox-metrics`, `oncebox-dlq-api`, `oncebox-inbox-cache` | - |

Then `@EnableOncebox` on the main class.

**Outbox:** `OutboxPublisher#publish(eventType, payload)` **inside the business `@Transactional` method** - or
`@OutboxPublish(eventType = "...")` on it, which takes the method result as the payload. Atomicity is the whole
point: a publish outside the transaction buys nothing over sending to the broker directly, so the library refuses
it - the call fails with `IllegalTransactionStateException` (no existing transaction found for propagation
mandatory).

**Inbox:** `IdempotentConsumer#consume(...)` inside the listener, with the event id taken from the message by
`OnceboxHeadersUtils::extractId` (the `oncebox_event_id` header the outbox writes). A redelivered id skips the
operation. Two implementations are registered and neither is `@Primary`, so the listener picks one by annotation:

| Inject with | Bean | Guarantee | When the operation throws |
|---|---|---|---|
| `@TransactionalConsumer` | `inboxTransactionalIdempotentConsumer` | effectively-once | id and operation roll back together, the redelivery runs it again |
| `@NonTransactionalConsumer` | `inboxNonTransactionalIdempotentConsumer` | at-most-once | the id stays recorded, the redelivery is skipped |

The transactional one is the default choice - a database change. The non-transactional one is for a side effect
that must not repeat, such as an email; a failed call is not retried, by design, and the ids it recorded are logged
at error. `@Qualifier` with the bean name works the same as the annotation.

The library creates its tables (`oncebox.tables.auto-create`, default `true`) - only the ones the enabled
features use. No `CREATE TABLE` or Flyway migration is written for `outbox_events`, `outbox_dlq_events`,
`inbox_events` or `oncebox_jobs`. Every time column is set by the database clock, so the application needs no
`Clock` bean and instances in different time zones agree.

## The traps

An `oncebox.outbox` or `oncebox.inbox.cache` block that is present with `enabled` unset counts as **on**; an absent
block means off. `oncebox.inbox` itself is the exception: it is on only with `enabled: true`. Two of the traps
below follow from that.

### Fails at startup

**A starter with no dialect.** `oncebox-starter` bundles none, so there is no DAO to wire and the context fails
to start - with an error that does not name the missing module.

**An `outbox` block in an inbox-only service.** The context dies on `'sender' cannot be null`. An inbox-only
service either leaves the `outbox` block out or writes `oncebox.outbox.enabled: false`.

**An `inbox.cache` block with no `cache-name`.** The context dies on `'cache-name' cannot be null`. The name must
also match a cache in a `CacheManager` bean the application provides, and `oncebox-inbox-cache` has to be on the
classpath.

**An `IdempotentConsumer` injected without a qualifier.** The context dies on `NoUniqueBeanDefinitionException`:
there are two, and neither is primary. Mark the injection point with `@TransactionalConsumer` or
`@NonTransactionalConsumer`. With Lombok's `@RequiredArgsConstructor` the annotation on the field does not reach the
constructor - and the same exception follows - unless `lombok.config` lists it:
`lombok.copyableAnnotations += io.github.dmitriyiliyov.oncebox.starter.inbox.TransactionalConsumer` (and the
`NonTransactionalConsumer` line likewise).

**An emergency timeout not below the stuck threshold.** The context dies on `'sender.emergency-timeout' (...) must be
less than 'stuck-recovery.max-batch-processing-time'`: a batch still being sent would be recovered as stuck and
published twice. Raise `oncebox.outbox.stuck-recovery.max-batch-processing-time` or lower
`oncebox.outbox.sender.emergency-timeout`.

**MySQL older than 8.0.13.** The tables use expression defaults (`DEFAULT (UTC_TIMESTAMP(6))`), so creating them
fails.

### Wrong behaviour, no error

**MySQL under the default `REPEATABLE READ`.** The polling query takes next-key (gap) locks, so concurrent
instances deadlock instead of each taking a disjoint batch. Run the datasource at `READ COMMITTED` - a poller
never needs repeatable reads. PostgreSQL and Oracle are unaffected.

**A missing `cache:` block in the inbox.** Every idempotency check then hits the database, with a warning in the
log and nothing else.

**One `cache-name` shared by several services.** The cache key is the event id alone, and a hit skips the
operation without reaching the database. Services consuming the same event from one store (one Redis) under one
name see each other's entries: the first to consume an event marks it for all, and the rest skip it silently -
the event is lost for them, not duplicated. The name carries the service -
`cache-name: "<service-name>:oncebox:inbox"`, never a bare `oncebox:inbox`; a separate
`spring.cache.redis.key-prefix` or a separate store per service works as well.

**The DLQ is disabled by default.** Events that exhaust their retries go nowhere until
`oncebox.outbox.dlq.enabled` is set.

**A broker with no acks.** At-least-once rests on the outbox waiting for the broker's acknowledgement, and the
library does not configure the broker. It only warns: on a Kafka producer without `acks=all`, and on a
`RabbitTemplate` without `mandatory=true`. Without them the guarantee is not there, whatever the outbox does.

**A thread pool sized for one event type.** The outbox needs `2 + n` threads - stuck recovery and cleanup plus one
per event type - and `5 + n` with the DLQ (transfer to, transfer from, DLQ cleanup); the inbox needs `1`. The
default is `min(available_processors, 5)`, so an outbox with several event types starves silently. Set
`oncebox.thread-pool-size`.

**Listeners on the wrong container factory.** The idempotent consumer needs
`containerFactory = "inboxKafkaListenerContainerFactory"` or `"inboxRabbitListenerContainerFactory"`, whose
message converter resolves the payload class from the event-type header through `oncebox.inbox.mappings`; on
Spring's default factory the payload arrives untyped.

**Kafka committing offsets before processing.** The default `AckMode` (`BATCH`) commits only after the listener
returns, so `ack-mode=manual` is not needed and only moves the commit into user code. What breaks at-least-once is
`enable.auto.commit=true` - the client commits on a timer, before the event is processed; the starter warns about
it. And the default `DefaultErrorHandler` commits a record that exhausted its retries, whatever the `AckMode`:
without a recoverer (a `DeadLetterPublishingRecoverer`, say) that event is gone.

**Two listeners of one service on the same event.** The inbox records an event by its id alone, whichever
consumer recorded it. If one listener writes to the database through the transactional consumer and another sends
an email through the non-transactional one, the first to record the id handles the event and the other skips it
silently. Do both in one listener, or give the side effect its own event.

**The non-transactional consumer for a database change.** A failed operation leaves the id recorded, so the
redelivery is skipped and the change is lost. That is its guarantee, not a bug - use `@TransactionalConsumer`.

**Inbox mappings keyed differently from the outbox.** The keys of `oncebox.inbox.mappings` (event type to payload
class) have to be exactly the event types the outbox was configured with and publishes under.

## Configuration

The full reference is the project README, https://github.com/dmitriy-iliyov/oncebox - link it rather than
restating it. By section: the outbox's `sender`, polling and backoff, `stuck-recovery`, `clean-up` and `dlq`; the
inbox's `source`, `mappings`, `clean-up` and `cache`; and the global `thread-pool-size` and `distributed-lock`.
Read the relevant one rather than guessing a property name - unknown keys under `oncebox` are not rejected, they
are ignored.
