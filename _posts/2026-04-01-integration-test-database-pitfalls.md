---
title: "Integration testing with databases - part II: Pitfalls"
excerpt: In the second part of this series on integration testing with databases, we explore common pitfalls and how to avoid them.
tags: [spring, testing, integration testing, databases, transactions, TestContext Framework]
---

# Overview
This is the second part of my miniseries about Spring integration testing with databases. 
In the [first part][first-part], we discussed the tools available in Spring and the wider Java ecosystem to set up integration tests that interact with databases.
In this part, we will explore some common pitfalls you may encounter when writing integration tests that interact with databases, and how to avoid them.
Fortunately, there aren't many, but those that do exist can lead to subtle false positives or negatives in your test results.

# Transaction Management
In one of my earlier posts, I addressed the importance of proactively considering cross-cutting concerns in our integration tests.
As I mentioned there, there are no clear dos and don'ts for handling cross-cutting concerns in integration tests.
However, when it comes to transaction management, I strongly urge everyone to incorporate it.
Mainly for two reasons:
* Unlike other cross-cutting concerns, such as security, transaction management does not require convoluted setup logic. With Spring Boot, it is almost automatic.
* Complex business logic often involves multiple database operations across different execution branches. We need to carefully design commit and rollback points to ensure data integrity around edge cases. Testing without transaction management will eventually defer nasty bugs to production.

Beforehand, I need to point out that everything I am about to explain can be found in the official Spring documentation, but it is scattered across multiple sections and not very easy to find.
I learned it the hard way, through iterations of trial, error, and querying the documentation.
My intention is to consolidate this knowledge in one place for easier consumption.

So the question is: _How exactly does transaction management work during integration tests?_

The naive answer is: "It works the same way as in production code."
While this is true to some extent, there are important differences to consider.

## Spring TestContext Framework
The [TestContext Framework][spring-test-cxt] integrates seamlessly with the test framework of your choice (JUnit, TestNG, etc.) and provides support for loading application contexts, managing transactions, and more.
It hooks into the test lifecycle steps to provide additional functionality; these hooks are called Test Execution Listeners.

The listener we are interested in is the `TransactionalTestExecutionListener`.
This listener is automatically registered by default in the TestContext Framework.
It ensures that each test method or test annotated with `@Transactional` is executed within a transaction that is rolled back after the test method completes.
Some of the test slice annotations, like `@DataJpaTest`, already include `@Transactional` by default.

Most of the time, this is exactly what we want, but there are some edge cases to consider.

# Pitfalls
## Nested Transactions
If our unit of work uses a single transaction, everything works as expected.
However, if our unit of work involves nested transactions (using `REQUIRES_NEW` propagation, for example), those inner transactions will be committed independently of the outer transaction.
These changes will persist in the database even after the test transaction is rolled back, potentially leading to data contamination between tests.

`REQUIRES_NEW` is not the only propagation type that can lead to anomalies in tests, but it is the most common one. Some lesser-used propagation types that can also cause issues are:

* `NEVER`: will always fail in tests annotated with `@Transactional` since it forbids running within a transaction.
* `MANDATORY`: may lead to false negatives on execution branches where the production code does not create a transaction beforehand, but the test does.

For example, consider the following simple service methods:
```java
    @Transactional
    void create(User user) {
        repository.save(user);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    User createWithCommit(User user) {
        return repository.save(user);
    }
```
The test case:
```java
@DataJdbcTest // for JDBC
// @DataJpaTest for JPA
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class PropagationTest {
    // ...
    @Test
    @Order(1)
    void createUsers() {
        User user = UserUtil.createUserWithUsername("user1");
        // propagation REQUIRES_NEW - this one will be committed immediately
        userService.createWithCommit(user);

        user = UserUtil.createUserWithUsername("user2");
        // this one joins the test transaction - will be rolled back
        userService.create(user);
    }

    @Test
    @Order(2)
    void verifyCommit() {
        // This one wasn't rolled back
        User user = userService.fetchUserByUserName("user1");
        assertThat(user).isNotNull();
        // This one was rolled back
        user = userService.fetchUserByUserName("user2");
        assertThat(user).isNull();
    }
}
```

You can find examples of other propagation discrepancies in the same test class (`PropagationTest`) in the [JDBC examples][jdbc-propagation-examples] and [JPA examples][jpa-propagation-examples].

## Combining Multiple Transactions
In rare cases, we need to split a single unit of work into multiple transactions.
For example, the logic is split into multiple execution steps, and at the end of each step, we want to commit some changes to the database.
In such cases, the production code will have multiple sibling transactions, so it is assumed that even if the n+1th transaction is rolled back, the changes from the nth transaction will still be present in the database.
However, if we annotate the test with `@Transactional`, all transactions will be combined into a single unit and committed or rolled back at once at the end of the test method, leading to false positives in our tests.

In the example below, using JPA, we want to create a user in draft state (0) and then, after some processing, activate it (1). If that activation fails, we still want the user in draft state to be present in the database.
```java
// Main service 
public void longRunningOperation(User draft, boolean shouldFail) {
    // Create user draft and commit immediately
    UUID userId = firstService.saveDraftUser(draft);

    // Do some other processing here that may fail
    try {
        secondService.failingLongRunningOperation(userId, shouldFail);
    } catch (Exception e) {
        // let test continue
    }
}
// First service
@Transactional(propagation = Propagation.REQUIRES_NEW) 
UUID saveDraftUser(User draft) {
    draft.setUserStatus(0);
    User user = repository.save(draft);
    return user.getId();
}
// Second service
@Transactional
void failingLongRunningOperation(UUID userId, boolean shouldFail) {
    // Some processing done here, may or may not commit
    User user = repository.findById(userId).orElseThrow();
    user.setUserStatus(100);

    if (shouldFail) {
        // But then fail
        throw new RuntimeException("Simulated failure during long running operation");
    }
    repository.save(user);
}
```
The test case:
```java
@DataJpaTest
class LongRunningOperationTest {
    // ...
    @Test
    void falseResult() {
        User user = UserUtil.createUser(FALSE_POSITIVE);
        userService.longRunningOperation(user, true);

        // seems to be committed with success - should be committed as draft
        user = userService.fetchUserByUserName(FALSE_POSITIVE);
        assertThat(user).isNotNull();
        assertThat(user.getUserStatus()).isEqualTo(100);
    }
}
```
If you argue that with better transaction design we could avoid such pitfalls, you are right. 
This is a fairly simple example, but one of the nastiest bugs I encountered in production stemmed from the very same misunderstanding of transaction management in tests.
There, the multistage transaction was split across thousands of lines of code and services.

You can find a complete example of this pitfall in the [JPA examples][jpa-multi-tx-examples].

## Verifying Rollback Behavior
In some cases, we want to verify that rollback behavior works as expected—for example, when testing error handling logic and ensuring that the database state is not corrupted after a specific exception is thrown.
When the exception passes a logical transaction boundary (e.g., a method annotated with `@Transactional`), the transaction is marked for rollback.
However, a query verifying the database state after the exception is thrown will be executed within the same transaction and will see the changes that are supposed to be rolled back, leading to false positives in our tests.

You can find a complete example of this pitfall in the [JDBC examples][jdbc-rollback-examples] and [JPA examples][jpa-rollback-examples].

## Hibernate Cache and Flush Behavior
In my [previous post][first-part], I already mentioned the usefulness of the `TestEntityManager` in JPA tests, and how flushing the persistence context can help reveal issues that may arise in production.
This is not only important when creating test data, but also when exercising the unit of work under test.
Spring documentation points this out as well at the very end of [this article][spring-tx-orm].

In production code, we want to interfere as little as possible with the ORM's own cache and flush behavior.
So we tend to avoid calling `flush()` explicitly, and we should not quit that habit for the sake of tests either.
Instead, inject the `TestEntityManager` in your tests and call `flush()` or `clear()` after exercising the unit of work under test, to make sure that all pending changes are synchronized with the database and any potential issues are revealed.

See [these examples][test-flush-examples].

# Working with Transactional Tests
When designing integration tests that interact with a database, the most important thing to consider is: 

_Does the transactional aspect of the unit of work under test matter for the test case?_

If not, then it is better to rely on test-managed transactions, since it will save us from the hassle of managing the database state manually.
Again, the test will run in a test-managed transaction if it is annotated with `@Transactional` either directly or indirectly (e.g., via `@DataJpaTest`).
When the transactional aspects are crucial to the unit of work under test, it is better to opt out of test-managed transactions and manage the database state manually. 

It may be tempting to always opt out of test-managed transactions, thinking that it will avoid all the pitfalls mentioned above.
This may be true; however, it comes at the cost of managing cleanup logic manually. Two common mistakes here are:
* Relying too much on drop-recreate mechanisms between tests. In the long run, this will lead to slow tests and may hide issues related to data contamination between tests.
* Trying to design non-overlapping test data between tests. This is a losing battle; sooner or later, the test dataset will be so large and complex that maintaining it will be a nightmare.

As a middle ground, the Spring TestContext Framework provides a set of tools to [control the transaction behavior in tests][spring-test-annotations]. 
These align with the test execution lifecycle methods. Let's briefly overview them:

* `@Rollback`, `@Commit`: These annotations can be used to override the default rollback behavior of test-managed transactions.
  Use `@Rollback(false)` to commit the transaction after the test method completes, and `@Commit` to explicitly commit the transaction.
  Note that the action applies to the entire test method, including any setup or teardown methods.
* `@BeforeTransaction`, `@AfterTransaction`: These annotations can be used to define methods that run before and after the test-managed transaction.
  They run before and after each transactional test method, respectively.
  `@BeforeTransaction` is useful for setting up and verifying preconditions. 
  `@AfterTransaction` can be used for cleanup or verification.
* `@SqlConfig`: In the previous post, I mentioned the `@Sql` annotation for executing SQL scripts before or after tests.
  The `@SqlConfig` annotation's `transactionMode` attribute allows us to specify whether the SQL scripts should run within the test-managed transaction or outside of it.
* `@Transactional`: Just like in production code, we can use this annotation to define transaction boundaries at the test method or class level.
  We can enable test-managed transactions by simply annotating the test class or method with `@Transactional`. 
  Conversely, we can disable test-managed transactions by annotating the test class or method with `@Transactional(propagation = Propagation.NOT_SUPPORTED)`.

The documentation also provides information about the transactional behavior of test-level lifecycle methods:

* `@BeforeAll`, `@AfterAll` (or respective lifecycle methods in other test frameworks): These methods run outside any test-managed transaction.
  Any database operations performed here will persist across tests.
  Use these methods for global setup and teardown tasks that should not be rolled back after each test.
* `@BeforeEach`, `@AfterEach` (or respective lifecycle methods in other test frameworks): These methods run within the test-managed transaction.
  Any database operations performed here will be rolled back after each test.
  Use these methods for setup and teardown tasks that should be isolated to each test.

One last thing to mention for the sake of completeness: the test-managed transaction obviously can't roll back changes to external systems.
If the test context is set up with `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)` or similar, the application—and thus the unit of work under test—runs in a separate process.
In such cases, the test-managed transaction can't roll back changes made by the application.

# Conclusion
In summary, understanding the nuances of transaction management and the potential pitfalls in integration testing with databases is essential for building reliable and maintainable tests. 
By being aware of how Spring’s TestContext Framework handles transactions, recognizing the impact of different propagation behaviors, and carefully choosing when to rely on test-managed transactions versus manual state management, you can avoid subtle bugs and false test results. 
Thoughtful test design, combined with the right use of Spring’s testing annotations and utilities, will help ensure your integration tests accurately reflect real-world scenarios and provide lasting value as your application evolves.

In my next and final post of this series, I will present a fully working, opinionated setup for integration testing with databases in Spring Boot applications.

You can find the complete code in my GitHub repository for [JDBC][jdbc-examples] and for [JPA][jpa-examples].

[first-part]: {% post_url 2026-03-01-integration-test-database %}
[jdbc-propagation-examples]: https://github.com/K987/spring-boot-integration-tests/blob/jdbc-test-transaction/integration-test-persistence/service/src/integrationTest/java/com/examples/application/user/PropagationTest.java
[jpa-propagation-examples]: https://github.com/K987/spring-boot-integration-tests/blob/jpa-test-transactions/integration-test-persistence/service/src/integrationTest/java/com/examples/application/user/PropagationTest.java
[jpa-multi-tx-examples]: https://github.com/K987/spring-boot-integration-tests/blob/jpa-test-transactions/integration-test-persistence/service/src/integrationTest/java/com/examples/application/user/LongRunningOperationTest.java
[jdbc-rollback-examples]: https://github.com/K987/spring-boot-integration-tests/blob/jpa-test-transactions/integration-test-persistence/service/src/integrationTest/java/com/examples/application/user/RollbackTest.java
[jpa-rollback-examples]: https://github.com/K987/spring-boot-integration-tests/blob/jpa-test-transactions/integration-test-persistence/service/src/integrationTest/java/com/examples/application/user/RollbackTest.java
[spring-tx-orm]: https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/tx.html#testcontext-tx-annotation-demo
[test-flush-examples]: https://github.com/K987/spring-boot-integration-tests/blob/jpa-test-transactions/integration-test-persistence/service/src/integrationTest/java/com/examples/application/user/NoFlushTest.java
[spring-test-annotations]: https://docs.spring.io/spring-framework/reference/testing/annotations/integration-spring.html
[jdbc-examples]: https://github.com/K987/spring-boot-integration-tests/tree/jdbc-test-transaction/integration-test-persistence
[jpa-examples]: https://github.com/K987/spring-boot-integration-tests/tree/jpa-test-transactions/integration-test-persistence
