# Kafka gotchas

There's a number of things that are common mistakes that everyone should avoid. We provide several that we're aware of and rationale for avoiding them. They are presented as strong opinions because they've bit us in the past.

## Delivery

Don't listen to anyone that tells you Exactly Once is possible. [It isn't](https://blog.bulloak.io/post/20200917-the-impossibility-of-exactly-once/), and the math disagrees with [Confluent themselves](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/). We trust the math, you should too. Pick At Least Once or At Most Once, depending only on your data loss tolerance. We recommend At Least Once and handling messages in an idempotent manner.

In all cases, ordering matters very little. If it does, it's usually within a small domain, like I need all updates to my account by ID. If that's the case, partition on account ID and my account changes will be in order on a single partition. This should be a rare need. If you need ordering, Kafka is only going to be in the way and expensive in all measures.

This plays into offset management. If your offsets get reset, can your consumer groups handle that? Offsets get messed up sometimes. You can manually fix that, but message processing is happening while you do so. It is far better to have the developer of the consumer code handle their offset management. They can provide specific functionality for overriding offsets, checks that they match some message timestamp, etc. Again, this is far easier to do in the Consumer code with the context of what offsets mean than it is from a CLI or something by a kafka administrator.

## Message Size

Never under any circumstances use large messages. It is important we recognize the folly here of not defining what a large message is. Large is anything you can't send 10,000 of in a second. That's a lot of room, but notice that for every kilobyte of message you add to the message size in that measure, you have about 10 megabytes of additional per second processing to do. If you started at 1 kilobyte, that's about 10 megabytes a second to start from, and then you double it. All of those messages have to fit in memory on brokers and on consumers. You're paying for network, storage, memory, and CPU for that.

You don't need it, we promise. There has never been a messaging app that processes large messages in their entirety at that rate. There are work queues that do that, such as image processing pipelines. Don't use Kafka as a work queue. If you have to process message, use Kafka as an eventing to notify via broadcast. It could be a broadcast to one consumer, but keep it in broadcast architecturally. The message should refer to an external storage location for the image data, which can then follow appropriate tooling for the very different paradigm that information represents. For instance, image data, who has access to it? What's the state of its being processed? Who processed it? How long did it take? These are questions Kafka actively can never answer because it is meant to move lots of messages very quickly in bulk.

A good rule of thumb is simply that if you care about the status of the message beyond "acked" or not, Kafka is a bad tool for that. It can be part of eventing to a system that keeps track of status in its own data store. Someone is going to read this and disagree, and that's fine. Let that person be your competitor, and they'll spend a lot more time managing Kafka and dealing with issues from their decisions while you ignore them and add value to your products and processes.

To put a number on large messages, we get concerned as messages approach a megabyte. Remember that Kafka was originally built to aggregate logs across 120K servers to a large data processing farm. Log messages don't typically get very large, even if they're full stack traces. 1 megabyte is a lot of UTF8 characters.

## Consumer Groups

Consumer groups are great. Keep their constraints in mind and we can keep them useful.

A partition can only have one consumer in the group. A consumer in the group can have multiple partitions. This means if you have 3 partitions and 18 consumers in the group, 15 consumers are sitting idle forever. Plan accordingly, and keep in mind that any time you change partition numbers, partition keys change, so data that was going to partition 1 may now go to partition 7. Note: This will destroy your ordering concerns. This is a common theme in Kafka management.

Also keep in mind that any time a consumer group changes, either because consumers were added or subtracted or the brokers need to reshuffle for some reason, they will have to do a rebalance. A rebalance is what assigns all the consumers to partitions and brokers for those partitions. This can take from seconds to minutes. Do not scale components often, even producers/consumers. Producers are less of an issue, but watch for stability issues there, too. If multiple producers are coming in and out of usage, brokers have to handle those connection negotiations, shuffle leadership information, and handle errors like a producer dying mid-way through sending a message. It adds up. Sloppy producer code can cause a lot of usage in Kafka to clean up that state.

Consumer groups are also really bad at maintaining offset information magically. Be wary, and learn how to change the offsets for a consumer group in your environment.

## Retention

Don't retain eternal messages, even if cost is no matter. Managing the storage is a huge headache where the scale doesn't play well with anything out there. You can expand to many more brokers, but now you're just idling. Instead, set a normal retention that you'd need to replay within, say a couple months maximum, and then punt old messages to longer term storage as a backup if you really need them.

The rule of thumb here is that Kafka should never be your source of truth. It's not a good database. Someone will advocate for KSQL here as a counter example. Let them go to your competitors as well. It's possible to do a great deal of things with Kafka that it wasn't well designed for. Use KSQL as a good example. If the performance doesn't matter very often, or the data isn't egregiously against it, KSQL can be used as a poor solution for a problem you don't need great efficiency for. It's win-win. When you start relying on this sort of functionality, retention included, the opportunity for sadness grows.

Overall, when choosing to use these kinds of features or scaling out storage or whatnot, watch your metrics carefully. Watch them over time, compare them with changes before and after. IF you start using KSQL on one topic for one purpose, how much more memory gets used? How much network? Are your storage IOps affected? Be informed, and let minor inefficiency be.

## Risk Tolerance

As you grow your kafka installation, should you have to do such a thing, there's a couple other changes to be aware of that we have yet to cover. Kafka replicas are like RAID. If you have 3 drives, and your RAID 5 them, you can lose 1. IF you have 4 and your RAID 6, you can lose 2. This sounds great. If you expand to 12 drives and RAID 6 them, you can still only lose 2, but 2 is a much smaller percentage of your array.

Kafka is no different. If you can lose a broker, great. If you can lose 2, excellent. Can you lose half of them? Even at 3 brokers, if you're on 2 availability zones, you could lose 2. So, can you lose service and be ok? Kafka can lose 2 brokers, but it can't take new messages at that point. Plan your cluster accordingly as you grow. If you're at 3 brokers, this is sort of a non-issue. Know the scenarios you can handle and adjust accordingly.

An unrelated but no less headache inducing problem is zookeeper maintenance. 3 zookeeper nodes will last a long time, but with small storage they will have problems very early. If you grow your kafka cluster's topics, users, partitions, or even just messages, watch your zookeeper metrics and see if you need to give them more capacity.

## Overall

Just be smart. Don't follow some best practices document, even like this. These are all guidelines. As a Kafka cluster owner, you must think. How do the various systems work, how are they configured, and how will they behave in various scenarios? Can a topic handle 2,000 producers? Does it need to? If someone did that accidentally, what would happen?

You can't account for everything. We have been at this for years and we're still surprised at some failure modes we find. Instead, use good monitoring and do regular exploration of the metrics you're capturing to identify possible issues. When failure happens, and it will, be ready to think and challenge yourself.
