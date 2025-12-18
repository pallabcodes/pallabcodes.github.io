+++
title = "API Design: REST vs GraphQL vs gRPC"
date = 2024-12-10
description = "Choosing the right API paradigm for your use case."
[taxonomies]
tags = ["api-design", "rest", "graphql", "grpc"]
+++

Choosing an API style is one of the most impactful architectural decisions you'll make. Let's break down the trade-offs.

<!-- more -->

## The Three Contenders

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Format | JSON | JSON | Protobuf |
| Contract | Loose | Schema | Strict |
| Transport | HTTP | HTTP | HTTP/2 |
| Best for | CRUD APIs | Flexible queries | Microservices |

## REST: The Reliable Workhorse

REST is ubiquitous and well-understood:

```bash
# Create
POST /users
{"name": "Alice", "email": "alice@example.com"}

# Read
GET /users/123

# Update
PUT /users/123
{"name": "Alice Smith"}

# Delete
DELETE /users/123
```

### When to Use REST

✅ Public APIs (maximum compatibility)
✅ Simple CRUD operations
✅ Caching-friendly resources
✅ Team is familiar with it

### REST Pitfalls

❌ Over-fetching: Getting more data than needed
❌ Under-fetching: Multiple requests for related data
❌ Versioning: `/v1/users` vs `/v2/users`

## GraphQL: The Flexible Query Language

GraphQL lets clients specify exactly what they need:

```graphql
query {
  user(id: "123") {
    name
    email
    posts(limit: 5) {
      title
      publishedAt
    }
  }
}
```

One request, exactly the data you need.

### When to Use GraphQL

✅ Mobile apps (bandwidth-sensitive)
✅ Complex, nested data structures
✅ Rapidly evolving frontends
✅ Multiple clients with different needs

### GraphQL Pitfalls

❌ Caching is harder (POST for everything)
❌ N+1 query problems without DataLoader
❌ Complex queries can be expensive

```python
# GraphQL resolver with N+1 problem
def resolve_posts(user):
    # This runs for EACH user - N+1!
    return db.query("SELECT * FROM posts WHERE user_id = ?", user.id)

# Solution: DataLoader
loader = DataLoader(batch_load_fn=load_posts_for_users)
```

## gRPC: The Performance Beast

gRPC uses Protocol Buffers and HTTP/2 for maximum efficiency:

```protobuf
// user.proto
syntax = "proto3";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc StreamUsers(StreamRequest) returns (stream User);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}
```

```go
// Client code
user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: "123"})
```

### When to Use gRPC

✅ Internal microservices
✅ High-performance requirements
✅ Streaming data (bidirectional)
✅ Polyglot environments (auto-generated clients)

### gRPC Pitfalls

❌ Browser support requires proxy
❌ Not human-readable (debugging harder)
❌ Learning curve for protobufs

## Decision Framework

```
┌─────────────────────────────────────────────────────┐
│                  API Decision Tree                   │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Public API?                                         │
│     └── Yes → REST (widely supported)               │
│     └── No ↓                                        │
│                                                      │
│  Need streaming?                                     │
│     └── Yes → gRPC                                  │
│     └── No ↓                                        │
│                                                      │
│  Complex, nested queries?                           │
│     └── Yes → GraphQL                               │
│     └── No ↓                                        │
│                                                      │
│  High performance critical?                         │
│     └── Yes → gRPC                                  │
│     └── No → REST                                   │
│                                                      │
└─────────────────────────────────────────────────────┘
```

## Hybrid Approaches

Many companies use multiple paradigms:

- **REST** for public APIs and webhooks
- **GraphQL** for frontend/mobile teams
- **gRPC** between internal services

```
┌─────────┐      GraphQL      ┌─────────────┐
│ Mobile  │──────────────────▶│   API GW    │
└─────────┘                   │             │
                              │  (BFF/GQL)  │
┌─────────┐      REST         │             │
│ Partners│──────────────────▶│             │
└─────────┘                   └──────┬──────┘
                                     │ gRPC
                    ┌────────────────┼────────────────┐
                    │                │                │
              ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
              │ User Svc  │   │ Order Svc │   │ Auth Svc  │
              └───────────┘   └───────────┘   └───────────┘
```

## Key Takeaways

1. **REST** - Default choice for public APIs
2. **GraphQL** - When clients need flexibility
3. **gRPC** - For internal, high-performance services
4. **Combine them** - Use the right tool for each context
5. **Start simple** - You can always add complexity later

The best API is the one your team can build and maintain effectively.
