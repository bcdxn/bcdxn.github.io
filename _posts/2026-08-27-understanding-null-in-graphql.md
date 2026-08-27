---
title: "Understanding Null in GraphQL Schema Design"
date: 2026-08-27 07:30:00 -0400
last_modified_at: 2026-08-27 08:00:00 -0400
categories: [blog]
tags: [graphql, federation, schema-design]
excerpt: "When should a GraphQL field be null versus an empty list? How do non-nullable fields protect invariants while nullable fields enable partial responses. A practical guide to returning null in federated schemas."
image:
  path: /assets/images/banners/2026-08-27.jpg
  alt: header
header:
  og_image: /assets/images/banners/2026-08-27.jpg
---

[![trees](/assets/images/banners/2026-08-27.jpg)](https://unsplash.com/photos/aerial-view-of-trees-during-daytime-UxakjtPzIT0)

I've debugged schemas where a resolver throws in one subgraph and an unrelated field across the graph silently comes back null. The root cause is often a non-nullable field on a shared type; data guarantees becomes a failure mode that impacts other tenants in the supergraph.

Getting null right in GraphQL means answering two questions consistently: when should a field be nullable, and when there's no data, should you return `null` or a zero value like `[]`? The combination shapes how well your schema evolves, how much data survives a partial failure, and how straightforward your resolvers are to reason about.

## Non-Nullable Fields Protect Invariants

Mark a field non-nullable only to protect a business invariant. If the entity is meaningless without that piece of data, nullify the entire parent instead of returning partial data with required fields missing.

**For Example**

A logged-in customer must always have a customer identifier `cID`. If it's missing, the `Customer` object itself is invalid.

```graphql
type Customer {
  cID: ID!
  # ...
}
```

If `cID` cannot be resolved, the whole `Customer` should be nullified rather than returning a Customer with a null identifier.

## Prefer Nullable Fields

In practice, prefer nullable fields. A field should be non-nullable only when protecting a business or domain invariant.

1. **Evolution**: Nullable fields can become non-nullable without breaking changes; the reverse is a breaking change.
2. **Partial responses**: Errors on non-nullable fields bubble up to the nearest nullable ancestor, limiting how much data you can return. One of GraphQL's strengths is its ability to return partial success responses elegantly.
3. **Data inconsistencies**: Older datasets rarely have perfect coverage. For example, it may be desirable for a customer record to have a phone number, but not guaranteed.

> **<i class="fas fa-lightbulb"></i>** In supergraphs, where evolution is critical, prefer nullable fields over non-nullable by default.
> {: .notice--info}

**Example**

A middle name is optional. The customer data remains useful without it and it's unlikely that a consumer would rather have no data returned than simply the customer without the middle name.

```graphql
type Customer {
  cID: ID!
  middleName: String
}
```

## Zero Values vs Null

- **Zero value**: The default for a type when uninitialized, e.g., `[]` for lists.
- **Null**: Explicit absence of a value. GraphQL uses `null` to signal "no data".

### When to return null vs a zero value

#### Scalars

Return `null` when no data is found or an error occurs. Consider `Int` types for example. Is `0` a real amount or missing data? Using a `null` removes the ambiguity and allows consumers to distinguish missing data from a legitimate zero value.

#### Value Objects

A value object bundles tightly coupled properties.

**For Example**

```graphql
type Money {
  amount: String!
  currency: String!
}
```

If either field is missing, the whole object is meaningless. Nullify the entire `Money` response object rather than returning partial data or zero values for its fields. Essentially, we treat value objects as scalars.

#### Lists, Connections, Groupings

Return a zero value when the collection exists but is empty, and no error was thrown.

_Lists and Connections_

If a user has not listed any phone numbers on their account, the capability for the system to return phone numbers is still supported -- the relationship between customers and phone numbers still exists. But the collection is empty. The same can be said of connection objects.

**For Example**

```graphql
# Schema
type Customer {
  id: ID!
  name: String
  phones: [Phone!]
  addresses: AddressConnection
}

type AddressConnection {
  edges: [AddressEdge!]!
  pageInfo: PageInfo!
}

type AddressEdge {
  node: Address!
  cursor: String!
}
```

```json
// Query Response
{
  "data": {
    "customer": {
      "id": "...",
      "name": "John Doe",
      "phones": [],
      "addresses": {
        "edges": [],
        "pageInfo": {
          "hasNextPage": false,
          "endCursor": null
        }
      }
    }
  }
}
```

The list/connection field container is returned simply as an empty list or empty set of edges respectively.

_Groupings_

Groupings aren't a standard GraphQL concept — they're a schema design pattern. A grouping type is a plain object type that exists purely to create hierarchy and bundle related fields under a named namespace (e.g., a `programs` field that contains several loyalty sub-types). The grouping itself has no resolver logic; data is resolved by the fields nested within it. Because of this, groupings should typically not return `null` and instead return a zero value where the nested properties are `null` when no data is resolved.

**For Example**

```graphql
# Schema
type Customer {
  id: ID!
  name: String
  programs: LoyaltyPrograms
}

type LoyaltyPrograms {
  personalCareRewards: PersonalCareProgram
}
```

```json
// Query Response
{
  "data": {
    "customer": {
      "id": "...",
      "name": "John Doe",
      "programs": {
        "personalCareRewards": null
      }
    }
  }
}
```

The grouping container is returned even if a specific program is not initialized.

### TL;DR

| Type                                                          | Scenario            | Value                                 |
| ------------------------------------------------------------- | ------------------- | ------------------------------------- |
| [scalars](https://graphql.org/learn/schema/#scalar-types)     | no data / exception | `null`                                |
| value object types                                            | no data / exception | `null`                                |
| [lists](https://graphql.org/learn/schema/#list)               | no data found       | zero value: `[]`                      |
| lists                                                         | exception           | `null`                                |
| [connection types](https://relay.dev/graphql/connections.htm) | no data found       | zero value: `{ "edges": [] }`         |
| connection types                                              | exception           | `null`                                |
| grouping types                                                | no data found       | zero value container with null fields |
| grouping types                                                | exception           | `null`                                |

**Rules**

1. Scalars and value types always return null, regardless of error vs no data.
2. If a resolver throws, the field value becomes null.
3. Collection types return zero-value objects when empty and no exception occurred.

## Handling Errors and Null Values

Returning null for an error lets GraphQL frameworks append `path` and `locations` automatically. If you return a zero value and also add an error manually, you must derive path and locations yourself — extra complexity for your resolvers. It also introduces extra complexity for consumers who have to make sense of a data value along with an error pointing to said value.

**For Example**

```json
{
  "data": {
    "customer": {
      "name": "John Doe",
      "programs": {
        "personalCareRewards": null
      }
    }
  },
  "errors": [
    {
      "message": "internal server error",
      "path": ["customer", "programs", "personalCareRewards"],
      "locations": [{ "line": 3, "column": 6 }]
    }
  ]
}
```

The field is null, the grouping `programs` stays intact, and the error is properly attributed.

Grouping types rarely have fetching logic, so they should almost never throw or return null.

## Errors Bubble Up

Non-nullable fields cause errors to bubble up to the nearest nullable ancestor. In federation this can cause unintended side effects, like nullifying otherwise successful parts of a query response.

Consider two subgraphs owning different loyalty programs:

```graphql
type LoyaltyPrograms @key(fields: "id") {
  id: ID!
  personalCareRewards: PersonalCareProgram!
}
```

```graphql
type LoyaltyPrograms @key(fields: "id") {
  id: ID!
  foodLoyalty: FoodLoyaltyProgram
}
```

If `personalCareRewards` is non-nullable and its resolver throws an exception, the entire `programs` field becomes null, losing `foodLoyalty` data too.

**Bad:**

```json
{
  "data": {
    "programs": null
  },
  "errors": [...]
}
```

**Better:**

```json
{
  "data": {
    "customer": {
      "name": "John Doe",
      "programs": {
        "personalCareRewards": null,
        "foodLoyalty": { "purchaseHistory": [...] }
      }
    }
  },
  "errors": [...]
}
```

Partial success preserves usable data.

## Summary

The TL;DR table covers the mechanics. The harder part is the discipline. It's easy to slap `!` on a field to express confidence in your data, and then six months later regret it when a subgraph you don't own starts throwing exceptions and nullifying data. In a federated graph, non-nullable fields are a shared contract — one team's certainty can silently knock out another team's data.

Default to nullable. Reserve `!` for true invariants. Return zero values for empty collections, null for missing data and errors.
