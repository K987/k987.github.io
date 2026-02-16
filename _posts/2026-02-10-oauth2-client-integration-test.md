---
title: "Spring Boot REST Client Integration Testing - part II: Client authorization"
excerpt: How to design integration tests with client authorization in play.
tags: [integration testing, spring boot, rest client, oauth2, testcontainers, keycloak]
---

# Overview

In my [previous post][integration-test-external], we explored how to design integration tests for a Spring Boot REST client, covering test environment setup, server response mocking, and client validation.
While we made progress, we still need to address real-world scenarios, starting with authorization.

I’ve been eager to write this post because it goes beyond tools and techniques.
We’ll also tackle architectural design considerations, a common bottleneck, as testability is often overlooked during architecture planning.
You’ll see that the real challenge isn’t OAuth’s complexity or authentication/authorization hurdles, but rather handling cross-cutting concerns in a clean, maintainable way.

Designing for testability at the unit test level improves code quality and maintainability by promoting SOLID principles.
The same holds true for integration tests, where design for testability contributes to loosely coupled, highly cohesive modular application architecture.

Unit tests rarely deal with cross-cutting concerns; if they need to, it may be a sign of poor design. 
But at the integration level, these concerns are unavoidable.
We’ll explore how to address them effectively in test design.

_Before continuing, ensure you’re familiar with the example application from the [previous post][integration-test-external]._

# Adding OAuth2 client authorization

In a real-world scenario, the need to secure communication with downstream services is common.
In our example, we already have an API key in place.
When it comes to authentication and authorization, the three major aspects to consider are:
* _What is the authentication and authorization mechanism in place?_ It can be as simple as an API key, or Basic Auth, or more complex ones like SAML, OAuth2, OpenID Connect, mTLS, etc.
* _Who is the subject of authentication and authorization?_ It can be the application itself or the end user on behalf of whom the application is acting, or a combination of both in case of multitenant applications.
* _How are the credentials acquired?_ Are they static, like an API key or username and password, or dynamic, like token relay mechanisms? Are they acquired by the application itself, or provided by a third-party vault?

The more complex the answers, the less advisable it is to tightly couple these concerns to the business logic layer.

OAuth2 is a highly flexible standard that can be adapted to a wide variety of scenarios.
In the end, the client's request just needs to pass a valid token in the right header and format, and does not need to know how to acquire it.
We want to reflect that in our application design.

Let's start with a naive approach and follow what's written in the relevant documentation for [Spring Boot][spring-boot-oauth2] and [Spring Security][spring-security-oauth2].

We need to add the `spring-boot-starter-oauth2-client` dependency first.
It'll autoconfigure all we need to work with OAuth2 clients.
To inject a request interceptor that adds the authorization header, we need to change the `RestClient` configuration as follows:
```java
// PetWarehouseApiClientConfiguration.java

@Bean
PetWarehouseRepository petApiRepository(
        RestClient.Builder builder,
        PetWarehouseApiClientConfigurationProperties properties,
        // new dependencies
        OAuth2AuthorizedClientManager authorizedClientManager,
        OAuth2AuthorizedClientService authorizedClientService
) {
    // Configure OAuth2 hooks
    OAuth2ClientHttpRequestInterceptor requestInterceptor =
            new OAuth2ClientHttpRequestInterceptor(authorizedClientManager);

    OAuth2AuthorizationFailureHandler authorizationFailureHandler =
            OAuth2ClientHttpRequestInterceptor.authorizationFailureHandler(authorizedClientService);
    requestInterceptor.setAuthorizationFailureHandler(authorizationFailureHandler);
    
    // Build the RestClient with the interceptor
    RestClient client = builder
            // ...
            .defaultRequest(req ->
                    req.attributes(clientRegistrationId(properties.clientId))
            )
            .requestInterceptor(requestInterceptor)
            .build();
    // ...
}
```
An `OAuth2AuthorizedClientManager` is injected through an interceptor into the `RestClient`; this manager is responsible for acquiring and managing the OAuth2 tokens.

We also need to define the `clientRegistrationId` as a request attribute, so the framework knows which client registration to use when fetching the token.
In the example, I use the `defaultRequest` configurer to customize every request.

{% capture notice-1 %}
Alternatively, we could leverage the [HTTP Service Clients Integration][http-service-clients-integration].
This requires a new bean definition that will process the `@ClientRegistrationId` annotations on the repository interface or methods and add the aforementioned interceptor to the underlying `RestClient`:
```java
    @Bean
    OAuth2RestClientHttpServiceGroupConfigurer securityConfigurer(
            OAuth2AuthorizedClientManager manager) {
        return OAuth2RestClientHttpServiceGroupConfigurer.from(manager);
    }
```
In the scope of this example, it is indifferent which one you choose, and your choice may depend on your specific needs.
{% endcapture %}

<div class="notice">
  {{ notice-1 | markdownify }}
</div>

We also need to add some sort of client registration configuration to the configuration. 
I did it under the integration test resources of the service module:
```yaml
# service/src/integrationTest/resources/config/application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          dummy-client-id:
          # redacted for brevity
```

# Fixing the sliced integration test

If we run the sliced integration test `PetWarehouseApiClientTest` now, we see that the test fails to start up because it is missing the bean definition for `OAuth2AuthorizedClientManager`.

_How so?_ According to the documentation, it should be created by the framework.

The problem is that we have a sliced integration test, which only loads part of the application context, enough to bootstrap the `RestClient`, but not Spring Security.

If we insist on the configuration above, we have three options. Let's see them one by one:

## Adding autoconfiguration
Look up the right autoconfiguration class and import it manually via [`@ImportAutoConfiguration`][import-auto-configuration].
One may find these classes in the appendix of the [documentation][spring-boot-auto-configuration].
The problem with this approach is that it is often a slippery slope, as it will try to pull in additional dependencies, which in turn bring further autoconfiguration needs.
Understanding the autoconfiguration mechanism and its dependencies requires knowledge of the intricate internal details of the framework.
I'd want to be bothered with for test fixtures.

Most importantly, these additional auto-configurations break the core idea of sliced integration tests, which is to keep the test context as small as possible.
Combining sliced integration tests with auto-configuration is a bit of an anti-pattern, as it defeats the purpose of slicing the context in the first place.

## Configure the missing beans manually
Define the missing beans for yourself, manually in the test configuration.
Depending on the complexity and how deeply nested they are, this may be easy or completely impossible.
Even if possible, it will lay a considerable overhead by requiring the maintenance of framework-specific code in the test fixtures, which is not ideal.

## Mock the missing beans
Mock the missing bean. In this case, this seems the most feasible approach, as we don't care about the actual OAuth2 logic in this test, just need to satisfy the dependencies.
Using the `@MockitoBean` annotation, we can easily mock the missing bean in the test class:
```java
    // PetWarehouseApiClientTest.java

    @MockitoBean
    OAuth2AuthorizedClientManager authorizedClientManager;

    @MockitoBean
    OAuth2AuthorizedClientService authorizedClientService;
```
Try it... this seems to work, but it required some luck: we didn't have to define the behavior of these mocks, as the dependent code handle no-op behavior in this case.
It was pure luck.

Above all, remember: **you should not mock what you don't own.**

# Design for testability

The approaches above are all workarounds to patch up the design of the application under test.
All of them bring additional and superfluous complexity, hence code smell.
Let's reconsider the design, bearing in mind that the application will be deployed in several test environments before reaching production.
It is very likely that a full-grown authentication and authorization service won't be in place in all of those environments, but only in late stages of integration.
Most of these environments won't even have real users, production-grade data, or public network access.
So, building out and maintaining complex cross-cutting concerns will only burden the testability of business use cases.
A better approach is to conditionally enable or disable such features based on the environment and the use case under test.

_How is it possible?_

This is always a case by case answer. In this specific use case, we have two options:

1. Inject an OAuth2-customized RestClient builder when needed; otherwise, use a plain one. As you may see, `RestClient.Builder` is just a bean like any other, so if we define our own, or use a [customizer][rest-client-customizer], we can make the customized version injected.
   This is very convenient when all our client instances require the same configuration; otherwise, we would need to maintain multiple builder beans.
   This approach is not only valid for this case.
   We can override most of the default bean definitions when needed - it's even better if there are customizer interfaces available to alter the default behavior of the framework.
2. Use [conditional bean registration][conditional-bean-registration] to choose configuration on the fly.
   As the document points out, the most common example is using configuration profiles with the `@Profile` annotation, yet there exist several conditional variants in [Spring Boot][conditional-variants].

The latter is more explicit and flexible; the trade-off is code duplication. Below you see a possible implementation based on conditional bean registration:
```java
@Configuration
@EnableConfigurationProperties(PetWarehouseApiClientConfiguration.PetWarehouseApiClientConfigurationProperties.class)
class PetWarehouseApiClientConfiguration {

    @Bean
    @ConditionalOnProperty(value = "demo.pet.client.client-id")
    PetWarehouseRepository petApiRepositoryWithOauth(
            RestClient.Builder builder,
            PetWarehouseApiClientConfigurationProperties properties,
            OAuth2AuthorizedClientManager authorizedClientManager,
            OAuth2AuthorizedClientService authorizedClientService
    ) {
        // ...
    }

    @Bean
    @ConditionalOnProperty(value = "demo.pet.client.client-id", matchIfMissing = true, havingValue = "-1")
    PetWarehouseRepository petApiRepository(
            RestClient.Builder builder,
            PetWarehouseApiClientConfigurationProperties properties
    ) {
        //...
    }

    // extracted shared code
}
```
It is assumed that if the `client-id` configuration property is defined, then this specific client must use authorization; otherwise, it does not.
When we have mutually exclusive variants of a bean definition, both bean definition methods need to have a condition defined.

We could have used the built-in `@ConditionalOnOAuth2ClientRegistrationProperties` condition as well, or other approaches.

It is important to highlight that conditional bean registration increases the complexity of the overall application design. Special care must be taken to design around this aspect with as little added complexity as possible.
{: .notice--warning}

In general, we should favor implicit conditions over explicit ones, provided the former are robust enough for the use case at hand.
Implicit conditions, like the example above, deduce the condition to be applied from indirect factors.
E.g., if configuration property X (an OAuth2 Auth Server config) is defined, then component Y (OAuth2-capable `RestClient`) is needed.
On the other hand, with an explicit condition, we dedicate a config option, proliferating them and increasing the risk of errors by giving room for invalid configuration constellations.

# Mocking OAuth2 Authorization Server

If we run the integration test with the full application context configured, we might be tempted to enable OAuth2 to test if everything is wired up correctly.
We need something to act as an OAuth2 Authorization Server in our test environment.

In the [previous blog post][integration-test-external], I introduced Testcontainers to mock the downstream Pet API.
Looking at the available TestContainer modules, it is easy to find the [Keycloak Testcontainer][keycloak], which is an open-source IAM application.
It may be a bit of overkill for the use case at hand; however, for production-grade applications, it may be a viable option to have a real OAuth2 Authorization Server in place even in test environments.
My setup is mostly based on the solution provided in [this article][keycloak-integration].

First, we need to create the Keycloak realm definition file with the registered the client application.

{% capture notice-2 %}
The way the referenced article describes the realm export will not work, because the embedded h2 database allows only a single connection at a time.
I suggest running the Keycloak Docker container with the following command:
```bash
docker run -it -p 9090:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin --entrypoint=/bin/bash quay.io/keycloak/keycloak:25.0
```
Then start the server inside the container:
```bash
  /opt/keycloak/bin/kc.sh start-dev
```
Once the realm is configured, you can shut down the server and export the realm configuration.
{% endcapture %}

<div class="notice">
  {{ notice-2 | markdownify }}
</div>

Then, we configure our OAuth2 client in the configuration properties.
```yaml
# application/src/integrationTest/resources/config/application.yaml
demo:
  pet:
    client:
      # ...
      client-id: "pet-api-client"

spring:
  security:
    oauth2:
      client:
        registration:
          pet-api-client:
            provider: default
            client-id: pet-api-client
            client-secret: # your client secret here from the realm configuration
            authorization-grant-type: client_credentials
```

We also need to extend the Testcontainer configurations to start the Keycloak container alongside the other containers.
```java
public interface TestContainerConfiguration {

    @Container
    KeycloakContainer keyCloakContainer =
            new KeycloakContainer() //quay.io/keycloak/keycloak:25.0
                    // pointing to the exported realm definition file, which includes the client registration
                    .withRealmImportFile("keycloak/realm.json");

    @DynamicPropertySource
    static void registerContainerProperties(DynamicPropertyRegistry registry) {
        // register the various properties
        registerAuthServer(registry);
    }
    
    private static void registerAuthServer(DynamicPropertyRegistry registry) {
        registry.add("spring.security.oauth2.client.provider.default.issuer-uri",
                () -> TestContainerConfiguration.keyCloakContainer.getAuthServerUrl() + "/realms/spring-integration-test");
    }
}
```

We are ready to run the integration test now!
If we want to verify that the token is attached to the request, we can leverage WireMock's capabilities to verify headers as well.

First, we need to wrap the container in a `WireMock` instance. I created a helper for that:
```java
public final class WiremockHelper {

    private static final class WireMockClientHolder {
        private static final WireMock wireMockClient = WireMock.create()
                .host(TestContainerConfiguration.wiremockServer.getHost())
                .port(TestContainerConfiguration.wiremockServer.getPort())
                .build();
    }

    public static WireMock getWireMockClient() {
        return WireMockClientHolder.wireMockClient;
    }

    private WiremockHelper() {}
}
```
Then we can add the verification logic to the test method:
```java

    @Test
    void whenFetchingExistingPet_ThenResponseResolved() {
        // ... existing test code ...

        WireMock wireMockClient = WiremockHelper.getWireMockClient();

        wireMockClient.verifyThat(
                WireMock.getRequestedFor(
                        WireMock.urlEqualTo("/v1/pet/12345"))
                        .withHeader("Authorization", WireMock.matching("Bearer .*")
                        )
        );
    }
```

# Conclusion

In this post, we have seen how cross-cutting concerns - in this particular case, authorization with OAuth2 - can affect integration testing.
We have explored that, despite several workarounds, the most sustainable solution is to design the whole application with testability in mind.

In this specific case, we used conditional bean registration to separate the OAuth2-capable `RestClient` configuration from the plain one.

In the second half of the post, we have seen how we can extend the existing infrastructure of Testcontainers to involve a full-blown IAM solution like Keycloak to act as an OAuth2 Authorization Server in our integration tests.
The choice of Keycloak is somewhat arbitrary and serves only demonstration purposes.
Depending on your real-world use case, you may choose a different solution, such as the [JWT extension of WireMock][wiremock-jwt] or a custom-built mock server.

You can find the complete source code of the example application on [GitHub][oauth2-client-example].

[spring-boot-oauth2]: https://docs.spring.io/spring-boot/reference/web/spring-security.html#web.security.oauth2.
[spring-security-oauth2]: https://docs.spring.io/spring-security/reference/servlet/oauth2/client/authorized-clients.html#oauth2-client-rest-client
[http-service-clients-integration]: https://docs.spring.io/spring-security/reference/features/integrations/rest/http-service-client.html
[import-auto-configuration]: https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/ImportAutoConfiguration.html
[spring-boot-auto-configuration]: https://docs.spring.io/spring-boot/appendix/auto-configuration-classes/index.html
[rest-client-customizer]: https://docs.spring.io/spring-boot/reference/io/rest-client.html#io.rest-client.restclient.customization
[conditional-bean-registration]: https://docs.spring.io/spring-framework/reference/core/beans/java/composing-configuration-classes.html#beans-java-conditional
[conditional-variants]: https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html#features.developing-auto-configuration.condition-annotations
[keycloak]: https://testcontainers.com/modules/keycloak/
[integration-test-external]: {% post_url 2025-10-05-integration-test-external-service %}
[keycloak-integration]: https://testcontainers.com/guides/securing-spring-boot-microservice-using-keycloak-and-testcontainers/
[wiremock-jwt]: https://wiremock.org/docs/jwt/
[oauth2-client-example]: https://github.com/K987/spring-boot-integration-tests/releases/tag/oauth2-client-authorization