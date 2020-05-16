---
layout: post
title: "CAP theorem in a nutshell"
date: 2020-04-19
categories: distributed-systems
thumbnail: /images/cap-theorem-pragmatically/thumbnail.png
---

<div style="text-align: center; margin: 1em;">
  <img src="/images/cap-theorem-pragmatically/cap-diagram.png"
  title="CAP diagram" />
</div>

Why worth knowing?
------------------

When designing a new system __resiliency__ is usually made __the top priority__.
Most often the goal is to build a system that is as fault-tolerant and
immune to outages as possible. Whether this should be of such great importance
is another topic, but what is quite interesting is __how reliable a system can become__.
In other words, what is the limit we are going to eventually reach when trying
to max out the reliability of our system.

CAP theorem can be pretty helpful with establishing such limit. Or actually it can
help us become more __aware of the trade offs__ lurking behind the architectural
choices we are making.

Another down-to-earth argument for learning about CAP theorem is that it is
__often asked about during job interviews__ (especially when applying for backend roles)
so getting to know it better could potentially improve your performance on
the interviews.

What is it?
-----------

<img src="/images/cap-theorem-pragmatically/cap-diagram-extended.png"
title="Extended CAP diagram" style="clear: both;" />

CAP theorem states that at one time a distributed system can only provide
__2 out of 3 available guarantees__:

* <strong>C</strong> onsistency - every read returns the most recent write
(it looks like system contains a single up-to-date copy of the data)

<img src="/images/cap-theorem-pragmatically/consistency.png"
title="Consistency" style="clear: both;" />

* <strong>A</strong> vailability - every request receives a non-error response (system
  returns data although it may not be the latest update)

<img src="/images/cap-theorem-pragmatically/availability.png"
title="Availability" style="clear: both;" />

* <strong>P</strong> artition tolerance - system continues to operate despite
broken communication between nodes

<img src="/images/cap-theorem-pragmatically/partition-tolerance.png"
title="Partition tolerance" style="clear: both;" />

This sounds __similar to hiring a contractor__ in order to renovate
your apartment or house.

The contractor can be only two of:
* cheap
* good
* fast

If a contractor is cheap then you can expect either poor result or job not being
done in a timely manner. When the results are to be anywhere good then you should
either equip yourself with patience or be ready to pay more. Finally, when
you require all the work to complete in a short period of time then you have
to make a decision what is more important: reducing the costs or getting
better quality (which seems also relatable in the software world).

<img src="/images/cap-theorem-pragmatically/contractor-diagram.png"
title="Contractor diagram (equivalent to CAP)" style="clear: both;" />

All in all, __you cannot have all three__ and that is just the way it is.

Proof
---------------------

Let's try to use common sense to confirm that CAP theorem is indeed true.

Imagine we have a simple distributed system where data is written to a leader
that replicates data to a replica from which data is being read:

<img src="/images/cap-theorem-pragmatically/cap-system.png"
title="System" style="clear: both;" />

If nothing bad ever happened then such system would work well but in reality
there is always a chance that __connectivity__ between leader and replica
__breaks down__:

<img src="/images/cap-theorem-pragmatically/connection-breakdown.png"
title="Connection breakdown" style="clear: both;" />

Unfortunately, this means that if __replica__ contains stale data then it
__cannot be refreshed__:

<img src="/images/cap-theorem-pragmatically/network-partition-problem.png"
title="Connection breakdown" style="clear: both;" />

Availability or consistency?
----------------------------

We have to make a tough call between __sacrificing consistency__ and returning old data:

<img src="/images/cap-theorem-pragmatically/losing-consistency.png"
title="Loosing consistency" style="clear: both;" />

...or __losing availability__ by throwing back an error:

<img src="/images/cap-theorem-pragmatically/losing-availability.png"
title="Loosing availability" style="clear: both;" />

Practical application
---------------------

It sounds really bad that we are forced to make such decisions. Luckily, though
CAP theorem stands true and we have to accept it as a fact, the problem
is not so black and white. Quite the opposite, there is __a spectrum of available
options__ for us to choose from.

First of all, there is a wide range of countermeasures to greatly __reduce
the probability of a network partition__, which happens rarely anyways.
Then we can usually pick __different strategies depending
on a scenario__: we might not be willing to give up on consistency when dealing
with money but being presented with somewhat older number of users currently
signed in to our application could be perfectly acceptable.

What's important, a choice how to react to a network partition can be made on
a case-by-case basis within the same system. Most modern database and messaging
systems usually provide a number of parameters we can pass to read and write
operations that affect whether we opt for consistency or availability.

Want to learn more?
---------

In order to learn more [here you can read an article](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) containing Eric Brewer's
considerations many years after he came up with the CAP theorem.
