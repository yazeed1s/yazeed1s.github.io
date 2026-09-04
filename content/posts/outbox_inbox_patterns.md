+++
title = "Outbox and Inbox Patterns"
date = 2025-07-08
description = "Why you need transactional outbox and idempotent inbox when building event-driven systems."
[taxonomies]
tags = ["distributed systems", "event driven", "patterns"]
+++

I was working on an event-driven system and hit a classic problem, which is that when I save to the database I need to make sure the event actually gets published to the broker, and on the consumer side I need to avoid processing the same event twice when the broker redelivers.

These are the Outbox and Inbox patterns, not new ideas, but I only really understood them after implementing them.

## the fundamental problem

Say you save a user to the database and then publish a `UserCreated` event:

```
1. save user to DB
2. publish event to broker
3. done
```

What if you crash between step 1 and step 2? The user is saved, but the event is never published, and the system is now inconsistent because some service was waiting for that event to do something and it will never hear about this user.

The obvious thought is "just do both in a transaction," but you can't because the database and the message broker are two separate systems. You can't wrap them in the same ACID transaction, and distributed transactions (2PC) exist but they're slow and fragile and most message brokers don't even support them.

## the outbox pattern

The outbox pattern helps with the publishing side, where instead of publishing directly to the broker you write the event to a table in the same database, in the same transaction as your domain data.

```
1. BEGIN transaction
2. save user to DB
3. save "UserCreated" event to outbox_events table
4. COMMIT
```

If step 2 and 3 succeed, they succeed together, and if anything fails, both roll back. Now the database is the source of truth for "what events need to be published."

A separate background worker polls the outbox table, picks up pending events, publishes them to the broker, and when it gets confirmation it marks the row as sent. The table looks something like:

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

The worker reads rows where `status = 'PENDING'`, publishes to the broker, and updates to `SENT`, and if publish fails it leaves it pending or marks it failed after some retry threshold.

## what can go wrong with outbox

Ordering is one. If you run multiple worker instances for throughput they might publish events out of order, where event B commits after event A but worker 2 publishes B before worker 1 publishes A, so if your consumers care about ordering you need to partition by aggregate or use a single worker.

The dual write illusion is another. The outbox doesn't magically solve the dual write problem, it just moves it, so now the dual write is between "mark row as SENT" and "broker actually received it", and if you mark SENT before the broker confirms and the broker was down you lose the event, and if you mark SENT after the broker confirms and you crash before marking you'll republish on restart. The second is safer because inbox handles duplicates.

And the broker can be down. If it's unreachable for a long time your outbox table grows unbounded, and depending on your write rate this can become a real problem, so you need monitoring and maybe backpressure if the table gets too large.

## the inbox pattern

Publishing is one half, consuming is the other.

Delivery guarantees depend on your broker, so some offer exactly-once like Kafka with transactions, some offer at-least-once like most durable brokers with acknowledgment, and some offer at-most-once which is fire and forget. If you're using something like core NATS with no persistence you get at-most-once, where the message is gone the moment it's published if no one is listening. With a durable broker that retries on no-ack you get at-least-once, which means your handler might receive the same event multiple times, and processing it every time means you double-create things, double-charge customers, or whatever your handler does happens twice.

The inbox is the mirror of the outbox, where before processing you record that you received this event, and if you see it again you skip it.

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

The unique constraint on `(event_id, handler_name)` is the key. When a message arrives, you try to insert into inbox with that event_id and handler_name; if insert fails (constraint violation) you already handled this event so you ACK and return; if insert succeeds you process the event and update status to `PROCESSED`. This gives you idempotency at the handler level, where the same handler won't process the same event twice.

## what can go wrong with inbox

Ghosting is the main one. You insert the inbox row, then crash before doing the actual work, and now the event is "locked" because the row exists but the work never happened, so the next delivery sees the row and skips it. This is a real problem, and one mitigation is to only insert the inbox row after successful processing, but then you're back to the crash-between-work-and-record problem just in a different order, and another is to have a timeout or "last touched" timestamp and treat old pending rows as abandoned.

Pruning is the other one, since same as with the outbox, `PROCESSED` rows accumulate and need cleanup.

## some brokers handle parts of this

Before implementing all of this, check what your broker does. Some brokers (like Kafka or NATS JetStream) persist messages to disk and support exactly-once semantics with proper configuration, tracking what has been delivered and acknowledged, and if a subscriber is down, the message waits. If you publish with a deduplication ID, the broker won't double-publish.

This doesn't eliminate the need for these patterns entirely, but it might simplify things. With a proper durable broker, the outbox is still useful if you need the DB write and event publish to be atomic (broker acknowledgment happens after publish, not as part of the same transaction), and the inbox might be less critical if the broker guarantees exactly-once delivery, but "exactly-once" is tricky and often has caveats so I would still keep inbox for safety.

If you're using a fire-and-forget broker with no persistence (like core NATS), you absolutely need both patterns because the message is gone the moment it's published if no one is listening.

## the at-least-once guarantee

This whole setup gives you at-least-once delivery from end to end, so you're guaranteed the event will eventually reach the handler because the outbox retries until success, and the handler is guaranteed not to double-process because the inbox deduplicates.

You don't get exactly-once because the outbox worker might publish, then crash before marking `SENT`, then on restart publish again. That's fine because the inbox catches the duplicate.

## operational stuff

Both tables need maintenance, so batch reads instead of one row at a time (read 100 or whatever), a polling interval that balances CPU waste against latency (we use 500ms), a cleanup job that deletes `SENT` and `PROCESSED` rows older than X days, monitoring to track how many `PENDING` rows are piling up and alert if it's growing, and failure handling that marks events `FAILED` after N retries and maybe routes them to a dead letter table for manual inspection.


## notes

- Polling isn't the only way, some DBs support change data capture (CDC) which can trigger on outbox inserts, and Debezium does this with Postgres
- Some people use the inbox pattern even for HTTP requests, not just events, same idea: deduplicate by request ID
- The outbox pattern is sometimes called "transactional outbox" or "outbox polling"
- Kafka has built-in exactly-once if you use transactions correctly, still worth understanding what's happening underneath
