---
title: Integration testing with databases
excerpt: In this miniseries, I want to cover several aspects of integration testing with the Spring framework, focusing on databases. This first part provides an overview of the tools available for test data management and database provisioning.
toc: true
---

# Overview

In this miniseries, I want to cover several aspects of integration testing with the Spring framework.

I'll begin with an overview of the tools provided by the framework and third-party integrations.

In the second part, I'll address some common pitfalls that hinder the reliability and maintainability of integration tests. 
For that, we'll have to look a bit under the hood to understand what exactly the Spring TestContext Framework does.

In the third part, I'll show a robust configuration using the tools introduced.

Integration tests are excellent for verifying stateful, lengthy business scenarios. 
When a real RDBMS is part of the test infrastructure, we can also test transactional logic. 
The downside is test data management, which can become cumbersome and error-prone as we introduce more and more tests.

By "test data management," I mean the process of creating and making the test data available, accessing the test data during tests, and finally cleaning up the test data. The process must obviously meet several requirements, such as:
* Easy creation and maintenance of test data.
* Reproducible tests.
* Tests that don't interfere with one another, even when running in parallel.
* Cross-cutting concerns, like transaction management, must not alter the outcome.

To begin, let's see the tools offered.

# Tools for Test Data Management

The technology selection of the production code influences the tools we can use for testing. 
Sometimes, incompatibility between these tools is clear; other times, a combination is just extremely inconvenient to use. 
Try to use a limited set of tools compatible with the level of data access abstraction in your production code.
Carefully evaluate the trade-offs of your selection. 
Arbitrarily introducing a tool that fits a momentary use case best will eventually complicate your test infrastructure.

## Generating Test Data

Beyond the simplest schemas, generating and maintaining test data manually is a tedious and error-prone task. 
In the age of AI tools, this should not be a problem anymore; unless we don't intend to share the details of our database schema with a third-party service. 
Beyond AI, there are plenty of online test data generators, some even open-source.

Honestly, I don't know much about large-scale test data generation, but I want to mention two tools that match the scale of integration tests.

The main reason I strongly suggest using either or both of these tools is maintainability. 
As our database schema evolves, we will need to keep our tests up to date as well. 
I am sure we've all suffered the consequences of adding _just another column_ to a table and consequently having to update dozens of test data setups manually, even for cases where the new column is not relevant. 
With the tools below, we can not only automate test data generation, but also maintain changes from a single place.

### Instancio
As it states itself, [Instancio][https://www.instancio.org] is primarily designed for unit tests where actual values are not that important. 
However, you may still find it useful to generate entity objects for populating test data in integration tests as well. 
It is suitable for generating large numbers of complex objects with nested relationships, when the actual values are not that important. 
You can still customize the generation process, and thus the values, via its fluent API. It can therefore be combined with other tools, like Datafaker, to generate realistic test entities.

### Datafaker
[Datafaker][https://www.datafaker.net] is better suited for generating realistic test data for databases. 
At its core are providers, which can generate fake data for various domains, such as names, addresses, companies, products, etc.
The feature that makes it most appealing for database test data generation is the ability to export generated data in SQL format, via its so-called transformers. 
It can also generate Java objects.

## Loading Test Data

Part of the test fixture is loading the test data into the database before test execution. 
We may need to load a chunk of data for a single test or a shared dataset for multiple tests.
If you seek a tool to initialize the database schema and seed data for the entire test suite, please check the _Initializing database_ section below.

### Plain SQL Scripts
The `spring-jdbc` module offers multiple interfaces to run SQL scripts as part of the setup and teardown phase, either programmatically or declaratively.

* [`ScriptUtils`][script-utils] is a low-level utility class to execute SQL scripts from a Resource (such as files). It is a very low-level tool; a `java.sql.Connection` instance must be provided to the database. 
  It is intended for internal use, but can be used directly if needed.
* [`ResourceDatabasePopulator`][resource-database-populator] is intended for populating, initializing, or cleaning up a database using SQL scripts defined in external resources.  
  A single instance can manage multiple scripts. It can accept a `javax.sql.DataSource` or a `java.sql.Connection` instance to execute the scripts. [`DatabasePopulatorUtils`][database-populator-utils] is a utility class to conveniently execute a `DatabasePopulator`.
* `@Sql` and `@SqlGroup` annotations can be used to [declaratively define SQL scripts][sql-annotation] to be executed before or after a test method or class.
  It requires not only the `spring-jdbc` but also the `spring-tx` module. 
  That said, script execution by default is part of the test execution transaction context. 
  This is a versatile and convenient tool for many reasons:
   * A single annotation can manage multiple scripts.
   * It can process script files from different locations, inline statements, and SpEL expressions.
   * The execution phase can be set (before/after test method/class).
   * Via the `@SqlConfig` annotation, it is possible to set the encoding, separator, comment prefix, error handling strategy, and most importantly, transaction mode.
   * If multiple `@Sql` annotations are defined, `@SqlMergeMode` can control how they are combined.

Developers using ORM solutions, such as JPA/Hibernate or Spring Data JPA, often prefer using entity objects to set up test data programmatically over plain SQL scripts. 
That may seem the obvious choice, and it does have some advantages, like:
* It keeps the relational database details hidden behind the entity model. No need to understand the database schema or SQL types.
* Writing SQL scripts is error-prone, especially when dealing with complex schemas, relationships, and constraints, unless you generate them automatically.
* The ORM will take care of populating synthetic keys, foreign keys, audit fields, etc.

Personally, I prefer using the declarative way to define test fixtures for three main reasons:
* The test data setup and teardown are clearly separated from the test logic.
* It provides detailed control over what, when, and how is executed against the database.
* It does not interfere with assumptions built into the entity model or the data access layer.

In the next article, I'll go deeper into the pros and cons of these approaches.

### Hibernate / JPA Support
When using an ORM solution, such as JPA/Hibernate or Spring Data, we can leverage the entity model to set up test data programmatically.

If you are using JPA from the `spring-boot-data-jpa` module, you can make use of [`TestEntityManager`][test-entity-manager], a tiny wrapper around `EntityManager` specifically designed for tests. 
Its `persistFlushFind` and `persistAndFlush` methods are very useful to set up test data in a way that respects the entity mappings and relationships. 
This is helpful when ensuring that entity data is actually written and read from the underlying database correctly.

As stated above, combining ORM with SQL scripts is a valid approach.

### Flyway Extension
If you are already using Flyway (see below) for schema migration, the `flyway-spring-test` [module][flyway-extension] is a great extension to manage test data as well.
By using the `@FlywayTest` annotation, you can not only declaratively execute SQL scripts as migrations, similar to the `@Sql` annotation, but also recreate the database schema before each test method or class, ensuring a clean state. 
However, this approach may be too heavy-weight for many use cases.

## Initializing the Database

Initialization can include two steps: creating the database schema and populating initial data.

Test data initialization can be split into two parts as well:
1. Creating the seed data that is part of the baseline and therefore shared across multiple tests. This dataset must be treated as read-only by the tests, so that they don't interfere with one another.
2. Executing test-specific data setup and teardown before and after each test method. These datasets can be mutable as they are isolated per test and cleaned up afterward. 
   If you execute tests in parallel on a single database instance, you must ensure that the test-specific datasets are isolated as well.

This approach helps reduce the amount of test data setup and teardown needed per test method, improving maintainability and performance.
Such baseline data can include reference data, lookup tables, common entities, user accounts, etc.

### Spring JDBC
We can also bootstrap the initial database state via SQL scripts executed automatically when the application context is created.

If you are using [Spring Boot with JDBC][spring-jdbc-init], you can place `schema.sql` and `data.sql` on your test classpath. 
This so-called _script-based approach_ lets you initialize the database from scripts. 
The scripts in those files will be executed automatically against the configured DataSource when the application context is created. 
They initialize the database schema and populate initial data, respectively.

Properties with the prefix `spring.sql.init` are available to customize the behavior of this feature, such as enabling/disabling it, changing the script locations, encoding, and selecting platforms.

### Hibernate and JPA
When using JPA/Hibernate, the [initialization feature][jpa-hibernate-init] of the underlying persistence provider is available as well. 
The property values `create` and `create-drop` of `spring.jpa.hibernate.ddl-auto` will create the database schema based on the entity mappings. 
On top of that, you can place a file named `import.sql` on the test classpath to populate initial data.

In rare cases, you may need to combine these approaches. In that case, the `spring.jpa.defer-datasource-initialization` property lets you define the execution order.

### Database Migration Tools
Lastly, migration tools such as Flyway and Liquibase can be used to [initialize test data][flyway-liquibase-init] as well. Be careful not to execute test data migrations against production databases!

Even if migrations run as a separate process in production (for example, to avoid lengthy application startup times), you can still benefit from embedded execution during tests. 
For that, just add them as test dependencies and point them to the migration scripts folder.

# Verification

For the sake of completeness, I mention `JdbcTestUtils`[jdbc-test-utils] here, which provides some utility methods to verify the state of the database, such as counting rows matching a criterion in a table or deleting from a table. 
To use it, you must pass a `JdbcClient` instance, which can be created from a `DataSource`. `JdbcTemplate` and `JdbcClient` beans are automatically created when using Spring Boot.

# Databases

We generally have two options for the database in integration tests: either use an in-memory database or a standalone RDBMS instance. 
Spring and Spring Boot provide support for the first option out of the box; the second option usually requires some additional setup.

## Embedded Databases

An embedded database is a lightweight, usually in-memory database that runs within the same process as the application. 
These qualities make them ideal for integration testing, as they are fast to start up and tear down, superfast, and do not require any external dependencies.

The downside is that they may not fully replicate the behavior of a production RDBMS, leading to discrepancies between test and production environments. 
This can be a significant problem when your application uses database-specific features, native queries, or SQL dialects. Some engines, like H2, provide compatibility modes for popular RDBMSs, but they are not bulletproof.

Spring supports three types of embedded databases out of the box: H2, HSQLDB, and Apache Derby. 
If you are using Spring Boot, the [presence of these databases][spring-boot-embedded-db] on the classpath will trigger autoconfiguration of a DataSource bean using the embedded database. 
That means you don't even need to provide the datasource URL. You can also configure whether a single database instance is shared across tests or a new instance is created for each test context via the `spring.datasource.embedded.unique-name` property.

If Spring Boot is not used, or a custom DataSource of an embedded database is needed, you can use `EmbeddedDatabaseBuilder` from the `spring-jdbc` module to [programmatically create an embedded database][spring-embedded-db]. 
You can further customize them via `EmbeddedDatabaseConfigurer` instances.

## Standalone Databases

When using a standalone RDBMS instance for integration tests, you must ensure that the database is available when the tests run. This can be achieved in several ways:
* A dedicated test database server is set up and running all the time.
* A test database server is started and stopped as part of the build process.
* A test database server is started and stopped as part of the test execution.

The first option is often preferred in integration environments where the database is reset to a known state before running the tests (e.g., by a CI/CD pipeline).
Dev environments may use pre-built DB images with seed data loaded.

The second approach can be used in CI/CD pipelines where a database server is started as part of the build process, tests are executed against it, and then the server is stopped. 
Build tools like Maven and Gradle provide several plugins to facilitate this process, eg: creation of containers, running database migration tools, etc.

The third option is often implemented using containerization technologies like Docker. 
Unsurprisingly, [Testcontainers][testcontainers] is a popular choice for this use case as well. It supports a [wide range of RDBMSs][testcontainers-db-modules] and provides a convenient API to manage the lifecycle of the containers.

## Zonky.io

It is absolutely imperative to mention the [Zonky.io][zonky] project when discussing databases for integration tests. 
Under the hood, it relies on Testcontainers (unless another provider is chosen), but it provides a higher-level abstraction specifically designed for database testing. 
What it brings to the table is automatic management of database instances, caching, and resetting. 
You no longer need to think about when and how to reset the database state, Zonky.io handles that automatically, following the declarative instructions you provide via the `@AutoConfigureEmbeddedDatabase` annotation. 
The biggest selling point is that it does not create a new database instance for each test context; instead, it caches the baseline state and copies it when needed. 
Its prefetching and caching mechanism significantly speed up test execution compared to starting a new container each time.

# Conclusion

In conclusion, effective integration testing with databases in Spring requires thoughtful selection and configuration of tools for test data management and database provisioning. 
By leveraging the right combination of SQL scripts, ORM utilities, and database management solutions like Flyway or Zonky.io, you can ensure your tests are reliable, maintainable, and closely aligned with production environments.

Prioritizing reproducibility, isolation, and maintainability in your test setup will help you catch issues early and build confidence in your application’s data access logic.

In the next post, I'll dive deeper into common pitfalls and best practices for integration testing with databases in Spring.

[script-utils]: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/init/ScriptUtils.html
[resource-database-populator]: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/init/ResourceDatabasePopulator.html
[database-populator-utils]: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/init/DatabasePopulatorUtils.html
[sql-annotation]: https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/executing-sql.html
[test-entity-manager]: https://docs.spring.io/spring-boot/3.4/api/java/org/springframework/boot/test/autoconfigure/orm/jpa/TestEntityManager.html
[flyway-extension]: https://github.com/flyway/flyway-test-extensions
[spring-jdbc-init]: https://docs.spring.io/spring-boot/how-to/data-initialization.html#howto.data-initialization.using-basic-sql-scripts
[jpa-hibernate-init]: https://docs.spring.io/spring-boot/how-to/data-initialization.html#howto.data-initialization.using-hibernate
[flyway-liquibase-init]: https://docs.spring.io/spring-boot/how-to/data-initialization.html#howto.data-initialization.migration-tool
[jdbc-test-utils]: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/jdbc/JdbcTestUtils.html
[spring-boot-embedded-db]: https://docs.spring.io/spring-boot/reference/data/sql.html#data.sql.datasource.embedded
[spring-embedded-db]: https://docs.spring.io/spring-framework/reference/data-access/jdbc/embedded-database-support.html
[testcontainers]https://testcontainers.com/guides/replace-h2-with-real-database-for-testing/
[testcontainers-db-modules]: https://java.testcontainers.org/modules/databases/
[zonky]: https://github.com/zonkyio/embedded-database-spring-test