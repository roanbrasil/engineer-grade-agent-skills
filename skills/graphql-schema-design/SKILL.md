---
name: graphql-schema-design
description: Design production-quality GraphQL schemas — type system, mutation patterns, pagination, N+1 DataLoader, federation, security, and schema evolution
---

# GraphQL Schema Design — Expert Reference

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                     GraphQL Request Lifecycle                     │
│                                                                    │
│  Client                GraphQL Server                             │
│  ┌──────┐  POST /graphql  ┌────────────┐                         │
│  │      │────────────────►│  Parse &   │                         │
│  │      │  { query, vars }│  Validate  │◄── Schema (SDL)         │
│  │      │                 └─────┬──────┘                         │
│  │      │                       │                                 │
│  │      │                 ┌─────▼──────┐                         │
│  │      │                 │  Execute   │                          │
│  │      │                 │  (resolve  │                          │
│  │      │                 │   each     │                          │
│  │      │                 │   field)   │                          │
│  │      │                 └─────┬──────┘                         │
│  │      │   { data, errors }    │                                 │
│  │      │◄──────────────────────┘                                 │
│  └──────┘                                                         │
│                                                                    │
│  Schema is the contract — define it FIRST, then implement         │
└──────────────────────────────────────────────────────────────────┘
```

## Schema-First Principle

Define the schema before writing a single resolver. The schema IS the API contract. Clients and servers can be developed in parallel against the schema. Use SDL (Schema Definition Language) committed to source control.

```graphql
# schema.graphql — the contract
# Write this before any resolver code

type Query {
  order(id: ID!): Order
  orders(filter: OrderFilter, sort: OrderSort, pagination: PaginationInput): OrderConnection!
}

type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  cancelOrder(input: CancelOrderInput!): CancelOrderPayload!
}

type Subscription {
  orderStatusChanged(orderId: ID!): OrderStatusEvent!
}
```

---

## Type System

### Scalar Types

```graphql
# Built-in scalars
String    # UTF-8 string
Int       # 32-bit signed integer
Float     # 64-bit double
Boolean   # true / false
ID        # opaque identifier (serialized as String)

# Custom scalars (define in schema + implement serialization)
scalar DateTime   # ISO 8601: "2024-01-15T10:30:00Z"
scalar UUID       # "550e8400-e29b-41d4-a716-446655440000"
scalar Money      # { amount: Int, currency: String }
scalar EmailAddress
```

### Non-Null Discipline — nullable by default

```graphql
# RULE: fields are nullable by default
# Add ! only when the field is GUARANTEED to never be null in ANY scenario

type Order {
  id:         ID!          # always present (primary key)
  status:     OrderStatus! # always has a status
  customer:   Customer     # nullable — customer might be deleted
  cancelledAt: DateTime    # nullable — only set if cancelled
  total:      Money!       # always present
  lineItems:  [LineItem!]! # list always exists, items always valid
}

# [LineItem!]! breakdown:
#   outer !  = the list itself is never null (but may be empty)
#   inner !  = each element in the list is never null
#   [LineItem]  = list can be null AND elements can be null (rare, avoid)
```

### Enums

```graphql
enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}

enum SortDirection {
  ASC
  DESC
}
```

### Interfaces and Unions

```graphql
# Interface — shared fields, specific implementations
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Order implements Node & Timestamped {
  id:        ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  status:    OrderStatus!
}

# Union — no shared fields, client switches on __typename
union SearchResult = Order | Product | Customer

type Query {
  search(query: String!): [SearchResult!]!
}

# Client query with fragments:
# search(query: "laptop") {
#   ... on Order   { id status }
#   ... on Product { id name price }
#   ... on Customer { id email }
# }
```

---

## Query Design

### Field Naming

```graphql
# GOOD: descriptive camelCase
type Customer {
  id:           ID!
  emailAddress: String!
  totalOrders:  Int!
  loyaltyTier:  LoyaltyTier!
}

# BAD: generic names that tell clients nothing
type Customer {
  id:     ID!
  data:   String!   # what data?
  result: Int!      # result of what?
  info:   String!   # which info?
}
```

### Arguments and Input Types

```graphql
# Use input types for complex arguments — never long flat argument lists
input OrderFilter {
  status:    [OrderStatus!]
  customerId: ID
  dateRange:  DateRangeInput
  minTotal:   Money
  maxTotal:   Money
}

input DateRangeInput {
  from: DateTime!
  to:   DateTime!
}

input OrderSort {
  field:     OrderSortField!
  direction: SortDirection!
}

enum OrderSortField {
  CREATED_AT
  TOTAL
  STATUS
}

type Query {
  # GOOD: structured types for filter/sort/pagination
  orders(
    filter:     OrderFilter
    sort:       OrderSort
    pagination: PaginationInput
  ): OrderConnection!

  # BAD: flat argument list — hard to extend, no grouping
  # orders(status: OrderStatus, customerId: ID, page: Int, limit: Int): [Order]
}
```

---

## Mutation Design

### Naming Convention

```graphql
# GOOD: verb + noun, action-oriented
type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  cancelOrder(input: CancelOrderInput!): CancelOrderPayload!
  updateOrderShipping(input: UpdateOrderShippingInput!): UpdateOrderShippingPayload!
  addOrderLineItem(input: AddOrderLineItemInput!): AddOrderLineItemPayload!
}

# BAD: noun-first, unclear intent
type Mutation {
  order(input: OrderInput!): Order          # create? update? unclear
  orderMutation(type: String!): OrderResult # stringly typed "type"
}
```

### Input + Payload Pattern

```graphql
# Every mutation: one input type + one payload type
input CreateOrderInput {
  customerId: ID!
  lineItems:  [CreateLineItemInput!]!
  shippingAddress: AddressInput!
}

input CreateLineItemInput {
  productId: ID!
  quantity:  Int!
}

# Payload wraps the result AND errors
type CreateOrderPayload {
  order:  Order            # null on failure
  errors: [UserError!]!    # empty on success; user errors as DATA, not exceptions
}

type UserError {
  field:   String   # which input field caused the error (null = general error)
  message: String!
}

# Usage pattern in client:
# mutation {
#   createOrder(input: { ... }) {
#     order { id status }
#     errors { field message }
#   }
# }
```

### Error Strategy

```
User errors (validation, business rules) → in payload { errors: [UserError] }
  - "Email already taken"
  - "Quantity exceeds stock"
  - "Cannot cancel a delivered order"

System errors (bugs, infra failures) → HTTP 500 + top-level GraphQL error
  - Database connection failure
  - Unhandled NullPointerException
  - Service unavailable
```

---

## Pagination — Relay Cursor Connection Spec

```graphql
# The standard: always use cursor-based pagination
type OrderConnection {
  edges:    [OrderEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!           # optional but useful
}

type OrderEdge {
  node:   Order!
  cursor: String!   # opaque; base64-encoded position
}

type PageInfo {
  hasNextPage:     Boolean!
  hasPreviousPage: Boolean!
  startCursor:     String
  endCursor:       String
}

# Input for forward pagination
input PaginationInput {
  first:  Int     # forward: take N after cursor
  after:  String  # forward: start after this cursor
  last:   Int     # backward: take N before cursor
  before: String  # backward: start before this cursor
}

type Query {
  orders(pagination: PaginationInput): OrderConnection!
}
```

```graphql
# Client query — infinite scroll pattern
query GetOrders($after: String) {
  orders(pagination: { first: 20, after: $after }) {
    edges {
      node { id status total { amount currency } }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

```
Why cursors beat offsets:
  Offset pagination:  page 3 of results → OFFSET 40 LIMIT 20
    Problem: if 5 new orders added while browsing, page 3 duplicates rows from page 2

  Cursor pagination:  "give me 20 orders after cursor ABC"
    Safe under concurrent writes — cursor anchors to a specific row position
    Works for infinite scroll, real-time feeds
```

---

## Subscriptions

```graphql
type Subscription {
  # GOOD: specific type with all needed fields
  orderStatusChanged(orderId: ID!): OrderStatusEvent!
}

type OrderStatusEvent {
  orderId:   ID!
  oldStatus: OrderStatus!
  newStatus: OrderStatus!
  timestamp: DateTime!
  actor:     String        # who triggered the change
}

# BAD: generic "updated" event that pushes entire object
# orderUpdated(orderId: ID!): Order
# Problem: re-fetches entire object; leaks all fields; hard to diff
```

```
When to use subscriptions:
  YES: real-time status updates (< 1 event/sec per user)
  YES: collaborative editing, live dashboards
  NO:  polling on a timer (use repeated queries)
  NO:  high-frequency data (use WebSocket directly)

Security:
  - Authenticate at subscription setup (not per-message)
  - Authorize per subscription topic (user can only subscribe to their own orders)
  - Always define a specific event type — never subscribe to "all changes"
```

---

## N+1 Problem and DataLoader

### The Problem

```graphql
# Query asks for 100 orders, each with customer name
query {
  orders(pagination: { first: 100 }) {
    edges {
      node {
        id
        customer { name email }  # N+1: fires 100 separate customer queries
      }
    }
  }
}

# Resolver without DataLoader:
#   1 query: SELECT * FROM orders LIMIT 100
#   100 queries: SELECT * FROM customers WHERE id = ?  (once per order)
#   = 101 queries total
```

### DataLoader Solution

```
DataLoader batching:
  tick 1: order resolver calls loader.load("cust-1")
  tick 1: order resolver calls loader.load("cust-2")
  ...
  tick 1: order resolver calls loader.load("cust-100")
  tick 2: DataLoader BATCHES all 100 keys into ONE call:
           SELECT * FROM customers WHERE id IN (cust-1, cust-2, ..., cust-100)
  = 2 queries total (1 orders + 1 customers)
```

### Java — graphql-java-dataloader

```java
// Define a batch loader function
BatchLoaderWithContext<String, Customer> customerBatchLoader =
    (customerIds, environment) -> {
        return CompletableFuture.supplyAsync(() ->
            customerRepository.findAllByIds(customerIds)  // one DB call
        );
    };

// Register per-request (IMPORTANT: new DataLoader per request, not shared)
DataLoader<String, Customer> customerLoader =
    DataLoader.newDataLoader(customerBatchLoader);

DataLoaderRegistry registry = new DataLoaderRegistry()
    .register("customers", customerLoader);

// In resolver (DataFetcher)
public CompletableFuture<Customer> get(DataFetchingEnvironment env) {
    String customerId = env.getSource().getCustomerId();
    DataLoader<String, Customer> loader = env.getDataLoader("customers");
    return loader.load(customerId);  // batched automatically
}
```

### Python — strawberry-graphql + aiodataloader

```python
from strawberry.dataloader import DataLoader
from typing import List

async def load_customers(customer_ids: List[str]) -> List[Customer]:
    """Batch function — called once with all requested IDs"""
    rows = await db.fetch_all(
        "SELECT * FROM customers WHERE id = ANY($1)", customer_ids
    )
    # IMPORTANT: return in same order as input keys
    id_to_customer = {row["id"]: Customer(**row) for row in rows}
    return [id_to_customer.get(id) for id in customer_ids]

# Per-request DataLoader instance
@strawberry.type
class Query:
    @strawberry.field
    async def orders(self, info) -> List[Order]:
        loader = DataLoader(load_fn=load_customers)
        info.context["customer_loader"] = loader
        return await get_orders()

@strawberry.type
class Order:
    customer_id: str

    @strawberry.field
    async def customer(self, info) -> Customer:
        loader = info.context["customer_loader"]
        return await loader.load(self.customer_id)  # batched automatically
```

---

## Apollo Federation

### When to Use

```
Monolith GraphQL: one schema, one server — fine for small teams
Federation: multiple teams, each owns their subgraph schema

Supergraph:  the unified schema clients query
Subgraph:    one service's portion of the schema
```

```graphql
# orders-subgraph/schema.graphql
extend schema @link(url: "https://specs.apollo.dev/federation/v2.3",
                    import: ["@key", "@external", "@requires"])

type Order @key(fields: "id") {
  id:     ID!
  status: OrderStatus!
  # customer is owned by customers-subgraph — reference only
  customer: Customer
}

# Reference an entity from another subgraph
type Customer @key(fields: "id") @external {
  id: ID!
}
```

```graphql
# customers-subgraph/schema.graphql
type Customer @key(fields: "id") {
  id:    ID!
  name:  String!
  email: String!
  # extend Customer with orders — cross-service field
  orders: [Order!]!
}

type Order @key(fields: "id") @external {
  id: ID!
}
```

```
@key          — defines the entity's primary key for federation
@external     — this field is defined in another subgraph
@requires     — need a field from another subgraph to resolve this one
@provides     — this subgraph can provide fields usually from another
```

---

## Persisted Queries

```
Problem: clients can send arbitrary, potentially expensive queries
Solution: register queries by hash at deployment time; clients send hash only

Flow:
  1. At build time: client tools generate query hash → register {hash: query} with server
  2. At runtime: client sends { extensions: { persistedQuery: { sha256Hash: "abc..." } } }
  3. Server looks up hash → executes known query
  4. Unknown hash → 404 (prevents ad-hoc attacks)

Benefits:
  - Prevents expensive arbitrary queries
  - Smaller request payloads (hash vs. full query text)
  - Built-in query allowlist
```

---

## Security

```graphql
# 1. Depth limit — prevent deeply nested abuse
# query { orders { customer { orders { customer { orders ... } } } } }
# Set max depth: 7-10 levels typical

# 2. Complexity limit — assign cost to fields; reject if total > budget
# simple field = 1, list field = 10, DB call = 5
# Reject queries with total cost > 1000

# 3. Disable introspection in production
#    Introspection reveals your entire schema to attackers
#    Development: ENABLED
#    Production: DISABLED (or restricted to internal IPs)

# 4. Auth at resolver level — never at HTTP middleware level
#    Middleware is too coarse: one request can query multiple types
#    Each resolver checks permissions independently
```

```java
// Java: Disable introspection
GraphQL graphQL = GraphQL.newGraphQL(schema)
    .queryExecutionStrategy(new AsyncExecutionStrategy(
        new SimpleDataFetcherExceptionHandler()))
    .instrumentation(new MaxQueryDepthInstrumentation(10))
    .instrumentation(new MaxQueryComplexityInstrumentation(1000))
    .build();

// Disable introspection in production
NoIntrospectionOperationChecker checker = new NoIntrospectionOperationChecker();
// Register checker on schema validation
```

```python
# Python strawberry: depth limit
from strawberry.extensions import MaxTokensLimiter

schema = strawberry.Schema(
    query=Query,
    extensions=[MaxTokensLimiter(max_token_count=1000)]
)
```

---

## Schema Evolution

```graphql
# SAFE changes:
#   + Add new nullable field (existing clients ignore it)
#   + Add new type
#   + Add new mutation
#   + Add new enum value (clients must handle UNKNOWN)

# UNSAFE changes (breaking):
#   - Remove a field — clients querying it get errors
#   - Rename a field — all client queries break
#   - Change field type — type mismatch in client
#   - Remove an enum value — serialization error

# DEPRECATION workflow before removal:
type Order {
  id:     ID!
  status: OrderStatus!

  # Step 1: add new field, deprecate old
  customerEmail: String @deprecated(reason: "Use customer.email instead")
  customer:      Customer!  # the replacement

  # Step 2: wait for all clients to migrate (monitor usage in Apollo Studio)
  # Step 3: remove the deprecated field after migration confirmed
}
```

---

## Tooling

```bash
# GraphQL Codegen — generate type-safe client/server code
npm install -D @graphql-codegen/cli @graphql-codegen/typescript

# codegen.yml
schema: "./schema.graphql"
documents: "src/**/*.graphql"
generates:
  src/generated/types.ts:
    plugins: [typescript, typescript-operations, typescript-react-apollo]

npm run codegen  # generates TypeScript types + React hooks from schema + queries
```

```
Apollo Studio:
  - Schema registry (version history, breaking change alerts)
  - Field usage analytics (safe to deprecate when usage drops to 0)
  - Query explorer + documentation

Schema linting tools:
  - graphql-inspector: diff schemas, find breaking changes
  - eslint-plugin-graphql: validate query documents against schema
```

---

## Production Checklist

```
Schema design:
  [ ] Schema defined in SDL before resolvers written
  [ ] All mutation args wrapped in input types
  [ ] All mutations return payload type with errors field
  [ ] Nullable by default; ! only where guaranteed
  [ ] Cursor-based pagination on all list fields
  [ ] Enums defined for all categorical values
  [ ] @deprecated used before any field removal

Performance:
  [ ] DataLoader in place for every relationship resolver
  [ ] DataLoader instance is per-request (not singleton)
  [ ] Depth limit configured (7-10)
  [ ] Complexity limit configured
  [ ] N+1 queries profiled in staging (Apollo tracing)

Security:
  [ ] Introspection disabled in production
  [ ] Auth check in every resolver that needs it
  [ ] Subscriptions authenticated at setup time
  [ ] Persisted queries enabled (or allowlist enforced)

Evolution:
  [ ] Deprecated fields annotated with reason
  [ ] Field usage monitored before removal
  [ ] Schema changes reviewed for breaking changes (graphql-inspector in CI)
```

---

## Anti-Patterns

```
WRONG: REST-style "get everything" field
  type Query { getOrderData(id: ID!): JSON }  # untyped blob
RIGHT: typed fields that clients can select from.

WRONG: Mutations without payload type
  type Mutation { createOrder(input: CreateOrderInput!): Order }
  # Returns null on error with no way to surface validation errors
RIGHT: createOrder(input: CreateOrderInput!): CreateOrderPayload!
  # payload { order, errors }

WRONG: Throwing exceptions for user validation errors
  if (!email.valid()) throw new GraphQLException("Invalid email")
RIGHT: Return errors in payload: { errors: [{ field: "email", message: "..." }] }
  System errors (bugs) throw; user errors return in data.

WRONG: Offset pagination
  orders(page: Int, limit: Int): [Order]
  # Breaks under concurrent writes; duplicate rows on infinite scroll
RIGHT: Relay cursor connections.

WRONG: One resolver per type, no DataLoader
  customer { orders { customer { ... } } }  # explodes to N*M queries
RIGHT: DataLoader for every cross-entity relationship.

WRONG: Disable introspection... in development
  Makes it impossible to use GraphQL Playground, Postman, Codegen
RIGHT: Disable only in production.

WRONG: Generic subscription type
  subscription { entityChanged: JSON }
RIGHT: Specific subscription per business event with typed payload.
```
