+++
title = "Exactly-Once Is a Lie"
date = 2025-08-04
description = "What brokers actually give you when they say exactly-once."
[taxonomies]
tags = ["Distributed Systems", "Event Driven", "Messaging"]
+++

Every message broker eventually puts "exactly-once" somewhere in its docs. Kafka has it, Pulsar claims it, NATS JetStream has a version of it, and it sounds like the thing you want: send a message, it arrives once, nobody worries.

But exactly-once delivery doesn't exist. What exists is engineering around the fact that it doesn't.

## what can actually happen

A producer sends a message to a broker, and there are three outcomes: the broker receives it, acknowledges, and the producer gets the ack (success); the broker receives it, acknowledges, but the ack is lost so the producer doesn't know it worked and retries, creating two copies; or the broker never receives it and the message is gone.

That's it, those are the options on an unreliable network. You can avoid case 3 by retrying, but retrying introduces case 2, and you cannot eliminate both. At the network level you get at-most-once (don't retry, accept losses) or at-least-once (retry, accept duplicates), and there's no third option that the network gives you for free.

## what Kafka actually does

Kafka advertises exactly-once semantics, but what it actually implements is two things working together.

**Idempotent producer.** Each producer gets an ID, and each message gets a sequence number. If the broker sees the same producer ID + sequence number twice, it drops the duplicate, which handles the lost-ack problem: the producer retries, the broker deduplicates. This only works within a single producer session though; if the producer crashes and restarts with a new ID, the deduplication state is gone.

**Transactions.** Kafka lets you wrap a consume-process-produce cycle in a transaction: read from input topic, do work, write to output topic, commit offsets, all atomically. If anything fails, everything rolls back.

So Kafka doesn't deliver messages exactly once. It uses at-least-once delivery with deduplication and atomic commits to make the _processing_ behave as if messages were delivered once, but the duplicates still happen at the transport layer and the broker just hides them.

## delivery vs processing

These are different things. **Exactly-once delivery** would mean the network guarantees each message arrives once, and no system does this because the network is unreliable: acks get lost, connections drop, machines crash between receiving and acknowledging.

**Exactly-once processing** means the side effects of handling a message happen once, even if the message itself arrives more than once. That's achievable, but it requires work: deduplication, idempotency, transactions. It's not a delivery guarantee, it's an application-level property.

When brokers say "exactly-once" they mean the second thing, and they don't say that clearly.

## the broker can't save you everywhere

Even with Kafka transactions, exactly-once only works within Kafka: read from Kafka, write to Kafka, commit. That's a closed system where Kafka controls both sides.

The moment your consumer does something outside Kafka, the guarantee breaks. Write to a database? Send an HTTP request? Call an external API? Kafka doesn't know about those. If your consumer processes a message, writes to Postgres, then crashes before committing the Kafka offset, it'll reprocess on restart and Postgres gets the write twice.

This is where the inbox pattern from my [outbox/inbox post](@/posts/outbox_inbox_patterns.md) comes in: you need idempotency at your handler level, recording that you processed this message before doing the work, so if you see it again you skip it.

The broker gives you tools, it doesn't give you a complete solution.

## idempotency is always your problem

Regardless of what your broker promises, if your consumer has side effects you need to handle duplicates. Either make the operation naturally idempotent (SET is idempotent, INCREMENT is not), track processed message IDs and skip duplicates, or use transactions that span both the broker offset and your external state (hard, often impractical).

At-least-once delivery + idempotent handling gives you the effect of exactly-once, and that's what production systems actually do. The "exactly-once" label is marketing over a real but narrower mechanism.

## notes

- Two Generals Problem: you can't get agreement over an unreliable channel with finite messages, which is why ack-loss is unavoidable
- Kafka's exactly-once (KIP-98) was controversial when introduced, and the original blog post title was literally "Exactly-once Semantics are Possible" which tells you something
- NATS JetStream does deduplication by message ID with a configurable window, similar idea but simpler scope
- Pulsar transactions work similarly to Kafka: read-process-write atomically within Pulsar
- If you're using a broker without deduplication, you're getting at-least-once at best, plan for it
- "Effectively exactly-once" is the more honest term some people use, and I prefer it
