---
title: Testing Spring REST Webservices
excerpt: Learn how to effectively test Spring REST APIs using RestAssured, OpenAPI, and Spring Boot, with a focus on integration and end-to-end testing.
last_modified_at: 2026-01-30T10:00:00-05:00
---

The article has been updated to reflect changes in Spring Boot 4 and RestAssured 6.x on __2026.02.05__.
You may find additional notes at the end of the article.
{: .notice--warning}

# Overview

Testing RESTful web services is a critical part of ensuring your API behaves as expected. 
In this post, we’ll explore how to set up and run integration tests for a Spring Boot REST API using **RestAssured**, **OpenAPI Generator**, and **Spring Boot Test**. We’ll cover project structure, test configuration, and best practices for writing maintainable and robust tests.

# Web testing support

<table>
    <thead>
        <tr>
            <th>Tool</th>
            <th>Library/Module</th>
            <th>System under test</th>
        </tr>
    </thead>
    <tbody>
        {% for tool in site.data.rest-client-support %}
            <tr>
                <td>{{tool.name}}</td>
                <td>{{tool.library}}</td>
                <td>
                    <ul>
                        {% for sut in tool.sut %}
                            <li>{{sut}}</li>
                        {% endfor %}
                    </ul>
                </td>   
            </tr>
        {% endfor %}
    </tbody>
</table>

* **[MockMvc][mock-mvc]** is a server side test framework of the Spring framework. It offers a low level API and mocks servlet objects, so instead running real servlet container it directly calls the `DispatcherServlet`.
As it runs server-side test it allows access to, and therefore assertions on, various components involved in the request handling, such as handlers, interceptors, HTTP request and response objects, modelAndView objects. 
MockMvc is suitable for testing MVC applications, but for REST APIs it is not that handy mainly because it does not provide out of the box request serialization and response deserialization.
It supports `XPath` and `JsonPath` validations, but they are not so convenient.
* **[WebTestClient][web-test-client]** is the next level web testing API of the Spring framework. It can be used as an HTTP client testing web applications and, just like `MockMvc`, to perform server-side tests with mocked servlet environments. 
Besides JSON and XML validation capabilities it also supports request serialization and response deserialization.
* **[TestRestTemplate][test-rest-template]**: is a simple test REST client utility class of Spring Boot. It acts exactly the same way as a `RestTemplate`, but does not throw exceptions on 4xx and 5xx responses. As `RestTemplate` is deprecated in favor of `RestClient`, `TestRestTemplate` is also deemed deprecated in favor of `TestRestClient`.
* **[RestAssured][rest-assured]**: is a standalone library specifically for REST API testing. It offers [integration][rest-assured-spring] with the Spring framework to enable standalone unit testing of individual controllers.
It comes with an extensive test feature set: configurable serialization, schema support, authentication support, CSRF support, SSL support.
* **[RestTestClient][rest-test-client]**: introduced in Spring Framework 7.0, based on the `RestClient` interface. Fluent style, lightweight alternative of `MockMvc`, `WebTestClient` and `TestRestTemplate` for testing RESTful web services. Supports AssertJ.
* **[MockWebServiceClient][mock-web-service-client]**: is a mock SOAP service client. It mocks the servlet container similarly to `MockMvc`.

We should prefer either `RestAssured` or `RestTestClient` to test synchronous REST API.
While `RestTestClient` is native to Spring and well integrated with `AssertJ`, `RestAssured` is more feature rich in many aspects and has better support for OpenAPI generated clients.
If we had to test asynchronous REST APIs, then `WebTestClient` or `RestAssured` would be the best choices.

**For this example**, we’ll use **RestAssured** to test a traditional REST API in an end-to-end (e2e) style. 
The main reason to this choice is that RestAssured clients can be easily generated from OpenAPI specifications using the OpenAPI Generator tool.

# Project Setup

## Project Structure

The project follows standard multimodule structure:
```
application     -- Root, configuration
└── web         -- REST Controllers, generated server stubs
└── service     -- Domain services
└── persistence -- Data access layer (repositories)
└── domain      -- Domain objects (eg. entities)
```

## Model first approach

The example project implements the [Swagger Petstore][swagger-petstore] REST  API using a **model-first** approach.
The REST endpoints are generated from an `openapi.yaml` specification.

The REST API server interfaces are generated with Gradle `org.openapi.generator` plugin in the web module.

```groovy
// web/build.gradle
openApiGenerate {
	generatorName = 'spring'
	inputSpec = "${rootProject.projectDir}/openapi/pet-store.yaml"
	modelNameSuffix = 'Dto'
	configOptions = [
			library	: 'spring-boot',
			useSpringBoot3 : 'true',
			useSpringController : 'true',
			useSpringBuiltInValidation : 'true',
			skipDefaultInterface : 'true',
			unhandledException : 'true',
			interfaceOnly : 'true',
			generateBuilders : 'true',
			documentationProvider : 'source',
			apiPackage : 'com.examples.application.api.v1',
			modelPackage : 'com.examples.application.api.v1',
			useTags : 'true',
			openApiNullable	: 'false'
	]
	globalProperties = [
			apiTests : 'false',
			modelTests : 'false',
	]
}

sourceSets {
    main {
	    java {
			srcDir layout.buildDirectory.dir('generate-resources/main/src/main/java')
		}
	}
}

tasks.named('compileJava') {
	dependsOn 'openApiGenerate'
}
```

## Test Source Sets 

Each test level has its own source set and directory:

```groovy
// build.gradle
subprojects {
  pluginManager.withPlugin('java') {
    apply plugin: 'jvm-test-suite'
	testing {
	  suites {
	    test {
		  useJUnitJupiter()
		  dependencies {
		    implementation 'org.mockito:mockito-core'
			implementation 'org.assertj:assertj-core'
			implementation 'org.hamcrest:hamcrest'
		  }
		}
		integrationTest(JvmTestSuite) {
		  dependencies {
			implementation project()
			implementation 'org.springframework.boot:spring-boot-starter-test'
		  }
		}
	  }
    }
    
    //include implementation dependencies into integrationTestImplementation configuration
	configurations["integrationTestImplementation"].extendsFrom(configurations.implementation)
    
	tasks.named('check') {
	  dependsOn 'integrationTest'
	}
  }
}
```

Note: Avoid using org.springframework.boot:spring-boot-starter-aspectj-test as a unit test dependency. Unit tests should focus on business logic in POJOs, not framework dependencies.

For the sake of simplicity the database layer is an H2 instance.

# Test Configuration

## Where to Place Which Test?

* Web module: Integration tests targeting the web layer. These are grey-box tests and do not require a fully configured application context.
* Top-level module: Black-box tests exercising the whole application context. Test classes here should not reference other modules to avoid abstraction leaks.

A common mistake I often see is that various modules appearing as test dependencies in the top level module, eg: `integrationTestImplementation project(':web')`. This can be seen as an abstraction leak.
Most often this occurs to access the domain object classes or the DTO classes to use in request / response serialization / deserialization.

Let's see how this can be done differently.

## Generating Test Clients

The `org.openapi.generator` Gradle plugin is also used to generate a strongly typed test client for RestAssured:
```groovy
// application/build.gradle

dependencies {
	integrationTestImplementation 'com.fasterxml.jackson.core:jackson-annotations'
	integrationTestImplementation 'com.fasterxml.jackson.core:jackson-databind'
	integrationTestImplementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
	// RestAssured 5.x is incompatible with Groovy 5.x used by Spring Boot 4.x
	integrationTestImplementation 'io.rest-assured:rest-assured:6.0.0'
}

openApiGenerate {
	generatorName = 'java'
	inputSpec = "${rootProject.projectDir}/openapi/pet-store.yaml"
	modelNameSuffix = 'Dto'
	configOptions = [
	        library: 'rest-assured',
			useJakartaEe: 'true',
			serializationLibrary: 'jackson',
			openApiNullable: 'false'
	]
	globalProperties = [
	        apiTests: 'false',
			modelTests: 'false',
	]
}

sourceSets {
	integrationTest {
		java {
			srcDir layout.buildDirectory.dir('generate-resources/main/src/main/java')
		}
	}
}

tasks.named('compileJava') {
	dependsOn 'openApiGenerate'
}
```

## Running a Real Server

Since RestAssured cannot mock the servlet container, we need a running web server. Use the `@SpringBootTest` annotation to configure the web environment:

```java
@SpringBootTest(
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, 
        useMainMethod = SpringBootTest.UseMainMethod.ALWAYS)
```
* `WebEnvironment.RANDOM_PORT`: Creates a WebApplicationContext in a full servlet container on a random port.
* `UseMainMethod.ALWAYS`: Invokes the main method to set up the ApplicationContext, making the test environment more realistic.

The Spring Boot documentation highlights some of the among the different web environment configurations. The list are from complete, therefore to ensure the most realistic behaviour for such system-test like set up a real web environment is better.
{: .notice}

## Configuring the Test Client

So now we have a running server. How can it be accessed from the client? The entry point of the test client is the `ApiClient` class. Once it is configured we can reuse it as we need. To configure it properly at least the server port must be passed.
The easiest way is to create a test configuration and turn the APIClient into a bean:
```java
// in integrationTest source set

@TestConfiguration(proxyBeanMethods = false)
public class ApiClientConfiguration {

    @Bean
    @Lazy
    ApiClient apiClient(@LocalServerPort String port) {
        return ApiClient
                .api(ApiClient.Config
                        .apiConfig()
                        .reqSpecSupplier(() -> new RequestSpecBuilder()
                                .setBaseUri("http://localhost:" + port)
                                .setConfig(config().objectMapperConfig(objectMapperConfig().defaultObjectMapper(jackson())))
                        )
                );
    }
}
```

* `@TestConfiguration`: Adds additional, test related, configuration to the ApplicationContext.
* Our only bean is created only when requested with `@Lazy`. We can't create the bean eagerly because by the time it was created the web server, hence the server port, wouldn't be known.
* `@LocalServerPort` is just shorthand for `@Value("${local.server.port}")`.
* An APIClient instance is created passing the server base URI pointing to the test server and a default ObjectMapper configuration used for serialization.

## Writing the tests

Now we can finally run the tests.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, useMainMethod = SpringBootTest.UseMainMethod.ALWAYS)
@Import(ApiClientConfiguration.class)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class UserApiTest {

    @Autowired
    ApiClient apiClient;

    UserApi userApi;

    @BeforeEach
    void setUp() {
        userApi = apiClient.user();
    }

    @Test
    @Order(1)
    void createUserTest() {
        UserDto dto = new UserDto()
                .id(100L)
                .username("username")
                .firstName("firstName")
                .lastName("lastName")
                .userStatus(1)
                .email("test@test.com")
                .password("pwd");

        UserDto response = userApi.createUser()
                .body(dto)
                .executeAs(resp -> resp.then()
                        .contentType(ContentType.JSON)
                        .statusCode(HttpStatus.SC_OK)
                        .extract().response());

        assertThat(response).isNotNull();
        assertAll(
                () -> assertThat(response.getId()).isEqualTo(1L),
                () -> assertThat(response.getEmail()).isEqualTo("test@test.com"),
                () -> assertThat(response.getUsername()).isEqualTo("username")
                //...
        );
    }

    @Test
    @Order(2)
    void queryExistingUserTest() {

        UserDto response = userApi.getUserByName()
                .usernamePath("username")
                .executeAs(resp -> resp.then()
                        .contentType(ContentType.JSON)
                        .statusCode(HttpStatus.SC_OK)
                        .extract().response());

        assertThat(response).isNotNull();
        assertAll(
                () -> assertThat(response.getId()).isEqualTo(1L),
                () -> assertThat(response.getEmail()).isEqualTo("test@test.com"),
                () -> assertThat(response.getUsername()).isEqualTo("username")
                //...
        );
    }
    
    // ...
}
```

Unlike the `MOCK` web environment set up the transactions are committed. The server transactions can not be rolled back at the end of the test method as they executed separately.
Hence, we can  order the dependent test cases in a sequence.
{: .notice}

# Conclusion

In this post, we demonstrated how to use OpenAPI Generator and RestAssured to create robust, type-safe integration tests for a Spring Boot REST API.
By running tests against a real server instance and exercising the API as a black box, we ensure our tests are as realistic as possible.

For code-first projects, you can still generate test clients by first generating the OpenAPI spec.

The full example is available on my [GitHub][git-hub-example].

## Notes on Spring Boot 4 upgrade

As the inline comment `application/build.gradle` points out we need to use RestAssured 6.x. Using an older version with Spring Boot 4.x will eventually lead to cryptic NPE errors due to Groovy version incompatibility.
Spring Boot 4.x upgrades to Groovy 5.x, RestAssured is only compatible with Groovy 4.x up to version 5.x.

**Why this poses a risk?**

We are generating our client API with OpenAPI Generator, in the supported [library definitions][open-api] we can find the dependency versions required by the generated code. In fact those can be also found in build configuration of the client: `application/build/generate-resources/main/build.gradle`
It is 5.5.6 for RestAssured, hence we are using a potentially incompatible version. 
Nevertheless, the generated classes does not miss a dependency declaration for RestAssured, so it may be safe to override in our build configuration.

Happy testing!

[mock-mvc]: https://docs.spring.io/spring-framework/reference/testing/mockmvc.html
[web-test-client]: https://docs.spring.io/spring-framework/reference/testing/webtestclient.html
[test-rest-template]: https://docs.spring.io/spring-boot/reference/testing/test-utilities.html#testing.utilities.test-rest-template
[rest-assured]: https://rest-assured.io
[rest-assured-spring]: https://github.com/rest-assured/rest-assured/wiki/Spring
[mock-web-service-client]: https://docs.spring.io/spring-ws/docs/current/reference/html/#_ii_reference
[swagger-petstore]: https://petstore3.swagger.io
[git-hub-example]: https://github.com/K987/spring-boot-integration-tests/tree/master/system-test-restassured
[rest-test-client]: https://docs.spring.io/spring-framework/reference/testing/resttestclient.html
[open-api]: https://openapi-generator.tech/docs/generators/java