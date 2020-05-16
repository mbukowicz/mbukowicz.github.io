---
layout: post
title: "Paging in Spring Data"
date: 2020-05-12
categories: java spring-data
thumbnail: /images/paging-in-spring-data/thumbnail.jpg
---

<div style="text-align: center; margin: 1em;">
  <img src="/images/paging-in-spring-data/book-1738609_1280.jpg"
  title="Pages in a book" class="rounded" />
</div>

Intro
-----

Presenting a larger set of data can be challenging. Spring Data can ease some
of the pains by providing various paging capabilities.

Basics
------

Suppose you would like to display such paging component:

<img src="/images/paging-in-spring-data/paging.png"
title="Simple paging component" style="clear: both;" />

First of all, you need to compute the total number of elements so that you can
estimate the number of pages.

Spring Data provides a special abstraction for that purpose:

{% highlight java %}
public interface Page<T> extends Slice<T> {
  // returns the number of all pages
  int getTotalPages();

  // returns the number of all elements
  long getTotalElements();

  // returns element on this page
  List<T> getContent();

  ...
}
{% endhighlight %}

The __Page__ interface is most often accompanied with __Pageable__ which is used
for selecting a specific page containing up to a certain number of elements:

{% highlight java %}
public interface Pageable {
  int getPageNumber();
  int getPageSize();
  ...
}
{% endhighlight %}

Both can be easily used within your @Repository interfaces:

{% highlight java %}
@Entity
public class Employee {

  @Id
  private Long id;
  private String name;
  private BigDecimal salary;
}

@Repository
public interface EmployeeRepository
  extends CrudRepository<Employee, Long> {

  Page<Employee> findAll(Pageable pageable);
}
{% endhighlight %}

Then getting a certain page can be accomplished like this:

{% highlight java %}
private EmployeeRepository repository;

...
int page = 1;
int pageSize = 3;
Pageable pageable = PageRequest.of(page, pageSize);
Page<Employee> secondPageWithUpTo3Elements =
  repository.findAll(pageable);
{% endhighlight %}

You do not need to explicitly add this method to your repository interface
if it already extends __JpaRepository__ or __PagingAndSortingRepository__:

{% highlight java %}
public interface JpaRepository<T, ID>
  extends PagingAndSortingRepository<T, ID> { ... }

public interface PagingAndSortingRepository<T, ID> {
  Page<T> findAll(Pageable pageable);
  ...
}
{% endhighlight %}


Dynamic queries with paging
---------------------------

Now imagine you need to not only split the results into pages but also add
filtering based on various criteria (e.g. salary range):

<img src="/images/paging-in-spring-data/filtering-and-paging.png"
title="Filtering and then paging" style="clear: both;" />

__Pageable__ stores only information about the page request so it cannot be
used for passing criteria, however, it can be used together with other APIs.


Paging with Specification
-------------------------

You could use <strong>Specification</strong>s based on the JPA criteria API
by extending the __JpaSpecificationExecutor__ interface:

{% highlight java %}
public interface JpaSpecificationExecutor<T> {
  Page<T> findAll(Specification<T> spec, Pageable pageable);
  ...
}

@Repository
public interface EmployeeRepository
  extends JpaSpecificationExecutor<Employee> { ... }
{% endhighlight %}

Here you can find a snippet showing how to pass a __Specification__:

{% highlight java %}
public static Specification<Employee> hasSalaryBetween(
    BigDecimal from, BigDecimal to) {
  return (Specification<Employee>) (employee, query, builder) ->
      builder.between(employee.get("salary"), from, to);
}

...

PageRequest pageRequest = PageRequest.of(0, 5);
Specification<Employee> spec = hasSalaryBetween(
  new BigDecimal("4000"), new BigDecimal("11000"));
Page<Employee> result = repository.findAll(spec, pageRequest);
{% endhighlight %}


Paging with Querydsl
--------------------

Alternatively, you could use Querydsl which can make your code more type-safe
and it also accepts __Pageable__ as a parameter:

{% highlight java %}
public interface QuerydslPredicateExecutor<T> {
  Page<T> findAll(Predicate predicate, Pageable pageable);
  ...
}
{% endhighlight %}

In order to achieve the same result as before you can write code like this:

{% highlight java %}
@Repository
public interface EmployeeRepository extends
  QuerydslPredicateExecutor<Employee> { ... }

...

QEmployee employee = QEmployee.employee;
Predicate predicate = employee.salary.between(
    new BigDecimal("4000"), new BigDecimal("11000"));
PageRequest pageRequest = PageRequest.of(0, 5);
Page<Employee> result = repository.findAll(predicate, pageRequest);
{% endhighlight %}


Avoiding paging
---------------

If your database grows large enough you could get into a situation where
__counting matching elements is rather expensive__ (especially if you
apply dynamic filtering on top of basic pagination). And you should be aware
that calculating the number of pages requires knowing the total number
of rows. In a nutshell, creating a Page object entails executing
two SQL queries.

For example, to retrieve the third page these queries will need to be run:

{% highlight sql %}
-- fetch the third page (pageSize=5)
SELECT *
FROM employee
WHERE ... -- some criteria
OFFSET 10 ROWS
FETCH NEXT 5 ROWS ONLY;

-- count all matching elements
-- (could become your bottleneck)
SELECT COUNT(*)
FROM employee
WHERE ...; -- the same criteria as above
{% endhighlight %}

First _SELECT_ should finish rather quickly since it expects a small
number of results. However, the second query, assuming you have a huge dataset,
could significantly slow down the retrieval of the current page. The problem is
that __it needs to count all the rows that match__ to get a precise total value.

Your users are unlikely to appreciate the value of knowing that there are
exactly 1771571534 elements to browse:

<img src="/images/paging-in-spring-data/too-many-pages.png"
title="Too many pages" style="clear: both;" />


Slice as an alternative
-----------------------

Fortunately, there is another useful abstraction called __Slice__ that can
be used in this scenario:

{% highlight java %}
public interface Slice<T> {
  int getNumberOfElements(); // on this Slice
  List<T> getContent();
  boolean hasNext();
  Pageable nextPageable(); // get next Slice
  ...
}
{% endhighlight %}

Actually, you may have noticed that __Page__ derives from __Slice__ but please
note that their implementations can vary quite a lot. Most importantly,
__Slice__ does not involve the expensive _COUNT_ operation we have just discussed.

Loading of the next __Slice__ can be invoked after clicking a _"Load more"_ button:

<img src="/images/paging-in-spring-data/loading-button.gif"
title="Loading button" style="clear: both;" />

...or when user reaches the end of the page (commonly called _infinite scroll_):

<img src="/images/paging-in-spring-data/infinite-scroll.gif"
title="Infinite scroll" style="clear: both;" />

To add a method returning a __Slice__ you can just write:

{% highlight java %}
@Repository
public interface EmployeeRepository extends ... {
  Slice<Employee> findAll(Pageable pageable);
}
{% endhighlight %}

In case you would like to keep the method returning <strong>Page</strong>s
here is how you can workaround the method name conflict:

{% highlight java %}
// you would like to preserve this
Page<Employee> findAll(Pageable pageable);

// ...so you need to give this method a different name
@Query("SELECT e FROM Employee e")
Slice<Employee> findAllSliced(Pageable pageable);
{% endhighlight %}

The code calling this method is almost the same as previously. The only difference
is that it returns a __Slice__ instead of a __Page__.

The end
-------

<div class="my-info">
For a complete set of examples please check <a href="https://github.com/mbukowicz/spring-data-queries">the following GitHub repository</a>.
</div>
