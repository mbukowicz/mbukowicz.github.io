---
layout: post
title: "Implementing a Kafka consumer in Java"
date: 2020-09-12
categories: kafka
thumbnail: /images/kafka-consumer/thumbnail.png
comments: true
---

<div style="text-align: center;">
  <img src="/images/kafka-consumer/kafka-consumer.png"
  title="Consumers reading from Kafka" class="rounded" />
</div>


Introduction
------------

In this post we will take a look at __different ways how messages can be read__ from
Kafka. Our goal will be to find the simplest way to implement __a Kafka consumer
in Java__, exposing potential traps and showing interesting intricacies.

<div class="my-info">
The code samples will be provided in <strong>Java 11</strong> but they could
be also easily translated to other versions of Java (or even to other JVM languages,
like Kotlin or Scala).
</div>


Naive implementation
--------------------

Getting started with Kafka is incredibly straightforward. You just need to
add an official Kafka dependency to your preferred dependency management tool,
e.g. add this to your pom.xml :

{% highlight xml %}
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>${kafka.version}</version> <!-- e.g. 2.6.0 -->
</dependency>
{% endhighlight %}

Then you just need to prepare configuration for your cluster and topic:

{% highlight java %}
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;

// ...

var props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.deserializer", StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());
{% endhighlight %}

Finally, you can create a consumer using this config:

{% highlight java %}
var consumer = new KafkaConsumer<String, String>(props);
{% endhighlight %}

<code>KafkaConsumer</code> like any other resource needs to be released
so you need to ensure it is closed in the finally block:

{% highlight java %}
var consumer = new KafkaConsumer<String, String>(props);
try {
  // ...
} finally {
  consumer.close();
}
{% endhighlight %}

Or even better, wrap it in _try-with-resources_:

{% highlight java %}
try (var consumer = new KafkaConsumer<String, String>(props)) {
  // ...
}
{% endhighlight %}


Manual or automatic consumer-partition assignment
-------------------------------------------------

Now we need to tell the consumer to read messages from a particular
set of topics.

One way do to this is to __manually assign
your consumer to a fixed list of topic-partition pairs__:

{% highlight java %}
var topicPartitionPairs = List.of(
    new TopicPartition("my-topic", 0),
    new TopicPartition("my-topic", 1)
);
consumer.assign(topicPartitionPairs);
{% endhighlight %}

Alternatively, __you can leave it to Kafka__ by just providing a name of
the consumer group the consumer should join:

{% highlight java %}
props.put("group.id", "my-consumer-group");
{% endhighlight %}

...and then __subscribing to a set of topics__:

{% highlight java %}
consumer.subscribe(List.of("my-topic"));
{% endhighlight %}

<div class="my-info">
<code>group.id</code> parameter is also required if you wish to
store offsets of consumed messages in Kafka (more on that later so stay tuned).
</div>

Although __assigning partitions manually makes the code easier to reason about__,
because later you can easily pinpoint the process and the machine
where the consumer run and what it tried to do, unfortunately it __is not very reliable__.
__If this consumer dies then message processing stops__.
You could programmatically implement some form of
a fail-over but it could be quite tricky (as with any distributed system
things tend to be more complicated than we have initially anticipated).

On the other hand, the __subscription model assumes you are
running multiple consumers that will evenly split topic-partitions__
between each other:

<img src="/images/kafka-consumer/consumer-group.png"
title="Partitions evenly distributed over consumers in a consumer group"
style="clear: both;" />

<div class="my-info">
Consumer groups are uniquely identified by the <code>group.id</code> parameter.
</div>

<div class="my-info">
It is important that <strong>all consumers</strong> within one consumer group
(<code>my-consumer-group</code> in this case) <strong>subscribe to the same list of
topics</strong>. Otherwise the final list of topics from which your consumer group
will receive messages will be hard to predict (you can expect a race condition
making debugging particularly miserable). This means that <strong>the same consumer
group cannot be re-used to listen to a different subset of topics</strong>.
</div>

Then __when one consumer crashes other consumers take over__ the
partitions previously processed by this dead consumer:

<img src="/images/kafka-consumer/consumer-group-dead-consumer.png"
title="When consumer dies other take over"
style="clear: both;" />

Not only you get automatic fail-over but it also makes scaling-out
easier when you add more partitions to your topics.
In that case __the existing consumers proactively divide the
new partitions among themselves__:

<img src="/images/kafka-consumer/consumer-group-new-partition.png"
title="When consumer dies other take over"
style="clear: both;" />


Consumer rebalancing
--------------------

The way it works is through __consumer rebalancing which gets triggered whenever
a consumer joins or leaves a consumer group__.

Consumer rebalancing is a huge topic, deserving a separate discussion,
but it is definitely worthwhile to explore because it can later reward you
with better understanding of what your consumers are doing when you need to
investigate a problem.

There are some amazing talks about rebalancing available on the Internet,
like for example these two:
* [The Magical Rebalance Protocol of Apache Kafka](https://www.youtube.com/watch?v=MmLezWRI3Ys) by Gwen Shapira
* [Everything You Always Wanted to Know About Kafka’s Rebalance Protocol but Were Afraid to Ask](https://www.confluent.io/kafka-summit-lon19/everything-you-wanted-to-know-kafka-afraid/) by Matthias J. Sax

And actually, rebalancing is an area of Kafka gaining quite
a lot of traction lately. Here are two recent performance improvements
that are fixing some of the most burning issues related with consumer rebalancing:
* [static membership](https://www.confluent.io/blog/kafka-rebalance-protocol-static-membership/) - introduces <code>group.instance.id</code> param used for preserving
partition assignment between application restarts
* [incremental cooperative rebalancing](https://www.confluent.io/blog/incremental-cooperative-rebalancing-in-kafka/) - a new rebalancing algorithm that avoids
stop-the-world pauses (instead it allows more than one rebalancing round to
obtain final assignment and reduces the number of partitions that switch
hands during rebalancing)


Poll for messages in a loop
---------------------------

Now that our consumer is ready to accept new messages we can start polling:

{% highlight java %}
var records = consumer.poll(Duration.ofSeconds(1));
records.forEach(record ->
    System.out.println(
        record.offset() + " -> " + record.value()));
{% endhighlight %}

This is normally done in a long-running loop:
1. poll for new messages: <code>consumer.poll(...)</code>
2. process messages: <code>records.forEach(...)</code>
3. repeat (go back to step 1.)


The complete code of a naive implementation
-------------------------------------------

Putting it all together this is how the consumer's code in
the subscribing variant can look like:

{% highlight java %}
var props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.deserializer", StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());
props.put("group.id", "my-consumer-group");

try (var consumer = new KafkaConsumer<String, String>(props)) {
  consumer.subscribe(List.of("my-topic"));
  while (true) {
    var records = consumer.poll(Duration.ofSeconds(1));
    records.forEach(record ->
        System.out.println(
            record.offset() + " -> " + record.value()));
  }
}
{% endhighlight %}


Saving offsets
--------------

This code works fine, however if you are new to Kafka you may be wondering
how consumer knows where it finished reading, meaning how and where does
it store offsets.

It might be confusing at first, but offsets of read messages are saved
in a special topic called <code>__consumer_offsets</code>.

<div class="my-info">
In the past <strong>offsets used to be stored in ZooKeeper</strong>
which is probably even more puzzling.
</div>


Auto-commit
-----------

Alright, but the above code does not contain any logic for saving offsets.

And that is fine, because by default the consumer will commit
offsets automatically every interval configured in <code>auto.commit.interval.ms</code>
(5 seconds by default).

<div class="my-info">
You might be worried that this auto-commit functionality runs in the background
and therefore it could incorrectly mark a message as consumed when consumer
crashes or when an exception gets thrown. Actually, <strong>offsets are not
committed in some background thread, but are saved when you call
<code>consumer.poll(...)</code></strong>. So basically, if consumer crashes
then <code>poll</code> is not called again and records from the previous
<code>poll</code> do not get marked as consumed.
</div>


Where to start reading?
-----------------------

Another surprise that you can encounter is when you publish a message
before your consumer polls for new records:

{% highlight java %}
//  1. send
try (var producer = new KafkaProducer<String, String>(props)) {
  producer.send(new ProducerRecord<>("my-topic", value));
}

...

//  2. poll
var records = consumer.poll(Duration.ofSeconds(1));
{% endhighlight %}

If there is no offset stored for a consumer group (it is the first poll or
previously committed offset expired) then your consumer will look at the
<code>auto.offset.reset</code> parameter and either:

* start from the first record available if <code>auto.offset.reset=earliest</code>
* start from the end (awaiting new messages) if <code>auto.offset.reset=latest</code>

<img src="/images/kafka-consumer/earliest-latest.png"
title="Earliest vs latest" style="clear: both;" />

<div class="my-info">
Please note the <strong>first available offset can be larger than 0</strong>
(due to old messages being deleted according to your retention policy).
</div>

And as you may suspect, it is set by default to
<code>latest</code> which means __your consumer will not even try to
read messages produced before it first joined__.


Committing manually
-------------------

Although __committing less often is the preferred option__ from the performance standpoint,
__it can result in more messages being re-processed__ when a consumer crashes.
Another consumer has to read the same uncommitted messages and do the same
work once again.

__Problems could also arise if you would like to batch messages in memory__ so that
you can later process them in bulk. Remember that whenever you call poll then
records from the previous poll are being committed.
It means that all the messages batched so far will be considered consumed,
even though they were not yet processed. Now when a consumer crashes,
these messages will not be processed by another live consumer because they were
already marked as consumed. In other words, this will get you into trouble:

{% highlight java %}
var buffer = new ArrayList<ConsumerRecord<String, String>>();
while (true) {
  // DO NOT DO THIS WHEN enable.auto.commit=true
  var records = consumer.poll(Duration.ofSeconds(1));
  records.forEach(buffer::add);
  if (buffer.size() > THRESHOLD) {
    buffer.forEach(record -> ...);
    buffer.clear();
  }
}
{% endhighlight %}

For cases like that you may decide to take a fine-grained control over
how and when messages are committed. You can disable auto-commit by
setting <code>enable.auto.commit=false</code> and then commit manually by calling
either <code>commitSync()</code> or <code>commitAsync()</code>, depending on your
use-case.

If you just call the parameterless variant of the method
then it will commit all the records returned by this poll:

{% highlight java %}
var records = consumer.poll(Duration.ofSeconds(1));
...
consumer.commitSync(); // commits all the records from the current poll
{% endhighlight %}

But if you wish to be more specific then you can also provide
exactly which offsets should be committed for which
partitions. For instance like this:

{% highlight java %}
consumer.commitSync(Collections.singletonMap(
  partition, new OffsetAndMetadata(lastOffset + 1)
));
{% endhighlight %}


Single- vs multi-threaded
-------------------------

It is important to keep in mind that __<code>KafkaConsumer</code> is not
thread-safe__. Therefore it is __easier to correctly implement a consumer
with the "one consumer per thread" pattern__. In this pattern scaling boils down to
just adding more partitions and more consumers.

As an alternative, you can __decouple consumption and processing__ which should result in
__reduced number of TCP connections__ to the cluster and __increased
throughput__. On the downside, you will end up with __much more complicated code__
because not only will you need to coordinate threads but also commit only
specific offsets while preserving order between processed messages.

<div class="my-info">
You can <a href="https://kafka.apache.org/24/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html#multithreaded">read more about concurrency patterns in the
official JavaDoc</a> of <code>KafkaConsumer</code>.
</div>

If you are not discouraged by potential code complexity then [Igor Buzatovic
recently wrote an article explaining how to write a multi-threaded consumer](https://www.confluent.io/blog/kafka-consumer-multi-threaded-messaging/). Igor also provided
[a ready-to-use implementation](https://github.com/inovatrend/mtc-demo/blob/master/src/main/java/com/inovatrend/mtcdemo/MultithreadedKafkaConsumer.java)
if you would like to jump straight to the code.


Spring Kafka
------------

Up so far we have mostly complicated the code and discussed what can
go wrong. And it is good to be aware of various gotchas you can run into.

But we have strayed from our original goal which was to __come up with
the simplest Kafka consumer possible__.

When thinking about code simplification in Java then usually it is a good idea
to __check if Spring Framework__ offers an abstraction that __can make our code
more compact and less error-prone__.

Actually, there is more than one way how Spring can simplify our integration
with Kafka. For example you can use Spring Boot togther with Spring Kafka:

{% highlight xml %}
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
{% endhighlight %}

Then we only need to configure connection details in <code>application.yml</code>
(or <code>application.properties</code>):

{% highlight yml %}
spring.kafka:
  bootstrap-servers: localhost:9092
  consumer:
    group-id: my-group
    key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
{% endhighlight %}

...and add <code>@KafkaListener</code> annotation on a method that
will process messages:

{% highlight java %}
@Component
public class MyKafkaListener {

  @KafkaListener(topics = "my-topic")
  public void processMessage(ConsumerRecord<String, String> record) {
    System.out.println(record.offset() + " -> " + record.value());
  }
}
{% endhighlight %}

If you do not need to access messages' metadata (like partition, offset
or timestamp) then you can make the code even shorter:

{% highlight java %}
@KafkaListener(topics = "my-topic")
public void processMessage(String content) {
  System.out.println(content);
}
{% endhighlight %}

There is also an alternative way to access this kind of information
using annotations:

{% highlight java %}
@KafkaListener(topics = "my-topic")
public void processMessage(
    @Header(KafkaHeaders.PARTITION_ID) int partition,
    @Header(KafkaHeaders.OFFSET) int offset,
    @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long timestamp,
    @Payload String content) {
  // ...
}
{% endhighlight %}


Commit modes in Spring Kafka
----------------------------

We have already discussed that you can safely rely
on automatic committing for a wide range of use-cases. Surprisingly, __Spring Kafka by
default sets <code>enable.auto.commit=false</code>__ but actually makes it
work in a very similar way. What Spring does instead, is __emulate
auto-commit__ by explicitly committing after all the records from the poll
are finally processed.

This acknowledgment mode is called <code>BATCH</code> because Spring commits
messages from the previous batch of records returned by <code>poll(...)</code>

However, if you wish to manually commit offsets you can switch to <code>MANUAL</code> or
<code>MANUAL_IMMEDIATE</code> ACK mode. This can be accomplished by
changing Kafka Listener mode in your Spring Boot config:

{% highlight yml %}
spring.kafka:
  # ...
  listener:
    ack-mode: MANUAL_IMMEDIATE
{% endhighlight %}

Then you are expected to use the <code>Acknowledgment</code> object to mark
messages as consumed:

{% highlight java %}
@KafkaListener(topics = "my-topic")
public void processMessage(String content, Acknowledgment ack) {
  // ...
  ack.acknowledge();
}
{% endhighlight %}

<div class="my-info">
There is <a href="https://docs.spring.io/spring-kafka/reference/html/#committing-offsets">
a dedicated section about different acknowledgment modes</a>
in the official Spring Kafka documentation if you would like to read more about it.
</div>


Spring Cloud Stream
-------------------

Another option  is to use __Spring Cloud Stream with
Kafka Binder__:

{% highlight xml %}
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-stream</artifactId>
</dependency>
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-stream-binder-kafka</artifactId>
</dependency>
{% endhighlight %}

Spring Cloud Stream can help you __write even more generic code__ that you can
quickly integrate with a variety of modern messaging systems (e.g. RabbitMQ,
Apache Kafka, Amazon Kinesis, Google PubSub and more).

The basic concept is that you just provide implementations
of Java functional interfaces:

* <code>java.util.function.Supplier</code> for sources
* <code>java.util.function.Consumer</code> for sinks
* <code>java.util.function.Function</code> for processors

...and then in your configuration you bind them to specific queues or topics
in the messaging system(s) of your choice.

<div class="my-info">
There is <a href="https://youtu.be/5Mgni6AYnWg">a very good talk by
Soby Chacko and Oleg Zhurakousky from Pivotal</a> about integrating
Spring Cloud Stream with Kafka, explaining the approach with Java
functional interfaces. Oleg Zhurakousky also wrote
<a href="https://spring.io/blog/2019/10/14/spring-cloud-stream-demystified-and-simplified">an interesting article explaining the motives behind
the move  to functional programming model</a> in Spring Cloud Stream.
</div>

For example, you can quickly put together a Spring Boot application like this:

{% highlight java %}
// ...
import java.util.function.Consumer;

@SpringBootApplication
public class KafkaConsumerApplication {

  public static void main(String[] args) {
    SpringApplication.run(KafkaConsumerApplication.class, args);
  }

  @Bean
  public Consumer<String> myMessageConsumer() {
    return content -> System.out.println(content);
  }
}
{% endhighlight %}

Please note the <code>java.util.function.Consumer</code> implementation
(returned from <code>myMessageConsumer()</code> method) that you can replace
with your own logic for processing messages. After you finish the implementation
of this interface you just need to properly bind it. As an example, you can configure
the consumer to read from <code>my-topic</code> using <code>my-group</code> as
consumer group:

{% highlight yml %}
spring.cloud.stream:
  bindings:
    myMessageConsumer-in-0:
      destination: my-topic
      group: my-group
{% endhighlight %}

The name of the binding looks peculiar, but in most cases this will just be
<code>&lt;function name&gt;-in-0</code> for inbound and
<code>&lt;function name&gt;-out-0</code> for outbound topics.

<div class="my-info">
Please refer to <a href="https://cloud.spring.io/spring-cloud-stream/reference/html/spring-cloud-stream.html#_binding_and_binding_names">documentation on binding names</a>
for more details (especially if you are curious about the <code>-0</code> suffix
representing the index of the binding).
</div>

You might also notice that we have not specified any <code>bootstrap.servers</code>.
By default it connects to a Kafka cluster running on <code>localhost:9092</code>.

It can be easily changed to a different list of brokers:

{% highlight yml %}
spring.cloud.stream:
  kafka.binder:
    brokers: my-node1:9090,my-node2:9090,my-node3:9090
{% endhighlight %}

Leveraging Spring Cloud Stream totally __decoupled our code from Kafka__.
Now it is possible to switch to an entirely different message broker without
having to do a single code change. All this can be done through configuration
changes.


Parting thoughts
----------------

To sum up, __Spring Kafka and Spring Cloud Stream with Kafka Binder offer quick
ways to build incredibly succinct code for reading messages from Kafka__.

That being said, even though the abstractions provided by Spring framework
are usually very good, it can be beneficial to know what happens under the hood.

Here are some topics worth exploring:
* [consumer rebalancing](https://kafka.apache.org/20/documentation.html#impl_consumerrebalance)
* [committing offsets](https://docs.confluent.io/current/clients/consumer.html#offset-management)
* [multi-threaded consumer](https://www.confluent.io/blog/kafka-consumer-multi-threaded-messaging/)
