---
layout: post
title:  "Thoughts on Software Architecture"
date:   2016-04-03 18:27:38
categories: architecture
---
I have just finished reading [Software Architecture for Developers](https://leanpub.com/software-architecture-for-developers) by Simon Brown. It is a very nice and well-structured book. Any aspiring or
experienced architect should find something useful in it. In this post I
would like to summarize what I have found especially interesting and reference
some other great sources on this topic.

A few words about the author
----------------------------
If you have not met Mr Simon Brown yet, he has done a lot of work on preaching
gospel about software architecture. Aside from this great book he made tons
of presentations on multiple conferences. To just link one of his cool
presentation [checkout this one](https://www.youtube.com/watch?v=oDpdaXt0HQI).
In the above video Brown describes succinctly his approach to software
architecture.

So what really is software architecture?
----------------------------------------
Simon provides a lot of views on what different people think software architecture
is. His definition says that it is **anything related to the
significant elements of a software system**. I like the explanation that it is
**everything that is hard to change**, everything that will cost us a lot of money
and time to change. Grady Booch quote fits here perfectly:

> Architecture represents the significant design decisions that shape a system,
> where significance is measured by cost of change.

Do we really need it?
---------------------
As Brown highlights many people think that when incorporating agile in their
project then architecture and documentation are no longer needed. They want to avoid
*big design upfront*, but actually go to the other extreme. There is also
a problem in thinking that *code is the documentation*. This approach is
somewhat true, however there are a lot of questions that the code itself cannot
answer (like 'why these technologies have been chosen', 'how scalability
is achieved', 'how security is ensured', etc.). We should bear in mind that
too much documentation is bad, but too little is also not any good. I like
Brown's clarification on this topic:

> Just enough up front architecture and design

Benefits
--------
So what can be accomplished with software architecture? You get a fair amount
of benefits:

* express goals of the system (requirements and constraints)
* create a clear vision and roadmap
* identify and avoid risks
* ignite discussions across the team about how software should work (*architecture
  as a platform for conversation*)
* introduce standards and guidelines

The C4 model
------------
With the strong decline of UML use in modern IT projects the model suggested by
Brown seems to be a lighter and more sensible solution.

C4 stands for:

* context - a **birds-eye view of the system** with its dependencies and actors
  (systems/users) that even **non-technical people** should understand
* containers - a more-detailed diagram showing **technology choices and distribution
  of responsibilites** accross the system; this diagram is meant for **technical
  people** that are trying to grasp what containers are in the system (by container
  you should understand web server, database, file system, etc.); mentioning
  protocols used by the system (HTTP, JMS, etc.) can be very beneficial here
* components - a diagram explaining the components/services: how the **system is
  structured** and what are the **containers consisting of**?
* classes - **optional** diagram of classes in the system (usually code is enough
  at this point)

Software architecture guide book
--------------------------------
Simon published [a free guide book](https://leanpub.com/techtribesje) on how
you can introduce the C4 model to your projects. Beneath are some diagrams
presented in the book. I would be more than grateful, if I could receive
such documentation for a system I am taking over.

Context
=======
![techtribes.je context diagram](/images/thoughts-on-software-architecture/context.png)

Containers
==========
![techtribes.je containers diagram](/images/thoughts-on-software-architecture/containers.png)

Components
==========
![techtribes.je components diagram](/images/thoughts-on-software-architecture/components.png)

Structurizr
-----------
In order to incorporate C4 model Simon is recommending [Structurizr](https://www.structurizr.com).
It is not free, but still worth considering. You can [check here](https://www.structurizr.com/public/21#Context)
how the [techtribes.je project](http://techtribes.je) is visualised in Structurizr.

Software architecture checklist
-------------------------------
Brown provides a great checklist that can be run against the C4 model (this is
an excerpt from his book):

* I can see and understand the solution from multiple levels of abstraction.
* I understand the big picture; including who is going to use the system (e.g. roles, personas, etc) and what the dependencies are on the existing IT environment (e.g. existing systems).
* I understand the logical containers and the high-level technology choices that have been made (e.g. web servers, databases, etc).
* I understand what the major components are and how they are used to satisfy the important user stories/use cases/features/etc.
* I understand what all of the components are, what their responsibilities are and can see that all components have a home.
* I understand the notation, conventions, colour coding, etc used on the diagrams.
* I can see the traceability between diagrams and diagramming elements have been used consistently.
* I understand what the business domain is and can see a high-level view of the functionality that the software system provides.
* I understand the implementation strategy (frameworks, libraries, APIs, etc) and can almost visualise how the system will be or has been implemented.

Some more stuff to ponder on...
-------------------------------
O'Reilly has a very cool set of videos on software architecture presented by
Neal Ford and Mark Richards - two fantastic architecture masters. I would
recommend to start with "[Understanding the Basics](http://shop.oreilly.com/product/110000195.do)"
and then move on to "[Beyond the Basics](http://shop.oreilly.com/product/110000197.do)".
After that your brain will be filled with a lot of concepts around software
architecture.
