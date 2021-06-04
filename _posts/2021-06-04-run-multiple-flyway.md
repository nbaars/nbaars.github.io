---
layout: post
title: Run multiple separate Flyway migrations
categories: [Spring Boot, Flyway]
---

When running database migration scripts together with Spring Boot Flyway is the go-to solution. In this blog, I will show you how to run two Flyway migrations separately within the same application...

## Introduction

In our project [WebGoat](https://owasp.org/www-project-webgoat/) we store information about a user, track progress within the lessons, etc. On the other hand, we need to store information specifically related to a lesson. For example, the SQL injection lesson stores information in the database when the user does a couple of attempts and completely breaks the database we have to be able to reset the lesson to its initial state. On the other hand, we do not want to reset our user administration data within the application otherwise the user(s) need to create a new account when one of the users clicks the reset lesson button.

![WebGoat](https://nbaars.github.io/images/webgoat.png)

## Structure of WebGoat 

One of the more complicating factors and the reason we implemented the option we will describe below is that it is important to have the SQL statements needed for a specific lesson part of a particular lesson. For example, we have:

```
- webgoat-container
    - src/main/resoures/db
- webgoat-lessons
    - sql-injection
        - src/main/resoures/db
    - xxe
        - src/main/resoures/db
```

As you can see we want the database scripts as part of a lesson and not have it as part of the container which handles all the administration around the users etc. 

## Solution

For this, to work, we created two separate Flyway beans one for running the container-related scripts and one which will run all the migration scripts found in all the lessons.

```java
@Bean(initMethod = "migrate")
  public Flyway flywayContainer() {
      return Flyway
              .configure()
              .configuration(Map.of("driver", driverClassName))
              .dataSource(dataSource)
              .schemas("container")
              .locations("db/container")
              .load();
    }
    
@Bean(initMethod = "migrate")
@DependsOn("flyWayContainer")
public Flyway flywayLessons() {
      return Flyway
              .configure()
              .configuration(Map.of("driver", driverClassName))
              .dataSource(dataSource)
              .load();
    }
```

Both beans call the `migrate` method of the Flyway bean after the bean has been initialized. The first bean runs the scripts found in `/db/container` and the second one will run the scripts found in `/db/migration` which is the default location Flyway looks for scripts to run.
When the application starts two beans will be initialized:

```
2020-06-14 13:04:25.381  INFO 5698 --- [  restartedMain] o.f.c.internal.database.DatabaseFactory  : Database: jdbc:hsqldb:hsql://127.0.0.1:9001/webgoat (HSQL Database Engine 2.5)
2020-06-14 13:04:25.418  INFO 5698 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 10 migrations (execution time 00:00.024s)
2020-06-14 13:04:25.423  INFO 5698 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema "PUBLIC": 2019.11.10.1
2020-06-14 13:04:25.424  INFO 5698 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Schema "PUBLIC" is up to date. No migration necessary.
2020-06-14 13:04:25.532  INFO 5698 --- [  restartedMain] o.f.c.internal.database.DatabaseFactory  : Database: jdbc:hsqldb:hsql://127.0.0.1:9001/webgoat (HSQL Database Engine 2.5)
2020-06-14 13:04:25.537  INFO 5698 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 3 migrations (execution time 00:00.003s)
2020-06-14 13:04:25.541  INFO 5698 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema "container": 2
2020-06-14 13:04:25.541  INFO 5698 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Schema "container" is up to date. No migration necessary.
```

Flyway creates a separate table inside the database to keep track of which migration has already been applied. To make the solution work, it is necessary to define two schemas in the 
database one writes to the `PUBLIC` schema and one writes to a schema called `container`.


## Resetting a lesson
 
When a user wants to reset a lesson we do not want to reset the information stored about the user like username and password, how many attempts were made etc. The only thing we need to reset is data related to the lesson. 

```java
private final Flyway flywayLessons;

public void restartLesson() {
 Lesson al = webSession.getCurrentLesson();
 UserTracker userTracker = userTrackerRepository.findByUser(webSession.getUserName());
 userTracker.reset(al);
 userTrackerRepository.save(userTracker);

 flywayLessons.clean();
 flywayLessons.migrate();
}
```

We first wire the Flyway bean `flywayLesson` as defined above, in the method we first reset the progress of the user so whether the lesson was solved or not. In the last part, we call `clean` and `migrate` which will recreate tables again.