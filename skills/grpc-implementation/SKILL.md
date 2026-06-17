---
name: grpc-implementation
description: Implement gRPC services end-to-end — proto schema design, code generation, Java/Kotlin/Python stubs, streaming, interceptors, TLS, health checks, and production hardening
---

# gRPC Implementation — Expert Reference

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        gRPC Architecture                         │
│                                                                   │
│  Client Process                      Server Process              │
│  ┌─────────────────┐                ┌─────────────────────────┐  │
│  │  Application    │                │  Application            │  │
│  │  Code           │                │  Code                   │  │
│  ├─────────────────┤                ├─────────────────────────┤  │
│  │  Generated      │                │  Generated Base Class   │  │
│  │  Stub (Blocking │                │  (ServiceImplBase)      │  │
│  │  / Async /      │                ├─────────────────────────┤  │
│  │  Future)        │                │  gRPC Server Runtime    │  │
│  ├─────────────────┤    HTTP/2      ├─────────────────────────┤  │
│  │  gRPC Channel   │◄──────────────►│  Netty / okhttp         │  │
│  │  (ManagedChannel│  multiplexed   │                         │  │
│  │  / insecure /   │  streams over  │                         │  │
│  │  TLS)           │  single TCP    │                         │  │
│  └─────────────────┘                └─────────────────────────┘  │
│                                                                   │
│  Proto file defines the contract — compiled to both sides        │
└─────────────────────────────────────────────────────────────────┘
```

## Streaming Types

```
Unary (request/response):
  Client ──[Req]──► Server
  Client ◄──[Res]── Server

Server Streaming (one request, many responses — e.g., log tail, search results):
  Client ──[Req]──────────────────► Server
  Client ◄──[Res1]──[Res2]──[Res3]── Server

Client Streaming (many requests, one response — e.g., file upload, sensor data):
  Client ──[Req1]──[Req2]──[Req3]──► Server
  Client ◄──[Res]──────────────────── Server

Bidirectional Streaming (chat, real-time collaboration):
  Client ──[Req1]──[Req2]──────────► Server
  Client ◄──[Res1]──────[Res2]────── Server
  (independently send/receive at any pace)
```

---

## Protocol Buffers (proto3)

### Field Types, Numbers, and Rules

```protobuf
syntax = "proto3";

package com.example.order.v1;

option java_package = "com.example.order.v1";
option java_multiple_files = true;

// Enums — start at 0, 0 is the "unknown/unset" value
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;  // always required; default value
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_CONFIRMED   = 2;
  ORDER_STATUS_SHIPPED     = 3;
  ORDER_STATUS_CANCELLED   = 4;
}

// Nested messages
message Money {
  string currency_code = 1;  // "USD", "EUR"
  int64  units         = 2;  // whole units
  int32  nanos         = 3;  // fractional nanos (0-999_999_999)
}

// Maps
message Order {
  string                   order_id   = 1;
  string                   customer_id = 2;
  repeated OrderLineItem   items      = 3;
  OrderStatus              status     = 4;
  Money                    total      = 5;
  map<string, string>      metadata   = 6;  // key-value tags

  // oneof — only one field set at a time
  oneof fulfillment {
    ShippingInfo shipping = 7;
    PickupInfo   pickup   = 8;
  }

  // reserved — blocks re-use of removed field numbers and names
  reserved 9, 10;
  reserved "legacy_field", "old_status";
}

message OrderLineItem {
  string product_id = 1;
  int32  quantity   = 2;
  Money  unit_price = 3;
}

message ShippingInfo {
  string address    = 1;
  string carrier    = 2;
  string tracking   = 3;
}

message PickupInfo {
  string location_id  = 1;
  string pickup_time  = 2;  // ISO 8601
}
```

### Service Definition

```protobuf
// order_service.proto
service OrderService {
  // Unary
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder   (GetOrderRequest)    returns (GetOrderResponse);

  // Server streaming — stream real-time order updates to client
  rpc WatchOrder (WatchOrderRequest)  returns (stream OrderEvent);

  // Client streaming — batch-upload orders
  rpc BulkCreateOrders(stream CreateOrderRequest) returns (BulkCreateResponse);

  // Bidirectional — real-time order negotiation
  rpc NegotiateOrder(stream NegotiationMessage) returns (stream NegotiationMessage);
}

message CreateOrderRequest  { Order order    = 1; }
message CreateOrderResponse { Order order    = 1; string order_id = 2; }
message GetOrderRequest     { string order_id = 1; }
message GetOrderResponse    { Order order    = 1; }
message WatchOrderRequest   { string order_id = 1; }
message OrderEvent          { OrderStatus status = 1; string timestamp = 2; }
message BulkCreateResponse  { int32 created = 1; repeated string failed_ids = 2; }
message NegotiationMessage  { string content = 1; }
```

### Schema Evolution Rules

```
SAFE changes (backward + forward compatible):
  + Add a new field with a new field number
  + Add a new enum value
  + Add a new RPC method
  + Change a field from singular to repeated (mostly safe)

UNSAFE changes (break compatibility — never do these):
  - Remove a field without reserving its number
  - Renumber a field
  - Change a field's wire type (e.g., int32 -> string)
  - Rename a package
  - Remove an enum value without reserving it

PROTOCOL: when removing a field, always add to reserved:
  reserved 5;
  reserved "old_field_name";
```

---

## Code Generation

### protoc (classic)

```bash
# Install protoc + plugins
brew install protobuf          # macOS
apt install -y protobuf-compiler  # Debian/Ubuntu

# Java plugin
# Download protoc-gen-grpc-java from grpc.io/docs/languages/java/
# Place in $PATH as protoc-gen-grpc-java

# Generate Java
protoc \
  --proto_path=src/main/proto \
  --java_out=src/main/java \
  --grpc-java_out=src/main/java \
  src/main/proto/order_service.proto

# Generate Python
pip install grpcio-tools
python -m grpc_tools.protoc \
  -Isrc/proto \
  --python_out=src/generated \
  --grpc_python_out=src/generated \
  src/proto/order_service.proto
```

### buf (modern — preferred)

```yaml
# buf.yaml
version: v2
lint:
  use:
    - DEFAULT
breaking:
  use:
    - FILE          # detect breaking changes at file level

# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/java
    out: src/main/java
  - remote: buf.build/grpc/java
    out: src/main/java
  - remote: buf.build/protocolbuffers/python
    out: src/generated
  - remote: buf.build/grpc/python
    out: src/generated
```

```bash
buf lint                           # check style + conventions
buf breaking --against .git#branch=main  # catch breaking changes in CI
buf generate                       # generate code per buf.gen.yaml
buf push                           # publish schema to Buf Schema Registry
```

---

## Java / Kotlin Implementation

### Gradle Dependencies

```kotlin
// build.gradle.kts
plugins {
    id("com.google.protobuf") version "0.9.4"
}

dependencies {
    implementation("io.grpc:grpc-netty-shaded:1.63.0")
    implementation("io.grpc:grpc-protobuf:1.63.0")
    implementation("io.grpc:grpc-stub:1.63.0")
    implementation("com.google.protobuf:protobuf-java:3.25.3")

    // Spring Boot integration
    implementation("net.devh:grpc-spring-boot-starter:3.1.0.RELEASE")
}

protobuf {
    protoc { artifact = "com.google.protobuf:protoc:3.25.3" }
    plugins {
        id("grpc") { artifact = "io.grpc:protoc-gen-grpc-java:1.63.0" }
    }
    generateProtoTasks {
        all().forEach { it.plugins { id("grpc") } }
    }
}
```

### Server Implementation

```java
// OrderServiceImpl.java
import io.grpc.Status;
import io.grpc.StatusRuntimeException;
import io.grpc.stub.StreamObserver;

public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {

    private final OrderRepository repository;

    public OrderServiceImpl(OrderRepository repository) {
        this.repository = repository;
    }

    // ── Unary ──────────────────────────────────────────────────────────
    @Override
    public void getOrder(GetOrderRequest request,
                         StreamObserver<GetOrderResponse> responseObserver) {
        try {
            Order order = repository.findById(request.getOrderId())
                .orElseThrow(() -> Status.NOT_FOUND
                    .withDescription("Order not found: " + request.getOrderId())
                    .asRuntimeException());

            responseObserver.onNext(GetOrderResponse.newBuilder()
                .setOrder(order)
                .build());
            responseObserver.onCompleted();

        } catch (StatusRuntimeException e) {
            responseObserver.onError(e);
        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL
                .withDescription("Internal error")
                .withCause(e)
                .asRuntimeException());
        }
    }

    // ── Server Streaming ───────────────────────────────────────────────
    @Override
    public void watchOrder(WatchOrderRequest request,
                           StreamObserver<OrderEvent> responseObserver) {
        String orderId = request.getOrderId();
        // subscribe to event bus; push events until client cancels
        EventSubscription sub = eventBus.subscribe(orderId, event -> {
            if (responseObserver.isReady()) {
                responseObserver.onNext(OrderEvent.newBuilder()
                    .setStatus(event.getStatus())
                    .setTimestamp(event.getTimestamp())
                    .build());
            }
        });
        // Clean up when client disconnects
        ((io.grpc.ServerCallStreamObserver<OrderEvent>) responseObserver)
            .setOnCancelHandler(sub::cancel);
    }

    // ── Client Streaming ───────────────────────────────────────────────
    @Override
    public StreamObserver<CreateOrderRequest> bulkCreateOrders(
            StreamObserver<BulkCreateResponse> responseObserver) {
        List<String> failedIds = new ArrayList<>();
        AtomicInteger created = new AtomicInteger(0);

        return new StreamObserver<>() {
            @Override
            public void onNext(CreateOrderRequest req) {
                try {
                    repository.save(req.getOrder());
                    created.incrementAndGet();
                } catch (Exception e) {
                    failedIds.add(req.getOrder().getOrderId());
                }
            }
            @Override
            public void onError(Throwable t) {
                // client-side error; clean up
            }
            @Override
            public void onCompleted() {
                responseObserver.onNext(BulkCreateResponse.newBuilder()
                    .setCreated(created.get())
                    .addAllFailedIds(failedIds)
                    .build());
                responseObserver.onCompleted();
            }
        };
    }
}
```

### Server Bootstrap

```java
// GrpcServer.java
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.protobuf.services.ProtoReflectionService;
import io.grpc.services.HealthStatusManager;

public class GrpcServer {

    public static void main(String[] args) throws Exception {
        HealthStatusManager health = new HealthStatusManager();

        Server server = ServerBuilder.forPort(9090)
            .addService(new OrderServiceImpl(new OrderRepository()))
            .addService(ProtoReflectionService.newInstance())   // grpcurl/Postman
            .addService(health.getHealthService())              // health checks
            .intercept(new AuthServerInterceptor())
            .intercept(new LoggingServerInterceptor())
            .maxInboundMessageSize(4 * 1024 * 1024)            // 4 MB
            .build()
            .start();

        // Mark service as SERVING
        health.setStatus("com.example.order.v1.OrderService",
            HealthCheckResponse.ServingStatus.SERVING);

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            health.enterTerminalState();
            server.shutdown();
        }));

        server.awaitTermination();
    }
}
```

### Client — Three Stub Types

```java
// GrpcClient.java
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import java.util.concurrent.TimeUnit;

public class OrderClient {

    private final ManagedChannel channel;
    private final OrderServiceGrpc.OrderServiceBlockingStub blockingStub;
    private final OrderServiceGrpc.OrderServiceStub asyncStub;
    private final OrderServiceGrpc.OrderServiceFutureStub futureStub;

    public OrderClient(String host, int port) {
        this.channel = ManagedChannelBuilder.forAddress(host, port)
            .usePlaintext()           // no TLS (dev only)
            .build();

        this.blockingStub = OrderServiceGrpc.newBlockingStub(channel)
            .withDeadlineAfter(5, TimeUnit.SECONDS);   // ALWAYS set deadline

        this.asyncStub  = OrderServiceGrpc.newStub(channel);
        this.futureStub = OrderServiceGrpc.newFutureStub(channel);
    }

    // Blocking — simple, single thread
    public Order getOrder(String orderId) {
        return blockingStub.getOrder(
            GetOrderRequest.newBuilder().setOrderId(orderId).build()
        ).getOrder();
    }

    // Async — non-blocking, uses StreamObserver callback
    public void getOrderAsync(String orderId, Consumer<Order> onSuccess) {
        asyncStub.withDeadlineAfter(5, TimeUnit.SECONDS)
            .getOrder(
                GetOrderRequest.newBuilder().setOrderId(orderId).build(),
                new StreamObserver<>() {
                    public void onNext(GetOrderResponse r) { onSuccess.accept(r.getOrder()); }
                    public void onError(Throwable t)       { /* handle */ }
                    public void onCompleted()              { }
                }
            );
    }

    // Future — integrates with ListenableFuture / CompletableFuture
    public ListenableFuture<GetOrderResponse> getOrderFuture(String orderId) {
        return futureStub.withDeadlineAfter(5, TimeUnit.SECONDS)
            .getOrder(GetOrderRequest.newBuilder().setOrderId(orderId).build());
    }

    public void shutdown() throws InterruptedException {
        channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

### Interceptors

```java
// Auth server interceptor
public class AuthServerInterceptor implements ServerInterceptor {

    private static final Metadata.Key<String> AUTH_KEY =
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <Req, Res> ServerCall.Listener<Req> interceptCall(
            ServerCall<Req, Res> call,
            Metadata headers,
            ServerCallHandler<Req, Res> next) {

        String token = headers.get(AUTH_KEY);
        if (token == null || !isValid(token)) {
            call.close(Status.UNAUTHENTICATED.withDescription("Missing or invalid token"), new Metadata());
            return new ServerCall.Listener<>() {};
        }
        return next.startCall(call, headers);
    }

    private boolean isValid(String token) { /* JWT validation */ return true; }
}

// Client interceptor adds auth header to every call
public class AuthClientInterceptor implements ClientInterceptor {

    private final String token;

    public AuthClientInterceptor(String token) { this.token = token; }

    @Override
    public <Req, Res> ClientCall<Req, Res> interceptCall(
            MethodDescriptor<Req, Res> method,
            CallOptions callOptions,
            Channel next) {

        return new ForwardingClientCall.SimpleForwardingClientCall<>(next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<Res> responseListener, Metadata headers) {
                headers.put(
                    Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER),
                    "Bearer " + token
                );
                super.start(responseListener, headers);
            }
        };
    }
}
```

### Spring Boot Integration

```java
// Server-side service with @GrpcService
@GrpcService
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {
    // Spring DI works normally; no manual ServerBuilder needed
    @Autowired private OrderRepository repository;

    @Override
    public void getOrder(GetOrderRequest request,
                         StreamObserver<GetOrderResponse> responseObserver) {
        // ...
    }
}

// Client-side with @GrpcClient
@Service
public class OrderApiClient {

    @GrpcClient("order-service")  // matches application.yml key
    private OrderServiceGrpc.OrderServiceBlockingStub orderStub;

    public Order get(String id) {
        return orderStub.withDeadlineAfter(5, TimeUnit.SECONDS)
            .getOrder(GetOrderRequest.newBuilder().setOrderId(id).build())
            .getOrder();
    }
}
```

```yaml
# application.yml
grpc:
  server:
    port: 9090
  client:
    order-service:
      address: static://order-service:9090
      negotiation-type: PLAINTEXT
```

---

## Python Implementation

### Dependencies

```bash
pip install grpcio grpcio-tools grpcio-health-checking
```

### Server (sync)

```python
# server.py
from concurrent import futures
import grpc
from grpc_health.v1 import health, health_pb2, health_pb2_grpc
import order_service_pb2
import order_service_pb2_grpc

class OrderServiceServicer(order_service_pb2_grpc.OrderServiceServicer):

    def GetOrder(self, request, context):
        order = db.find_order(request.order_id)
        if order is None:
            context.abort(grpc.StatusCode.NOT_FOUND,
                          f"Order not found: {request.order_id}")
        return order_service_pb2.GetOrderResponse(order=order)

    def WatchOrder(self, request, context):
        """Server streaming"""
        for event in event_bus.subscribe(request.order_id):
            if context.is_active():
                yield order_service_pb2.OrderEvent(
                    status=event.status,
                    timestamp=event.timestamp
                )
            else:
                break  # client disconnected

    def BulkCreateOrders(self, request_iterator, context):
        """Client streaming"""
        created, failed_ids = 0, []
        for req in request_iterator:
            try:
                db.save_order(req.order)
                created += 1
            except Exception:
                failed_ids.append(req.order.order_id)
        return order_service_pb2.BulkCreateResponse(
            created=created, failed_ids=failed_ids
        )

def serve():
    health_servicer = health.HealthServicer()
    server = grpc.server(
        futures.ThreadPoolExecutor(max_workers=10),
        options=[
            ("grpc.max_receive_message_length", 4 * 1024 * 1024),
        ]
    )
    order_service_pb2_grpc.add_OrderServiceServicer_to_server(
        OrderServiceServicer(), server
    )
    health_pb2_grpc.add_HealthServicer_to_server(health_servicer, server)

    health_servicer.set("com.example.order.v1.OrderService",
                        health_pb2.HealthCheckResponse.SERVING)

    server.add_insecure_port("[::]:9090")
    server.start()
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
```

### Async Server (asyncio)

```python
# async_server.py
import asyncio
import grpc
import order_service_pb2_grpc

class AsyncOrderServiceServicer(order_service_pb2_grpc.OrderServiceServicer):

    async def GetOrder(self, request, context):
        order = await db.async_find_order(request.order_id)
        if order is None:
            await context.abort(grpc.StatusCode.NOT_FOUND, "Not found")
        return GetOrderResponse(order=order)

    async def WatchOrder(self, request, context):
        async for event in event_bus.asubscribe(request.order_id):
            yield OrderEvent(status=event.status)

async def serve():
    server = grpc.aio.server()
    order_service_pb2_grpc.add_OrderServiceServicer_to_server(
        AsyncOrderServiceServicer(), server
    )
    server.add_insecure_port("[::]:9090")
    await server.start()
    await server.wait_for_termination()

asyncio.run(serve())
```

### Client

```python
# client.py
import grpc
import order_service_pb2
import order_service_pb2_grpc

def get_order(order_id: str):
    with grpc.insecure_channel("localhost:9090") as channel:
        stub = order_service_pb2_grpc.OrderServiceStub(channel)
        try:
            response = stub.GetOrder(
                order_service_pb2.GetOrderRequest(order_id=order_id),
                timeout=5.0   # always set timeout
            )
            return response.order
        except grpc.RpcError as e:
            if e.code() == grpc.StatusCode.NOT_FOUND:
                return None
            raise

# Async client
async def get_order_async(order_id: str):
    async with grpc.aio.insecure_channel("localhost:9090") as channel:
        stub = order_service_pb2_grpc.OrderServiceStub(channel)
        response = await stub.GetOrder(
            order_service_pb2.GetOrderRequest(order_id=order_id),
            timeout=5.0
        )
        return response.order
```

---

## TLS / mTLS

```java
// Server with TLS
Server server = NettyServerBuilder.forPort(443)
    .sslContext(GrpcSslContexts.forServer(certChainFile, privateKeyFile)
        .trustManager(caFile)                       // for mTLS: verify client cert
        .clientAuth(ClientAuth.REQUIRE)             // enforce mTLS
        .build())
    .addService(new OrderServiceImpl())
    .build();

// Client with mTLS
ManagedChannel channel = NettyChannelBuilder.forAddress("api.example.com", 443)
    .sslContext(GrpcSslContexts.forClient()
        .trustManager(caFile)
        .keyManager(clientCertFile, clientKeyFile) // present client cert
        .build())
    .build();
```

```python
# Python TLS client
credentials = grpc.ssl_channel_credentials(
    root_certificates=open("ca.crt", "rb").read(),
    private_key=open("client.key", "rb").read(),   # mTLS
    certificate_chain=open("client.crt", "rb").read()
)
channel = grpc.secure_channel("api.example.com:443", credentials)
```

---

## gRPC-Gateway (gRPC as REST)

```protobuf
// Annotate proto with HTTP options
import "google/api/annotations.proto";

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse) {
    option (google.api.http) = {
      get: "/v1/orders/{order_id}"
    };
  }
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse) {
    option (google.api.http) = {
      post: "/v1/orders"
      body: "order"
    };
  }
}
```

```
┌──────────┐   REST/JSON   ┌─────────────────┐   gRPC    ┌─────────────┐
│  Browser │──────────────►│  gRPC-Gateway   │──────────►│  gRPC Server│
│  Mobile  │◄──────────────│  Reverse Proxy  │◄──────────│             │
└──────────┘               └─────────────────┘           └─────────────┘
   HTTP/1.1 JSON              Transcodes JSON                HTTP/2 Protobuf
```

---

## Error Handling — Status Codes

```
NOT_FOUND          (5)  — entity does not exist
INVALID_ARGUMENT   (3)  — bad input from client (do not retry)
ALREADY_EXISTS     (6)  — create conflict
PERMISSION_DENIED  (7)  — authenticated but not authorized
UNAUTHENTICATED   (16)  — no valid credentials
RESOURCE_EXHAUSTED (8)  — rate limited or quota exceeded
UNAVAILABLE       (14)  — server temporarily down (safe to retry with backoff)
DEADLINE_EXCEEDED  (4)  — timeout expired
INTERNAL           (13) — server bug (do not retry)
UNIMPLEMENTED      (12) — method not implemented
```

```java
// Throw with metadata
Metadata metadata = new Metadata();
metadata.put(Metadata.Key.of("request-id", Metadata.ASCII_STRING_MARSHALLER), requestId);
throw Status.INVALID_ARGUMENT
    .withDescription("Field 'quantity' must be > 0")
    .asRuntimeException(metadata);

// Catch on client
try {
    stub.createOrder(request);
} catch (StatusRuntimeException e) {
    switch (e.getStatus().getCode()) {
        case NOT_FOUND        -> handleNotFound();
        case INVALID_ARGUMENT -> handleBadInput(e.getStatus().getDescription());
        case UNAVAILABLE      -> retry();
        default               -> throw e;
    }
}
```

---

## Health Checking

```bash
# grpc_health_probe CLI
grpc_health_probe -addr=localhost:9090 -service=com.example.order.v1.OrderService

# grpcurl
grpcurl -plaintext localhost:9090 grpc.health.v1.Health/Check
```

```yaml
# Kubernetes liveness/readiness probe
livenessProbe:
  exec:
    command: ["/bin/grpc_health_probe", "-addr=:9090"]
  initialDelaySeconds: 10
readinessProbe:
  exec:
    command: ["/bin/grpc_health_probe", "-addr=:9090",
              "-service=com.example.order.v1.OrderService"]
```

---

## Testing with grpcurl

```bash
# List services (needs reflection enabled)
grpcurl -plaintext localhost:9090 list

# Describe a service
grpcurl -plaintext localhost:9090 describe com.example.order.v1.OrderService

# Call unary RPC
grpcurl -plaintext -d '{"order_id": "ord-123"}' \
  localhost:9090 com.example.order.v1.OrderService/GetOrder

# Call with metadata (auth header)
grpcurl -plaintext \
  -H 'authorization: Bearer eyJ...' \
  -d '{"order_id": "ord-123"}' \
  localhost:9090 com.example.order.v1.OrderService/GetOrder
```

---

## Production Checklist

```
Proto design:
  [ ] All field numbers stable; reserved used for removed fields
  [ ] Enums have _UNSPECIFIED = 0 value
  [ ] buf lint passes (no style violations)
  [ ] buf breaking passes (no wire-incompatible changes)
  [ ] Package uses versioning: com.example.order.v1

Server:
  [ ] Deadlines enforced server-side with context cancellation
  [ ] StatusCode returned (not generic Exception)
  [ ] Health service registered (grpc.health.v1)
  [ ] Reflection enabled (non-prod) / disabled (prod)
  [ ] maxInboundMessageSize set explicitly
  [ ] Interceptors: auth, logging, tracing in place
  [ ] TLS/mTLS configured for inter-service calls

Client:
  [ ] withDeadlineAfter() on every call (default: 5s)
  [ ] Long-lived ManagedChannel (one per target, not per call)
  [ ] Channel keepalive configured for long-idle connections
  [ ] StatusRuntimeException handled with code-specific logic
  [ ] Retry with exponential backoff only on UNAVAILABLE / DEADLINE_EXCEEDED

Performance:
  [ ] Streaming used where sending >1 message in sequence
  [ ] Connection reuse (channel is thread-safe, do not recreate)
  [ ] Message size < 4 MB; use streaming for large payloads
```

---

## Anti-Patterns

```
WRONG: Create a new channel per request
  ManagedChannel ch = ManagedChannelBuilder.forAddress(...).build();  // expensive!
  ch.newCall(...);
  ch.shutdown();
RIGHT: Share channel; it manages connection pooling internally.

WRONG: No deadline
  stub.getOrder(request);
RIGHT: Always set deadline:
  stub.withDeadlineAfter(5, TimeUnit.SECONDS).getOrder(request);

WRONG: Retry on INVALID_ARGUMENT / INTERNAL
  Indicates a client bug or server bug. Retrying makes it worse.
RIGHT: Retry only on UNAVAILABLE or DEADLINE_EXCEEDED with backoff.

WRONG: Large monolithic .proto file with 50 messages
RIGHT: Split by domain — one proto per aggregate / service boundary.

WRONG: No buf lint or breaking change detection in CI
RIGHT: buf lint && buf breaking in every PR pipeline.

WRONG: Remove a field number and reuse it for a different field
  message Foo { string name = 1; }     // remove name
  message Foo { int32 count = 1; }     // reuse field 1 — WIRE CORRUPTION
RIGHT: reserved 1; reserved "name"; then add count = 2;

WRONG: Use streaming for simple request/response
RIGHT: Only use streaming when there are genuinely multiple messages.
```
