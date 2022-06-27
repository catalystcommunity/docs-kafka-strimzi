# Break Some Brokers

This tutorial, will take a multi-broker cluster and break one of those brokers to show how we might look at logs and operationally figure out what's wrong with the cluster. It will follow with some tips about moving beyond local troubleshooting via probes.

## Install Requirements

For this work, you'll need some tools installed locally. If you don't already have a functional Kafka cluster in a Kubernetes cluster, we have [an example guide](https://github.com/catalystsquad/example-kafka-client) that works if you have enough RAM to run it all (about 12GB free). The tools needed are:

- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) (or this comes with many other tools like several cloud CLI providers)
- A Kubernetes cluster (see guide mentioned before) capable of running 3 Zookeeper pods and 3+ Kafka broker pods.
- A Kafka cluster in that kubernetes cluster, with 3 Zookeeper and 3+ Kafka brokers. Again, see guide mentioned before if you don't already have this, as that's the fastest way to obtain such a cluster.
- [stern](https://github.com/wercker/stern#installation) for looking at logs across multiple pods
- [helm (3+)](https://helm.sh/docs/intro/install/)
- [jq](https://stedolan.github.io/jq/) is also just something you should have for working with kubernetes anyway
- [yq](https://github.com/mikefarah/yq#install) for similar reasons, though it is not used currently directly in this or the example repo.

Note: We highly recommend using the [example kafka client](https://github.com/catalystsquad/example-kafka-client) unless you're comfortable adjusting commands to match your kafka cluster including keystore and user/topic management. We would have a separate terminal window open to the root of a clone of that repo. All our commands are based on that example, so adjust as necessary if you don't use it.

## Producing Messages

You should have a topic with at least 3 partitions, and at least 3 replicas per partition. If you are using the Catalyst Squad example client this topic is called `tuttopic` in the `mykafka` namespace.

It is worth noting that what you call the cluster in strimzi has no bearing on how it is externally accessed, so configuring proper DNS and hostname resolution for both internal and external access to a kafka cluster is not recommended but is an exercise we'll leave to the reader.

Begin by producing a number of messages. 20 should do, make sure they're produced to different partitions.

How do you make sure? We'll, in this tutorial we're using a client that should pull a message, write it out to console, and give us information about the partition.

## Consuming messages

If you haven't already, use the consumer in the example or your own to consume from the topic you've chosen to create with the user certs you also created.

You should have a set of messages and information about the partition it came from and other information like timestamps of messages.

Note that if you run this multiple times, it should start from the beginning of the topic, but the ordering will change. If you have multiple partitions, ordering of messages only matters in one partition, and each client in a group can consume from 1 or more partitions.

In another terminal window you can use stern to watch logs across all kafka brokers:

```bash
stern -n mykafka -l strimzi.io/name=mykafkacluster-kafka -s 10s
```

Run your consumer a few times and see the logs it produces in a healthy state. There are likely very little.

## Breaking a broker

Now we're going to do something a little unorthodox. We're going to delete data from a live broker.

Make sure you've created a bunch of messages before we do this, and then be prepared with at least a stern command actively watching logs on brokers, and a producer and consumer running for a while (increase delay and number of messages, and have one terminal skip consuming, and one skip producing)

Something like this:

```bash
export MYKAFKA_SKIP_CONSUME=false
export MYKAFKA_SKIP_PRODUCE=true
export MYKAFKA_MESSAGE_NUM=200
export MYKAFKA_CLIENT_DELAY_SECONDS=1
```

Now do what one should avoid doing with great effort in all kubernetes applications: exec into a broker. Any broker will do.

Once in, change directories to your data volume, which in strimzi is `/var/lib/kafka/data` and in there should be a directory like `kafka-log1` which will change depending on broker. In that directory, find your partition data for your topic. It will follow the scheme `<TOPICNAME>-<PARTITIONNUMBER>`

If you delete all those directories, and produce/consume, you are likely to see nothing unusual show up in the logs. Producing and consuming will still continue. If you have monitoring in place, you may see that In Sync Replicas is not at the appropriate level, but we're going to just light everything on fire.

Delete everything in the `kafka-log1` or equivalent directory, and then produce a large number of messages. Eventually, the producer will crash.

Why is this occurring? Simply put, the broker that is a leader for some partitions is out of commission, and failures are expected in this scenario. Producers and consumers should be made to handle such exceptions and reconnect to the cluster, because the cluster will shuffle leadership and data as needed.

If you're using Strimzi, you'll see a bunch of errors in the logs from the other brokers, and kubernetes will delete the pod of the broken broker.

Watch the logs. Nothing should ever come from the broker you crashed that tells you it's healed. You have to get that state from metrics, but for clients this doesn't matter. Clients talk to the cluster, and only know of brokers because the cluster tells them to. The clients will adjust to the new leaders when they next connect.

When the new broker pod comes up, remote in and look at the data again. All of the partition data is back!

This is the self-healing nature of Kafka.

If your clients are written to handle the exceptions of disconnects and reconnect to the cluster, they'll get rebalanced. Rebalances happen all the time for consumers for instance. Fear not the failures.

## Some alternative scenarios

This was great and self-healing. If you weren't using a Strimzi operator, you'd have to notice and take the actions of restarting that broker yourself.

There's other data syncing issues that will be more nuanced and are less straightforward than deleting a broker pod to resync it. However, this is always a possibility. If you are absolutely sure that you have redundancy, you can delete the volume and let the brokers resync to each other.

In large kafka cluster, this can take some time. You may need to be more surgical.

For example, what happens if you delete just the topic data and then restart the broker pod with corruption on it? You can try this with the above scenario of course. It's just getting involved earlier than the kafka cluster would notice on its own.

It should come back and resync without needing to resync all the other topics. This is good to keep in mind. Often times we can be much more specific with the tools we're resolving incidents with.

You may also notice that this will interrupt producers and consumers, but again if you handle exceptions and simply reconnect, it should take care of itself. Make sure you keep this common behavior in mind when you're talking about possible time delay guarantees. Kafka is not great for real-time messaging if it can't tolerate such jitter scenarios.

## Final thoughts

This is a good scenario to give you the basics, but it illustrates a very important point: do not rely on logs.

As you can see, the logs are not contextual. This is common. Logs can sometimes expose something in a live debug scenario, but only when trying to look for something specific. There's always more noise, and Kafka is no exception to this.

There were no logs regarding a problem until exceptions were thrown in clients. How would one proactively notice the problem?

Metrics are needed. Metrics about in-sync replicas, leader elections, offset lag, and anything else that can be measured. They're much easier to piece together problems from and be proactive, but also see historically how behavior is anomalous.

Get monitoring in place and metrics scraping solid before you need it. Learn to watch those more and alert on those more. Logs are secondary, and Kafka makes this very easy to see.
