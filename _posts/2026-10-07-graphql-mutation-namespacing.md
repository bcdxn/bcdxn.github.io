---
title: "Namespacing Mutations in a Federated Graph"
date: 2026-10-07 07:30:00 -0400
categories: [blog]
tags: [graphql, federation, apollo, architecture]
excerpt: ""
image:
  path: /assets/images/banners/2026-09-14.jpg
  alt: header
header:
  og_image: /assets/images/banners/2026-09-14.jpg
---

<script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
<script type="module">
  mermaid.initialize({ startOnLoad: true });
</script>

In a large enterprise GraphQL API, there may be thousands of mutations and queries. Namespacing queries is one approach to adding structure to your graph; it prevents a flat graph with thousands of root-level queries and improves the logical organization. This makes relationships clear and helps developers (and agents) find the functionality they're looking for in large, complex GraphQL APIs. For more details on why namespacing in GraphQL is a good idea, see Apollo's technical note on [Namespacing by Separation of Concern](https://www.apollographql.com/docs/graphos/schema-design/guides/namespacing-by-separation-of-concerns).

Namespacing mutations receives more pushback, but I believe it's still a valuable pattern. There are some oddities that I'll discuss later, but overall it presents a more hierarchical graph where the relationships between your entities are represented more clearly. It creates a more readable and discoverable API.

### An Example of a Flat GraphQL API

<pre class="mermaid">
%%{init: {'themeVariables': { 'edgeLabelBackground': '#fff'}}}%%
flowchart TD
  classDef default fill:none,stroke-width:2px
  classDef grouping fill:none,stroke:#999,stroke-width:2px
  classDef user fill:none,stroke:#333,stroke-width:3px

  mutation([mutation])
  mutation --> updatePreferredName[updatePreferredName]
  mutation --> updateName[updateName]
  mutation --> updateEmailAddress[updateEmailAddress]
  mutation --> updateFoodLoyaltyPreferences[updateFoodLoyaltyPreferences]
  mutation --> updatePersonalCarePreferences[updatePersonalCarePreferences]
  mutation --> updateHomeAndCleaningPreferences[updateHomeAndCleaningPreferences]

  class mutation user
</pre>

### An Example of a Hierarchical, Namespaced GraphQL API

<pre class="mermaid">
%%{init: {'themeVariables': { 'edgeLabelBackground': '#fff'}}}%%
flowchart TD
  classDef default fill:none,stroke-width:2px
  classDef grouping fill:none,stroke:#999,stroke-width:2px
  classDef user fill:none,stroke:#333,stroke-width:3px

  mutation([mutation])
  customer(CustomerMutation)
  profile(ProfileMutation)
  programs(ProgramsMutation)
  foodLoyalty(FoodLoyaltyMutation)
  personalCareRewards(PersonalCareRewardsMutation)
  homeAndCleaning(HomeAndCleaningMutation)

  mutation --> customer
  customer --> profile
  customer --> programs
  programs --> foodLoyalty
  programs --> personalCareRewards
  programs --> homeAndCleaning

  profile --> updatePreferredName[updatePreferredName]
  profile --> updateName[updateName]
  profile --> updateEmailAddress[updateEmailAddress]
  foodLoyalty --> foodLoyaltyUpdatePreferences[updatePreferences]
  personalCareRewards --> personalCareUpdatePreferences[updatePreferences]
  homeAndCleaning --> homeAndCleaningUpdatePreferences[updatePreferences]

  class mutation user
  class customer,profile,programs,foodLoyalty,personalCareRewards,homeAndCleaning grouping
</pre>

In general, your namespaces should align with your API domain model. A graph could also align with a formal data taxonomy, but in general an API domain model based on domain-driven design creates a clearer, more succinct graph. Exposing the internals of your data taxonomy can lead to overexposing data and underexposing capabilities -- think [anemic domain models](https://martinfowler.com/bliki/AnemicDomainModel.html)

Note that you can go overboard here. You should aim to have less than a few hundred mutations or queries in each namespace, but don't create namespaces with a granularity so fine that it begins to negatively impact readability. Think of namespaces like genres in a bookstore: they help you get to a set of related books quickly, and then you can navigate alphabetically to find the book you're looking for. In our case, namespaces should help you quickly jump to a specific domain. From there, you should rely on standard naming conventions to find the specific capability you're after.

So what does this look like in practice?

## Namespacing Mutations in a Stand-Alone Graph

When federation is not a concern, i.e., when we are working on a stand-alone graph, namespacing is straightforward. We simply create types (`CustomerMutation`, `ProfileMutation`, and `ProgramsMutation` in the example below) to act as namespaces and add our mutations as fields to these types.

### For Example

```graphql
type Mutation {
  customer: CustomerMutation!
}

type CustomerMutation {
  profile: ProfileMutation!
  programs: ProgramsMutation!
}

type ProfileMutation {
  updatePreferredName(
    input: CustomerProfileUpdatePreferredNameInput!
  ): CustomerProfileUpdatePreferredNamePayload!
  updateName(
    input: CustomerProfileUpdateNameInput!
  ): CustomerProfileUpdateNamePayload!
  updateEmailAddress: CustomerProfileUpdateEmailAddressPayload!
}

type ProgramsMutation {
  foodLoyalty: FoodLoyaltyMutation!
  personalCareRewards: PersonalCareRewardsMutation!
  homeAndCleaning: HomeAndCleaningMutation!
}

type FoodLoyaltyMutation {
  updatePreferences(
    input: FoodLoyaltyUpdatePreferencesInput!
  ): FoodLoyaltyUpdatePreferencesPayload!
}

type PersonalCareRewardsMutation {
  updatePreferences(
    input: PersonalCareRewardsUpdatePreferencesInput!
  ): PersonalCareRewardsUpdatePreferencesPayload!
}

type HomeAndCleaningMutation {
  updatePreferences(
    input: HomeAndCleaningUpdatePreferencesInput!
  ): HomeAndCleaningUpdatePreferencesPayload!
}
```

> **<i class="fas fa-circle-info"></i>** Note that I am using the payload response format described by [Marc Andre](https://magiroux.com) in [Production Ready GraphQL](https://productionreadygraphql.com/2020-08-01-guide-to-graphql-errors), but I have left the actual object definitions out of the examples for brevity.
> {: .notice--info}

## Namespacing Mutations in Federated Subgraphs

Namespacing mutations in a federated graph is similar to namespacing them in a stand-alone graph; we'll still use types to logically organize our mutations. However, in a federated graph, we will likely want multiple subgraphs to contribute to a single mutation namespace. To do this, we must turn the namespace into an entity by adding the `@key` directive. Different subgraphs will contribute fields to the _same_ namespace, so those fields must be shareable using the `@shareable` directive.

### For Example

```graphql
# Subgraph A
type Mutation {
  customer: CustomerMutation! @shareable
}

type CustomerMutation @key(fields: "id") {
  id: ID! @shareable
  profile: ProfileMutation! @shareable
  programs: ProgramsMutation! @shareable
}

type ProfileMutation @key(fields: "id") {
  id: ID! @shareable
  updatePreferredName(
    input: CustomerProfileUpdatePreferredNameInput!
  ): CustomerProfileUpdatePreferredNamePayload!
  updateName(
    input: CustomerProfileUpdateNameInput!
  ): CustomerProfileUpdateNamePayload!
}

type ProgramsMutation @key(fields: "id") {
  id: ID! @shareable
}
```

```graphql
# Subgraph B
type Mutation {
  customer: CustomerMutation! @shareable
}

type CustomerMutation @key(fields: "id") {
  id: ID! @shareable
  profile: ProfileMutation! @shareable
  programs: ProgramsMutation! @shareable
}

type ProfileMutation @key(fields: "id") {
  id: ID! @shareable
  updateEmailAddress: CustomerProfileUpdateEmailAddressPayload!
}

type ProgramsMutation @key(fields: "id") {
  id: ID! @shareable
  foodLoyalty: FoodLoyaltyMutation!
}

type FoodLoyaltyMutation @key(fields: "id") {
  id: ID!
  updatePreferences(
    input: FoodLoyaltyUpdatePreferencesInput!
  ): FoodLoyaltyUpdatePreferencesPayload!
}
```

So now we know what the schema looks like, but what does a subgraph actually provide as the value of a mutation namespace's `id` field? Before we can answer that, it's important to understand the purpose of the `id` field. Apollo Router uses it to merge data in responses from multiple subgraphs that are related to the same entity. This is easier to understand with a query example: a customer's loyalty information comes from one subgraph, while personal care information for the same customer comes from another subgraph, and the router returns a single customer object.

```json
{
  "customer": {
    "id": "123456",
    "foodLoyaltyProgram": { ... }
  }
}
```

```json
{
  "customer": {
    "id": "123456",
    "personalCareRewards": { ... }
  }
}
```

```json
{
  "customer": {
    "id": "123456",
    "foodLoyaltyProgram": { ... },
    "personalCareRewards": { ... }
  }
}
```

Remember that our namespace types are just logical groupings of mutations -- they aren't themselves true entities like a customer or an account. In fact, there is only a single instance of a mutation namespace entity. _Because of this, the ID doesn't matter as long as all subgraphs agree on what it is._

In the past, I have used the convention that the value of the `id` field is simply a static string that matches the name of the type. For example, the ID of the `ProgramsMutation` namespace is `"ProgramsMutation"`, the ID of the `ProfileMutation` namespace is `"ProfileMutation"`, and so on.

## Closing Thoughts

A mutation namespace is more than a container for fields. It is a map for navigating a large graph. Hierarchical graphs communicate that certain queries and mutations belong to different parts of the customer domain before a consumer ever reads a resolver or opens a documentation page.

That structure becomes especially valuable as a graph grows and more teams contribute to it. A well-designed namespace gives people and agents a reliable place to look, while Federation lets multiple subgraphs extend that place without forcing the graph back into a flat list. The goal is not maximum nesting. The goal is a graph whose shape explains the domain, makes capabilities discoverable, and stays readable as the organization behind it changes.
