---
layout: post
title: "Scaling Kafka"
date: 2020-06-22
categories: kafka
thumbnail: /images/scaling-kafka/thumbnail.png
---

<div style="text-align: center;">
  <img src="/images/scaling-kafka/scaling-kafka.png"
  title="Scaling Kafka" class="rounded" />
</div>

Foreword
--------

In this post we will explore the basic ways __how Kafka cluster can grow to handle
more load__. We will go 'from zero to hero' so even if you have never worked
with Kafka you should find something useful for yourself.

<div class="my-info">
For the sake of simplicity we will ignore the problems of data replication
as well as how to span Kafka across multiple data centres or cloud regions.
These subjects deserve separate posts to even
scratch the surface.
</div>


Basics
------

Let's start with basic concepts and build from there.

In Kafka __producers publish messages to topics__ from which these messages
are read by consumers:

<img src="/images/scaling-kafka/producer-topic-consumer.png"
title="Producer writes to topic, consumer reads from it" style="clear: both;" />


Partitions
----------

If we zoom in we can discover that __topics consist of partitions__:

<img src="/images/scaling-kafka/topic-partition.png"
title="Topics consist of partitions" style="clear: both;" />


Segments
--------

Partitions in turn are split into segments:

<img src="/images/scaling-kafka/partitions-segments.png"
title="Partitions consist of segments" style="clear: both;" />

You can think of segments as log files on your permanent storage where each
segment is a separate file. If we take a closer look we will finally see that
__segments contain the actual messages__ sent by producers:

<img src="/images/scaling-kafka/segments-messages.png"
title="Partitions consist of segments" style="clear: both;" />

__New messages are appended at the end__ of the last segment, which is called
the active segment.

Each message has an offset which uniquely identifies a message on a partition.

<div class="my-info">
If you would like to learn more how messages are organized inside partitions
<a href="/kafka/2020/05/31/how-kafka-stores-messages.html">here is an article
you may find interesting</a>.
</div>


Brokers
-------

Going back to the overview, there has to be some entity that handles
producers' and consumers' requests and this component is called a broker:

<img src="/images/scaling-kafka/kafka-broker.png"
title="Kafka Broker" style="clear: both;" />

__A Kafka broker is basically a server handling incoming TCP traffic__, meaning either
storing messages sent by producers or returning messages requested by consumers.


Multiple partitions
-------------------

The simplest way your Kafka installation can __grow__ to handle more requests
is __by increasing the number of partitions__:

<img src="/images/scaling-kafka/multiple-partitions.png"
title="Multiple partitions" style="clear: both;" />

From the producers perspective this means that it can now __simultaneously publish
new messages__ to different partitions:

<img src="/images/scaling-kafka/producer-multiple-partitions.png"
title="Producer sends to multiple partitions" style="clear: both;" />

Messages could potentially be sent from producers running in different
threads/processes or even on separate machines:

<img src="/images/scaling-kafka/multiple-producers.png"
title="Multiple producers writing to multiple partitions" style="clear: both;" />

However, in such case you could consider splitting this big topic
into specific sub-topics, because a single consumer can easily read from a list
of topics.

Producer implementations try to evenly spread messages across all partitions,
but it is also __possible to programmatically specify the partition__ to which
a message should be appended:

{% highlight java %}
var producer = new KafkaProducer(...);

var topic = ...;
var key = ...;
var value = ...;

// partition can be handpicked
int partition = 42;

var message = new ProducerRecord<>(topic, partition, key, value);
producer.send(message);
{% endhighlight %}

More on that can be found in the JavaDocs:

<img src="/images/scaling-kafka/producer-record-javadoc.png"
title="ProducerRecord JavaDoc" style="clear: both;" />

In short, __you can pick the partition yourself or rely on the producer to do it
for you__. Producer will do its best to distribute messages evenly.

<div class="my-info">
The official Kafka producer used to assign messages to partitions using
round-robin algorithm, but recently <a href="https://www.confluent.io/blog/apache-kafka-producer-improvements-sticky-partitioner/?_ga=2.35903392.1439487451.1592284671-1299957806.1592284671">the partitioning strategy changed to sticky partitions.</a>
</div>

Consumers
---------

On the other side we have Kafka consumers. Consumers are services that __poll
for new messages__. Consumers could be running as separate threads, but could
be as well totally different processes running on different machines. And
physically separating your consumers could actually be a good thing, especially
from the high-availability standpoint, because if one machine crashes
there is still a consumer to take over the load.

In case of consumers you have to make a similar choice as with producers:
you can take control and __assign partitions to consumers manually
or you can leave the partition assignment to Kafka__.
The latter means that consumers can subscribe to a topic
at any time and Kafka will distribute the partitions between the consumers that
joined.


Consumer groups
---------------

In this subscription model messages are consumed within consumer groups:

<img src="/images/scaling-kafka/consumer-groups.png"
title="Consumer groups" style="clear: both;" />

Consumers join consumer groups and Kafka takes care of even distribution
of partitions:

<img src="/images/scaling-kafka/consumer-to-group-assignment.png"
title="Assigning consumers to groups" style="clear: both;" />

Whenever a new consumer joins a consumer group __Kafka does rebalancing__:

<img src="/images/scaling-kafka/consumer-group-rebalancing.png"
title="Consumers group rebalancing" style="clear: both;" />

But if there are more consumers than partitions then some consumers
will stay idle (i.e. will not be assigned any partition):

<img src="/images/scaling-kafka/idle-consumer.png"
title="Idle consumer if too few partitions" style="clear: both;" />

It could be beneficial to keep such idle consumers so that when some consumer
dies the idle one jumps in and takes over.


Multiple brokers in a cluster
----------------------------------

Eventually, you will no longer be able to handle the load with just one broker.

You could then add more partitions, but this time these partitions will be handled
by __a new broker__. And you can scale horizontally in such way as often as
you need:

<img src="/images/scaling-kafka/multiple-brokers.png"
title="Multiple brokers" style="clear: both;" />


Dedicated clusters or multi-tenancy?
-----------------------------------

Since now you know the basics of scaling a Kafka cluster, there is one important
thing you should take into consideration: how are you going to run Kafka
within your organisation?

A more __cost-effective approach__ might be to run __a single multi-tenant cluster__.
The idea here is that all of your company's applications will connect to this
one cluster. Obviously, having just one service to maintain is more manageable
and also cheaper from the operational perspective.

But then you need to deal with the 'noisy neighbour' problem where one
application hammers your Kafka cluster disturbing work of other applications.
It requires some more thought in the form of capacity planning and
[setting up quotas](https://kafka.apache.org/documentation/#design_quotas)
(i.e. limits) on how much resources (e.g. network bandwidth)
a particular client can use.

Another option is to __spawn a dedicated Kafka cluster__ handling only requests
from a particular application (or a subset of your company's applications).
And actually you may be forced into this solution if you have a critical
application that requires better guarantees and more predictability.


Summary
-------

As you can see scaling Kafka is not that complicated. In a nutshell, you
just __add more partitions or increase the number of servers__ in your cluster.
