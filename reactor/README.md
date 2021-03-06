Overview
========
Reactor-gRPC is a set of gRPC bindings for reactive programming with [Reactor](http://projectreactor.io/).

### Android support
Reactive gRPC supports Android to the same level of the underlying reactive technologies. Spring Reactor
does [not officially support Android](http://projectreactor.io/docs/core/release/reference/docs/index.html#prerequisites),
however, "it should work fine with Android SDK 26 (Android O) and above."

Installation
=====
### Maven
To use Reactor-gRPC with the `protobuf-maven-plugin`, add a [custom protoc plugin configuration section](https://www.xolstice.org/protobuf-maven-plugin/examples/protoc-plugin.html).
```xml
<protocPlugins>
    <protocPlugin>
        <id>reactor-grpc</id>
        <groupId>com.salesforce.servicelibs</groupId>
        <artifactId>reactor-grpc</artifactId>
        <version>[VERSION]</version>
        <mainClass>com.salesforce.reactorgrpc.ReactorGrpcGenerator</mainClass>
    </protocPlugin>
</protocPlugins>
```

### Gradle
To use Reactor-gRPC with the `protobuf-gradle-plugin`, add the reactor-grpc plugin to the protobuf `plugins` section.
```groovy
protobuf {
    protoc {
        // The artifact spec for the Protobuf Compiler
        artifact = "com.google.protobuf:protoc:${protobufVersion}"
    }
    plugins {
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:${grpcVersion}"
        }
        reactor {
            artifact = "com.salesforce.servicelibs:reactor-grpc:${reactiveGrpcVersion}"
        }
    }
    generateProtoTasks {
        ofSourceSet("main")*.plugins {
            grpc { }
            reactor {}
        }
    }
}
```
And add the following dependency: `"com.salesforce.servicelibs:reactor-grpc-stub:${reactiveGrpcVersion}"`

*At this time, Reactor-gRPC with Gradle only supports bash-based environments. Windows users will need to build using 
Windows Subsystem for Linux (win 10) or invoke the Maven protobuf plugin with Gradle.*

Usage
=====
After installing the plugin, Reactor-gRPC service stubs will be generated along with your gRPC service stubs.

* To implement a service using an Reactor-gRPC service, subclass `Reactor[Name]Grpc.[Name]ImplBase` and override the Reactor-based
  methods.

  ```java
  ReactorGreeterGrpc.GreeterImplBase svc = new ReactorGreeterGrpc.GreeterImplBase() {
      @Override
      public Mono<HelloResponse> sayHello(Mono<HelloRequest> request) {
          return request.map(protoRequest -> greet("Hello", protoRequest));
      }

      ...

      @Override
      public Flux<HelloResponse> sayHelloBothStream(Flux<HelloRequest> request) {
          return request
                  .map(HelloRequest::getName)
                  .buffer(2)
                  .map(names -> greet("Hello", String.join(" and ", names)));
      }
  };
  ```
* To call a service using an Reactor-gRPC client, call `Reactor[Name]Grpc.newReactorStub(Channel channel)`.

  ```java
  ReactorGreeterGrpc.ReactorGreeterStub stub = ReactorGreeterGrpc.newReactorStub(channel);
  Flux<HelloRequest> req = Flux.just(
          HelloRequest.newBuilder().setName("a").build(),
          HelloRequest.newBuilder().setName("b").build(),
          HelloRequest.newBuilder().setName("c").build());
  Flux<HelloResponse> resp = stub.sayHelloBothStream(req);
  resp.subscribe(...);
  ```

## Don't break the chain
Used on their own, the generated Reactor stub methods do not cleanly chain with other Reactor operators.
Using the `compose()` and `as()` methods of `Mono` and `Flux` are preferred over direct invocation.

#### One→One, Many→Many
```java
Mono<HelloResponse> monoResponse = monoRequest.compose(stub::sayHello);
Flux<HelloResponse> fluxResponse = fluxRequest.compose(stub::sayHelloBothStream);
```

#### One→Many, Many→One
```java
Mono<HelloResponse> monoResponse = fluxRequest.as(stub::sayHelloRequestStream);
Flux<HelloResponse> fluxResponse = monoRequest.as(stub::sayHelloResponseStream);
```

## Retrying streaming requests
`GrpcRetry` is used to transparently re-establish a streaming gRPC request in the event of a server error. During a
retry, the upstream rx pipeline is re-subscribed to acquire a request message and the RPC call re-issued. The downstream
rx pipeline never sees the error.

```java
Flux<HelloResponse> fluxResponse = fluxRequest.compose(GrpcRetry.ManyToMany.retry(stub::sayHelloBothStream));
```

For complex retry scenarios, use the `Retry` builder from <a href="https://github.com/reactor/reactor-addons/blob/master/reactor-extra/src/main/java/reactor/retry/Retry.java">Reactor Extras</a>.

## gRPC Context propagation
Reactor does not have a convenient mechanism for passing the `ThreadLocal` gRPC `Context` between threads in a reactive
call chain. If you never use `observeOn()` or `subscribeOn()` the gRPC context _should_ propagate correctly. However,
the use of a `Scheduler` will necessitate taking manual control over Context propagation.

Two context propagation techniques are:

1. Capture the context in a `Tuple<Context, T>` intermediate type, transforming an indirect static context reference
   into an explicit reference.
2. Make use of Reactor's [`subscriberContext()`](https://github.com/reactor/reactor-core/blob/master/docs/asciidoc/advancedFeatures.adoc#adding-a-context-to-a-reactive-sequence)
   API to capture the gRPC context in the call chain.

Modules
=======

Reactor-gRPC is broken down into four sub-modules:

* _reactor-grpc_ - a protoc generator for generating gRPC bindings for Reactor.
* _reactor-grpc-stub_ - stub classes supporting the generated Reactor bindings.
* _reactor-grpc-test_ - integration tests for Reactor.
* _reactor-grpc-tck_ - Reactive Streams TCK compliance tests for Reactor.
