---
layout: post
title: "Snapshot isolation in PostgreSQL"
date: 2020-05-01
categories: databases
---

<div style="text-align: center; margin: 1em;">
  <img src="/images/snapshot-isolation-in-postgresql/block-chain-2853046_1280.jpg"
  title="Fancy transactions flying around" />
</div>

Intro
-----

No matter whether your application operates on a common RDBMS or a bleeding-edge
NoSQL database you still need to __cater for proper isolation__ of operations done
by your end users.

Although the trade-offs made by the creators of each database can be slightly
different the mechanics and limitations stay the same. If you understand
the basic concepts and possible strategies for implementing isolation then
later it can make your life much easier when you need to quickly get up to
speed with a new database.

In this article we will take a look at how PostgreSQL solves the isolation problem
by leveraging optimistic approach to locking.

The problem
-----------

Even the most trivial-sounding problems require some understanding how different isolation levels are implemented in a particular database. Take for example the classic
__problem of debit and credit in an account__.

Let's start with something simple. Suppose we just want to add 100$ to Bob's account.
We could write SQL code so that we first fetch the current value:

{% highlight sql %}
SELECT balance FROM accounts WHERE owner = 'Bob'; -- balance: 500$
{% endhighlight %}

...and then increment it by 100. Assuming the returned balance was 500 we just
need to add 100:

{% highlight sql %}
UPDATE accounts
SET balance = 600 -- 500$ + 100$
WHERE owner = 'Bob';
{% endhighlight %}


Lost updates
------------

There is, however, a potential problem that __other transaction could jump
in-between__ and some data would get lost. It means that an overlapping
transaction aiming to increase Bob's balance by 300$ could finish successfully,
only that Bob would not receive his money.

<div style="width: 50%; float: left;">
{% highlight sql %}
-- TRANSACTION #1

SELECT balance
FROM accounts
WHERE owner = 'Bob';
-- balance: 500$












UPDATE accounts
SET balance = 600
WHERE owner = 'Bob';
COMMIT;

-- balance: 600$
-- (300$ was lost)
{% endhighlight %}
</div>

<div style="width: 50%; float: left;">
{% highlight sql %}
-- TRANSACTION #2






SELECT balance
FROM accounts
WHERE owner = 'Bob';
-- balance: 500$

UPDATE accounts
SET balance = 800
WHERE owner = 'Bob';
COMMIT;
-- balance: 800$






-- balance: should be 900$
-- (500$ + 300$ + 100$)
{% endhighlight %}
</div>

<div style="clear: both;"></div>


Is this an isolation level problem?
-----------------------------------

According to SQL standard there are
[4 isolation levels](https://en.wikipedia.org/wiki/Isolation_(database_systems))
to pick from and it is quite common in relational databases to have READ COMMITTED
as the default isolation mode.

And if you stick to the default isolation level in PostgreSQL you might run into
the exact problem we have just experienced because it happens only in READ UNCOMMITTED
and READ COMMITTED.

One way to overcome this is to just raise the isolation level to
__REPEATABLE READS to prevent losing updates__.

When you run the following code in PostgreSQL one of interleaving transactions will
crash and it will need to be manually retried from your application:

{% highlight sql %}
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT balance
FROM accounts
WHERE owner = 'Bob';

UPDATE accounts
SET balance = ...
WHERE owner = 'Bob';
-- it will crash here if Bob's account has been modified
-- since the beginning of this transaction

COMMIT;
{% endhighlight %}

If you are curious this is the error that will be thrown:

{% highlight plain %}
ERROR:  could not serialize access due to concurrent update
SQL state: 40001
{% endhighlight %}

But how was this implemented in PostgreSQL? How did PostgreSQL realise
that a record was modified from another transaction?


Snapshot isolation
------------------

A popular solution to this problem is called snapshot isolation. When working in
REPEATABLE READ mode, at the beginning of each transaction
__PostgreSQL takes a consistent snapshot of the database__. When you
then query the data you only get records committed up to this instant in time.
Later changes are not visible and conflicts result in aborting the transaction
(as we have just observed).

Taking a snapshot sounds like a really expensive operation that is going to
absolutely kill the performance of PostgreSQL...


MultiVersion Concurrency Control (MVCC)
---------------------------------------

But actually it is not that terrible thanks to a technique called MVCC, standing
for MultiVersion Concurrency Control, which does the job of providing consistent
views of the data while keeping the performance in check.

<div class="my-info">
MVCC is <a href="https://en.wikipedia.org/wiki/List_of_databases_using_MVCC">
commonly used in many relational, as well as in NoSQL, databases</a>,
including Oracle, SQL Server, PostgreSQL, MongoDB and CouchDB just to name a few.

The way MVCC is implemented may vary significantly, but it is widely used in
modern databases.
</div>

MVCC, as the name suggests, maintains __several versions of the same record__,
meaning instead of overwriting the changed data a new version of the record
is created.

Before we get to how PostgreSQL knew it needed to abort the conflicting transaction
we need to go through some concepts so please bear with me.


xid
---

First of all, in order to distinguish transactions, each one of them receives
a special identifier, called __xid__,
which is __a sequential number uniquely identifying a transaction__.

<img src="/images/snapshot-isolation-in-postgresql/xid.png"
title="Transaction id taken from a sequence" style="clear: both;" />


xmin and xmax
-------------

Then each version of a record gets additional metadata:

* __xmin__ - xid of the __transaction that created__ the record
* __xmax__ - xid of the __transaction that deleted__ the record

When a new account is INSERTed PostgreSQL creates a record with xmin set to the current
transaction id (xid) and leaves xmax empty. If you later decide to UPDATE the account
then __instead of overwriting the existing record a new one is created__. The old
record becomes invisible by setting xmax to the transaction that updated it.
Finally, if the account was meant to be DELETED then all there is to do is to
properly set xmax of the last existing record.

<img src="/images/snapshot-isolation-in-postgresql/xmin_xmax.png"
title="How xmin/xmax is set when doing INSERT, UPDATE or DELETE"
style="clear: both;" />


Other in-progress transactions
------------------------------

The last piece of the puzzle is information about __other active transactions__.
Whenever PostgreSQL takes a snapshot it collects information about the
currently open transactions that have not yet finished. With such information
it can later efficiently resolve conflicts because then it can discover
that another transaction has just modified the record that
the current transaction was about to update. Active transactions are
described in PostgreSQL in the following format:

xmin | : | xmax | : | xip_list |
-----|---|------|---|----------|
earliest active transaction | | first as-yet-unassigned transaction | | list of in-progress transactions |

In the simplest case all transactions have already finished (some finished
successfully and some were aborted):

<img src="/images/snapshot-isolation-in-postgresql/xmin_xmax_xip_1.png"
title="Simple scenario" style="clear: both;" />

But most likely there will be some other transaction(s) currently running in
the system:

<img src="/images/snapshot-isolation-in-postgresql/xmin_xmax_xip_2.png"
title="Snapshot with other active transactions" style="clear: both;" />

While the current transaction is still open another transaction
could start and then complete before the current one finishes (here xid=45
started after snapshot for xid=44 was taken):

<img src="/images/snapshot-isolation-in-postgresql/xmin_xmax_xip_3.png"
title="Snapshot with transactions completed in the future" style="clear: both;" />


How does it all play together?
------------------------------

Returning to our example, below you can find a diagram showing how
PostgreSQL came to the conclusion that a conflict has occurred and that
transaction #2 needs to be aborted:

<img src="/images/snapshot-isolation-in-postgresql/mvcc_conflict_1.png"
title="MVCC conflict explained" style="clear: both;" />

The first transaction (#1) succeeds which results in a new version of Bob's
account to be created.

But when transaction #2 tries to update Bob's account PostgreSQL discovers that
a new version of the record was created since transaction #2 has began and
it can only be resolved by retrying from the application code (and so it crashes).

This was possible to deduce because xid of transaction #1 (__xid=41__) was on
the list of active transactions (41:42:__[41]__) when transaction #2 started
(it means that transaction #1 had to change the record in the meantime).

It could also happen that transaction #2 updated sooner than #1:

<img src="/images/snapshot-isolation-in-postgresql/mvcc_conflict_2.png"
title="Another scenario of MVCC conflict" style="clear: both;" />

Transaction #1 figures out there is a conflict because __xid=42__ is greater than xmax
in the snapshot (41:__41__:) so it had to be updated by a transaction that started
later (a future transaction from the standpoint of transaction #1).

<div class="my-info">
<p>This is only a part of the story. PostgreSQL could also realise during UPDATE
that another transaction is updating the same record. In that scenario
it will wait for this transaction to finish. If the other
transaction aborts then it can proceed with its update. Otherwise
it needs to fail. <strong>First updater wins</strong>.</p>
<p style="margin-bottom: 0;">How all possible scenarios are handled is
<a href="http://www.interdb.jp/pg/pgsql05.html#_5.8.1.">explained here</a>.</p>
</div>


Drawbacks of PostgreSQL's MVCC implementation
---------------------------------------------

As you can see, MVCC is quite a clever technique.
On the downside, it obviously __takes more disk space__ because all those
additional versions need to be stored somewhere.

The challenge here is to protect the database from bloating caused by
the old versions of records that are not used anymore by any transaction but
are still taking up the space. There needs to be a __process sweeping the
database__ and removing such unnecessary data (think garbage collector for
databases). In PostgreSQL such process is called autovacuum ("auto" because
you can also run vacuum manually whenever you think is the right moment).

Hypothetically, this could be turned into an advantage if your use case was to
store historical data. You could then go back in time and compare values
from different points in the past.

Another potential issue is the wraparound of the counter generating sequential
transaction identifiers. Counters have limits and eventually they reach their max values.
At some point __xid will need to roll over and start from zero__. If incorrectly
set up, or even worse: when disabled, it can hit back in the least
expected moment ([which can have unpleasant consequences for you and your users](https://blog.sentry.io/2015/07/23/transaction-id-wraparound-in-postgres)).


Displaying balances of two accounts
------------------------------------

Going back to the accounts problem, what if now __Bob wanted to transfer money
to Alice__? It would be convenient for him if he could monitor the progress of
this action.

__While the transfer takes place he would like to see both his and Alice's balance__.
Let's assume for the sake of this example that it is perfectly legal for Bob to
see how much money Alice has in her account. How does MVCC comes into play?

Here is what would happen if a transaction reading balances interleaved with
a transaction that updated both accounts:

<img src="/images/snapshot-isolation-in-postgresql/mvcc_snapshot_isolation.png"
title="Snapshot isolation in MVCC" style="clear: both;" />

Transaction #1 (xid=48) starts first and manages to fetch balance of Bob's
account before the second transaction (xid=49) updates this account.

But then transaction #2 updates Alice's account before transaction #1 gets
a chance to check its balance.

Now, depending on the isolation level, transaction #1 either:

* returns the new value (700$) in READ COMMITTED mode, since this is what was
  literally just "committed" (but then Bob would see inconsistent view of
  the data)
* ...or returns 500$ if in REPEATABLE READ mode (the value is older but it
  coherently shows a state of the database at some point of time)

In this case, the second option sounds like a more reasonable thing to do. At least
it should behave less surprising for Bob.


Read more
---------

If you have stayed till the end and it made you eager to learn more then Martin
Kleppmann's book, ["Designing Data-Intensive Applications"](https://dataintensive.net),
should be able to satisfy your curiosity. It is a tough and long read but
it is definitely worth the time and effort.

Another good source is PostgreSQL documentation. There is
[a whole chapter](https://www.postgresql.org/docs/current/mvcc.html)
that contains a succinct explanation of concurrency problems that could happen
and how they were solved in PostgreSQL.

However, you may find those docs a little bit shallow because they are lacking a
detailed explanation how each concurrency mechanism was actually implemented.
Fortunately, there exists [an extensive documentation](http://www.interdb.jp/pg/pgsql05.html) that will take you on a deep dive on all the intricacies
of PostgreSQL internals.
