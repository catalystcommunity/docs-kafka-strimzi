# Tools

Kafka itself is fairly mundane and simple. Interfacing with it is through some haphazard shell scripts that wrap java jars. A number of other tools are useful in specific cases and may be used. We don't necessarily recommend them, but we'll talk about some and where they can be most useful

## MirrorMaker

MirrorMaker and MirrorMaker2 are common enough. MirrorMaker2 is not a strict superset or improvement, and people use both sometimes, usually where they have a lot of legacy setup in MirrorMaker. You can probably do everything with just MirrorMaker2 if you start with it.

Use this when you need to get data from one Kafka cluster to another. It just sets up a Consumer set and Producer set to take messages and reproduce them over in the other cluster. You can combine all sorts of ways. 3 source topics to one destination topic for instance.

The biggest caveat is that there's nothing promising your partitions or message orders will be preserved. You can have a topic named the same thing on the second cluster, but with a very different configuration. Even if you have the exact same server configuration, as a Producer dev, I have a lot more functionality available to me than the Producer in MirrorMaker2 has just due to the abstraction layer that MirrorMaker2 is providing. Pigeonhole principle applies.

## UIs

A number of UIs exist for Kafka. They all serve different purposes and have features for one audience or another.

Strimzi included functionality for [Cruise Control](https://github.com/linkedin/cruise-control) which looks ok and provides some automation for rebalancing Kafka data. That should raise some red flags for anyone looking to get a UI going. This UI is meant to take some action, not just be a display of information and exploration tool. If we install Cruise Control, there's potential for out-of-band changes to happen.

This is a common theme with Kafka UIs, where the desire has been to reduce reliance on the fiddly shell scripts for administration. Strimzi reduces a lot of that risk for us. Cruise Control does action that Strimzi doesn't, which is why it's included in Strimzi. It can be a nice addition. Learn the tool before you deploy it, and make sure it doesn't have the ability to do what you don't want it to.

Overall our experience forces us to recommend avoiding UIs. It can be faster to use a UI for some exploration aspects of administration of Kafka or anything else. However, that comes at a cost. UIs make it difficult to actually learn how to operationally interact with these tools, and Kafka is no exception. Struggle through that once and you'll forever be more capable of manipulating the system via the CLI afterwards. Use the UI and it will always be a path to avoiding that struggle, making you unlikely to learn and even after learning the temptation to abdicate thinking continues.

## Schema Registry

The Schema Registry is very common if you're using Avro messages. It's important to recognize its limitations and value additives. The Schema Registry allows you to version, reference, and modify schemas for Avro messages. They allow users to validate that this should conform to a schema, that versions 1 through N are compatible with each other or provide breaking changes or whatnot.

This can be useful information for automating data transformations from one schema version to another. This can be useful for seeing what should be available on a message, especially programmatically. There is, however, no enforcement of any of this.

The Schema Registry will take schemas, namespace them, version them, and tell you about them. It doesn't prevent publishing of messages that don't conform to the schemas available, and it doesn't even validate that a message matches the format of the schema referenced by the producer. The only thing it can do is tell you that there is a schema with that name and version in the registry. It doesn't even look at the messages, they're separate.

Another caveat is that the Avro library and spec has a different lifecycle from Kafka, so now you have another functionality mismatch possibility. For instance, the Avro library spec was updated to include Decimals as a binary type. The Kafka project did not update to this avro library for at least 18 months (we stopped tracking it at that point since we weren't going to wait for the functionality). This affects what clients and the schema registry can account for.

This is an incredibly useful tool for typing and data engineering, and it is recommended if that use case matches yours. Just keep in mind the caveats and the fact that it is in no way magic.

## Kafka Bridge

If you need a simple way to map a REST API to Kafka, this is the tool for you. This is also kind of an anti-pattern, and often you'll want to do some data validation so writing your own REST API and producing to Kafka from there is usually what you want. Still, low hanging fruit, worth keeping in mind.

## Kafka Connect

Kafka Connect is useful if you can find an open source connector, or you could pay for one if you wish. These will either pull data in or export data out from Kafka topics to various data sources. For instance, a Postgres connector can take Change-Data-Capture (CDC) data from Postgres and put messages on a Kafka topic. This is usually more useful for data engineering purposes to get data out of systems and into a warehouse of some kind, but it can be useful for other specific use cases for smaller eventing purposes.

Be careful as they are billed as more magical than they are. Connectors are simply Producers and/or Consumers with an abstraction wrapped around them.
