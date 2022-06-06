# Kafka Security

Kafka was not originally built with security in mind, its goal was to move massive amounts of log data, not authentication and access controls. It still has had some options added to consider in more depth when it comes to the security side.

## Users are Principals

In Kafka, some docs will call them Users, but in the code, they're just Principals. They are identified internally to Kafka as a string, typically of the format `type:name` but it is worth noting that there's nothing enforcing that format.

The concept of a Principal allows us to think about things as entities that may not be people and may not have a username/password (or could, but that would be the least secure option for services to use).

Most Principals will be of type `User` for our purposes, and tooling will follow the convention by and large.

For Plaintext these will be defaulted to `User:Anonymous` or some tooling will display that as just `Anonymous` in some contexts. For TLS/SSL these will be something like `User:"CN=Catalyst, OU=Squad, C=US"` like you might be used to seeing in TLS Certificate display formats. SASL will change Anonymous to the username so it will be like `User:MyUser` and such. For OAuth, it will similarly be `User:Username` but the username will be dependent on the IDP claims.

All of these can be overridden, but don't do that unless you've got a lot of bandwidth to invest for whatever your desired outcome is. Maintaining these mappings alongside Kafka's updates could be a heavy distraction.

## Permissions and ACLs

Within Kafka, the core workhorse is the Partition, but you manage ACLs at the Topic level. With the CLI tooling, you could set an ACL by doing something like this:

`bin/kafka-acls.sh --authorizer-properties zookeeper.connect=myzookeeper.address.xyz:2181 --add --allow-principal User:ServiceA --producer --topic ServiceATopicA`

This gives producer privileges to `User:ServiceA` to the topic `ServiceATopicA` which includes WRITE, DESCRIBE and CREATE permissions to the topic. Consumers also need privileges, specifically READ and DESCRIBE on the topic, as well as READ on the consumer group, which makes sense but gets missed sometimes when doing this by hand.

Don't manage these by hand. This is fine for learning or creating a test scenario from scratch, but Strimzi provides CRDs for KafkaUser and KafkaTopic so you can do this declaratively. This is good security practice to have automated configuration-as-code that is applied via CI/CD pipelines on that code, and limit access to humans.

## Encryption

It's worth saying at all times that Encryption is not security by itself. Encryption is a tool with which to build security with other tools as well. Kafka is a fine example of this.

All traffic between brokers is by default encrypted with TLS negotiated communications. You can easily configure all communication to Zookeeper to be similarly negotiated, and most tooling already does this by default. Out of the box, no client Producer or Consumer traffic is encrypted. You can communicate on an encrypted port rather than an unencrypted one, and this is better than nothing.

The ideal is to use TLS authentication so keys are signed in a trust structure, and speak TLS with those keys and certs. This requires ACLs be made, but that's simpler as Kafka has matured over the years. The encryption will require more CPU usage for both cluster and clients, so keep that in mind on heavily utilized Kafka deployments.

## Network Security

Kafka was born out of the old era of perimeter security. By default if you have network access to the cluster, you have control of the cluster. While authentication and authorization are excellent additions, it's worth also thinking about the network perimeter to make things easier on us all.

First, don't add listeners for any services you won't use. If you aren't using components such as Kafka Bridge, don't add them. Don't add access that might be nice to have some day, just what you're using today. This includes access from outside the Kubernetes cluster, even for limited users. If you have an administrative GUI for instance, don't add an Ingress when the same audience has `kubectl port-forward` access. Not only is such a GUI a potential vulnerability and further maintenance responsibility, it also enforces thinking about the right habits. If it's really easy to get in and make a change by clicking in a UI, we all tend to choose the easy thing instead of using our gitops workflow, and then those changes get clobbered the next time a sync with the infra-as-code repo happens.
