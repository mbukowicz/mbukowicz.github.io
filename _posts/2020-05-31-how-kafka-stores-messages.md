---
layout: post
title: "How Kafka stores messages"
date: 2020-05-31
categories: kafka
thumbnail: /images/how-kafka-stores-messages/thumbnail.jpg
---

<div style="text-align: center;">
  <img src="/images/how-kafka-stores-messages/audit-3737447_1280.jpg"
  title="Kafka under magnifying glass" class="rounded" />
</div>

Intro
-----

Apache Kafka, like any other messaging or database system,
is a complicated beast. But when divided into manageable chunks it can be
much easier to understand how it all works. In this post let's
focus on one area of Kafka's inner workings and try to figure out __how
Kafka writes messages to disk__.

Create a topic
--------------

First we will __create a test topic__ using an utility script provided with Kafka:

{% highlight text %}
> $KAFKA_HOME/bin/kafka-topics.sh \
  --create \
  --bootstrap-server localhost:9092 \
  --replication-factor 1 \
  --partitions 1 \
  --topic my-topic

Created topic "my-topic".


> $KAFKA_HOME/bin/kafka-topics.sh \
  --describe \
  --bootstrap-server localhost:9092 \
  --topic my-topic

Topic: my-topic	PartitionCount: 1	ReplicationFactor: 1	Configs:
	Topic: my-topic	Partition: 0	Leader: 1001	Replicas: 1001	Isr: 1001
{% endhighlight %}

__<code>my-topic</code> is the simplest topic__ possible. There is only __a single
copy of each message__ ( <code>replication-factor&nbsp;1</code> ) so everything will
be stored in one of the nodes, which means data could
be lost in case of just one node failure. On top of that it has only
__one partition__ ( <code>partitions&nbsp;1</code> ) which greatly __limits parallelism__ but
on the other hand makes things much easier to understand.

<div class="my-info">
Partitions play an important role in increasing throughput. Basically,
if you have <strong>more partitions</strong> you can <strong>write and
read more</strong> at the same time. The downside of partitions is that you may
<strong>lose message ordering</strong>. In other words, it is possible that
<strong>messages written later will be read sooner</strong> than other
messages which were produced earlier.
</div>

Having a single partition sounds perfectly suited to our case since
we are just trying to get our heads around how things work. More partitions
would only complicate things.


Send a test message
-------------------

To get started we will __send just one message__. Again Kafka has
an out-of-the-box tool we can use:

{% highlight text %}
> $KAFKA_HOME/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic

my test message<enter>
<ctrl+c>
{% endhighlight %}


Where is the message?
-----------------------

The message should be stored in the folder pointed by <code>logs.dirs</code>
parameter in your Kafka's broker configuration, e.g.:

{% highlight text %}
# server.properties
log.dirs=/var/lib/kafka
{% endhighlight %}

If you listed files in this folder you would see something like this:

{% highlight text %}
> ls /var/lib/kafka

cleaner-offset-checkpoint
log-start-offset-checkpoint
meta.properties
my-topic-0
recovery-point-offset-checkpoint
replication-offset-checkpoint
{% endhighlight %}

There is a lot of interesting stuff here, but from our point of view
__<code>my-topic-0</code> is the folder we should focus&nbsp;on__. This folder contains messages
(and their metadata) sent to the newly created topic.

Why was <code>0</code> appended to the folder name? Well, we asked for
a topic with just one partition so we only have <code>my-topic-0</code>.
If our topic had multiple partitions and these partitions were assigned to this
node we would see <code>my-topic-1</code> , <code>my-topic-2</code> and so on
(one folder per partition).

Now let's dig deeper and reveal the contents of <code>my-topic-0</code>.

{% highlight text %}
> ls /var/lib/kafka/my-topic-0

00000000000000000000.index      
00000000000000000000.log        
00000000000000000000.timeindex  
leader-epoch-checkpoint
{% endhighlight %}

Once again, quite interesting things are going on here, but we will limit our
experiment to the file which contains our message, which is the
<code>.log</code> file:

{% highlight text %}
> cat /var/lib/kafka/my-topic-0/00000000000000000000.log

GM??rat??rat????????????????*my test message
{% endhighlight %}

We can obviously see the exact same text that was sent (
<code>my test message</code> ), but what is this gibberish in front of it?


Message format
--------------

It probably does not come as a surprise that the message was saved in
a binary format and each message has some metadata together with the payload:

{% highlight text %}
> hexdump -C /var/lib/kafka/my-topic-0/00000000000000000000.log

00000000  00 00 00 00 00 00 00 00  00 00 00 47 00 00 00 00  |...........G....|
00000010  02 4d 1a e9 ea 00 00 00  00 00 00 00 00 01 72 61  |.M............ra|
00000020  74 a0 a7 00 00 01 72 61  74 a0 a7 ff ff ff ff ff  |t.....rat.......|
00000030  ff ff ff ff ff ff ff ff  ff 00 00 00 01 2a 00 00  |.............*..|
00000040  00 01 1e 6d 79 20 74 65  73 74 20 6d 65 73 73 61  |...my test messa|
00000050  67 65 00                                          |ge.|
00000053
{% endhighlight %}

A message stored in this binary format can be (fairly) easily decoded by referring to
[Kafka's official documentation](https://kafka.apache.org/documentation/#messageformat).

You should be aware that Kafka,
[for performance's sake](https://kafka.apache.org/documentation/#maximizingefficiency),
instead of writing one message at a time stores incoming messages in batches:

<img src="/images/how-kafka-stores-messages/records_in_batches.png"
title="Kafka stores messages in batches" style="clear: both;" />

So from the output of the <code>.log</code> file above we can distinguish
the batch's header:

<div class="code-with-tooltips">
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">base offset</span>
    00 00 00 00 00 00 00 00
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">batch length</span>
    00 00 00 47
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">partition leader epoch</span>
    00 00 00 00
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">magic</span>
    02
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">crc</span>
    4d 1a e9 ea
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">attributes</span>
    00 00
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">last offset delta</span>
    00 00 00 00
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">first time stamp</span>
    00 00 01 72 61 74 a0 a7
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">max time stamp</span>
    00 00 01 72 61 74 a0 a7
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">producer id</span>
    ff ff ff ff ff ff ff ff
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">producer epoch</span>
    ff ff
  </span>
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">base sequence</span>
    ff ff ff ff
  </span>
</div>

And batch's header is followed by a record:

<div class="code-with-tooltips">
  00 00 00 01 2a 00 00 00 01 1e
  <span class="code-with-tooltip">
    <span class="code-tooltip-text">"my test message" in ASCII</span>
    6d 79 20 74 65 73 74 20 6d 65 73 73 61 67 65 00
  </span>
</div>

Our batch is quite special because it actually contains just one record:

<img src="/images/how-kafka-stores-messages/single_message_batch.png"
title="Batch with just one record" style="clear: both;" />

<div class="my-info">
Please take note that <strong>message format may be subject to change</strong>
as Kafka continuously evolves and new KIPs are being implemented. The example
above shows how Kafka v2.5.0 serialized messages to bytes.
</div>


Better way to decode messages
-----------------------------

Dealing with bits and bytes is fun and makes us, programmers, look like
those hackers from NCIS.

But a more efficient way would be to write a tool that automatically transforms
binary representation to something easily interpretable by human beings.

Fortunately, such utility is already bundled with Kafka and here is how you
can __decode all messages from a <code>.log</code> file__:

{% highlight text %}
> $KAFKA_HOME/bin/kafka-run-class.sh kafka.tools.DumpLogSegments \
  --print-data-log \
  --files /var/lib/kafka/my-topic-0/00000000000000000000.log

Dumping /var/lib/kafka/my-topic-0/00000000000000000000.log
Starting offset: 0
baseOffset: 0
lastOffset: 0
count: 1
baseSequence: -1
lastSequence: -1
producerId: -1
producerEpoch: -1
partitionLeaderEpoch: 0
isTransactional: false
isControl: false
position: 0
CreateTime: 1590772932775
size: 83
magic: 2
compresscodec: NONE
crc: 1293609450
isvalid: true
| offset: 0
CreateTime: 1590772932775
keysize: -1
valuesize: 15
sequence: -1
headerKeys: []
payload: my test message
{% endhighlight %}


Add more messages
-----------------

Before we continue our little experiment, to spice things up,
let's add another 2 test messages. To send them in the same batch we will
add them at once (note that messages are divided with <code>\n</code>):

{% highlight text %}
> printf "next test message\nyet another message\n" | \
  $KAFKA_HOME/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic
{% endhighlight %}

If we now explore the contents of the <code>.log</code> file we could see
something like this:

{% highlight text %}
> hexdump -C /var/lib/kafka/my-topic-0/00000000000000000000.log

00000000  00 00 00 00 00 00 00 00  00 00 00 47 00 00 00 00  |...........G....|
00000010  02 4d 1a e9 ea 00 00 00  00 00 00 00 00 01 72 61  |.M............ra|
00000020  74 a0 a7 00 00 01 72 61  74 a0 a7 ff ff ff ff ff  |t.....rat.......|
00000030  ff ff ff ff ff ff ff ff  ff 00 00 00 01 2a 00 00  |.............*..|
00000040  00 01 1e 6d 79 20 74 65  73 74 20 6d 65 73 73 61  |...my test messa|
00000050  67 65 00 00 00 00 00 00  00 00 01 00 00 00 63 00  |ge............c.|
00000060  00 00 00 02 cd 0b 06 ba  00 00 00 00 00 01 00 00  |................|
00000070  01 72 61 d2 37 37 00 00  01 72 61 d2 37 63 ff ff  |.ra.77...ra.7c..|
00000080  ff ff ff ff ff ff ff ff  ff ff ff ff 00 00 00 02  |................|
00000090  2e 00 00 00 01 22 6e 65  78 74 20 74 65 73 74 20  |....."next test |
000000a0  6d 65 73 73 61 67 65 00  32 00 58 02 01 26 79 65  |message.2.X..&ye|
000000b0  74 20 61 6e 6f 74 68 65  72 20 6d 65 73 73 61 67  |t another messag|
000000c0  65 00                                             |e.|
{% endhighlight %}

After a closer look you may notice that the new messages are part of
the same batch:

{% highlight text %}
> $KAFKA_HOME/bin/kafka-run-class.sh kafka.tools.DumpLogSegments \
  --print-data-log \
  --files /var/lib/kafka/my-topic-0/00000000000000000000.log

Dumping /var/lib/kafka/my-topic-0/00000000000000000000.log

Starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 ...
| offset: 0 CreateTime: 1590779007825 keysize: -1
  valuesize: 15 sequence: -1 headerKeys: []
  payload: my test message
baseOffset: 1 lastOffset: 2 count: 2 ...
| offset: 1 CreateTime: 1590779066167 keysize: -1
  valuesize: 17 sequence: -1 headerKeys: []
  payload: next test message
| offset: 2 CreateTime: 1590779066211 keysize: -1
  valuesize: 19 sequence: -1 headerKeys: []
  payload: yet another message
{% endhighlight %}

<img src="/images/how-kafka-stores-messages/records_batched.png"
title="Two new messages batched" style="clear: both;" />


Offsets
-------

You may have noticed that this concept of <code>offsets</code>
popped up several times already.

__Offset is a unique record identifier__ which allows to quickly find
a message on a partition (using binary search).Offset is an always increasing
number where next offset is calculated by adding 1 to the last offset:

<img src="/images/how-kafka-stores-messages/records_with_offsets.png"
title="Records with offsets" style="clear: both;" />


Log segments
------------

What about the filename of the log file ( <code>000...000.log</code> )?
The name of the file is the first offset that it contains. You can expect the
next file to be something like <code>00000000000000671025.log</code> .

The <code>.log</code> files within partition folders
(e.g. <code>my-topic-0</code> ) are the main part of so called log segments
(together with <code>.index</code> and <code>.timeindex</code> ).

Log segments have this important characteristic that __messages are only appended
at the end of a <code>.log</code> file in the currently active segment__.
Another thing is that already __stored records cannot be modified__ (although
they can be deleted but more on that later).

I guess it all boils down to architectural decisions that Kafka creators had
to make. And setting this limitations should simplify some
performance and replication considerations.


Rolling segments
----------------

You can control when a new log segment gets created by changing Kafka broker
config parameters:

* <code>log.roll.ms</code> or <code>log.roll.hours</code> to
create a new segment log file after some time is elapsed

* <code>log.segment.bytes</code> to limit segment log file size in bytes

For example, if you set <code>log.roll.hours = 3</code> then every 3 hours
a new log segment file will be created.

Alright, but you may be asking yourself, why those log segments
are needed at all?


Removing old messages
---------------------

One of the main aspects of log segments is related with cleaning up no longer
needed messages.

Kafka, unlike other "message brokers", does not remove a message after consumer
reads it. Instead it __waits a certain amount of time before a message is eligible
for removal__. The fun part is, because messages are kept for some time, __you can
replay the same message__. This can be handy after you fix a bug that earlier
crashed message processing because you can later reprocess the problematic message.

So for example, if you want to only keep messages sent within last 30 days
you can set <code>retention.ms = 2592000000</code>. Then Kafka will get rid
of segments older then 30 days. At first, all the filenames related with that
segment will get the <code>.deleted</code> suffix:

{% highlight text %}
> ls /var/lib/kafka/my-topic-0

00000000000000000000.log.deleted
00000000000000000000.index.deleted
00000000000000000000.timeindex.deleted
...
{% endhighlight %}

Then after <code>log.segment.delete.delay.ms</code> elapses these files
will be gone from your storage.

Flushing to disk
----------------

There seems to be many similarities between Kafka log files and write-ahead
logs (WAL) commonly used in databases for transaction processing.

Usually, such databases have some clever strategy as for when data is flushed
to permanent storage to ensure integrity without hindering performance
with excessive flushing (e.g. multiple transactions can be flushed together in one go).
Kafka, however, takes a slightly different approach and __it is suggested to leave
flushing to the operating system__.

The recommended approach to guaranteeing durability is to __instead rely
on replication__, meaning spreading data across many machines/disks
on the application level.

But if durability is of highest importance to you then it is still possible
to force fsync with these properties (keep in mind performance will likely degrade):

* <code>flush.messages</code> - data will be flushed to disk ever N-th message

* <code>flush.ms</code> - data will be explicitly flushed after some timeout

What is important, is that a hybrid approach is possible to achieve. As an example,
you could rely on OS to do the flushing for most of the topics but
explicitly fsync on topics where you refuse to compromise on durability.

So if you are a little bit paranoid about a certain topic you may force fsync
after every single message:

{% highlight text %}
> $KAFKA_HOME/bin/kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --add-config flush.messages=1

Completed updating config for topic my-topic.
{% endhighlight %}

Summary
-------

As you can see the way Kafka stores messages on disk is not that scary.
It is not based on a complicated data structure, but rather it is just
__a log file where each message is simply appended at the end__.

We have skipped a lot of details, but at least now, if you ever need
to investigate a low-level data-related Kafka issue you know what
tools and config parameters you can use.

For example, you can utilise <code>kafka.tools.DumpLogSegments</code>
to confirm that a particular
message was stored in one of Kafka nodes (if a need for such precise
confirmation ever arises).
