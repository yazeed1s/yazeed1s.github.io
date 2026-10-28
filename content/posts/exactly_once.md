+++
title = "Exactly-Once Is a Lie"
date = 2025-08-04
description = "What brokers actually give you when they say exactly-once."
[taxonomies]
tags = ["distributed systems", "event driven", "messaging"]
+++

Every broker puts "exactly-once" somewhere in its docs, Kafka has it, Pulsar has it, NATS JetStream has its own version, and it sounds like what everyone wants, send a message and it arrives once. The more I read about how it works the less I believe the label.

## the three outcomes

A producer sends a message and one of three things happens. The broker gets it and the ack comes back, fine. The broker gets it but the ack is lost, so the producer thinks it failed, retries, and now there are two copies. Or the broker never gets it and it's gone.

You can kill the third case by retrying, but retrying is exactly what creates the second one, and you can't remove both on a network that loses packets. So the network gives you at-most-once (don't retry, lose some) or at-least-once (retry, get duplicates), and that's the full menu.

## what Kafka does

Kafka's exactly-once is two mechanisms stacked together. The idempotent producer gives each producer an ID and each message a sequence number, so when a retry shows up with the same ID and sequence the broker drops it, which fixes the lost-ack case, at least while the producer session lives (restart with a new ID and that dedup state is gone). Then transactions let you read from a topic, write to another topic and commit your offsets atomically, so a failure in the middle rolls back all of it.

The duplicates still happen on the wire, Kafka just hides them before they reach your processing. So what you get is exactly-once *processing* built on at-least-once delivery, and when brokers say "exactly-once" this is what they mean, they just don't say it very clearly.

## outside Kafka it breaks

All of that only holds while you stay inside Kafka. Your consumer reads a message, writes a row to Postgres, crashes before committing the offset, and on restart it reads the same message again and Postgres gets the row twice. Kafka has no idea Postgres exists, so it can't roll that write back.

Which means duplicates are your problem no matter what the broker promises. Either the operation is idempotent by nature (setting a value is, incrementing isn't), or you record processed message IDs and skip the ones you've seen (that's the inbox pattern from my [outbox/inbox post](@/posts/outbox_inbox_patterns.md)), or you try a transaction across the broker offset and your database, which is hard and often not practical.

At-least-once plus idempotent handlers is what real systems run on, I'd guess almost all of them. "Effectively exactly-once" is a term some people use for this and I like it more, it's at least honest.

## notes

- The reason a lost ack can't be fixed is the Two Generals problem, you can't reach agreement over a channel that drops messages with a finite number of messages.
- Confluent's original blog post for this was titled "Exactly-once Semantics are Possible", which kinda tells you people expected it not to be.
- NATS JetStream dedups by message ID inside a time window, same idea, smaller scope.
- Pulsar transactions work like Kafka's, read-process-write atomically inside Pulsar.
- No dedup on your broker means at-least-once at best, plan for it.
