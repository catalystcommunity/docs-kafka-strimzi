# Availability and Data Loss

Kafka is often chosen for data redundancy and availability. It is true that Kafka is a highly available system when well cared for. Like all systems, failure is expected, and how well we plan for failure determines how available and performant we are when failure happens.

No system can be immune to data loss. Let's discuss how to plan for the worst and recovery modes.

## Can I Delete This?

Cloud Native thinking aspires to treat as many things as ephemeral as possible. Chaos Engineering is a way to test a lot of the fault tolerant assumptions we make. Catalyst Squad likes to ask this a simple way: can I delete this?

If the answer is yes because the system will continue running and the service is still available, the system is fault tolerant to that item being deleted. If data loss will occur if I delete this, it is not fault tolerant.

Kafka's workhorse concept is the Partition. An In Sync Replica (ISR) is the copy of that Partition's data, where it could resume the responsibilities of the Partition leader and be the actual Partition instead of an ISR.

You can delete an ISR's files off disk, but Kafka will have errors and not know what to do. Deleting an entire node will only result in failures of any Partitions that don't have any ISRs left. This is the critical point. ISRs must exist for fault tolerance and recovery. If Kafka can't figure out how to have the minimum required ISRs, it can't accept new messages. It won't lost data, but Producers might. Now we have to think about Producers in addition to Kafka. Well, one thing at a time.

## Producers too?

When talking about Data Loss, there is an entire Data Lifecycle. Producers and Consumers are the edges of what we consider Kafka as a System. That System, however, means that we don't necessarily control Producers and Consumers in configuration or behavior. Thus, we can not produce a system that is resilient against their errors. Instead, we must define expectations for which availability can be guaranteed at a certain level.

Our advice is to set expectations in the team's documentation and have it monitored. Tell Producers not to produce messages over a certain size, and then measure the size of messages coming in so we can alert when Producers are getting close to breaking that expectation, as well as crossing the line.

Set ISR requirements to at least 1 extra Partition than the Partition Leader. If we set it to all available, then the Kafka cluster will fail when it loses a broker. We must be able to have less than a full cluster to function, and this failure state will still result in an available service while we work to restore the rest of the cluster. The best failures are the ones users don't get affected by.

This puts the onus on Producers to keep their configuration in line with these expectations. Make that responsibility clear.

## I can lose a Broker now

Being able to lose a broker is a good thing. This does not prevent a user from deleting a topic, misconfiguring a topic to delete old messages, or producing bad messages.

Which of these things is our responsibility? At a certain point, users must know how to use Kafka as good citizens in the shared system. ACLs will reduce the responsibility we have to keep in the Kafka system itself and how much we can push to individual topic owners. We can also offer some additional safety nets if we have time to tool them, such as being able to snapshot disk states, or have a backup of configuration. Gitops flows enable that configuration backup/history naturally, and we recommend them very strongly.

Snapshots of disk states are not helpful in a large system with many topics with different owners. Resetting all state of all topics to even an hourly snapshot is throwing away a lot of data. At scale, this is very untenable. Kafka clusters can have Petabytes of data running through them fairly regularly. Having snapshots to prevent data loss of one kind is ensuring data loss of other kinds as complexity is compounded. Do you restore to a shadow cluster and copy over files? Re-produce the snapshot onward to the bad message?

This is not a happy path. It is far better for everyone if responsibility for data integrity is on Producers and Consumers. Error handling is incredibly important for any Kafka user, and not the server side responsibility. Bad log messages for instance can happen because of active memory corruption on a system. This is not possible to prevent on Producers. It is worth noting that this bad message is also good signal. An error could simply ignore this message and log it as an exception to something like Sentry for someone to handle. Then no complicated error handling tooling needs to be created on Producers nor do we need bespoke tooling to handle excising a bad message. Again, Kafka is meant for scalability. Scalability means error handling live.

## What about dead letter queues?

It is not an uncommon request to get Kafka users asking about dead letter queues. This is a good sign of two things. First, Kafka is being used as a queue, which is very bad. Kafka is a parallelized append-only log, not a work queue. Nothing can change this. Consumer offsets can get reset in a variety of scenarios. Will items in the topic be reprocessed or do they expect at-most-once semantics? Second, this problem never ends, so this is a time for conversation about properly engineering a Kafka use case.

If a dead letter queue is requested, what happens when the error happens on the dead letter queue Consumer? Does progress halt there? Do we need a dead letter queue for the dead letter queue? We think the answer is obviously no, but we've also seen this done more than once by very different groups.

Our advice is to never dead letter queue, and if you need to reprocess something, capture the errors in a different context. This could even look very much like a dead letter queue, but it should in actuality be more like a Sentry style capturing mechanism where errors can be handled out of band. This means that the ability to handle a message days late all while handling all the other messages live. Some will say this isn't tenable, but this is necessary with a dead letter queue as well.

At the end of the day, either every message needs to be processed within a certain time, or they can be safely dropped. That's not a problem Kafka can help solve, that is a business logic problem and the logistics do not change no matter your technology choice. That's just messaging.

## Best case redundancy

Ultimately, with a minimal cluster of 3 brokers and a topic with N partitions, we want each partition to have a leader, and 1 ISR minimum before the message is considered produced. Then have another ISR and any broker can be lost and we should still have redundancy with 1 ISR. No configuration changes, and we can alert that the broker is down.

We can also see problem topics where a single broker is out of sync for a topic and take appropriate action, but the system is still available and data flowing.

If we get more brokers, we can do more ISRs if we're willing to pay for storage, and we can still keep the Producer expecting only one ISR beyond Partition leader before moving on.

In rare instances, having a second Kafka cluster that everything is synced with is also appropriate, but not for redundancy, and redundancy should not be a criteria for a second Kafka cluster. In most scenarios, without significant tooling, a second Kafka cluster is a liability to data availability, not a help.

## Storage redundancy

It is worth noting that Kafka is only redundant in itself via ISRs, which for all purposes are copies to make brokers more ephemeral. Kafka can also benefit from storage that is redundant unbeknownst to the Kafka brokers themselves. RAID systems, filesystem snapshots, all can be used in addition for further redundancy or to limit the scope of data loss. Since these are all outside the scope of Kafka, that's all we'll dive into for this document, but obviously a healthy Kafka cluster is built on healthy foundations that are not Kafka, such as Kubernetes being the expectation of this set of documentation.
