---
layout: post
title: "Replication in Kafka"
date: 2020-07-26
categories: kafka
thumbnail: /images/replication-in-kafka/thumbnail.png
---

<div style="text-align: center;">
  <img src="/images/replication-in-kafka/kafka-replication.png"
  title="Kafka replication" class="rounded" />
</div>

Intro
-----

It is hard to deal with the thought of losing data, even the non-critical
kind of data, like application logs. But if we start considering what could
possibly happen, from short power outages through disk failures
to losing the whole datacenter due to natural hazards, then we can realise
that it is all just a matter of probabilities. And we, as engineers,
should try to __reduce the probabilities that we lose our precious data__.

In this post let's take a look at the approach Kafka takes on replication
in order to prevent data loss.


Why replicate?
--------------

The motivation to replicate is rather simple: we would like to save our future
selves from having to explain to a business person how and why our system
lost their data.


Tuning knobs
------------

Losing data is bad, no doubts about that. But __one type of data may be
more important__ than the other. We may get away with losing metrics related
with customers' shopping habits during last year's Christmas season,
but having to figure out why €10 mln disappeared from a system can
significantly lower one's life expectancy.

Another aspect of replication is that it has its costs: you simply need __more
storage space__ and you can also expect __increased latency__ because it takes
more time to distribute copies of the same data and then collect confirmations
that it was correctly saved.

The great thing about Kafka is that it provides __different ways to control
durability__, meaning you have various knobs and levers at your disposal with
which you can choose different trade-offs and control the risk exposure.

The most common durability-related parameters are:

* ```replication.factor``` - the number of Kafka brokers
where your message will be stored (basically, the number of copies
of your data)
* ```min.insync.replicas``` - at least this number of caught-up
replicas need to confirm before write operation is assumed to be successful
* ```acks``` - how many confirmations need to be received before write is
considered complete (available options are: ```0```, ```1``` or ```all```)


Replicas
--------

Let's visualise what each of those params actually does, starting with
```replication.factor```. Here you can see 3 topics with ```replication.factor=2```:

<img src="/images/replication-in-kafka/kafka-replication-factor-2.png"
title="3 topics with 2 replicas each" style="clear: both;" />

<div class="my-info">For simplicity's sake, it was assumed that
<strong>each topic has just one partition</strong>.</div>

Setting ```replication.factor``` to ```2``` means that each message sent to one
of these 3 topics will be saved to 2 brokers (e.g. messages in topic 1 are stored
at broker 1 and broker 2).

Then if one of the brokers dies the other one can take over and still
provide access to these messages.
In other words, our cluster can __survive a single-broker failure__ without any
data loss (we have at least one copy of our data):

<img src="/images/replication-in-kafka/kafka-survives-single-broker-failure.png"
title="Cluster survives single broker failure" style="clear: both;" />

If we increase ```replication.factor``` to ```3``` then we get additional copy
of each message:

<img src="/images/replication-in-kafka/kafka-replication-factor-3.png"
title="3 topics with 3 replicas each" style="clear: both;" />

...which in turn means we can still access data in our topics
even if 2 brokers are down (in this case we end up with exactly one copy
of each message available only from the 3rd broker):

<img src="/images/replication-in-kafka/kafka-survives-2-brokers-failure.png"
title="Cluster survives two brokers failure" style="clear: both;" />


Leaders and followers
---------------------

From the replicas one is elected to be a leader, while the rest
become followers:

<img src="/images/replication-in-kafka/leaders-followers.png"
title="Leaders and followers" style="clear: both;" />

<div class="my-info">Again, it was assumed that each topic has just a single
partition so that it is easier to understand. In case of topics containing
multiple partitions the process is similar, the only difference being
that <strong>each partition on a topic can have different leaders</strong>.</div>

Producers write messages to leaders. Then each message is propagated
by the leader to followers:

<img src="/images/replication-in-kafka/producer-leader-followers.png"
title="Producer writes to leader, leader writes to followers" style="clear: both;" />

The interesting thing about leaders is that by default consumers,
similarly to producers, read only from the leaders:

<img src="/images/replication-in-kafka/consumers-read-from-leader.png"
title="Consumer read only from the leader by default" style="clear: both;" />

And in the face of a network partition this can make problem resolution
a bit easier as it reduces the number of moving parts (write and read operations
are performed on the same broker).

On the other hand, it could make sense to take advantage of data
locality and fetch messages from a follower:

<img src="/images/replication-in-kafka/consumers-read-from-follower.png"
title="Consumer read from follower" style="clear: both;" />

So if a consumer and a follower reside in the same datacenter then this could
potentially help us reduce latency.

Fortunately, it recently became also possible to
[read messages from a follower](https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica). David Arthur from
Confluent wrote
[an in-depth blog post about it](https://www.confluent.io/blog/multi-region-data-replication/).


In-sync replicas
----------------

Another important concept in Kafka is ISR (In-Sync Replica). ISR is a copy
of a partition which is up-to-date (i.e. it is alive and caught-up
to the leader). In order to prevent losing data, when leader election
takes place __only replicas that are in-sync can be selected as leaders__.

<div class="my-info">If availability is more important than durability
then you may decide to accept the risk and set
<code>unclean.leader.election.enable=true</code> which allows for out-of-sync
replica to become a leader at the cost of potential data loss.</div>

So for example, if broker 3 becomes slow (e.g. due to infrastructure-related
problems or something as trivial as a GC pause) then it falls out of
in-sync replicas set:

<img src="/images/replication-in-kafka/broker3-out-of-sync.png"
title="Broker 3 drops out of in-sync replicas set" style="clear: both;" />

And this means it is not eligible to become a leader until it caughts up.

It also means that if ```min.insync.replicas``` was set to a value
higher than ```2``` then all consecutive writes are going to fail with
an error. This way we can __prevent our Kafka cluster from accepting a message
that can be potentially lost__ due to not enough back-up copies.

Okay, but what does it exactly mean that a replica is "in-sync"? This varies
and can be controlled using ```replica.lag.time.max.ms``` param
which sets a time limit on how "slow" a replica can become before it is
dropped from the ISRs set.

<div class="my-info">There used to be a <code>replica.lag.max.messages</code>
parameter which allowed to control when replica is assumed to be
out-of-sync based on the number of messages it is lagging behind,
but it was removed in Kafka 0.9.0.0.
Still, a good rule of thumb is to <strong>monitor how much each replica is falling
behind</strong> and set up proper alerting in case things go south.</div>


Acks
----

The last piece of the puzzle is the ```acks``` parameter which allow us to
control the certainty with which we can ascertain that a message was successfully
written.

### acks=0

If we set ```acks=0``` we get no guarantees and producer assumes its work
is finished as soon as the write request is pushed out the door:

<img src="/images/replication-in-kafka/acks-0.png"
title="acks set to zero" style="clear: both;" />

This helps achieve the highest throughput possible, because producer does
not need to wait,
but at the same time it gives absolutely no confidence that our write succeeded.

Although it does not seem very useful,
```acks=0``` can actually be a viable option in certain cases.
For example, if you continuously send measurements
(e.g. temperature, current, voltage, etc.)
where the latest measurement discards the older ones
then you can ignore losing a single message.

### acks=1

Alternatively, you can wait for a confirmation from the leader
by setting ```acks=1```:

<img src="/images/replication-in-kafka/acks-1.png"
title="acks set to one (wait for leader)" style="clear: both;" />

It gives some level of confidence that a write was successful, but if the leader
fails before a message gets replicated then it is lost.

### acks=all

If durability is your top priority then you should set ```acks=all```. It makes
producer wait for confirmation from all ISRs:

<img src="/images/replication-in-kafka/acks-all.png"
title="acks set to all (wait for acknowledgments from all in-sync replicas)"
style="clear: both;" />


Typical durability setup
------------------------

[As stated in the docs](https://kafka.apache.org/documentation/#min.insync.replicas)
it is quite common to set ```replication.factor=3```, ```min.insync.replicas=2``` and ```acks=all```. It offers good-enough durability
and is considered to be the sweet spot between greater durability and
better performance.


Learn more
----------

We have just touched upon the topic of replication in Kafka, but if
you have around 37 minutes to spare then I recommend
[watching this talk by Jun Rao from Confluent](https://www.youtube.com/watch?v=li2aowPnezA). Jun explores replication and leader election in great detail.
