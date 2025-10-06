---
title: Spring Boot REST Client Integration Testing - part I
excerpt: Learn how to test and configure Spring Boot integration tests with external webservice dependencies
classes: wide

---

# Overview

In a distributed system, services frequently depend on external APIs to perform their functions. 
A typical scenario involves calling a downstream REST API during request processing. However, even simple API dependencies in production require careful attention to several factors, including:

* **Request/response serialization and deserialization,**
* **Error handling,**
* **Retry mechanisms for transient failures,**
* **Authentication and authorization.**

While testing against a live API might appear straightforward, it presents significant challenges:

* **Unstable APIs** can lead to unreliable test results.
* **Too complex** API workflows are difficult to maintain.
* **Testing isolated components** may necessitate full API configurations, adding unnecessary complexity.
* **APIs may be inaccessible** in the test environment.

Of course, testing against a live system is ideal — but only in later stages, such as in an integration environment.

In **Part I** of this blog series, I demonstrate how to set up integration testing using a simple REST client implementation. I explore:

* **Test slices** for focused testing,
* **System test** scenarios running against the full application context.

In **Part II**, I’ll expand on production-ready features — such as retry logic and authorization — and examine how to effectively test them.

## Project Setup

For this tutorial, I’ll reuse the same project setup as in my previous post, [Testing Spring REST Webservices][rest-assured-post]. 
This ensures consistency and allows us to focus directly on implementing integration tests for REST client interactions.

# Project Structure

At a high level, the project consists of the following components:

* **PetV1Controller**: A REST controller that exposes the API for our service.
* **PetService**: A service layer that processes requests and delegates them to the PetWarehouseApiClient, our REST API client.
* **PetPhotoRepository**: A mocked service to populate photo URLs.
* **PetWarehouseApiClient**: The REST API client implementation. 
Mock Service: Used to populate pet photo URLs.

The REST client and its dependencies are located in the com/examples/application/pet/client package within the service layer.
Let’s examine the key components in detail:

* Two DTOs are used for serialization and deserialization. These are typically generated from API definitions and owned by the external service:

```java
// Pet resource representation
record PetDto(
        Long id,
        String name,
        String status
) {}
```
```java
// Error response body representation
record ErrorDto(
        String message,
        Long petId
) {}
```

* The `PetApiException` class maps error responses from HTTP error codes:

```java
public class PetApiException extends RuntimeException {

    enum ErrorCode {
        PET_EXISTS,
        PET_NOT_FOUND,
        OTHER_ERROR
    }

    @Getter
    private final Long petId;
    private final ErrorCode errorCode;

    PetApiException(HttpStatusCodeException cause, ErrorDto error, ErrorCode errorCode) {
        super(error.message(), cause);
        this.petId = error.petId();
        this.errorCode = errorCode;
    }

    // omitted for brevity
}
```

* The 2 central components of the REST API client implementation
* `PetWarehouseRepository`: An interface leveraging Spring's [HTTP services][spring-http-services], similar to Spring Data repositories. 
It saves use from boilerplate code by using declarative HTTP client. 
It is not exposed to services, only accessible to the `PetWarehouseApiClient`.
* `PetWarehouseApiClient`: The API client facade that abstracts the external API, maps request and response DTOs to domain objects and handles protocol-specific exceptions.

```java
@HttpExchange(accept = MediaType.APPLICATION_JSON_VALUE)
interface PetWarehouseRepository {

    @GetExchange(value = "/{petId}")
    PetDto findPet(@PathVariable Long petId);

    @PostExchange(value = "/",
            contentType = MediaType.APPLICATION_JSON_VALUE)
    PetDto create(@RequestBody PetDto pet);

    @DeleteExchange("/{petId}")
    void delete(@PathVariable Long petId);

    @GetExchange(value = "/search")
    List<PetDto> search(@RequestParam MultiValueMap<String, String> params);

    @PatchExchange(value = "/{petId}",
            contentType = MediaType.APPLICATION_JSON_VALUE)
    PetDto update(@PathVariable Long petId, @RequestBody PetDto petDto);
}
```
```java
@Component
@RequiredArgsConstructor
public class PetWarehouseApiClient {

    private final PetWarehouseRepository repository;

    public Pet findPet(Long petId) {
        try {
            return toPet(repository.findPet(petId));
        } catch (HttpClientErrorException.NotFound | HttpClientErrorException.Gone e) {
            throw toException(e, PetApiException.ErrorCode.PET_NOT_FOUND);
        } catch (HttpClientErrorException.BadRequest e) {
            throw new IllegalArgumentException("Invalid petId: " + petId, e);
        }
    }
    // rest omitted for brevity
    private Pet toPet(PetDto pet) {
        // rest omitted for brevity 
    }

    private PetApiException toException(HttpClientErrorException e, PetApiException.ErrorCode code) {
        // rest omitted for brevity
    }
}
```

* `PetWarehouseApiClientConfiguration`: `@Configuration` class  wires everything together

```java
@Configuration
@EnableConfigurationProperties(PetWarehouseApiClientConfiguration.PetWarehouseApiClientConfigurationProperties.class)
class PetWarehouseApiClientConfiguration {

    private static final String API_KEY_HEADER = "X-API-KEY";

    @ConfigurationProperties(prefix = "demo.pet.client")
    @Validated
    record PetWarehouseApiClientConfigurationProperties(
            @NotNull URL basePath,
            @NotNull String apiKey
    ) {}

    @Bean
    PetWarehouseRepository petApiRepository(
            RestClient.Builder builder,
            PetWarehouseApiClientConfigurationProperties properties
    ) {

        RestClient client = builder
                .baseUrl(properties.basePath.toString())
                .defaultHeaders(httpHeaders ->
                    httpHeaders.add(API_KEY_HEADER, properties.apiKey)
                )
                .build();

        HttpServiceProxyFactory factory = HttpServiceProxyFactory
                .builderFor(
                        RestClientAdapter.create(client)
                ).build();
        return factory.createClient(PetWarehouseRepository.class);
    }
}
```

While this setup is far from being production-ready, it includes several complex elements already. 
In the next section, we’ll explore how to test these components effectively.

# Test slice with `@RestClientTest`

While most of the test-worthy logic resides in the `PetWarehouseApiClient` class — and could be covered by unit tests — there’s a catch: **unit testing requires mocking the PetWarehouseRepository**.

_Why Is This a Problem?_

Mocking comes with a fundamental principle: _Do not mock types you don’t own_. Here’s why this matters:

* When foreign objects are mocked we essentially become a maintainer of someone else's code. Every bit of change they make to their code will potentially break our assumptions.
* There is a good chance that we bake in wrong assumptions into mocks in the absence of sufficient knowledge on their behaviour.
* We may hide important complexity.

Think about `PetWarehouseRepository` we are safe to assume that
```java 
PetDto findPet(@PathVariable Long petId);
```
will eventually return a `PetDto` of the same id provided in the method parameter.
That is a simple, fair assumption. What do we know about the edge cases, for example what would happen if a resource for that id does not exist? Is it going to throw `NoSucElementException` or an `HttpStatusCodeException`.
We can dig into the code or the documentation to find it out of course, but it will remain a brittle process.

Eventually the implementation details depend on the underlying REST API Client let it be `WebClient`, `RestClient`, `RestTemplate`. 

More on this: [Don't mock types you don't own][dont-mock].

_Test slices to the rescue!_ 

Spring Boot introduces the concept of [test slices][test-slices], which load only a portion of the application context. 
Think of them as the integration testing equivalent of auto-configurations.

[RestClientTest][rest-client-test] test slice does two important things:

* Creates beans related to REST request processing (e.g., HTTP message converters, `RestClient.Builder`, JSON support) via auto-configuration.
* Creates and configures a [MockRestServiceServer][mock-rest-service-server] bean to be used in our test to mock the remote API.

**Note**: `MockRestServiceServer` isn’t an actual mock server. 
Requests don’t pass through the network. 
Instead, it uses MockClientHttpRequestFactory to mock the HTTP client used by RestClient. Nevertheless the `RestClient` is a real object.
{: .notice}

Here’s how a scoped integration test looks:

```java
@RestClientTest(
        properties = {
                "demo.pet.client.basePath=http://dummy.org/v1/pet",
                "demo.pet.client.apiKey=THIS_IS_SECRET"
        })
@ContextConfiguration(classes = {
        PetWarehouseApiClientConfiguration.class,
        PetWarehouseRepository.class,
        PetWarehouseApiClient.class
})
public class PetWarehouseApiClientTest {

    @Autowired
    ObjectMapper objectMapper;

    @Autowired
    MockRestServiceServer mockServer;

    @Autowired
    PetWarehouseApiClient warehouseApiClient;

    @Test
    void whenFetchingExistingPet_ThenResponseResolved() throws JsonProcessingException {
        long petId = 1234L;
        String petName = "testName";
        String petStatus = "ON_STOCK";
        List<String> tags = List.of("test-tag-1", "test-tag-2");
        mockServer
                .expect(requestToUriTemplate("http://dummy.org/v1/pet/{petId}", petId))
                .andExpect(header("X-API-KEY", "THIS_IS_SECRET"))
                .andExpect(method(HttpMethod.GET))
                .andRespond(withAccepted()
                        .contentType(MediaType.APPLICATION_JSON)
                        .body(objectMapper.writeValueAsString(new PetDto(petId, petName, petStatus, tags))
                        )
                );
        Pet pet = warehouseApiClient.findPet(petId);
        assertAll(
                () -> mockServer.verify(),
                () -> assertThat(pet.getId()).isEqualTo(petId),
                () -> assertThat(pet.getName()).isEqualTo(petName),
                () -> assertThat(pet.getStatus()).isEqualTo(Pet.Status.AVAILABLE),
                () -> assertThat(pet.getTags()).isEqualTo(tags)
        );
    }
    // rest omitted for brevity
}
```

_Voilà!_ 

* `@RestClientTest`: Instructs the test framework to create dependencies for the test slice. Configuration properties are also passed here.
* `@ContextConfiguration`: Defines the Spring components to populate the application context.

**Note:** In simpler cases—such as when a `RestClient` is directly injected into a component—using the `@RestClientTest#components` attribute is sufficient for defining dependencies.
{: .notice}

# Testing on the whole application context

Let’s explore what changes when we move from testing a single, isolated slice to running tests on the entire application context.

Moving to the top level `application` modules integration test, let’s attempt to define the setup with the following code:

```java
@SpringBootTest
@RestClientTest
public class PetApiTest {

    @Autowired
    MockRestServiceServer mockRestServiceServer;

    @Test
    void someTest() {
        mockRestServiceServer.expect(requestTo("http://localhost"));
        // rest omitted for brevity
    }
}
```

However, running this code results in an initialization error:

```
Configuration error: found multiple declarations of @BootstrapWith for test class [com.examples.application.pet.PetApiTest]: [@org.springframework.test.context.BootstrapWith(value=org.springframework.boot.test.context.SpringBootTestContextBootstrapper.class), @org.springframework.test.context.BootstrapWith(value=org.springframework.boot.test.autoconfigure.web.client.RestClientTestContextBootstrapper.class)]
java.lang.IllegalStateException: Configuration error: found multiple declarations of @BootstrapWith for test class [com.examples.application.pet.PetApiTest]: [@org.springframework.test.context.BootstrapWith(value=org.springframework.boot.test.context.SpringBootTestContextBootstrapper.class), @org.springframework.test.context.BootstrapWith(value=org.springframework.boot.test.autoconfigure.web.client.RestClientTestContextBootstrapper.class)]
```

`@SpringBootTest` initializes the entire application context, including the parts needed by the REST API Client. 
However, we still need a `MockRestServiceServer` bean instance. 
Fortunately, the `@AutoConfigureMockRestServiceServer` annotation creates one for us.

Now, let’s try changing the web environment setup of the `@SpringBootTest`:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
```

This results in a different error:

```
Unable to use auto-configured MockRestServiceServer since mock server customizers have been bound to both a RestTemplate and a RestClient
java.lang.IllegalStateException: Unable to use auto-configured MockRestServiceServer since mock server customizers have been bound to both a RestTemplate and a RestClient
```

_Why is that?_ 

The test context bootstrapper creates a `TestRestTemplate` instance, and the framework doesn’t know which `RESTClient` instance the mock REST server should bind to.

It is easy there exists two solutions, one may think:

1. *Refrain to the Mocked Web Environment*
This works until another `RESTclient` is introduced in the production code, at which point the framework again won’t know where to bind the mock server. 
2. *Manually Wire the MockRestServiceServer*. 
This is a more interesting take, at least it will take more time to hit the wall...

## Wiring up our own MockRestServiceServer

While it’s a good idea to manually wire the `MockRestServiceServer`, there’s a challenge within: **we need a reference to the `RestClient` implementation**.
Looking at the bean method definition, we see that this isn’t directly possible.
RestClient is created is only referenced by a local variable.

```java
  @Bean
  PetWarehouseRepository petApiRepository(
          RestClient.Builder builder, 
          PetWarehouseApiClientConfigurationProperties properties) {

        RestClient client = builder
                .baseUrl(properties.basePath.toString())
                .defaultHeaders(httpHeaders ->
                    httpHeaders.add(API_KEY_HEADER, properties.apiKey)
                )
                .build();
        // omitted for brevity
    }
```

One might consider turning the `RestClient` into a bean, but this isn’t scalable for multiple `RestClient` instances unless using bean qualifiers.
Also, my gut feeling is that `RestClient`s are not intended to be used as beans.
As a last resort, you might try something like this:

```java
@SpringBootTest
public class PetApiTest {

    @Autowired
    RestClient.Builder builder;

    @Test
    void test() throws Exception {
        MockRestServiceServer build = MockRestServiceServer.bindTo(builder).build();
    }
}
```
This won’t work either because the `RestClient` has already been created by the time the mock server is bound to the builder, rendering the mock server configuration ineffective.

Luckily there exists a better alternative.

## WireMock to the rescue

Instead of struggling with these limitations, we can use [WireMock][wire-mock], a versatile mock server that can be integrated with Spring Boot many ways.
The most straightforward way is to use the [Spring Boot Test integration][wire-mock-spring-boot]. However, I will use an alternative approach that is more versatile and scalable.

Spring Boot provides first-class support for [Testcontainers][spring-test-containers]. Here’s how to configure a test using WireMock with Testcontainers:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ImportTestcontainers(TestContainerConfiguration.class)
public class PetApiTest {

    @Autowired
    TestRestTemplate testRestTemplate;

    @Test
    void whenFetchingExistingPet_ThenResponseResolved() {
        RequestEntity<?> request = RequestEntity
                .get(URI.create("/v1/pet/12345"))
                .accept(MediaType.APPLICATION_JSON)
                .build();
        ResponseEntity<Map<String, Object>> response = testRestTemplate.exchange(
                request,
                new ParameterizedTypeReference<>() {});

        assertAll(
                () -> assertThat(response.getStatusCode().value()).isEqualTo(200),
                () -> assertThat(response.getBody()).isNotNull(),
                () -> assertThat(response.getBody().size()).isEqualTo(6),
                () -> assertThat(response.getBody().get("id")).isEqualTo(12345),
                () -> assertThat(response.getBody().get("name")).isEqualTo("dog"),
                () -> assertThat(response.getBody().get("status")).isEqualTo("pending"),
                () -> assertThat(response.getBody().get("category")).isEqualTo(null),
                () -> assertThat(response.getBody().get("tags")).isInstanceOfSatisfying(
                        List.class,
                        list -> assertThat(list.size()).isEqualTo(2)
                ),
                () -> assertThat(response.getBody().get("photoUrls")).isEqualTo(
                        List.of(
                                "http://my.cdm.com/pet/12345/1",
                                "http://my.cdm.com/pet/12345/2",
                                "http://my.cdm.com/pet/12345/3"
                        )
                )
        );
    }
}
```

**Note**: This test method can be simplified using a generated RestAssured client, as described in my [previous post][rest-assured-post].
{: .notice}

The WireMock container is integrated via `@ImportTestcontainers(TestContainerConfiguration.class)`.
This annotation allows Spring Boot to hook the container configurations into the test `ApplicationContext`.
In the `TestContainerConfiguration` class, we define all our containers in one place.

```java
public interface TestContainerConfiguration {

    @Container
    WireMockContainer wiremockServer =
            new WireMockContainer(WIREMOCK_2_LATEST)
                    .withMappingFromResource(PetApiTest.class, "wiremock-config.json");

    @DynamicPropertySource
    static void wiremockProperties(DynamicPropertyRegistry registry) {
        registry.add("demo.pet.client.host", TestContainerConfiguration.wiremockServer::getBaseUrl);
    }
}
```

Using `@ImportTestcontainers` allows us to store container configurations in a central place, making them easily reusable.
Both the container and the DynamicPropertyRegistry could be defined in the test class itself.
{: .notice}

Both `@Container` and `@Testcontainers` are [JUnit5 Extension][junit-testcontainers] annotations that embed the container lifecycle into the test engine lifecycle. 
The `wiremockServer` field is static, because the container in intended to be reused across test executions, thus set up before the first test and teared down after the last test of the class. he `withMappingFromResource` method configures the mock server mappings, pointing to a stub mapping in `src/integrationTest/resources/PetApiTest/wiremock-config.json`.
See its contents below.

The [`@DynamicPropertySource`][dynamic-property-source] is part of Spring’s Test Context Framework, allowing dynamic property injection into the `Environment`. 
Here, we use it to configure the host of `PetWarehouseApiClient` with the WireMock server URL.

```json
{
  "mappings": [
    {
      "request": {
        "method": "GET",
        "urlPattern": "/v1/pet/12345"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "jsonBody":
          {
            "id": 12345,
            "name": "dog",
            "status": "ORDERED",
            "tags": [
              "tag-1",
              "tag-2"
            ]
          }
      }
    }
  ]
}
```

# Conclusion

In this tutorial, we explored two distinct approaches to setting up integration tests for external API calls:

1. **Test Slices**:
  * Allow you to isolate a module fragment from the entire application context.
  * Simplify test configuration and provide **better control** over the test environment.
  * Ideal for extensive testing** of the API client itself.
2. **WireMock with Testcontainers**:
  * Enable integration tests with **real network calls**, simulating external APIs.
  * Perfect for **system tests** that exercise the entire application stack.

Each approach has its strengths, and the choice depends on your testing goals. Use **test slices** for focused, modular testing and **WireMock with Testcontainers** for comprehensive, end-to-end validation.

The full example is available on my [GitHub][git-hub-example].

[rest-assured-post]: {% post_url 2025-09-09-rest-assured-integration-test %}
[spring-http-services]: https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-interface
[test-slices]: https://docs.spring.io/spring-boot/appendix/test-auto-configuration/slices.html
[rest-client-test]: https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html#testing.spring-boot-applications.autoconfigured-rest-client
[mock-rest-service-server]: https://docs.spring.io/spring-framework/docs/6.2.x/javadoc-api/org/springframework/test/web/client/MockRestServiceServer.html
[wire-mock]: https://wiremock.org
[wire-mock-spring-boot]: https://wiremock.org/docs/spring-boot/
[spring-test-containers]: https://docs.spring.io/spring-boot/reference/testing/testcontainers.html
[git-hub-example]: https://github.com/K987/spring-boot-integration-tests/tree/master/integration-test-external-service
[dont-mock]: https://davesquared.net/2011/04/dont-mock-types-you-dont-own.html
[junit-testcontainers]: https://testcontainers.com/guides/testcontainers-container-lifecycle/#_using_junit_5_extension_annotations
[dynamic-property-source]: https://docs.spring.io/spring-framework/reference/testing/annotations/integration-spring/annotation-dynamicpropertysource.html