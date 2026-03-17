# RSocket Broker

[![Gitter](https://badges.gitter.im/rsocket-routing/community.svg)](https://gitter.im/rsocket-routing/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

An implementation of the [RSocket Broker Specification](https://github.com/rsocket-broker/rsocket-broker-spec) — a message broker that forwards RSocket requests between routable destinations using tag-based routing and binary metadata frames.

## Overview

RSocket Broker acts as an intermediary between RSocket clients, decoupling producers and consumers. Instead of point-to-point connections, clients connect to the broker and announce their capabilities via tags. The broker maintains a routing table and forwards requests to the appropriate destination based on tag queries found in the request's `ADDRESS` metadata.

Key capabilities:

- **Tag-based routing** — Routes are determined by bitmap-indexed tag queries, not string/regex matching
- **Multiple routing modes** — Unicast (single destination with load balancing), Multicast (broadcast to all matches), and Shard routing
- **Full RSocket interaction model support** — Fire-and-forget, request/response, request/stream, request/channel, and metadata push
- **Broker clustering** — Brokers exchange `BROKER_INFO`, `ROUTE_JOIN`, and `ROUTE_REMOVE` frames to share routing information
- **Spring Boot integration** — Auto-configuration with `@ConfigurationProperties` support

## Modules

| Module                  | Description                                                                                                |
| ----------------------- | ---------------------------------------------------------------------------------------------------------- |
| `rsocket-broker`        | Core broker logic: routing table, RSocket index, socket acceptors, locators, and multicast/unicast routing |
| `rsocket-broker-spring` | Spring Boot auto-configuration, properties binding, cluster support, and metadata extraction               |

## Architecture

```
┌─────────────┐     ROUTE_SETUP      ┌──────────────────┐     ROUTE_SETUP      ┌─────────────┐
│   Client A   │ ──────────────────► │   RSocket Broker  │ ◄────────────────── │   Client B   │
│  (requester) │                     │                    │                     │  (responder) │
│              │     ADDRESS frame   │  ┌──────────────┐ │                     │              │
│              │ ──────────────────► │  │ RoutingTable │ │ ──── forward ────► │              │
│              │                     │  │ RSocketIndex │ │                     │              │
│              │ ◄── response ────── │  └──────────────┘ │ ◄── response ────── │              │
└─────────────┘                      └──────────────────┘                      └─────────────┘
```

1. **Client B** connects to the broker and sends a `ROUTE_SETUP` frame with a service name and tags
2. The broker registers the route in the `RoutingTable` and indexes the RSocket connection in the `RSocketIndex`
3. **Client A** sends a request with an `ADDRESS` frame containing tags that identify the destination
4. The broker queries the routing table using the tags, locates the matching RSocket, and forwards the request

## Key Components

### RoutingTable

Maintains a bitmap-indexed map of `RouteJoin` entries. Supports fast lookup by `Id` or `Tags` using [Roaring Bitmaps](https://roaringbitmap.org/). Emits reactive join/leave events via Reactor `Sinks`.

### RSocketIndex

Maps route IDs to live `RSocket` connections. When an `ADDRESS` arrives, the broker uses the `RSocketIndex` (via `CombinedRSocketQuery`) to find the actual RSocket to forward the request to.

### BrokerSocketAcceptor

The core `SocketAcceptor` that handles incoming connections. It distinguishes between:

- **RouteSetup** — a client registering a routable destination
- **BrokerInfo** — a peer broker establishing a cluster connection

On disconnect, routes are automatically cleaned up from the routing table and index.

### RSocketLocator

Strategy interface for selecting the target RSocket based on an `Address`. Implementations:

| Locator                   | Routing Type | Behavior                                                                                                                                                    |
| ------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `UnicastRSocketLocator`   | `UNICAST`    | Selects a single RSocket (load balanced if multiple matches). If no match exists, returns a `ResolvingRSocket` that waits for the route to become available |
| `MulticastRSocketLocator` | `MULTICAST`  | Returns a `MulticastRSocket` that broadcasts to all matching RSockets                                                                                       |
| `CompositeRSocketLocator` | All          | Delegates to the first locator that supports the routing type                                                                                               |

### Load Balancing

Unicast routing supports pluggable load balancing via RSocket's `LoadbalanceStrategy`. The default strategy is configurable, and clients can override it per-request using the `io.rsocket.broker.LBMethod` tag. Built-in strategies include round-robin and weighted (based on `WeightedStats`).

## Configuration (Spring Boot)

The broker is configured via `io.rsocket.broker.*` properties:

```yaml
io:
  rsocket:
    broker:
      uri: tcp://localhost:8001 # Broker listen URI
      default-load-balancer: round-robin # Default LB strategy
      cluster:
        enabled: false # Enable broker clustering
        uri: tcp://localhost:7001 # Cluster listen URI
      brokers: # Peer broker URIs (for clustering)
        - uri: tcp://broker2:7001
```

## Transport Protocols

The broker supports three URI schemes for its listen URI and cluster connections:

| Scheme   | Transport       | Example              |
| -------- | --------------- | -------------------- |
| `tcp://` | Raw TCP         | `tcp://0.0.0.0:8001` |
| `ws://`  | WebSocket       | `ws://0.0.0.0:8001`  |
| `wss://` | WebSocket + TLS | `wss://0.0.0.0:8443` |

### Configuring Secure WebSocket (wss://)

Using `wss://` requires TLS to be configured on the underlying Reactor Netty server. The broker's `DefaultServerTransportFactory` creates a `NettyRSocketServerFactory` and sets the transport to WebSocket, but does not configure SSL directly. You need to provide an `RSocketServerCustomizer` bean to configure TLS:

```java
@Bean
RSocketServerCustomizer rSocketServerCustomizer() {
    return server -> server.tcpConfiguration(tcp ->
        tcp.secure(ssl -> ssl.sslContext(
            SslContextBuilder.forServer(certFile, keyFile).build()
        ))
    );
}
```

Alternatively, when using Spring Boot's SSL bundles (Spring Boot 3.1+), you can configure SSL declaratively:

```yaml
io:
  rsocket:
    broker:
      uri: wss://0.0.0.0:8443

spring:
  ssl:
    bundle:
      jks:
        broker:
          keystore:
            location: classpath:broker-keystore.p12
            password: changeit
            type: PKCS12
```

Then reference the SSL bundle in a customizer:

```java
@Bean
RSocketServerCustomizer rSocketServerCustomizer(SslBundles sslBundles) {
    return server -> server.tcpConfiguration(tcp ->
        tcp.secure(ssl -> ssl.sslContext(sslBundles.getBundle("broker").createSslContext()))
    );
}
```

For cluster connections between brokers, the `ClientTransportFactory` handles `wss://` URIs. Reactor Netty's `WebsocketClientTransport.create(URI)` automatically negotiates TLS when the scheme is `wss`. The default JVM trust store is used; to customize the trust store, provide a custom `ClientTransportFactory` bean (see the rsocket-broker-client documentation for the extension pattern).

## Getting Started

### Prerequisites

- Java 21+
- The [rsocket-broker-client](https://github.com/rsocket-broker/rsocket-broker-client) library (version matching the broker)

### Dependencies

Add the Spring Boot starter to your project:

**Gradle:**

```groovy
implementation 'io.rsocket.broker:rsocket-broker-spring:0.4.0-SNAPSHOT'
```

**Maven:**

```xml
<dependency>
    <groupId>io.rsocket.broker</groupId>
    <artifactId>rsocket-broker-spring</artifactId>
    <version>0.4.0-SNAPSHOT</version>
</dependency>
```

The `rsocket-broker-spring` module transitively includes `rsocket-broker` (core) and the required `rsocket-broker-client` frame/common modules.

### Running the Broker

Create a Spring Boot application with the broker on the classpath:

```java
@SpringBootApplication
public class BrokerApplication {
    public static void main(String[] args) {
        SpringApplication.run(BrokerApplication.class, args);
    }
}
```

The `BrokerAutoConfiguration` will automatically set up the routing table, RSocket index, socket acceptors, and start listening for connections.

## Building from Source

```bash
./gradlew build
```

## Related Projects

- [rsocket-broker-spec](https://github.com/rsocket-broker/rsocket-broker-spec) — The RSocket Broker protocol specification
- [rsocket-broker-client](https://github.com/rsocket-broker/rsocket-broker-client) — Client library for connecting to an RSocket Broker
- [rsocket-java](https://github.com/rsocket/rsocket-java) — The RSocket Java implementation

## License

This project is released under the [Apache License 2.0](LICENSE.txt).
