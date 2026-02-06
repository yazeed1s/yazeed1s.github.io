+++
title = "Outbox and Inbox Patterns"
date = 2025-11-25
description = "Why you need transactional outbox and idempotent inbox when building event-driven systems."
[taxonomies]
tags = ["Distributed Systems", "Event Driven", "Patterns"]
+++

I've been working on an event-driven system and ran into a classic problem: how do I make sure that when I save something to my database, the corresponding event actually gets published to the message broker. And on the receiving side, how do I make sure I don't process the same event twice when the broker delivers it multiple times.

These are the Outbox and Inbox patterns. Not new ideas. But I had to actually implement them to understand why they matter.

## the fundamental problem

Say you save a user to the database and then publish a `UserCreated` event.

```
1. save user to DB
2. publish event to broker
3. done
```

What if you crash between step 1 and step 2? User is saved, but the event is never published. And the system is now inconsistent. Some service was waiting for that event to do something and it will never hear about this user.

The obvious thought is "just do both in a transaction." But you can't. The database and the message broker are two separate systems. You can't wrap them in the same ACID transaction. Distributed transactions (2PC) exist but they're slow and fragile and most message brokers don't even support them.

## the outbox pattern

The outbox pattern helps with one part of the problem; making sure the event gets published. The idea is surprisingly simple, you don't publish directly to the broker. Instead, write the event to a table in the same database, in the same transaction as your domain data.

```
1. BEGIN transaction
2. save user to DB
3. save "UserCreated" event to outbox_events table
4. COMMIT
```

If step 2 and 3 succeed, they succeed together. If anything fails, both roll back. This makes it an atomic operation. Now the database is the source of truth for "what events need to be published."

A separate background worker polls the outbox table, picks up pending events, and publishes them to the broker. When it gets confirmation, it marks the row as sent.

The table looks something like:

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('PENDING', 'SENT', 'FAILED')),
    created_at TIMESTAMPTZ DEFAULT now(),
    sent_at TIMESTAMPTZ
);

```

The worker reads rows where `status = 'PENDING'`, publishes to the broker, and updates to `SENT`. If publish fails, it leaves it pending or marks it failed after some retry threshold.

## what can go wrong with outbox

**Ordering.** If you run multiple worker instances for throughput, they might publish events out of order. Event B commits after event A, but worker 2 publishes B before worker 1 publishes A. If your consumers care about ordering, you need to partition by aggregate or use a single worker.

**Dual write illusion.** The outbox doesn't magically solve the dual write problem, it just moves it. Now the dual write is between "mark row as SENT" and "broker actually received it." If you mark SENT before the broker confirms, and the broker was down, you lose the event. If you mark SENT after the broker confirms, and you crash before marking, you'll republish on restart. The second is safer because inbox handles duplicates.

**Broker down.** If the broker is unreachable for a long time, your outbox table grows unbounded. Depending on your write rate this can become a real problem. You need monitoring and maybe backpressure if the table gets too large.

## the inbox pattern

Publishing is one half. Consuming is the other.

Delivery guarantees depend on your broker. Some offer exactly-once (Kafka with transactions, for example), some offer at-least-once (most durable brokers with acknowledgment), some offer at-most-once (fire and forget). If you're using something like core NATS with no persistence, you get at-most-once: message is gone the moment it's published if no one is listening. With a durable broker that retries on no-ack, you get at-least-once, which means your handler might receive the same event multiple times. Process it every time and you double-create things, double-charge customers, whatever your handler does happens twice.

The inbox is the mirror of the outbox. Before processing, you record that you received this event. If you see it again, you skip it.

```sql
CREATE TABLE inbox_events (
    id UUID PRIMARY KEY,
    event_id UUID NOT NULL,
    handler_name VARCHAR(100) NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('PENDING', 'PROCESSED', 'FAILED')),
    received_at TIMESTAMPTZ DEFAULT now(),
    processed_at TIMESTAMPTZ,
    UNIQUE(event_id, handler_name)
);
```

The unique constraint on `(event_id, handler_name)` is the key. When a message arrives:

1. Try to insert into inbox with that event_id and handler_name
2. If insert fails (constraint violation), you already handled this event. ACK and return.
3. If insert succeeds, process the event.
4. Update status to `PROCESSED`.

This gives you idempotency at the handler level. Same handler won't process the same event twice.

## what can go wrong with inbox

**Ghosting.** You insert the inbox row, then crash before doing the actual work. The event is now "locked" because the row exists, but the work never happened. Next delivery will see the row and skip it.

This is a real problem. One mitigation is to only insert the inbox row after successful processing, but then you're back to the crash-between-work-and-record problem, just in a different order. Another is to have a timeout or "last touched" timestamp and treat old pending rows as abandoned.

**Pruning.** Same as outbox. `PROCESSED` rows accumulate. Need cleanup.

## some brokers handle parts of this

Before implementing all of this, check what your broker does.

Some brokers (like Kafka or NATS JetStream) persist messages to disk and support exactly-once semantics with proper configuration. They track what has been delivered and acknowledged. If a subscriber is down, the message waits. If you publish with a deduplication ID, the broker won't double-publish.

This doesn't eliminate the need for these patterns entirely, but it might simplify things. With a proper durable broker:

- Outbox still useful if you need the DB write and event publish to be atomic. Broker acknowledgment happens after publish, not as part of the same transaction.
- Inbox might be less critical if the broker guarantees exactly-once delivery. But "exactly-once" is tricky and often has caveats. I would still keep inbox for safety.

If you're using a fire-and-forget broker with no persistence (like core NATS), you absolutely need both patterns. Message gone the moment it's published if no one is listening.

## the at-least-once guarantee

This whole setup gives you at-least-once delivery from end to end. You're guaranteed the event will eventually reach the handler (outbox retries until success). The handler is guaranteed to not double-process (inbox deduplicates).

You don't get exactly-once. The outbox worker might publish, then crash before marking `SENT`, then on restart publish again. That's fine because the inbox catches the duplicate.

## operational stuff

Both tables need maintenance:

- **Batch reads.** Don't poll one row at a time. Read 100 or whatever.
- **Polling interval.** Too fast wastes CPU. Too slow adds latency. We use 500ms.
- **Cleanup job.** Delete `SENT`/`PROCESSED` rows older than X days.
- **Monitoring.** Track how many `PENDING` rows are piling up. Alert if growing.
- **Failure handling.** After N retries, mark `FAILED`. Maybe route to a dead letter table for manual inspection.

## when you actually need this

If you're building anything where `"database saved but event not published"` would cause real problems, you probably need outbox. If your handlers have side effects that shouldn't happen twice, you need inbox.

If you're doing toy projects or your consistency requirements are loose, maybe you can skip it. But in production systems with money or important state involved, I wouldn't trust the happy path.

## notes

- Polling isn't the only way. Some DBs support change data capture (CDC) which can trigger on outbox inserts. Debezium does this with Postgres.
- Some people use the inbox pattern even for HTTP requests, not just events. Same idea: deduplicate by request ID.
- The outbox pattern is sometimes called "transactional outbox" or "outbox polling."
- Kafka has built-in exactly-once if you use transactions correctly. Still worth understanding what's happening underneath.
