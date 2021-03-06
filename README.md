# OSGi Configuration Support for Apache HttpComponents Client

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/net.distilledcode/httpclient-configuration-support/badge.svg?style=flat-square)](https://maven-badges.herokuapp.com/maven-central/net.distilledcode/httpclient-configuration-support)

The OSGi Configuration Support for Apache HttpComponents Client implements a set of configurable OSGi services that allow `HttpClient` and `HttpClientBuilderFactory` instances to be configured via Configuration Admin and registered in the OSGi service registry, ready for other services to consume.

## Motivation

There is no default request timeout set for `HttpClient` instances, which has caused production systems to become unavailable due to a single unresponsive backend system. Implementing configuration of the request timeout (and possibly other settings) for each service using an `HttpClient` is a burden on developer productivity. Rather than forcing developers to create ad-hoc configurations, a consistent approach to `HttpClient` configuration centralises the logic and can provide a sensible default configuration (e.g. with a short request timeout) and guarantees that forgetting to implementing these configuration options never brings down a system again.

The life-cycle of Apache HttpComponents `HttpClient` instance is not trivial to understand, and developers wanting to _just use_ an `HttpClient` often do not want to (and arguably should not need to) concern themselves with these aspects.

While many aspects of an `HttpClient` can be easily configured declaratively, configuring other aspects requires complex objects or even custom code. To get the best of both worlds, a pre-configured `HttpClientBuilderFactory` is registered for each HttpClient configuration. This allows a developer to only focus on the complex bits of configuration, while all other configuration options are automatically available.  

## Benefits
- editable default configuration for `HttpClient` instances
- editable non-default configuration for `HttpClient` instances
- possibility to assign non-default configurations by ID to a service (at development or runtime)
- automatic management of the life-cycle of `HttpClient` instances
- no more inconsistent ad-hoc configurations
- less work for developers using `HttpClient` in OSGi
- editable default configuration for `HttpClientBuilderFactory` instances
- editable non-default configuration for `HttpClientBuilderFactory` instances

## Requirements

- Framework 1.8
- MetaType 1.3 (e.g. org.apache.felix.metatype 1.1.2)
- Declarative Services
- Configuration Admin

## Where can I get it?

This bundle is [available from maven central](http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22httpclient-configuration-support%22).

## How can I use it with `HttpClient` instances?

By default a configurable HttpClient is registered. This client will be injected into any field annotated with a `@Reference` (and no target filter), because its `service.ranking` is higher than the `service.ranking` of a HttpClient instance configured via a factory configuration.

To get started, the following is sufficient:

```java
@Component(service = TestService.class, immediate = true)
public class TestService {

    private static final Logger LOG = LoggerFactory.getLogger(TestService.class);

    @Reference
    private HttpClient httpClient;

    @Activate
    private void activate() {
        LOG.info("Bound HttpClient {}", httpClient);
    }
}
```

The __default HttpClient__ can be configured using the configuration-pid `net.distilledcode.httpclient.Configuration.default`.

Example:

```
# file: net.distilledcode.httpclient.Configuration.default.config
request.config.socket.timeout=I"2000"
```

In order to use a specific configuration fo `TestService`, a configuration can be created

```
# file: TestService.config
httpClient.target="(httpclient.config.name\=test-service)" # note the escaped "=" sign
```

This instructs `TestService` to require an `HttpClient` service registered with a matching property. By default this is not satisfied. However, a configuration can be provided:

```
# file: net.distilledcode.httpclient.Configuration-test-service.config
httpclient.config.name="test-service"
request.config.socket.timeout=I"5000"
```

It is also possible to define a target filter in the service's source code:

```java
@Component(service = TestService.class, immediate = true)
public class TestService {

    private static final Logger LOG = LoggerFactory.getLogger(TestService.class);

    @Reference(target = "(httpclient.config.name=test-service)")
    private HttpClient httpClient;

    @Activate
    private void activate() {
        LOG.info("Bound HttpClient {}", httpClient);
    }
}
```

The implementation dynamically exposes all suitable setters (the ones with a single primitive type or String argument) of [`RequestConfig.Builder`](https://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/client/config/RequestConfig.Builder.html) and [`HttpClientBuilder`](https://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/impl/client/HttpClientBuilder.html) as configuration properties. 

## How can I use it with `HttpClientBuilderFactory` instances?

Using a pre-configured `HttpClientBuilderFactory` is very much like using Apache HttpComponents Client OSGi without configuration support and the life-cycle of the `HttpClient` needs to be managed by hand.  

Take note as well of one pitfall: because both the Apache HttpComponents Client OSGi bundle and the configuration support bundle register an `HttpClientBuilderFactory` service. This bundle registers the service with `service.ranking=100`. This gives it precedence over named configurations as well as over the service registered by Apache HttpComponents Client OSGi. However, in order to ensure the pre-configured factory is used, the reference needs to be declared to be greedy.
  
```java
@Component(service = BuilderTestService.class, immediate = true)
public class BuilderTestService {
    
    // GREEDY reference so the Apache HttpComponents Client's original
    // factory is substituted once the pre-configured one becomes available
    @Reference(policyOption = ReferencePolicyOption.GREEDY)
    private HttpClientBuilderFactory factory;
    
    private CloseableHttpClient httpClient;
    
    @Activate
    private void activate() {
        HttpClientBuilder builder = factory.newBuilder();
        // ... perform advanced configurations on builder ...
        httpClient = builder.build();
    }
    
    
    @Deactivate
    private void deactivate() {
        // take care of managing the HttpClient's life-cycle manually
        if (httpClient != null) {
            httpClient.close();      
        }
    }
}
```

Assigning a configuration to be used works the same way as in the example for an `HttpClient`. Set the target of the field holding the `HttpClientBuilderFactory` (e.g. `factory`) to a named configuration. 
 
 ```
 # file: BuilderTestService.config
 factory.target="(httpclient.config.name\=test-service)" # note the escaped "=" sign
 ```
 
 This instructs `BuilderTestService` to require an `HttpClientBuilderFactory` service registered with a matching property. By default this is not satisfied. However, a configuration can be provided:
 
 ```
 # file: net.distilledcode.httpclient.Configuration-test-service.config
 httpclient.config.name="test-service"
 request.config.socket.timeout=I"5000"
 ```

Note that this configuration is identical to the one in the `HttpClient` example. This is indeed the case, as for each configuration, both an `HttpClient` service and an `HttpClientBuilderFactory` service are registered.
