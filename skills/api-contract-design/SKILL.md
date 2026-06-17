---
name: api-contract-design
description: Design production-quality API contracts — REST, gRPC, GraphQL — covering OpenAPI 3.1, versioning, breaking changes, error handling, pagination, security, and contract testing
---

# API Contract Design

## API-First Design

```
  Traditional:              API-First:
  ┌───────────────┐         ┌───────────────┐
  │ Implement     │         │ Design API    │  ← OpenAPI spec / proto
  │ backend logic │         │ Contract      │    agreed by all parties
  └───────┬───────┘         └───────┬───────┘
          │                         │
  ┌───────▼───────┐         ┌───────▼───────┐
  │ Write API     │         │ Generate:     │
  │ docs after    │         │ - Mock server │  ← Frontend/consumers
  └───────┬───────┘         │ - Client SDKs │    can build immediately
          │                 │ - Test stubs  │
  ┌───────▼───────┐         └───────┬───────┘
  │ Consumers     │                 │
  │ discover      │         ┌───────▼───────┐
  │ surprises     │         │ Implement     │  ← Backend implements
  └───────────────┘         │ to the spec   │    to the agreed contract
                            └───────────────┘

  API-First benefits:
    - Contract is the source of truth, not the implementation
    - Consumers and providers can develop in parallel
    - Breaking changes are caught before deployment
    - Consistent documentation (generated, not manually written)
```

### OpenAPI 3.1 Spec Authoring

```yaml
# openapi.yaml — complete, production-quality example
openapi: "3.1.0"
info:
  title: Order Service API
  version: "1.0.0"
  description: |
    Manages customer orders from creation through fulfillment.
    
    ## Authentication
    All endpoints require Bearer JWT. See [Auth docs](/docs/auth).
    
    ## Rate Limiting
    1000 requests/minute per API key. See X-RateLimit-* response headers.
  contact:
    name: Platform Team
    email: platform@company.com

servers:
  - url: https://api.company.com/v1
    description: Production
  - url: https://api.staging.company.com/v1
    description: Staging

paths:
  /orders:
    post:
      operationId: createOrder
      summary: Create a new order
      description: |
        Creates an order and initiates the fulfillment process.
        This operation is idempotent when an `Idempotency-Key` header is provided.
      tags: [Orders]
      security:
        - BearerAuth: [orders:write]
      parameters:
        - $ref: '#/components/parameters/IdempotencyKey'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
            examples:
              standard:
                summary: Standard order
                value:
                  customer_id: "cust_8x4k2m"
                  items:
                    - sku: "PROD-001"
                      quantity: 2
      responses:
        "201":
          description: Order created
          headers:
            Location:
              schema:
                type: string
              description: URL of the created order
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        "400":
          $ref: '#/components/responses/BadRequest'
        "409":
          $ref: '#/components/responses/Conflict'
        "429":
          $ref: '#/components/responses/TooManyRequests'

components:
  schemas:
    Order:
      type: object
      required: [id, status, customer_id, items, created_at]
      properties:
        id:
          type: string
          description: Opaque order identifier
          example: "ord_3k9x2m"
          readOnly: true
        status:
          type: string
          enum: [pending, confirmed, shipped, delivered, cancelled]
        customer_id:
          type: string
        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItem'
        created_at:
          type: string
          format: date-time
          readOnly: true
        _links:                            # HATEOAS
          $ref: '#/components/schemas/OrderLinks'

    Problem:                               # RFC 7807
      type: object
      required: [type, title, status]
      properties:
        type:
          type: string
          format: uri
          example: "https://api.company.com/errors/insufficient-stock"
        title:
          type: string
          example: "Insufficient Stock"
        status:
          type: integer
          example: 422
        detail:
          type: string
          example: "SKU PROD-001 has only 1 unit available, 3 requested"
        instance:
          type: string
          format: uri
          description: URI specific to this occurrence
        trace_id:
          type: string
          description: Correlates to distributed trace

  responses:
    TooManyRequests:
      description: Rate limit exceeded
      headers:
        Retry-After:
          schema:
            type: integer
          description: Seconds to wait before retrying
        X-RateLimit-Limit:
          schema:
            type: integer
        X-RateLimit-Remaining:
          schema:
            type: integer
        X-RateLimit-Reset:
          schema:
            type: integer
          description: Unix timestamp when limit resets
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'

  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

---

## REST Principles

### Resource Modeling

```
  Resources are NOUNS, not verbs

  BAD:
    POST /createOrder
    POST /cancelOrder/{id}
    GET  /getUserOrders/{userId}

  GOOD:
    POST   /orders              → create order
    DELETE /orders/{id}         → cancel order (or PATCH status)
    GET    /users/{id}/orders   → list user's orders

  Resource hierarchy:
    /customers/{id}                 → single customer
    /customers/{id}/orders          → orders belonging to customer
    /customers/{id}/orders/{ordId}  → specific order

  Sub-resources vs query params:
    /orders/{id}/items          → owned collection (sub-resource)
    /orders?status=pending      → filtered view (query param)
    /orders?customer_id=xxx     → cross-resource lookup (query param)
```

### Uniform Interface

```
  HTTP method semantics:
  ┌────────┬──────────┬────────────┬─────────────────────────────────────────┐
  │ Method │ Safe?    │ Idempotent?│ Semantics                               │
  ├────────┼──────────┼────────────┼─────────────────────────────────────────┤
  │ GET    │ Yes      │ Yes        │ Retrieve resource; no side effects       │
  │ HEAD   │ Yes      │ Yes        │ GET without body; check existence/ETag   │
  │ POST   │ No       │ No*        │ Create resource or trigger action        │
  │ PUT    │ No       │ Yes        │ Replace entire resource                  │
  │ PATCH  │ No       │ No*        │ Partial update (JSON Merge Patch/Patch)  │
  │ DELETE │ No       │ Yes        │ Delete resource                          │
  │ OPTIONS│ Yes      │ Yes        │ CORS preflight; discover methods         │
  └────────┴──────────┴────────────┴─────────────────────────────────────────┘

  * POST/PATCH can be made idempotent with Idempotency-Key header

  PUT vs PATCH:
    PUT:   client sends complete representation, server replaces
    PATCH: client sends only changed fields
    
    PUT /orders/123
    { "status": "confirmed", "customer_id": "...", "items": [...] }  ← full

    PATCH /orders/123
    { "status": "confirmed" }  ← partial (JSON Merge Patch, RFC 7396)
```

### HATEOAS — When It Adds Value

```
  HATEOAS: responses include links to related actions

  Order response with HATEOAS:
  {
    "id": "ord_3k9x2m",
    "status": "confirmed",
    "_links": {
      "self":   { "href": "/orders/ord_3k9x2m" },
      "cancel": { "href": "/orders/ord_3k9x2m/cancel", "method": "POST" },
      "items":  { "href": "/orders/ord_3k9x2m/items" },
      "customer": { "href": "/customers/cust_8x4k2m" }
    }
  }

  When it adds value:
    → Workflows with dynamic state transitions (cancel only if pending)
    → Hypermedia-driven clients that discover capabilities at runtime
    → Reducing hardcoded URLs in clients

  When it's not worth it:
    → Simple CRUD APIs where all actions are always available
    → Internal service-to-service APIs with typed clients
    → Mobile apps with hardcoded URL patterns anyway
    → Teams that won't maintain the link metadata

  Pragmatic middle ground: include _links for the common workflow transitions,
  skip deep hypermedia for simple attribute resources.
```

---

## Versioning Strategies

```
  ┌──────────────────┬──────────────────┬──────────────────────────────────┐
  │ Strategy         │ Example          │ Tradeoffs                        │
  ├──────────────────┼──────────────────┼──────────────────────────────────┤
  │ URL Path         │ /v1/orders       │ + Obvious, cacheable             │
  │ (most common)    │ /v2/orders       │ + Easy to route at proxy level   │
  │                  │                  │ - "Version" in URI violates REST  │
  │                  │                  │ - Must update all hardcoded URLs  │
  ├──────────────────┼──────────────────┼──────────────────────────────────┤
  │ Accept Header    │ Accept:          │ + REST purist approach           │
  │                  │  application/    │ - Harder to test in browser      │
  │                  │  vnd.company     │ - Breaks HTTP caching            │
  │                  │  .v2+json        │ - Invisible in URLs/logs         │
  ├──────────────────┼──────────────────┼──────────────────────────────────┤
  │ Custom Header    │ X-API-Version: 2 │ + URL stable across versions     │
  │                  │                  │ - Non-standard, magic header     │
  │                  │                  │ - Breaks CDN caching (Vary:)     │
  └──────────────────┴──────────────────┴──────────────────────────────────┘

  Recommendation: URL path versioning (/v1/, /v2/) for public APIs.
                  Custom header only for internal APIs with typed clients.

  Versioning granularity:
    Version the API, not individual endpoints.
    /v2/orders implies the entire API is v2.
    Avoid /v1/orders and /v2/customers mixed in same API.

  Version lifecycle:
    v1 → current stable
    v2 → current stable (deprecate v1)
    
    Sunset header on deprecated versions:
      Sunset: Sat, 01 Jan 2026 00:00:00 GMT
      Deprecation: true
      Link: <https://api.company.com/v2/orders>; rel="successor-version"
```

---

## Breaking vs Non-Breaking Changes

```
  SAFE TO ADD (non-breaking):
  ✓ New optional request fields (clients that don't send them still work)
  ✓ New response fields (clients that don't read them still work)
  ✓ New endpoints / resources
  ✓ New optional query parameters
  ✓ New enum values in responses (warn: some clients fail on unknown enums)
  ✓ Making a required field optional
  ✓ New HTTP methods on existing resources
  ✓ New error codes (clients should handle unknown codes gracefully)

  BREAKING CHANGES (require new version):
  ✗ Renaming a field (old name disappears)
  ✗ Changing a field's type (string → integer, object → array)
  ✗ Removing a field from response
  ✗ Removing an endpoint
  ✗ Changing endpoint semantics (GET with side effects, etc.)
  ✗ Making an optional field required
  ✗ Removing enum values from request fields
  ✗ Changing authentication scheme
  ✗ Changing URL structure
  ✗ Reducing rate limits

  Gray area (technically non-breaking, practically may break):
  ~ Adding new required auth scopes
  ~ Changing default sort order
  ~ New enum values in request fields (strict validators may reject)
  ~ Reducing maximum field length
  ~ Changing error message format (if clients parse messages)

  Detection: contract testing (Pact) catches breaking changes in CI.
```

---

## gRPC and Protobuf

### Schema Design

```protobuf
// order_service.proto
syntax = "proto3";
package order.v1;

option java_package = "com.company.order.v1";
option java_multiple_files = true;
option go_package = "github.com/company/order/v1;orderv1";

// Service definition
service OrderService {
  // Unary: single request → single response
  rpc CreateOrder(CreateOrderRequest) returns (Order);

  // Server streaming: single request → stream of responses
  rpc WatchOrderStatus(WatchOrderRequest) returns (stream OrderStatusEvent);

  // Client streaming: stream of requests → single response
  rpc BulkCreateOrders(stream CreateOrderRequest) returns (BulkCreateResponse);

  // Bidirectional streaming: stream in, stream out
  rpc ProcessOrderFeed(stream OrderFeedItem) returns (stream OrderFeedResult);
}

message Order {
  string id = 1;
  OrderStatus status = 2;
  string customer_id = 3;
  repeated OrderItem items = 4;
  google.protobuf.Timestamp created_at = 5;
  // Field 6 is RESERVED — was removed in v1.1
  // Never reuse field numbers — wire format depends on them
  reserved 6;
  reserved "shipping_address_v1";  // also reserve names
  
  // Added in v1.2 — backward compatible (optional in proto3)
  string tracking_number = 7;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;  // always include 0 for unset detection
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  // Idempotency key — clients must set this for safe retries
  string idempotency_key = 3;
}

// Error details — use google.rpc.Status + details
// Status codes: NOT_FOUND, INVALID_ARGUMENT, ALREADY_EXISTS,
//               RESOURCE_EXHAUSTED (rate limit), UNAVAILABLE (retry)
```

### Protobuf Schema Evolution Rules

```
  SAFE:
  ✓ Add new fields (new field numbers)
  ✓ Rename fields (wire format uses numbers, not names)
  ✓ Change field from required → optional
  ✓ Add new enum values

  UNSAFE:
  ✗ Reuse field numbers (existing messages will misinterpret bytes)
  ✗ Change field types in incompatible ways
  ✗ Rename package (breaks generated code)
  ✗ Remove fields without reserving the number

  Always use `reserved` when removing fields or enum values:
    reserved 6, 7, 15 to 20;
    reserved "old_field_name";
```

---

## GraphQL Schema Design

### Schema First

```graphql
# schema.graphql

type Query {
  order(id: ID!): Order
  orders(
    customerId: ID
    status: OrderStatus
    first: Int = 20
    after: String    # cursor-based pagination
  ): OrderConnection!
}

type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  cancelOrder(id: ID!, reason: String): CancelOrderPayload!
}

type Subscription {
  orderStatusChanged(orderId: ID!): OrderStatusEvent!
}

type Order {
  id: ID!
  status: OrderStatus!
  customer: Customer!      # resolved by CustomerResolver
  items: [OrderItem!]!
  createdAt: DateTime!
  
  # Computed field — not stored, derived from items
  totalAmount: Money!
}

# Connection pattern for pagination (Relay spec)
type OrderConnection {
  edges: [OrderEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type OrderEdge {
  node: Order!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# Mutation payloads: always return the affected resource + errors
type CreateOrderPayload {
  order: Order
  errors: [UserError!]!
}

type UserError {
  field: [String!]    # which field caused the error
  message: String!
  code: String!
}

enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}
```

### N+1 Problem and DataLoader

```
  The N+1 Problem:
    Query: orders { customer { name } }
    Naive resolver:
      1 query → fetch 20 orders
      20 queries → fetch customer for each order (N+1 = 21 total)

    With DataLoader:
      1 query → fetch 20 orders
      1 query → fetch all 20 customers in ONE batched query (2 total)

  DataLoader pattern (Python/Strawberry):

    from strawberry.dataloader import DataLoader

    async def load_customers(customer_ids: list[str]) -> list[Customer]:
        # One DB query for all IDs
        customers = await db.get_customers_by_ids(customer_ids)
        # Return in same order as input IDs
        customer_map = {c.id: c for c in customers}
        return [customer_map.get(cid) for cid in customer_ids]

    # Register per-request (never share across requests — defeats batching)
    customer_loader = DataLoader(load_fn=load_customers)

    @strawberry.type
    class Order:
        customer_id: str

        @strawberry.field
        async def customer(self) -> Customer:
            return await customer_loader.load(self.customer_id)
            # DataLoader batches all calls in same execution tick

  Rule: Every field that resolves by ID → use DataLoader.
        Never fetch in resolver directly if there's a collection parent.
```

### Persisted Queries

```
  Problem: GraphQL queries sent as strings → variable size, introspection risk
  Solution: Persisted queries — send query hash, server resolves to full query

    Client sends:  { "id": "sha256:abc123" }
    Server returns: executes the pre-registered query

  Benefits:
    - Smaller payloads (hash vs full query string)
    - Can disable arbitrary query execution in production
    - Better caching (CDN-cacheable GET requests)
    - Security: unknown queries rejected

  Automatic Persisted Queries (APQ):
    1. Client sends hash only
    2. Server: "PersistedQueryNotFound"
    3. Client resends hash + full query
    4. Server stores, executes, returns result
    5. Future: hash only needed
```

---

## Error Responses: RFC 7807 Problem Details

```
  Content-Type: application/problem+json

  {
    "type": "https://api.company.com/errors/insufficient-stock",
    "title": "Insufficient Stock",          ← human-readable, stable
    "status": 422,
    "detail": "SKU PROD-001 has 1 unit available, but 3 were requested.",
    "instance": "/orders/requests/req_9x3k",  ← this specific occurrence
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    
    // Extension members (your domain-specific fields)
    "sku": "PROD-001",
    "requested_quantity": 3,
    "available_quantity": 1,
    "back_in_stock_date": "2024-02-15"
  }

  HTTP Status Code Mapping:
  ┌──────┬──────────────────────────┬──────────────────────────────────────┐
  │ Code │ Meaning                  │ When to use                          │
  ├──────┼──────────────────────────┼──────────────────────────────────────┤
  │ 400  │ Bad Request              │ Invalid syntax, missing required field│
  │ 401  │ Unauthorized             │ Missing or invalid authentication     │
  │ 403  │ Forbidden                │ Authenticated but not authorized      │
  │ 404  │ Not Found                │ Resource doesn't exist               │
  │ 409  │ Conflict                 │ Duplicate create (idempotency key)   │
  │ 410  │ Gone                     │ Resource existed and was deleted      │
  │ 422  │ Unprocessable Entity     │ Valid syntax, invalid business logic  │
  │ 429  │ Too Many Requests        │ Rate limit exceeded                  │
  │ 500  │ Internal Server Error    │ Unexpected server error              │
  │ 502  │ Bad Gateway              │ Upstream service error               │
  │ 503  │ Service Unavailable      │ Overloaded or down, retry later      │
  └──────┴──────────────────────────┴──────────────────────────────────────┘

  Machine-readable: `type` URI identifies error class (parse in code)
  Human-readable:   `detail` explains specific occurrence (show in UI/logs)
  Never: put implementation details, stack traces in error responses
```

---

## Pagination

### Cursor-Based vs Offset-Based

```
  OFFSET PAGINATION:
    GET /orders?offset=100&limit=20

    SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100;

    Pros: simple, supports jumping to arbitrary page
    Cons:
      - Slow on large tables (DB scans and discards rows)
      - Inconsistent: inserts/deletes shift items across pages
      - Page 5 might have duplicates if items added to page 1

  CURSOR PAGINATION (preferred):
    GET /orders?after=cursor_opaque_value&limit=20

    SELECT * FROM orders WHERE created_at < :cursor ORDER BY created_at DESC LIMIT 20;

    Pros:
      - O(log n) with index — fast at any dataset size
      - Stable: inserts/deletes don't corrupt pages
      - Works with real-time data streams
    Cons:
      - Cannot jump to arbitrary page ("go to page 47")
      - Cursor is opaque (can't decode position)

  Cursor encoding:
    cursor = base64(json({"created_at": "2024-01-15T10:23:41Z", "id": "ord_3k"}))
    → Never expose raw SQL values in cursor
    → Cursor should be treated as opaque by clients
    → Sign cursor to prevent tampering: hmac(cursor_payload, secret)

  Response format (Relay-compatible):
  {
    "data": [...],
    "pagination": {
      "has_next_page": true,
      "has_previous_page": false,
      "start_cursor": "eyJpZCI6Im9yZF8xIn0=",
      "end_cursor": "eyJpZCI6Im9yZF8yMCJ9"
    }
  }

  Rule: Use cursor pagination for any collection > 1000 records
        or any real-time feed. Offset only for small, static datasets.
```

---

## Rate Limiting

```
  Response headers — inform clients before they hit limits:
    X-RateLimit-Limit: 1000        ← requests allowed per window
    X-RateLimit-Remaining: 743     ← requests left in current window
    X-RateLimit-Reset: 1705316400  ← UTC epoch when window resets

  When limit is exceeded → 429 Too Many Requests:
    HTTP/1.1 429 Too Many Requests
    Retry-After: 47                ← seconds (or HTTP-date)
    X-RateLimit-Limit: 1000
    X-RateLimit-Remaining: 0
    Content-Type: application/problem+json

    {
      "type": "https://api.company.com/errors/rate-limit-exceeded",
      "title": "Rate Limit Exceeded",
      "status": 429,
      "detail": "You have exceeded 1000 requests per minute.",
      "retry_after_seconds": 47
    }

  Rate limit strategies:
    Fixed window:  count resets at :00 of each minute (burst at boundary)
    Sliding window: count over last N seconds (smoother)
    Token bucket:  refill at constant rate, burst up to bucket size
    Leaky bucket:  process at constant rate, queue excess

  Different limits for different consumers:
    Free tier:       100 req/min
    Pro tier:        1000 req/min
    Enterprise tier: 10000 req/min
    Internal services: unlimited (by IP, not API key)
```

---

## API Security

### Authentication

```
  Bearer JWT:
    Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...

    Validate:
      1. Signature (verify with public key)
      2. Expiry (exp claim)
      3. Audience (aud claim matches your service)
      4. Issuer (iss claim matches your auth server)
      5. Not before (nbf claim)

    Scopes in JWT:
      "scope": "orders:read orders:write customers:read"
      Check scope per endpoint, not just per token existence

  API Key:
    X-API-Key: ak_live_abc123...
    
    Store: hash(api_key) in DB — never store raw key
    Prefix with env: ak_live_ vs ak_test_ (visible in logs/code)
    Rotate: support multiple active keys per account during rotation
```

### Input Validation

```
  Validate at API layer, before any business logic:

  Java/Spring (Bean Validation + OpenAPI):
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(
        @Valid @RequestBody CreateOrderRequest request) { ... }

    public class CreateOrderRequest {
        @NotBlank
        @Size(max = 64)
        private String customerId;

        @NotEmpty
        @Size(max = 100)
        @Valid
        private List<OrderItem> items;
    }

  Python/FastAPI (Pydantic):
    class CreateOrderRequest(BaseModel):
        customer_id: str = Field(..., max_length=64)
        items: list[OrderItem] = Field(..., min_length=1, max_length=100)

        model_config = ConfigDict(extra="forbid")  # reject unknown fields

  Rules:
    → Reject unknown fields (don't silently ignore)
    → Validate lengths before processing
    → Sanitize string inputs (no raw SQL, no template injection)
    → Validate business constraints (item quantity > 0)
    → Return 400 with field-level error detail
```

---

## Contract Testing with Pact

```
  Consumer-driven contract testing:
    Consumer defines expectations → Provider verifies it can meet them
    Catches breaking changes before they reach staging/production

  ┌─────────────────────────────────────────────────────────────────────┐
  │  Consumer Test                  Provider Test                       │
  │                                                                     │
  │  1. Consumer writes test:       3. Provider fetches contracts       │
  │     "when I POST /orders           from Pact Broker                 │
  │      with {...}, I expect          ("What do consumers expect?")    │
  │      201 with {id, status}"                                         │
  │                                 4. Provider runs against            │
  │  2. Pact generates contract:       real implementation:             │
  │     interaction.json               "Can I fulfill this?"           │
  │     → published to Pact Broker                                      │
  │                                 5. If contract fails:               │
  │                                    CI fails on provider side        │
  └─────────────────────────────────────────────────────────────────────┘

  Python consumer test:
    def test_create_order(pact):
        pact.given("inventory available for SKU PROD-001") \
            .upon_receiving("a request to create an order") \
            .with_request("POST", "/v1/orders",
                body={"customer_id": "cust_1", "items": [...]}) \
            .will_respond_with(201, body={
                "id": Term(r"ord_[a-z0-9]+", "ord_abc"),
                "status": "pending"
            })

        with pact:
            result = order_client.create_order(customer_id="cust_1", ...)
            assert result.status == "pending"

  Java provider verification:
    @Provider("order-service")
    @PactBroker(url = "https://pact-broker.company.com")
    class OrderServiceProviderTest {
        @TestTarget
        public final MockMvcTarget target = new MockMvcTarget();

        @State("inventory available for SKU PROD-001")
        public void inventoryAvailable() {
            when(inventoryService.check("PROD-001")).thenReturn(available(3));
        }
    }

  Run in CI:
    Consumer CI: generate + publish pacts on every PR
    Provider CI: verify pacts on every PR + before deploy
    Can-I-Deploy: check Pact Broker before merging/deploying
```

---

## Idempotency

```
  Problem: network is unreliable. Client retries = duplicate operations.
  Solution: idempotency keys for non-idempotent operations.

  Client sends:
    POST /v1/orders
    Idempotency-Key: client-generated-uuid-v4

  Server behavior:
    1. First request: process + store (idempotency_key, response) in DB
    2. Duplicate request (same key): return stored response immediately
    3. Key expires after 24 hours (configurable)

  Response on duplicate:
    HTTP 200 (not 201) to signal "already processed"
    — or — HTTP 201 with same body as original
    Include header: Idempotency-Replayed: true

  Implementation:
    Store: idempotency_key, request_hash, response_status, response_body,
           expires_at, created_at
    Lock: optimistic or pessimistic while processing
    Validate: request body matches original (same key, different body = 422)

  PUT is naturally idempotent — no key needed.
  POST /orders → needs key.
  POST /orders/{id}/cancel → needs key (or model as PATCH to status).

  Stripe, Braintree, Twilio all use this pattern for payment/message APIs.
```

---

## API Documentation

```
  Generated from OpenAPI spec (never manually written):
    Swagger UI:      self-hosted, interactive try-it
    Redoc:           clean, read-focused, great for external docs
    Stoplight:       API design platform with hosted docs
    Scalar:          modern, beautiful, open source

  Changelog (CHANGELOG.md, committed with each release):
    ## [1.4.0] - 2024-01-15
    ### Added
    - `tracking_number` field on Order response
    - `GET /orders/{id}/timeline` endpoint for status history

    ## [1.3.0] - 2024-01-08
    ### Deprecated
    - `GET /v1/orders/search` — use `GET /v1/orders?q=` instead
    - Removal planned for 2024-07-01

  Deprecation headers:
    Deprecation: true
    Sunset: Mon, 01 Jul 2024 00:00:00 GMT
    Link: <https://docs.company.com/migration/v1-to-v2>; rel="deprecation"
```

---

## Practical Checklist

### Design Phase

- [ ] API-first: OpenAPI spec committed before implementation starts
- [ ] Resources are nouns; HTTP methods carry the verb semantics
- [ ] URL versioning: /v1/ prefix on all endpoints
- [ ] Defined breaking vs non-breaking change policy
- [ ] Pagination strategy chosen (cursor for large/realtime data)
- [ ] Idempotency-Key supported for all POST operations
- [ ] Error format: RFC 7807 Problem Details with machine-readable `type`
- [ ] Rate limit headers defined (X-RateLimit-*, Retry-After)

### Implementation Phase

- [ ] Input validation at API layer (before business logic)
- [ ] No PII in URLs (in logs), use opaque IDs
- [ ] Auth scope checked per endpoint
- [ ] DataLoader pattern for GraphQL (no N+1)
- [ ] Protobuf reserved field numbers when removing fields

### Testing Phase

- [ ] Pact consumer tests for all provider integrations
- [ ] Pact provider verification in CI
- [ ] Can-I-Deploy gate before deployment
- [ ] Contract versioned alongside code

### Anti-Patterns

- [ ] NEVER: versioning in individual field names (name_v2)
- [ ] NEVER: actions as resource names (/createOrder)
- [ ] NEVER: 200 OK for errors (with error in body)
- [ ] NEVER: stack traces in error responses
- [ ] NEVER: high-cardinality route params in metrics labels
- [ ] NEVER: breaking changes without version bump
```
