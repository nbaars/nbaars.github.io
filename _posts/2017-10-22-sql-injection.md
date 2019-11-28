---
layout: post
title: SQL injection -   when a prepared statement is not enough...
categories: [Security]
---

There are situations when a prepared statement is not enough to protect yourself against an SQL injection, in this 
blog we will explore a case where more protection is required...

## Introduction

An SQL injection attack consists of insertion or "injection" of a malicious data via the SQL query input from the client
to the application. In our example project we have a small Spring Boot based blog application. This
application exposes an endpoint to fetch blog articles based on the author:

```kotlin
@GetMapping("/author/{author}")
fun blogsByAuthorSqlInjection(@PathVariable author: String) = blogRepository.findByAuthor1(author)
```

Our (naive) implementation of the repository looks like:

```kotlin
@Repository
class BlogRepository(val entityManager: EntityManager) {

  fun findByAuthor1(name: String): Object = entityManager
    .createNativeQuery("select * from blogs b where b.author = '" + name + "'", BlogEntry::class.java)
    .resultList as Object
}
``` 

When we call the endpoint, we will receive:

```shell

curl -i http://localhost:8080/blogs/author/Anne%20Wilson

HTTP/1.1 200
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 02 Oct 2017 20:19:50 GMT

[ {
  "title" : "Spring Boot with Kotlin",
  "publishDate" : "2017-09-28",
  "author" : "Anne Wilson",
  "contents" : "Spring 5.0 ..."
}, {
  "title" : "New Spring Boot version is available",
  "publishDate" : "2017-10-02",
  "author" : "Anne Wilson",
  "contents" : "A new version ..."
} ]
```

If the attacker however supplies the following:

```shell
curl -i "http://localhost:8080/blogs/author/Smith'%20or%20'1'='1"
HTTP/1.1 200
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 02 Oct 2017 20:21:23 GMT

[ {
  "title" : "First blog",
  "publishDate" : "2017-09-27",
  "author" : "Peter Jackson",
  "contents" : "This is ..."
}, {
  "title" : "Spring Boot with Kotlin",
  "publishDate" : "2017-09-28",
  "author" : "Anne Wilson",
  "contents" : "Spring 5.0 ..."
}, {
  "title" : "A new Spring Boot version is available",
  "publishDate" : "2017-10-02",
  "author" : "Anne Wilson",
  "contents" : "Version 2 ..."
}, {
  "title" : "Spring 5 is on its way",
  "publishDate" : "2017-10-02",
  "author" : "Eric William",
  "contents" : "Spring 5 ..."
} ]
```

So due to the fact that the application is not escaping the quotes an attacker is able to modify the query and list all
the blogs.

When you read about how to prevent SQL injections the most common advise is: "Use a prepared statement or
parametrized queries." If we change our repository method accordingly we will get:

```kotlin
fun findByAuthor(name: String): List<BlogEntry> =
  return entityManager.createQuery("select b from BlogEntry b where b.author = :name")
           .setParameter("name", name)
           .resultList as List<BlogEntry>
```

in which case the call made by the attacker will result in an empty JSON message because there is no author with the name `Smith'%20or%20'1'='1`
The attacker can no longer change the meaning of the query. ORM implementations like JPA/Hibernate will help you automatically to prevent SQL injections but as we will see
in the next paragraph it is not enough.

## Not enough

Although using prepared statements is a great step in the defense of preventing SQL injections there are cases
where this is not enough, suppose we have the following query: "`select * from blogentry order by author`" The `order by`
clause will normally be a column name, however if we look at the SQL grammar definition it can be:

```
SELECT ...
FROM tableList
[WHERE Expression]
[ORDER BY orderExpression [, ...]]

orderExpression:
{ columnNr | columnAlias | selectExpression }
    [ASC | DESC]

selectExpression:
{ Expression | COUNT(*) | {
    COUNT | MIN | MAX | SUM | AVG | SOME | EVERY |
    VAR_POP | VAR_SAMP | STDDEV_POP | STDDEV_SAMP
} ([ALL | DISTINCT][2]] Expression) } [[AS] label]
```

Looking at this we can see we can use functions inside the `order by` clause. Let's change our example and add the
following code to our controller and repository:

```kotlin
@GetMapping("/author/{author}")
fun blogsByAuthor(@PathVariable author: String, @RequestParam sortBy: String?) =
  blogRepository.findByAuthorContaining(author, sortBy)


fun findByAuthorContaining(name: String, orderBy: String): List<BlogEntry> =
  return entityManager.createQuery("select b from BlogEntry b where b.author :name order by " + orderBy)
            .setParameter("name", name)
            .resultList as List<BlogEntry>
```

As you can see the `sortBy` parameter is passed in through the REST endpoint. The prepared statement can only
deal with query parameters (single value) and cannot be used with a column names, table names, expressions etc.
This means the `order by` is just appended to the given query string.


One way to test whether this query is vulnerable for SQL injection is:

```
(select * from BlogEntry where b.author = :name order by case when true=true then title else contents)

curl -i "http://localhost:8080/blogs/author/Anne%20Wilson?sortBy=case%20when%20true=true%20then%20title%20else%20contents%20end"
HTTP/1.1 200
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 02 Oct 2017 21:16:16 GMT

[ {
  "title" : "Spring Boot with Kotlin",
  "publishDate" : "2017-09-28",
  "author" : "Anne Wilson",
  "contents" : "Spring 5.0 ..."
}, {
  "title" : "A new Spring Boot version is available",
  "publishDate" : "2017-10-02",
  "author" : "Anne Wilson",
  "contents" : "Version 2 ..."
} ]
```

If we flip the `when` statement the result will be sorted based on the contents instead of the title.
This means we can substitute random expressions for the `sortBy` query parameter, which means
we can ask the database questions like:

```
select * from BlogEntry order by case when substring(h2version(),1,1)='1' then title else contents end
select * from BlogEntry order by case when substring(h2version(),2,1)='.' then title else contents end
select * from BlogEntry order by case when substring(h2version(),3,1)='4' then title else contents end
...
```

which will give you the database version(1.4.9) of H2 used in our example project. The `order by` clause is
just an example it also applies `group by` clauses etc. You can also start asking system tables like `INFORMATION_SCHEMA`
for other interesting tables in the database, sqlmap(http://sqlmap.org/) is a tool which can help you automate the
extraction process.

## Spring Data JPA

Our small example above was also present in an earlier version of Spring Data where you were able to
specify a sort expression, like:

```kotlin
@Query("select p from Person p where LOWER(p.lastname) = LOWER(:lastname)")
List<Person> findByLastname(@Param("lastname") String lastname, Sort sort)
```

In the old version of Spring Data you could call this method as follows: `findByLastname("Johnson", new Sort("LENGTH(firstname)"))`
which means you can repeat the same SQL injection as we described above. In the
newer version of Spring Data an exception is thrown whenever you use a function, you still
can use a function but you have to use: `JpaSort.unsafe("LENGTH(lastname)"` which clearly indicates a potential
dangerous operation when the input can be adjusted by an attacker.
More details can be found here: https://pivotal.io/security/cve-2016-6652

## Mitigation

If you need to provide a sorting column in your web application you should implement a whitelist to validate the value of
the `order by` statement it should always be limited to something like 'firstname' or 'lastname' or 1,2 etc.
Remember it is a combination of both, you should use both *always* use a prepared statement when dealing with
SQL but *also* use input validation on parts of the query which are not seen as a dynamic query parameter when using a prepared statement.
