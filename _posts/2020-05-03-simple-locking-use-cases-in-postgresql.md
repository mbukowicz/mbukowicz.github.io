---
layout: post
title: "Simple locking use-cases in PostgreSQL"
date: 2020-05-03
categories: databases
thumbnail: /images/simple-locking-use-cases-in-postgresql/security-2168233_1280.jpg
---

<div style="text-align: center;">
  <img src="/images/simple-locking-use-cases-in-postgresql/security-2168233_1280.jpg"
  title="Locks in PostgreSQL" class="rounded" />
</div>

Intro
-----

Although relying on techniques based on optimistic locking
([like MVCC for example](/databases/2020/05/01/snapshot-isolation-in-postgresql.html))
is usually a better choice, especially in terms of performance,
there are other locking mechanisms that are worth consideration. Contrary to
optimistic strategies, where it is assumed that conflicting updates tend to happen
rather rarely, locking introduces contention. But it can also make
the code simpler and easier to reason about.

What's more, in case of __PostgreSQL__ it __automatically acquires exclusive locks
on rows when they are updated or deleted__ so you might not get away without
blocking anyway.


Lost updates problem
--------------------

When running in READ COMMITTED mode you may stumble upon the lost updates
problem. In the most basic scenario it can occur if you select a record, run
some computation and then update it with the results of that computation.

For example, __transferring 100$ to Bob's account__ could be implemented
__in 2 steps__:

{% highlight sql %}
-- 1) read the current balance
SELECT balance
FROM accounts
WHERE owner = 'Bob'; -- balance: 500$

-- 2) increase the balance by 100$
UPDATE accounts
SET balance = 600 -- 500$ + 100$
WHERE owner = 'Bob';
{% endhighlight %}

If two transactions fetch the same balance before one of them updates and commits
the updated account the value will be overwritten and an update will be lost
(hence the name of this phenomena - lost updates).

And this __can be simply solved by levelling up to REPEATABLE READ__
as [described in more detail in my previous post](/databases/2020/05/01/snapshot-isolation-in-postgresql.html).


Explicit locking as an alternative
----------------------------------

But we could also take an alternative approach and use some form of explicit
locking while still running in READ COMMITTED.

PostgreSQL (like many other RDBMS) has a quite useful _SELECT ... FOR UPDATE_
construct to exclusively lock only the selected record(s) until the transaction finishes:

{% highlight sql %}
SELECT balance
FROM accounts
WHERE owner = 'Bob'
FOR UPDATE; -- locks the row
{% endhighlight %}

We then just need to update the locked row and commit the transaction:

{% highlight sql %}
UPDATE accounts
SET balance = 600
WHERE owner = 'Bob';
COMMIT; -- releases the lock
{% endhighlight %}

It has a similar effect to UPDATE or DELETE which acquires an exclusive row-level
lock for each of the affected row for the duration of the transaction.


Atomic increment
----------------

This works fine, but we can actually make it better. Operations such as incrementation
can be usually done in an atomic fashion, so we can get the job done with __just
one statement__:

{% highlight sql %}
UPDATE accounts
SET balance = balance + 100
WHERE owner = 'Bob';
{% endhighlight %}

<div class="my-info">
A small note: in case of databases other than PostgreSQL
you should <strong>consult the docs of your DBMS</strong>
to confirm that this is indeed an atomic operation
as it may not always be true for all databases.
</div>

Please bear in mind that since it modifies rows in a table __it also implicitly
makes use of locks__, which can have some undesirable side effects if not used
carefully (but more on that in a second).

Transfer money between accounts
-------------------------------

Let's complicate our example a little bit by bringing another account
into the picture. Imagine we need to __transfer 100$ from Bob to Alice__. We could
take advantage of what we have used so far and implement it like this:

{% highlight sql %}
UPDATE accounts
SET balance = balance - 100
WHERE owner = 'Bob';

UPDATE accounts
SET balance = balance + 100
WHERE owner = 'Alice';

COMMIT;
{% endhighlight %}

Deadlock
--------

It pleases the eye with its simplicity, but there is one caveat.
If two transactions interleave in a certain way we could run into __a deadlock__:

<div style="width: 49%; float: left; margin-right: 2%">
{% highlight sql %}
-- TRANSACTION #1

UPDATE accounts
SET balance = balance - 100
WHERE owner = 'Bob';
-- locks Bob's account





UPDATE accounts
SET balance = balance + 100
WHERE owner = 'Alice';
-- waits for a lock on Alice's account
{% endhighlight %}
</div>

<div style="width: 49%; float: left;">
{% highlight sql %}
-- TRANSACTION #2





UPDATE accounts
SET balance = balance + 300
WHERE owner = 'Alice';
-- locks Alice's account

UPDATE accounts
SET balance = balance - 300
WHERE owner = 'Bob';
-- waits for a lock on Bob's account
{% endhighlight %}
</div>

<div style="clear: both;"></div>

To automatically unstuck in such situation you would need to run a database
that actively looks out for deadlocks.

For example, __PostgreSQL would discover that you are in trouble__ and it would
cancel one of the transactions allowing the second one to finish successfully:

{% highlight plain %}
ERROR:  deadlock detected
DETAIL:  Process 57351 waits for ShareLock on transaction 1269; blocked by process 57321.
Process 57321 waits for ShareLock on transaction 1270; blocked by process 57351.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
SQL state: 40P01
{% endhighlight %}

If you would like to prevent deadlocks from ever happening in your application
then you should __acquire the locks in a certain order__.

As an example, we could first try to take
lock on an account with a higher id or owned by a user whose name is alphabetically
higher on the list of accounts (assuming names are unique).


The end
-------

I hope you have enjoyed this post. But before we part ways here is
[a link to an information-rich article about locks
in PostgreSQL](https://engineering.nordeus.com/postgres-locking-revealed/)
that you might be interested in.
