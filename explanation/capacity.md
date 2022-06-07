# Kafka Capacity Planning

One of the most common questions for newcomers to Kafka is how big does the Kafka cluster need to be? Nobody has an easy answer, and this document isn't going to change that. Instead, lets walk through some of the variables to consider and a couple approaches to apply those variables. You can invent your own approach, mix and match pieces, or whatever else helps you decide where to draw the lines and adjust later when things change.

Before diving in, we should note that Confluent themselves have [system requirements](https://docs.confluent.io/platform/current/installation/system-requirements.html) as a starting point for some discussion.

## Factors That Affect Capacity

Typically the root inputs to capacity planning will be the number of Partitions, and the use of those Partitions. Consumers and Producers have connections to Partitions, replicas will require more connections, and all of these require resources. Higher throughput on certain partitions is going to change the storage needs, and higher read capacity will benefit from different configuration items.

To list out the big inputs to keep track of, we'd ideally like to know all of the following:

* How many partitions total?
* How many topics?
* What is the largest number of partitions per a topic, and what is the average?
* What is the largest number of consumers per a topic, and what is the average?
* How big are consumer groups for topics?
* How many messages per second are we putting through each topic? What is the average and what is the largest?
* When and how many spikes happen in a given cycle?
* What are the message sizes we will have? What are the largest, on what topics? What are the averages on each topic?
* How much data are we putting through each topic total?

We can see how a lot of these are related with some overlapping in their data. Most teams will not know the answer to a lot of these until they start using Kafka. This is good to understand ahead of those conversations so the correct behaviors can be set up before the data arrives. For instance, do not use Kafka to store files, however small they may be. Images, binary blobs, or anything else that you wouldn't consider an interchange encoding for systems to message to each other. It increases the message size, and becomes a much more burdensome medium for processing such data. Stick those files in an object or file store appropriate for the use case, and reference the location in a message.

You might see that this makes Kafka responsible for a lot less. That's how Kafka speeds are achieved. Kafka focuses on batch movement of messages at scale. You can't have a lot of checks and variability and round trip chatting if you want achieve that scale. Again, keep this all in mind when architecting your Kafka cluster because it will be the starting point for messaging maturity in the team's engineering culture.

## Start at the Bottom

A minimal production Kafka cluster only needs Zookeeper and Brokers. For reliability, you need a minimum 3 Zookeeper nodes, and 3 Broker nodes. Zookeeper can be expanded to 5 for larger clusters, but Brokers can grow to many nodes before the 3 Zookeeper nodes will need expansion. At least 10s of thousands of partitions, unless they're being burdened with extremely large messages or something.

Zookeeper needs a minimum of 4GB RAM for each node, and a couple CPU cores. A larger installation typically doesn't increase memory unless there's pressure, and instead it will be largely storage bound. Use SSDs for the storage, and keep the iops high if you keep the storage low. A couple Terabytes is not unheard of for a zookeeper database in a Kafka cluster.

Brokers are where things get heavy. The minimum requirements are 64 GB of RAM and 24 CPU cores. This is absolutely overkill for most Kafka clusters, but again worth noting that Kafka was not meant for small loads. That said, we can safely pare this down a bit for a small deployment, and work up to it.

Keep brokers well stocked with storage, preferably RAID that increases speed as well as redundancy will make this more effective. Keep all brokers the same size whenever possible. If you have one broker with a partition that runs a little wild, fix that imbalance or eat the cost of expanding the whole fleet to meet the demands of the noisy partition.

You can have an equally write and read heavy Kafka cluster, but most will have more read than write. Reading a message only once is more of a work queue pattern, which Kafka very much isn't. Hopefully you have several TBs of storage room for each Broker. Keep their iops bottlenecks separate, so Brokers aren't all vying for throughput from the same NAS or such.

We don't recommend going below 16 GB RAM and 4 CPU cores for Brokers. Even if there aren't a lot of partitions or Producers/Consumers, Kafka will also have housekeeping items to perform, such as compacting topics.

## Monitoring Usage

The most important part of capacity planning is that it is done continuously. There is no capacity today that will not be wrong tomorrow. Luckily, this is enabled very easily by what we should be doing already, [monitoring!](https://computingforgeeks.com/monitor-apache-kafka-with-prometheus-and-grafana/)

Keep a track record of metrics on a variety of pieces of Kafka's performance. Keep track of lagging Consumer groups, Partitions, ISRs, storage capacity, CPU and Memory usage, and anything else you can measure. If using Prometheus metrics, you want to keep in mind the cardinality of labels you add so you don't lose monitoring capabilities as labels explode, but measure as much as possible within that constraint. Even if something isn't a core metric, it can help paint a picture of why the core metric behaves the way it does. For example, your Consumer lag may happen because open file handles has reached your OS limit, so reads are having contention on file handles but you won't see that in CPU or Memory stats, and not likely on storage capacity.

Have these metrics stored outside Kafka, as you never want the system to watch itself. So if you are using Kafka to ship logs or metrics to your monitoring system, figure out a different patch for these metrics specifically.

## Expanding Later

As your usage of Kafka grows, you may need to expand capacity. Sometimes, you can add more storage if you aren't hitting RAM or CPU throttling points. This is usually true of Kafka clusters with low utilization but long retention. Keep in mind in these kinds of systems that a replay, where a Consumer group has to go back to much earlier in a topic, is a recipe for a spike in storage throughput, CPU and Memory where the working memory has to be shuffled quite differently than other scenarios. Your Kafka cluster has to handle the extreme use cases, not the normal operational load.

If you are beyond the RAM and CPU for 3 brokers in a minimum requirements cluster (meaning 64GB RAM per broker, and 24 CPU cores per broker), it's time to consider when to add more brokers. Until then, vertically scale your brokers until they reach that requirement set.

When adding a broker, a few scenarios will bring sadness. First, when you do expand, you have to be prepared to get rebalanced ISRs. This takes time and a lot of network bandwidth. On an already stressed cluster, this takes time and resources, which can cascade failures. Note that in this scenario, the issue isn't that we are expanding a cluster and it just isn't fast at doing so. The issue is that we didn't capacity plan earlier to see that the extreme use cases would never be handled by this cluster because we were stressing it at normal usage levels. Messaging systems spike.

Second, any time we have partitions too low or too high, balance and client usage levels are going to be off. If we have 5 Partitions and 4 brokers, 1 broker is going to have 2 of those partitions while the other 3 have 1 each. If that's how we have all my partitions by default, spikes are going to hit 1 broker in painful ways. That said, if we have 1000 partitions, Producers and Consumers have to manage that many connections and states along with our Kafka cluster. Be ready to strike a balance, and be ready to help users of the Kafka cluster to understand their responsibilities and tradeoffs in that balance.

Third, and classic to the problem of capacity planning, is planning for failure. This falls into the SRE discipline of SLO defining, and Kafka isn't any different than any other service in this regard. When we plan for 3 brokers, we're planning a level of failure and a level of partition tolerance. If you have 3 brokers and 3 zookeepers, with 2 brokers and 1 zookeeper in Availability Zone A, with 1 broker and 2 zookeepers in Availability Zone B, what happens when Zones A and B can't communicate with each other for 10 minutes?

There's no number of brokers where this gets better either. We start with 3 zookeeper and 3 broker clusters because that's the minimum, but the broker number can increment with single brokers, or 2 at a time to stay odd, or by percentage rounded some way, etc. There's not a real limitation here and not a singular position to stake on. We can have [rack awareness](https://strimzi.io/docs/operators/latest/overview.html#configuration-points-broker_str) in Strimzi's configuration so we can spread things for data redundancy, but availability will require quorum of zookeeper and data availability.

There's many other scenarios that make expanding a consideration of trade-offs, and there's more that involve failure modes of Kafka, but we should now have a good idea of the kinds of thinking needed to approach those when they come up.

## Playing Around

When exploring the actual deployment and usage of Kafka in a local dev scenario, ignore capacity and simply keep the dev environment of similar enough configuration behaviorally. Kafka can run in a dev machine on a smaller amount of RAM, single zookeeper and broker. This is not going to help with load testing, but functionality testing it can, and we can simulate multiple nodes and start failing over things. Some of the tutorials may call for that exact scenario.
