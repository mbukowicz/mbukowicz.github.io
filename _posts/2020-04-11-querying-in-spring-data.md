---
layout: post
title:  "8 ways to write queries in Spring Data"
date:   2020-04-13 18:59:19
categories: java spring-data
thumbnail: /images/8-ways-to-write-queries-in-spring-data/thumbnail.png
---

<div style="text-align: center; margin: 1em;">
  <img src="/images/8-ways-to-write-queries-in-spring-data/database-149760_960_720.png"
  title="Spring logo on a database" />
</div>

Intro
-----

Spring Data is a great way to simplify your code accessing the database. It also
__gives a lot of flexibility__ in terms of available abstractions. Programmers can
choose from __high-level constructs__ requiring minimum amount of code __or__ dive deep into
__nitty-gritty implementation details__, while accepting a more verbose solution.

Kickstarting a project with Spring Data is as simple as adding this dependency:

{% highlight xml %}
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
{% endhighlight %}

Assuming we have such entity:

{% highlight java %}
@Entity
public class Employee {

  @Id
  @GeneratedValue
  private Long id;
  private String name;
  private BigDecimal salary;

  public Employee(Long id, String name, BigDecimal salary) {
    this.id = id;
    this.name = name;
    this.salary = salary;
  }

  // getters and setters
}
{% endhighlight %}

...stored in the following table:

{% highlight sql %}
CREATE TABLE employee (
  id BIGINT NOT NULL PRIMARY KEY,
  name VARCHAR(255),
  salary DECIMAL(19,2)
);
{% endhighlight %}

Let's add the following test data:

{% highlight sql %}
INSERT INTO employee (id, name, salary) VALUES (1, 'Bob', 5000.00);
INSERT INTO employee (id, name, salary) VALUES (2, 'Trent', 7500.00);
INSERT INTO employee (id, name, salary) VALUES (3, 'Alice', 10000.00);
{% endhighlight %}

1) Filtering based on method names
-------------------------------

By carefully [crafting a repository method name](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods) we can avoid writing even a single
line of SQL code.

For example this method signature:

{% highlight java %}
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
  List<Employee> findByNameContainingIgnoreCaseAndSalaryGreaterThanOrderBySalaryDesc(
      String name, BigDecimal salary);
}
{% endhighlight %}

...will cause Hibernate to spit out something along the lines of:

{% highlight sql %}
select employee0_.id as id1_0_,
       employee0_.name as name2_0_,
       employee0_.salary as salary3_0_
from employee employee0_
where (upper(employee0_.name) like upper(?) escape ?)
and employee0_.salary > ?
order by employee0_.salary desc
{% endhighlight %}

We can easily select employees whose name contains a given string of characters
and whose salary is above a threshold. No SQL, no JPQL, no HQL, just Java code.

Now, with the test data inserted earlier, when we call this:

{% highlight java %}
repository.findByNameContainingIgnoreCaseAndSalaryGreaterThanOrderBySalaryDesc(
    "LIC", new BigDecimal("8000"));
{% endhighlight %}

...we should get a one-element list containing only _Alice_:

{% highlight plain %}
[Employee{id=3,name='Alice',salary=10000.00}]
{% endhighlight %}

2) Using @Query annotation
-----------------------

The whole process of experimentation and trying to come up with the right method
name is fun and exciting, but sometimes the end result is not that satisfying.

_findByNameContainingIgnoreCaseAndSalaryGreaterThanOrderBySalaryDesc_ is quite
long and looks clunky. Try to say it in one breath. It feels like we have traded
readability for a quicker result.

It may be worth to replace this long method name with a shorter equivalent,
where we can decide how to call the method, but the implementation in JPQL has to be
provided in return (there is no free lunch I am afraid):

{% highlight java %}
@Query(
  "SELECT e FROM Employee e " +
  "WHERE lower(e.name) LIKE lower(concat('%', :name, '%')) " +
  "AND e.salary > :salary " +
  "ORDER BY e.salary DESC"
)
List<Employee> findByNameLikeAndSalaryAbove(
    @Param("name") String name, @Param("salary") BigDecimal salary);
{% endhighlight %}

Now the method name looks friendlier:

{% highlight java %}
repository.findByNameLikeAndSalaryAbove("LIC", new BigDecimal("8000"));
{% endhighlight %}

3) Native @Query
-------------

We have sorted out the problem of having too long method names, but in the process
we have introduced rather unpleasant implementation of case-insensitive LIKE
condition:

{% highlight sql %}
WHERE lower(e.name) LIKE lower(concat('%', :name, '%'))
{% endhighlight %}

In some databases there is a nicer ILIKE keyword that can be used to make
case-insensitive matching with less effort.

The code could then be refactored to this (please note _nativeQuery_ attribute set to true):

{% highlight java %}
@Query(
  value = "SELECT * FROM employee e " +
          "WHERE e.name ILIKE %:name% " +
          "AND e.salary > :salary " +
          "ORDER BY e.salary DESC",
  nativeQuery = true
)
List<Employee> findByNameLikeAndSalaryAbove(
    @Param("name") String name, @Param("salary") BigDecimal salary);
{% endhighlight %}

The caveat here is that we expose ourselves to a possible vendor lock-in so
usually it is better to stay on the safe side and __use _nativeQuery_ sparingly__.

Here _nativeQuery_ just makes the code nicer to look at and it might be up for
debate whether it is the best solution in this scenario. There are, however,
situations when you absolutely need it. It is up to you to make the call
when it should be used and when not. The decision is not always easy, but
programming is the art of trade-offs after all.

4) Query by example
-------------------

Another interesting alternative is querying by providing an example record:

{% highlight java %}
Employee employee = new Employee(null, "LIC", null);
ExampleMatcher matcher = ExampleMatcher.matching()
    .withMatcher("name", contains().ignoreCase());
List<Employee> result = repository.findAll(Example.of(employee, matcher));
// result=[Employee{id=3,name='Alice',salary=10000.00}]
{% endhighlight %}

Unfortunately, this is mostly limited to strings as
[stated in the docs](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#query-by-example.usage) which boils down to number expressions (e.g. greater/lower than) not being supported:

<img src="/images/8-ways-to-write-queries-in-spring-data/query-by-example-limitations.png"
title="Query by example limitations" style="clear: both; margin: 1em;" />


5) Specifications
-----------------

A more advanced option are [specifications](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#specifications), which were built on top of criteria API in JPA.

Here is how the previous conditions can be implemented as specifications:

{% highlight java %}
public class EmployeeSpecs {

  public static Specification<Employee> hasNameLike(String name) {
    return (Specification<Employee>) (employee, query, builder) ->
        builder.like(
          builder.lower(employee.get("name")),
          "%" + name.toLowerCase() + "%");
  }

  public static Specification<Employee> hasSalaryAbove(BigDecimal salary) {
    return (Specification<Employee>) (employee, query, builder) ->
        builder.greaterThan(employee.get("salary"), salary);
  }
}
{% endhighlight %}

After adding _JpaSpecificationExecutor_ to the list of interfaces
implemented by the repository:

{% highlight java %}
@Repository
public interface EmployeeRepository extends JpaSpecificationExecutor<Employee> {
  ...
}
{% endhighlight %}

...you can select elements based on specifications by calling an overloaded _findAll_
method:

{% highlight java %}
public interface JpaSpecificationExecutor<T> {

  /**
   * Returns all entities matching the given {@link Specification}.
   * @param spec can be {@literal null}.
   * @return never {@literal null}.
   */
   List<T> findAll(@Nullable Specification<T> spec);
}
{% endhighlight %}

The best part is that specifications can be mixed together allowing for
better customizability and reusability:

{% highlight java %}
repository.findAll(
        hasNameLike("LIC").and(hasSalaryAbove(new BigDecimal("8000"))));
{% endhighlight %}

6) Querydsl
-----------

One downside of specifications is that they are not strictly typed, meaning
we can run into issues when misstyping a property name or if we
assume incorrect property type.

To enforce a stricter name and type checking [Querydsl](http://www.querydsl.com)
can be used:

{% highlight xml %}
<dependencies>
  <dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <version>4.1.3</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <version>4.1.3</version>
  </dependency>
</dependencies>
{% endhighlight %}

It works by generating special classes prepended with _"Q"_ and
based on the entity classes. In our case Querydsl will generate _QEmployee_
class from the corresponding _Employee_ entity.

Below you can find Maven configuration for generating _Q*_ classes during
compilation:

{% highlight xml %}
<build>
  <plugins>
    <plugin>
      <groupId>com.mysema.maven</groupId>
      <artifactId>apt-maven-plugin</artifactId>
      <version>1.1.3</version>
      <executions>
        <execution>
          <goals>
            <goal>process</goal>
          </goals>
          <configuration>
            <outputDirectory>target/generated-sources</outputDirectory>
            <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
{% endhighlight %}

After setting it up you can write code like this:

{% highlight java %}
QEmployee employee = QEmployee.employee;
repository.findAll(
  employee.name.likeIgnoreCase("%LIC%")
    .and(employee.salary.gt(new BigDecimal("8000"))));
{% endhighlight %}

What's great about Querydsl is we get autocompletion:

<img src="/images/8-ways-to-write-queries-in-spring-data/querydsl-autocompletiong.gif"
title="Query by example limitations" style="clear: both;" />

Custom repositories
-------------------

It is very likely, that at one point you will reach a situation where all the
above solutions do not work. For such scenarios Spring Data provides a way
to strip all the layers of abstractions and get down to writing a more specific
implementation.

The assumption is that you will rarely fallback to writing low-level custom code
and most of the time you will stick to Spring Data repositories cherishing
the features described so far.

In order to extend an existing Spring Data repository you need to create
a dedicated repository interface:

{% highlight java %}
public interface EmployeeRepositoryCustom {
  List<Employee> findByNameLikeAndSalaryCustom(String name, BigDecimal salary);
}
{% endhighlight %}

...that is then extended by your original repository:

{% highlight java %}
public interface EmployeeRepository extends EmployeeRepositoryCustom {
  ...  
}
{% endhighlight %}

Finally, you should put your code inside an implementation of
the custom interface:

{% highlight java %}
public class EmployeeRepositoryCustomImpl implements EmployeeRepositoryCustom {

  @Override
  public List<Employee> findByNameLikeAndSalaryCustom(
    String name, BigDecimal salary) {
     // <custom code goes here>
  }
}
{% endhighlight %}

Spring Data will do all the plumbing for you so that when _findByNameLikeAndSalaryCustom_
is called on the interface the call is then correctly propagated to
the implementation inside _EmployeeRepositoryCustomImpl_:

{% highlight java %}
@Autowired
private EmployeeRepository repository;
...
repository.findByNameLikeAndSalaryCustom("LIC", new BigDecimal("8000"));
{% endhighlight %}

7) Custom repository with Criteria API
--------------------------------------

Going back to our initial problem of having to overcome difficulties with
previous solutions here is how JPA criteria API can play together with custom
repositories to implement dynamic queries:

{% highlight java %}
public class EmployeeRepositoryCustomImpl implements EmployeeRepositoryCustom {

  @PersistenceContext
  private EntityManager entityManager;

  @Override
  public List<Employee> findByNameLikeAndSalaryCustom(String name, BigDecimal salary) {
    CriteriaBuilder builder = entityManager.getCriteriaBuilder();
    CriteriaQuery<Employee> query = builder.createQuery(Employee.class);
    Root<Employee> employee = query.from(Employee.class);
    Predicate criteria = builder.conjunction();
    if (name != null) {
      Predicate nameLike = builder.like(
          builder.lower(employee.get("name")),
          "%" + name.toLowerCase() + "%");
      criteria = builder.and(criteria, nameLike);
    }
    if (salary != null) {
      Predicate salaryAbove = builder.greaterThan(employee.get("salary"), salary);
      criteria = builder.and(criteria, salaryAbove);
    }
    query.select(employee).where(criteria);
    return entityManager.createQuery(query).getResultList();
  }
}
{% endhighlight %}

What immediately stands out is the verbosity of this solution, but it works
quite well when you need to implement a search capability where criteria changes
frequently.

8) Old school approach using StringBuilder
------------------------------------------

You should be able to solve 99% of problems using one of the previous
7 different alternatives.

However, so far most of the solutions relied on JPA (and JPQL) which,
like any other abstraction, has its own shortcomings. You may want to
have in your toolkit something highly-customizable that can circumvent JPA.

If you mix the ol' school approach of building the SQL query yourself together
with custom repositories you might end up with something like this:

{% highlight java %}
public class EmployeeRepositoryCustomImpl implements EmployeeRepositoryCustom {

  @PersistenceContext
  private EntityManager entityManager;

  @Override
  public List<Employee> findByNameLikeAndSalaryCustom(String name, BigDecimal salary) {
    Map<String, Object> params = new HashMap<>();
    StringBuilder sql = new StringBuilder();

    sql.append("SELECT * FROM employee e WHERE 1=1 ");
    if (name != null) {
      sql.append("AND e.name ILIKE :name ");
      params.put("name", "%" + name + "%");
    }
    if (salary != null) {
      sql.append("AND e.salary > :salary ");
      params.put("salary", salary);
    }
    sql.append("ORDER BY e.salary DESC");

    Query query = entityManager.createNativeQuery(sql.toString(), Employee.class);
    for (Entry<String, Object> param : params.entrySet()) {
      query.setParameter(param.getKey(), param.getValue());
    }
    return query.getResultList();
  }
}
{% endhighlight %}

Such code has many disadvantages: it is incredibly __prone to typos__ and it is also
__quite verbose__. But this is the price to be paid for having the __flexibility__ of
accessing the database in a very direct way.

Still a good rule of thumb is to __use it as a last resort__. If it is unavoidable then
you have to do what you have to do, but considering the alternatives is advisable.

<div class="my-info">
For a complete set of examples please check <a href="https://github.com/mbukowicz/spring-data-queries">the following GitHub repository</a>.
</div>
