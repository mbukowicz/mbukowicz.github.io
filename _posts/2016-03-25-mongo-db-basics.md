---
layout: post
title:  "MongoDB Basics"
date:   2016-03-25 15:16:15
categories: no-sql mongo-db
thumbnail: /images/mongodb-basics/thumbnail.png
---
What is MongoDB?
----------------
MongoDB is a **document database**. What it means is that your data does not
have to conform to any schema. In contrary to relational databases
where records not having some attribute are expected to set a *NULL* value in
that column, in document databases it is totally acceptable.

How MongoDB stores data?
------------------------
MongoDB stores documents in JSON-like format called BSON. However, JSON is used
not only for storing and retrieving the documents. It is all over the MongoDB API,
so you will be using JSON for writing queries as well.

Differences in naming between relational and document DBs
-----------------------------------------------------
In case you are already familiar with some relational database (like Oracle)
it is much easier to wrap your head around MongoDB when comparing concepts
between this 2 types of DBs.

Below is a table showing how a concept in Oracle database matches a parallel
concept in the MongoDB realm.

Oracle       | MongoDB
-------------|---------
schema       | database
table        | collection
row          | document
ROWID        | \_id

Basics
------
When connecting to MongoDB using the *mongo* client the first thing you need to
do is select the database with the *use* command:

{% highlight plain %}
> show dbs
my-blog          0.034GB
> use my-blog
switched to db my-blog
> db
my-blog
{% endhighlight %}

Insert
---------
To insert a new *document* into a *collection* you can use the *insert* command:
{% highlight plain %}
> db.posts.insert(
  {
    "title": "First post",
    "content": "Wow, this is so cool!"
  }
)
{% endhighlight %}

Supported data types
-------------------
Generally, because MongoDB uses BSON under the hood, we can use JSON flavored
data types:

 Type                          | Example                
-------------------------------|------------------------
 Number (ints, longs, doubles) | 1234, 56.78            
 String                        | "First post"           
 Object                        | { "name": "Sam" }      
 Array                         | [ 1, 2, 3 ]            
 ObjectId                      | ObjectId("...")        
 Boolean                       | true/false             
 Date                          | new Date("2016-03-25")
 Timestamp                     | new Timestamp()        
 Null                          | null                   

MongoDB validations
-------------------
When changing data MongoDB does some simple validations:

* it checks whether *\_id* in the document is unique (no other document has the
  same *\_id*)
* there are no syntax errors
* document is less than 16mb

Find all
--------
To find every document stored inside the *posts* collection simply call *find*
without parameters:
{% highlight plain %}
> db.posts.find()
{ "title": "First post", ... }
{ "title": "Second post", ... }
{ "title": "Third post", ... }
{% endhighlight %}

If you look closer, you will find out that MongoDB has generated a unique
identifier for your documents:
{% highlight plain %}
> db.posts.find()
{
  "_id": ObjectId("123456789abcdefgh"),
  "title": "First post",
  ...
}
...
{% endhighlight %}

Find with query
---------------
To search for documents matching a specific criteria we can add an additional
parameter:
{% highlight plain %}
> db.posts.find({ "title": "First post" })
{ "title": "First post", ... }
{% endhighlight %}

Query embedded data
-------------------
How about a case when we would like to find all documents where "Bob" has left
a comment? Suppose that our *posts* collection looks like this:
{% highlight plain %}
{
  "title": "First",
  "comments": [ { "author": "Sam", ... }]
}
{
  "title": "Second",
  "comments": [ { "author": "Alice", ... }, { "author": "Bob", ... } ]
}
{
  "title": "Third",
  "comments": [ { "author": "Bob", ... }]
}
{% endhighlight %}

To write a query matching an embedded document we can use dot-notation:
{% highlight plain %}
> db.posts.find({"comments.author": "Bob"})
{
  "title": "Second",
  "comments": [ { "author": "Alice", ... }, { "author": "Bob", ... } ]
}
{
  "title": "Third",
  "comments": [ { "author": "Bob", ... }]
}
{% endhighlight %}

Embedding vs referencing
------------------------
The obvious advantage of embedding is that you can **fetch embedded documents
in one query**. Otherwise, you would **have to fetch referenced documents in
multiple queries**. Additionally, **with embedded documents you get atomic writes**,
whereas with reference documents we have to do some extra work to ensure
consistency.

To embed or to not embed?
---------------------------
You have to ask yourself a series of questions:

 * how often is the data used together (often -> go for embedded)
 * how many documents are expected to be embedded (more than a few hundred ->
   you should reconsider referencing)
 * how often will the embedded document change (often -> go for referencing)

The general rule of thumb is to start with embedding and move to referencing
when you need to access documents independently or you start to get very large
sets of embedded documents.

Remove
------
To remove documents you can simply use the *remove* command:
{% highlight plain %}
> db.posts.remove( { "title": "Unpopular post" } )
{% endhighlight %}

Update
------
As expected to update a document you can use the *update* command:
{% highlight plain %}
> db.posts.update(
  { "title": "Unpopular post" },
  { "content": "More attractive content..." }
)
{% endhighlight %}

**Watch out here!** This way you would replace the whole document with the
following content:
{% highlight plain %}
{ "content": "More attractive content..." }
{% endhighlight %}

What you probably expected is this result:
{% highlight plain %}
{
  "title": "Unpopular post",
  "content": "More attractive content..."
}
{% endhighlight %}

If what you really wanted is to only replace the *content* property then you
should use this operation instead:
{% highlight plain %}
> db.posts.update(
  { "title": "Unpopular post" },
  { "$set": { "content": "More attractive content..." } }
)
{% endhighlight %}

Another caveat is that this command will **change only the first matching
document**. If you want to update all the matching documents use *multi* option
instead:
{% highlight plain %}
> db.posts.update(
  { "title": "Unpopular post" },
  { "$set": { "content": "More attractive content..." } },
  { "multi": true }
)
{% endhighlight %}

Update operators
----------------
There are other useful operators you can use to modify your data. To name just
a few:

Operator    | Before                     | Operation                                 | After
------------|----------------------------|-------------------------------------------|------------------
$inc        | { "quantity": 2 }          | { "$inc": { "quantity": 3 }}              | { "quantity": 5 }
$mul        | { "price": 2.2 }           | { "$mul": { "price": 2.5 }}               | { "price": 5.5 }
$max        | { "highest": 6 }           | { "$max": { "highest": 9 }}               | { "highest": 9 }
            | { "highest": 6 }           | { "$max": { "highest": 5 }}               | { "highest": 6 }
$addToSet   | { "letters": ["a"] }       | { "$addToSet": { "letters": ["b"] }}      | { "letters": ["a", "b"] }
            | { "letters": ["a"] }       | { "$addToSet": { "letters": ["a"] }}      | { "letters": ["a"] }
$unset      | { "a": 1, "b": 2 }         | { "$unset": { "b": "" }}                  | { "a": 1 }
$rename     | { "oldy": "val" }          | { "$rename": { "oldy": "newy" }}          | { "newy": "val" }
$pop        | { "arr": [1, 2, 3] }       | { "$pop": { "arr": 1 }}                   | { "arr": [1, 2] }
            | { "arr": [1, 2, 3] }       | { "$pop": { "arr": -1 }}                  | { "arr": [2, 3] }
$push       | { "letters": ["a"] }       | { "$push": { "letters": ["a"] }}          | { "letters": ["a", "a"] }
$pull       | { "arr": [2, 2, 1, 3, 1] } | { "$pull": { "arr": 1 }}                  | { "arr": [2, 2, 3] }

Upsert
------
Be aware that when you try to update a document that does not yet exist nothing
will be modified and your update will be lost:
{% highlight plain %}
> db.posts.update(
  { "title": "Not yet existing post" },
  { "$set": { "content": "Lost content..." } }
)
{% endhighlight %}

To fix this you can use the *upsert* option:
{% highlight plain %}
> db.posts.update(
  { "title": "Not yet existing post" },
  { "$set": { "content": "Now not lost content..." } },
  { "upsert": true }
)
{% endhighlight %}

This way, if query does not match any document MongoDB will insert a new
document using the values from the query and the update section. It will result
in a document such as:
{% highlight plain %}
{
  "title": "Not yet existing post",
  "content": "Now not lost content..."
}
{% endhighlight %}

Advanced query operators
------------------------
Since now we have used only the simplest equality operator, however there are more:

 * $gt - greater than
 * $lt - less than
 * $gte - greater than or equals
 * $lte - less than or equals
 * $ne - not equals

Here is a command for finding all posts with more than 3 code snippets:
{% highlight plain %}
> db.posts.find({ "code_snippets_count": { "$gt": 3 } })
{% endhighlight %}

Did you know that you can specify multiple query operators?
{% highlight plain %}
> db.posts.find({ "code_snippets_count": { "$gt": 5, "$lt": 10 } })
{% endhighlight %}

Suppose that each post can be rated by our readers, i.e.:
{% highlight plain %}
{
  "title": "Controversial post",
  "ratings": [ 1, 2, 5, 9, 10 ]
}
{% endhighlight %}

Lets find a post that was at least once well rated:
{% highlight plain %}
> db.posts.find({ "ratings": { "$elemMatch": { "$gte": 9, "$lte": 10 } } })
{% endhighlight %}

Projections
-----------
Sometimes for the performance sake you would like to filter out the fields you
do not want to see in your result. To do that you have 2 options:

 * you either specify the fields you want to exclude
 * ...or you provide the fields you only want to be included

Here is an example returning only titles of the blog posts:
{% highlight plain %}
> db.posts.find(
  {},
  { "title": true }
)
{% endhighlight %}

This query will return only *title* and *_id* fields:
{% highlight plain %}
{ "_id": ObjectId(...), "title": "First post" }
{ "_id": ObjectId(...), "title": "Second post" }
{ "_id": ObjectId(...), "title": "Third post" }
{% endhighlight %}

This is the only case when we can use both inclusion and exclusion to get rid of
*_id* field:
{% highlight plain %}
> db.posts.find( {}, { "title": true, "_id": false } )
{ "title": "First post" }
{ "title": "Second post" }
{ "title": "Third post" }
{% endhighlight %}

Count
-----
How to count the total number of blog posts? You can use the *count()* method:
{% highlight plain %}
> db.posts.find().count()
{% endhighlight %}

Sort
----
What about sorting the posts alphabetically by title? *sort()* method may come
handy:
{% highlight plain %}
> db.posts.find().sort({ "title": 1 })
{% endhighlight %}

To reverse the order to be descending you can replace *1* with *-1* as in:
{% highlight plain %}
> db.posts.find().sort({ "title": -1 })
{% endhighlight %}

Pagination
----------
We can page results using *limit()* and *skip()* methods. In order to fetch
the first 10 posts we can execute this command:
{% highlight plain %}
> db.posts.find().limit(10)
{% endhighlight %}

For the next page you should use *skip()*:
{% highlight plain %}
> db.posts.find().skip(10).limit(10)
{% endhighlight %}

Aggregation
-----------
In relational databases we have this nice concept of aggregation using *GROUP BY*
together with *HAVING* and a number of aggregation functions like *AVG*, *SUM*, etc.
It happens that MongoDB implements this concept using *aggregate()* method.

As an example lets fetch the average blog posts ratings per year to see, if our
blog is getting better:
{% highlight plain %}
> db.posts.aggregate([
  {
    "$group": {
      "_id": "$published_year",
      "avg_rating": { "$avg": "$rating" }
    }
  }
])
{% endhighlight %}

For this set of documents:
{% highlight plain %}
{ "title": "First", "published_year": 2009, "rating": 5 }
{ "title": "Second", "published_year": 2009, "rating": 3 }
{ "title": "Last", "published_year": 2010, "rating": 2 }
{% endhighlight %}

...we would receive:
{% highlight plain %}
{ "_id": 2009, "avg_rating": 4 }
{ "_id": 2010, "avg_rating": 2 }
{% endhighlight %}

Aggregation pipeline
--------------------
What is cool about MongoDB is that we can build an aggregation pipeline where
we can split our query into a sequence of stages:
{% highlight plain %}
> db.posts.aggregate([stage1, stage2, stage3, stage4, ...])
{% endhighlight %}

Each stage can be one of:

 * *$group* - as shown above for grouping documents
 * *$match* - for filtering documents (similar to *WHERE* and *HAVING* in *SQL*)
 * *$sort* - for sorting
 * *$limit* - for narrowing the number of documents
 * *$project* - for narrowing the number of fields in documents

As our final example lets assume we are publishing sponsored posts on our blog.
Let us create aggregation pipeline showing top 5 sponsors with the best-rated
blog posts about Java:
{% highlight plain %}
> db.posts.aggregate([
  { "$match": { "tags": "Java" }},
  { "$project": { "_id": false, "sponsor": true, "rating": true }},
  { "$group": { "_id": "$sponsor", "avg_rating": { "$avg": "$rating" }}},
  { "$sort": { "avg_rating": 1 }},
  { "$limit": 5 }
])
{% endhighlight %}


What is MongoDB good for?
-------------------------
According to *NoSQL Distilled* by *Pramod J. Sadalage* and *Martin Fowler*
there are some common use cases for MongoDB:

 * event logging - as a central database for events in the whole app (storing
   unstructured data and scalability are MongoDB's advantages)
 * CMS and blogging - a blog together with comments can be retrieved in one query
 * e-commerce - products in an online shop have various attributes which makes
   relational database not very suitable for this purpose (no schema acts in
   favor of MongoDB)
 * website monitoring - you can fetch queries in real-time

Where to go next?
--------------
If you want to get your hands dirty *Code School* has a very nice course
on MongoDB.

However, if reading is your preferred way of learning you also have multiple
good options. As an introductory material the previously mentioned
*NoSQL Distilled* is a great starting point. If you want to dig deeper
*MongoDB: The Definitive Guide* by *Kristina Chodorow* would be your best bet.

Final words
-----------
I hope you enjoyed this article and you now have a grasp of what MongoDB is and
how can it be used.
