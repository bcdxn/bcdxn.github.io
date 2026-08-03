---
title: "The Customer Identifiers Pattern in Federated GraphQL"
date: 2026-08-03 07:30:00 -0400
categories: [blog]
tags: [graphql, federation, apollo, architecture]
excerpt: "How to unify customer data across siloed business units using GraphQL Federation's @requires and @context directives."
header:
  og_image: /assets/images/2026-08-03-ogimage.jpeg
---

[![path](https://images.unsplash.com/photo-1516441311474-499e4f3b32db?auto=format&fit=crop)](https://unsplash.com/photos/aerial-view-of-trees-during-daytime-UxakjtPzIT0)

Large organizations may end up with siloed divisions whether its simply due to operating model or from acquisitions. In these environments divisions often have their own, independent view of a customer, but experiences increasingly need a more holistic view. The instinctive fix is to push identifiers into custom request headers or build a Backend for Frontend (BFF) layer per calling context — but both approaches collapse under domain complexity and become a maintenance burden at scale. Let's walk through a pattern in Federated GraphQL schema design that addresses the root cause instead.

I'll use a fictional CPG company to illustrate the pattern. This fictional company has three divisions: Food & Beverage, Personal Care, and Home & Cleaning. Each division manages customers independently with its own ID system. A single person might be a loyalty member for the food brand, a rewards participant for the beauty line, and a bulk buyer for the household products — all under different identifiers.

> A person may have multiple IDs across divisions, but they're the same customer.

### Legacy Experiences

<pre class="mermaid">
%%{init: {'themeVariables': { 'edgeLabelBackground': '#fff'}}}%%
flowchart LR
  classDef default fill:none,stroke-width:2px
  classDef grouping fill:none,stroke:#999,stroke-width:2px
  classDef user fill:none,stroke:#333,stroke-width:3px

  User([User])


  fbExp(Food & Bev<br/>Experience)
  pcExp(Personal Care<br/>Experience)
  hcExp(Home & Cleaning<br/>Experience)


  subgraph Services
    fb[Food & Beverage]
    pc[Personal Care]
    hc[Home & Cleaning]
  end

  User --> fbExp
  User --> pcExp
  User --> hcExp
  fbExp -->|"Food & Beverage ID"| fb
  pcExp -->|"Personal Care ID"| pc
  hcExp -->|"Home & Cleaning ID"| hc

  class User user
  class Services grouping
</pre>

### Modern Experiences (What the People Want)

<pre class="mermaid">
%%{init: {'themeVariables': { 'edgeLabelBackground': '#fff'}}}%%
flowchart LR
  classDef default fill:none,stroke-width:2px
  classDef grouping fill:none,stroke:#999,stroke-width:2px
  classDef user fill:none,stroke:#333,stroke-width:3px

  User([User])

  exp(Unified Experience)

  subgraph Services
    fb[Food & Beverage]
    pc[Personal Care]
    hc[Home & Cleaning]
  end

  User --> exp

  exp -->|"Food & Beverage ID"| fb
  exp -->|"Personal Care ID"| pc
  exp -->|"Home & Cleaning ID"| hc

  class User user
  class Experiences grouping
  class Services grouping
</pre>

When you build a federated supergraph to unify these domains, you quickly run into two problems:

1. **Identifier mapping** — different subgraphs need different IDs to resolve their data
2. **Auth vs. resource context** — the logged-in user isn't always the owner of the resource being queried

The `CustomerIdentifiers` pattern solves both.

---

## The Problem

### 1. Identifier Complexity

In a federated system, subgraphs contribute fields to shared entities like a `Customer` type. But each division's backend services expect their own identifier. Looking up food loyalty accounts needs one ID; checking personal care rewards needs another, etc.

A single `id` on the `Customer` entity isn't enough. Each subgraph needs the right division-specific identifier to resolve its fields.

### 2. Auth Context vs. Resource Context

The logged-in user acting on a record isn't always the owner of that record. A support agent might look up a customer's profile using internal tooling — the agent's auth token has nothing to do with the customer being queried.

Historically, APIs have coupled these together by stuffing identifiers into custom headers. We can do better.

---

## The Pattern

The solution is a `CustomerIdentifiers` entity that abstracts the ID mapping logic into the schema itself. It's owned by a central identity subgraph and contributed to by other subgraphs as needed.

**Customer Identifiers Subgraph (mapping capability)**

```graphql
type Customer @key(fields: "id") {
  id: ID!
  identifiers: CustomerIdentifiers @inaccessible
}

# The customer identifiers resolver in this subgraph implements the
# mapping logic required to fetch division-specific identifiers
type CustomerIdentifiers @key(fields: "id") @inaccessible {
  id: ID!
  foodLoyaltyId: ID
  personalCareRewardsId: ID
  homeBulkBuyerId: ID
}
```

**Food Loyalty Subgraph**

Contributes the `foodLoyaltyProgram` property to the `Customer` entity requring the `foodLoyaltyId`.

```graphql
type Customer @key(fields: "id") {
  id: ID!
  identifiers: CustomerIdentifiers @external

  foodLoyaltyProgram: FoodLoyaltyProgram
    @requires(fields: "identifiers { foodLoyaltyId }")
}

type CustomerIdentifiers @key(fields: "id") @external {
  id: ID!
  foodLoyaltyId: ID
}

type FoodLoyaltyProgram @key(fields: "id") {
  id: ID!
  # ...
}
```

**Personal Care Subgraph**

Contributes the `personalCareRewards` property to the `Customer` entity requring the `personalCareRewardsId`.

```graphql
type Customer @key(fields: "id") {
  id: ID!
  identifiers: CustomerIdentifiers @external

  personalCareRewards: PersonalCareRewards
    @requires(fields: "identifiers { personalCareRewardsId }")
}

type CustomerIdentifiers @key(fields: "id") @external {
  id: ID!
  personalCareRewardsId: ID
}

type PersonalCareRewards @key(fields: "id") {
  id: ID!
  # ...
}
```

**Home & Cleaning Subgraph**

```graphql
type Customer @key(fields: "id") {
  # identity fields...

  wholesaleDiscounts: [WholesaleDiscount!]!
    @requires(fields: "identifiers { homeBulkBuyerId }")
}

# ...
```

A few things to notice:

- **`@inaccessible`** — the `identifiers` field can never be queried by API consumers. It's purely a server-side linking construct. Clients are completely unaware it even exists.
- **Declarative** — which subgraphs require which identifiers is visible in the schema, making it auditable during governance reviews.
- **Stateless** — no encrypted headers or session state flowing between services. Given a customer ID, the router resolves the mappings on every request.

> **<i class="fas fa-lightbulb"></i>key insight:**  
> By keeping identifiers _in the schema_ rather than in headers or session state, we decouple the auth context from the resource context. The same subgraph resolvers work whether a customer is querying their own data or a support agent is looking someone up.
> {: .notice--info}

### Two Entry Points with a Single Entity

The pattern supports multiple entry points into the graph without duplicating resolver logic:

```graphql
type Query {
  # Customer-facing: uses the logged-in user's auth context
  me: Customer

  # associate targets customer using global member ID
  customerById(id: ID!): Customer
}

# ...
```

Both entry points return the same `Customer` entity. All downstream subgraphs that contribute to `Customer` work identically regardless of how the query started. No duplicate resolvers. No BFF-per-context.

A consumer query looks the same either way:

```graphql
query Me {
  me {
    foodLoyaltyProgram {
      points
      tier
    }
    personalCareRewards {
      balance
      coupons
    }
    # ...
  }
}
```

```graphql
query CustomerById {
  customerById(id: "123") {
    foodLoyaltyProgram {
      points
      tier
    }
    personalCareRewards {
      balance
      coupons
    }
    # ...
  }
}
```

The consumer is blissfully ignorant of the ID mapping happening server-side.

---

## The Prop-Drilling Problem

The `CustomerIdentifiers` pattern works great for entities directly on `Customer`. But as your graph grows deeper, a new problem emerges: **prop drilling**.

Every nested entity that needs a division-specific identifier has to thread `identifiers` down from the root, level by level. The `Customer` subgraph starts knowing about domains it shouldn't care about. For example, if we wanted to logically group all of the loyalty programs in the graph to stop our `Customer` type from growing into an infinite list of properties, we need to define the grouping type `Programs` with its own `identifiers` property _in the Customer subgraph_. And the problem only gets worse with deeper levels of nesting. On the other hand, nesting is what allows us to express relationship hierarchies in the graph — we don't want to lose that.

```graphql
# Customer Subgraph — now coupled to Programs just to pass identifiers
type Customer @key(fields: "id") {
  id: ID!
  identifiers: CustomerIdentifiers @inaccessible

  programs: Programs
}

type Programs @key(fields: "id") {
  id: ID!
  identifiers: CustomerIdentifiers! @inaccessible # threaded down
}
```

Every logical grouping in the graph needs this treatment. It breaks domain boundaries and becomes unsustainable.

---

## The Solution: `@context`

Apollo Federation's [`@context`](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/entities/use-contexts) directive solves this elegantly. Instead of threading `identifiers` through every entity level, we capture it once in a named context and let downstream fields pull from it directly.

Here's how the schemas change:

The `Customer` subgraph no longer knows about `Programs`.

```graphql
# Customer Subgraph — clean, no knowledge of Programs
type Query {
  me: Customer
  customerById(id: ID!): Customer
}

type Customer @key(fields: "id") @context(name: "customerCtx") {
  id: ID!
  identifiers: CustomerIdentifiers @inaccessible
}

type CustomerIdentifiers @key(fields: "id") @inaccessible {
  id: ID!
  foodLoyaltyId: ID
  personalCareRewardsId: ID
  homeBulkBuyerId: ID
}
```

Each division subgraph declares the context once on `Customer` and pulls the identifier it needs directly from `$customerCtx`, regardless of nesting depth.

```graphql
# Food & Beverage Subgraph — pulls from context, no prop drilling
type Customer @key(fields: "id") @context(name: "customerCtx") {
  id: ID!
  identifiers: CustomerIdentifiers! @external
  programs: Programs @shareable
}

type Programs @key(fields: "id") {
  id: ID!
  foodLoyalty(
    _foodLoyaltyId: ID
      @fromContext(fields: "$customerCtx { identifiers { foodLoyaltyId } }")
  ): FoodLoyaltyProgram
}

type FoodLoyaltyProgram {
  id: ID!
  points: Int
  tier: String
  favoriteProducts: [String!]
}

type CustomerIdentifiers @key(fields: "id") @external {
  id: ID!
  foodLoyaltyId: ID
}
```

```graphql
# Personal Care Subgraph — same pattern, different identifier
type Customer @key(fields: "id") @context(name: "customerCtx") {
  id: ID!
  identifiers: CustomerIdentifiers! @external
  programs: Programs @shareable
}

type Programs @key(fields: "id") {
  id: ID!
  personalCareRewards(
    _personalCareRewardsId: ID
      @fromContext(
        fields: "$customerCtx { identifiers { personalCareRewardsId } }"
      )
  ): PersonalCareRewards
}

type PersonalCareRewards {
  id: ID!
  balance: Float
  coupons: [String!]
  beautyProfile: String
}

type CustomerIdentifiers @key(fields: "id") @external {
  id: ID!
  personalCareRewardsId: ID
}
```

And our queries express the logical relationships we expect out the graph. Behind the scenes, Router generates an efficient query plan that fetches all required identifiers in a single request and fans out to the division subgraphs in parallel.

```graphql
query Me {
  me {
    programs {
      foodLoyalty {
        points
        tier
      }
      personalCareRewards {
        balance
        coupons
      }
    }
    # other logical groupings of data ...
    profile {
      name
    }
  }
}
```

---

## Why This Matters

The `CustomerIdentifiers` pattern, combined with `@context`, gives you:

- **A single entity** for a holistic customer view, with ID mapping hidden from consumers
- **Decoupled auth and resource context** — the same resolvers work for self-service and agent-assisted flows
- **Clean domain boundaries** — no prop drilling, no cross-domain coupling in subgraph schemas
- **Auditability** — every identifier dependency is declared in the schema, visible during governance reviews
- **Future-proofing** — change the underlying ID systems without touching consumer APIs

Our fictional CPG company can now ship a single loyalty query that works identically whether a customer is checking their own rewards or a support agent is looking them up — no custom headers, no BFF-per-context, and no leaking of internal ID systems to consumers.

Large organizations don't need to choose between unified experiences and domain autonomy. GraphQL Federation gives you both — if you design the schemas right.
