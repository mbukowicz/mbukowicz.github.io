---
layout: post
title: "Kafka in multiple DCs"
date: 2020-08-31
categories: kafka
thumbnail: /images/kafka-in-multiple-dcs/thumbnail.png
---

<div style="text-align: center;">
  <img src="/images/kafka-in-multiple-dcs/kafka-in-multiple-dcs.png"
  title="Kafka in multiple Data Centers" class="rounded" />
</div>


Intro
-----

Shortly after you make a decision that Kafka is the right tool for solving
your problem you will probably wonder how to __install a Kafka
cluster that will survive various outage scenarios__ (no one likes to be woken
up in the middle of the night to handle production incidents, right?).

While studying the topic you may end up with a conclusion that running
your own Kafka cluster is not what you want as it can be both challenging
and time-consuming.
Fortunately, you can have someone else operate Kafka for you in
a Kafka-as-a-service way (e.g. Confluent Cloud, Amazon MSK or CloudKarafka
just to name a few).

But if you still decide to roll out your own Kafka cluster then you might
come to a realisation that the only way to have
a resilient Kafka installation is to __use multiple data centers__.

There are many ways how you can do this, each having their upsides and
downsides, and we will go through them in this post.


Active-passive
--------------

The simplest solution that could come to mind is to just have 2 separate
Kafka clusters running in two separate data centers and __asynchronously
replicate messages from one cluster to the other__.

In this approach, producers and consumers actively __use only one cluster__
at a time. The other cluster is passive, meaning it is not used if all goes well:

<img src="/images/kafka-in-multiple-dcs/active-passive.png"
title="Active-passive architecture" style="clear: both;" />

<div class="my-info">In the diagram there is only one broker per cluster
(per data center). Please note it is just a simplification. In a real cluster
you will most likely have multiple brokers.</div>

It is worth mentioning that because messages are replicated asynchronously
it is not possible to give confirmation back to a producer that
a message was stored not just in DC1 but also in DC2. In other words,
__producer could receive ACK for a particular message before it is sent to
data center 2__. As a consequence, __a message could get lost__ if the first data
center crashes before the message gets replicated.

<div class="my-info">By the way,
there are <strong>different ways to replicate data</strong>
between clusters. You could use <a href="https://kafka.apache.org/documentation/#basic_ops_mirror_maker">Mirror Maker</a>, as shown in the diagram,
which is basically a small utility program that serves both as
a Kafka consumer and a producer (it consumes messages from DC1 and produces them
to DC2).
You could also use a paid alternative called <a href="https://docs.confluent.io/current/multi-dc-deployments/replicator/index.html">Confluent Replicator</a> which
<a href="https://docs.confluent.io/4.1.2/multi-dc-replicator/mirrormaker.html#comparing-mirrormaker-to-confluent-replicator">comes with additional features</a>.</div>

Anyways, if the first data center goes down then the second one has to become active
and take over the load:

<img src="/images/kafka-in-multiple-dcs/active-passive-disaster.png"
title="Active-passive when disaster strikes" style="clear: both;" />

Apart from the potential loss of messages which did not get replicated,
another serious downside of this active-passive pattern is that it requires
a human intervention. __Someone has to be called in the middle of
the night in order to just pull the lever__ and switch to the healthy cluster
running in the other DC.

Unless consumers and producers are already running from a different data center
that still remains healthy they will also need to do the switch, making
the procedure even more complicated.

But probably the worst part is that you will __need to deal with aligning offsets__.
Because clusters are totally independent __the same message
in the active cluster can get an entirely different offset__ in the passive one.
And this can become a problem when you switch to the passive cluster because
now consumers will need to somehow figure out where they have ended up reading.
If done incorrectly the same messages will be read more than once,
or worse - they will not be read at all.

<div class="my-info">
You should be aware that Kafka by default
<a href="https://kafka.apache.org/documentation/#semantics">provides
at-least-once delivery guarantee</a>, meaning consuming <strong>applications
are expected to have some form of deduplication</strong>.
<br />
Kafka in version <strong>0.11.0.0 introduced exactly-once
semantics</strong>, which gives applications an option <strong>to avoid having to deal
with duplicates</strong>, but it requires a little bit more effort.
<br />
Please note that this exactly-once feature does not work across independent
Kafka clusters.
</div>

Unfortunately, a similar procedure needs to be applied when switching back
to the original cluster after it is finally restored.
You need to again find the place where your consumers left off and smoothly
switch to the repaired DC.

Yet another problem is __aligning configuration changes__. Both clusters
are totally independent which means that if you decide to modify a topic
in the active cluster (e.g. you add more partitions to a topic), you will need
to do the same in the passive cluster as well.


Active-active
-------------

All in all, paying for a stand-by cluster that stays idle most of the time is not the most
effective use of money. Alternatively, you could put the passive data
center to work and get better throughput:

<img src="/images/kafka-in-multiple-dcs/active-active.png"
title="Active-active architecture" style="clear: both;" />

This active-active configuration looks quite convoluted at first,
but starts to make more sense when you break it down. And it is worth
understanding as it is commonly used in LinkedIn (at least based on
the [blog posts](https://engineering.linkedin.com/kafka/running-kafka-scale)
and [tech talks](https://www.infoq.com/presentations/linkedin-scalability-arch)
they give) where Kafka was born.

As you can see, producers 1 and 2 __publish messages to local clusters__
(represented by brokers A1 and A2) which are then __propagated to aggregate
clusters__ (to which brokers B1 and B2 belong).

From the consumers perspective this __active-active architecture gives us
interesting options on what messages we can read__. We can decide
to __process only local messages__ (with consumer __1 and 2__) or __read messages
from both local data centers__ (using consumers __3 and 4__).

So imagine we have two data centers, one in San Francisco and one in New York.
We assign users to one of the data centers, whichever is closer to the user,
so that users can enjoy reduced latency. Now, if a user is somewhere in the bay area we will
assign her to the SF data center. And until the user stays close to this data center
we can quickly process her messages using a consumer which is reading from the local cluster.
But then if the same user decides to go on a business trip to the other coast
and her messages get published to the NY DC then the consumer
in the San Francisco data center will not get the message.
And this is where aggregate clusters come into play because they get messages
from both local DCs. Depending on a scenario, __we may choose to
read local messages and make our apps more responsive to users' actions
or wait for aggregate cluster to eventually get hold of these messages and
better handle the weird edge-cases where users' data is stored across
multiple data centers__.

But if you favour simplicity, __it could also make sense to allow consumption
only from the aggregate clusters__ (then only consumers 3 and 4 could read messages)
which can potentially make reasoning easier and help achieve a more straightforward
disaster-recovery procedure (at the cost of increased latency).

Going back to this complex active-active diagram, when looking at it you might wonder
why over-complicate and have those aggregate clusters if
instead you could just put mirror makers in each of the data centers where they
would copy data from A1 over to A2 and vice versa? That would have been
simpler, but unfortunately it would also introduce loops. So a message published
to A1 would have been replicated to A2 by mirror maker in DC2, but then mirror
maker in DC1 would have copied it back to A1. Here is an __example of a loop__
(just __follow the orange arrows__ from 1. to 5.):

<img src="/images/kafka-in-multiple-dcs/active-active-loops.png"
title="Loops in active-active architecture done incorrectly" style="clear: both;" />


Stretched cluster
-----------------

Whether you choose to go with active-passive or active-active you will still
need to deal with complicated monitoring as well as complicated recovery procedures.
And this can get pretty overwhelming when designing and setting up.  

But __if data centers are close to each other__ (e.g. availability zones within
the same region) then there is a much __simpler alternative__ commonly called
"stretched cluster".

It is basically __a one big cluster stretched over multiple data centers__ (hence
the name):

<img src="/images/kafka-in-multiple-dcs/stretched-cluster.png"
title="Stretched cluster architecture" style="clear: both;" />

Probably the best part about stretched cluster is that we are not forced
into devising a complex disaster-recovery instruction.
We just need to __keep our single cluster healthy__ by monitoring standard
Kafka's metrics __instead of having
to deal with__ 2 (active-passive) or 4 (active-active) __separate clusters__.

Another great thing is that we __do not need to worry about aligning offsets__
because data is no longer mirrored between independent clusters.
We can simply rely on Kafka's replication functionality to copy messages over to the
other data center while making sure all replicas are in-sync.


Rack-awareness
--------------

However, for this to work properly we need to ensure that __each partition
in one DC has a replica in the other DC__:

<img src="/images/kafka-in-multiple-dcs/stretched-cluster-replica-assignment.png"
title="Stretched cluster - replica assignment" style="clear: both;" />

It is necessary because when disaster strikes then __all partitions will need to
be handled by the remaining data center__:

<img src="/images/kafka-in-multiple-dcs/stretched-cluster-disaster.png"
title="Stretched cluster - when disaster strikes" style="clear: both;" />

By default Kafka is not aware that our brokers are running from different data
centers and __it could potentially put replicas of the same partition
inside one DC__.

But if we take advantage of the
[rack-awareness](https://cwiki.apache.org/confluence/display/KAFKA/KIP-36+Rack+aware+replica+assignment)
feature and [assign Kafka brokers to their corresponding data centers](https://kafka.apache.org/documentation/#basic_ops_racks) then __Kafka will try to evenly
distribute replicas over available DCs__.


What about ZooKeepers?
----------------------

Another important caveat when choosing stretched cluster is that it actually
__requires at least 3 data centers__. Otherwise quorum will not be possible
to achieve when one DC goes down because the remaining ZooKeeper
could not form the majority on its own:

<img src="/images/kafka-in-multiple-dcs/stretched-cluster-no-quorum.png"
title="Stretched cluster - ZooKeeper cannot achieve quorum" style="clear: both;" />

If we just __add a third ZooKeeper__ running somewhere off-site then we can
get the majority of votes (2 > 1) in case of an outage:

<img src="/images/kafka-in-multiple-dcs/stretched-cluster-quorum.png"
title="Stretched cluster - quorum possible" style="clear: both;" />

As shown on the diagram, the third data center does not necessarily
need to run any Kafka brokers, but __a healthy third ZooKeeper is a must
for the stretched cluster to keep on running__.

The good news is that there is [an improvement proposal to get rid of ZooKeeper](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum), meaning Kafka will provide its own
implementation of a consensus algorithm. Even though this will surely simplify
managing a Kafka installation it will unlikely render the third DC useless.
No matter the algorithm being used, we will still need another
data center to maintain quorum.


Last thoughts
-------------

Depending on the scale of a business, whether it is running locally
or all over the globe, different approaches can be used.
Please, do not get the wrong idea that one type of architecture is bad
while the other is superior. There is no silver bullet and each option
has its shortcomings.

Even when you look at how big tech giants (like for example the aforementioned LinkedIn)
are deploying Kafka then you could see they are often taking a mixed approach.
If it makes sense they run a passive cluster on a side, go for a stretched cluster
to handle users concentrated in one geographical region or choose active-active
configuration if data centers are further away. And none of these approaches
are bad, as long as they solve a certain use-case.


Want to learn more?
-------------------

Here are 2 tech talks by Gwen Shapira where she discusses different
cluster architectures in more detail:

* [One Data Center is Not Enough: Scaling Apache Kafka Across Multiple Data Centers](https://www.confluent.io/kafka-summit-sf17/Scaling-Apache-Kafka-Across-Multiple-Data-Centers/)
* [Common Patterns of Multi Data-Center Architectures](https://videos.confluent.io/watch/uvDmB9EiB8z3PGgJhp3pMH)
