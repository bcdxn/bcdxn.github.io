---
title: "Namespacing Mutations in a Federated Graph"
date: 2026-10-07 07:30:00 -0400
categories: [blog]
tags: [graphql, federation, apollo, architecture]
excerpt: "How hierarchical mutation namespaces improve discoverability and organization in large federated GraphQL APIs."
image:
  path: /assets/images/banners/2026-10-07.jpg
  alt: header
header:
  og_image: /assets/images/banners/2026-10-07.jpg
---

<script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
<script type="module">
  mermaid.initialize({ startOnLoad: true });
</script>

[![banner](/assets/images/banners/2026-10-07.jpg)](https://unsplash.com/photos/palm-leaf-against-blue-sky-TMxUnMAAwFA)

In a large enterprise GraphQL API, there may be thousands of mutations and queries. Namespacing queries is one approach to adding structure to your graph; it prevents a flat graph with thousands of root-level queries and improves the logical organization. This makes relationships clear and helps developers (and agents) find the functionality they're looking for in large, complex GraphQL APIs. For more details on why namespacing in GraphQL is a good idea, see Apollo's technical note on [Namespacing by Separation of Concern](https://www.apollographql.com/docs/graphos/schema-design/guides/namespacing-by-separation-of-concerns).

Namespacing mutations receives more pushback, but I believe it's still a valuable pattern. There are some oddities that I'll discuss later, but overall it presents a more hierarchical graph where the relationships between your entities are represented more clearly. It creates a more readable and discoverable API.

**An Example of a Flat GraphQL API**

<pre class="mermaid">
%%{init: {'themeVariables': { 'edgeLabelBackground': '#fff'}}}%%
flowchart LR
  classDef default fill:none,stroke-width:2px
  classDef grouping fill:none,stroke:#999,stroke-width:2px
  classDef user fill:none,stroke:#333,stroke-width:3px

  mutation([mutation])
  mutation --> updateEmailAddress[updateEmailAddress]
  mutation --> updateFoodLoyaltyPreferences[updateFoodLoyaltyPreferences]
  mutation --> updateHomeAndCleaningPreferences[updateHomeAndCleaningPreferences]
  mutation --> updateName[updateName]
  mutation --> updatePersonalCarePreferences[updatePersonalCarePreferences]
  mutation --> updatePreferredName[updatePreferredName]

  class mutation user
</pre>

**An Example of a Hierarchical, Namespaced GraphQL API**

<pre class="mermaid">
%%{init: {'themeVariables': { 'edgeLabelBackground': '#fff'}}}%%
flowchart LR
  classDef default fill:none,stroke-width:2px
  classDef grouping fill:none,stroke:#999,stroke-width:2px
  classDef user fill:none,stroke:#333,stroke-width:3px

  mutation([mutation])
  customer(Customer)
  profile(Profile)
  programs(LoyaltyPrograms)

  mutation --> customer
  customer --> profile
  customer --> programs
  programs --> updateFoodLoyaltyPreferences[updateFoodLoyaltyPreferences]
  programs --> updateHomeAndCleaningPreferences[updateHomeAndCleaningPreferences]
  programs --> updatePersonalCareRewardsPreferences[updatePersonalCareRewardsPreferences]

  profile --> updateEmailAddress[updateEmailAddress]
  profile --> updateName[updateName]
  profile --> updatePreferredName[updatePreferredName]
  class mutation user
  class customer,profile,programs grouping
</pre>

In general, your namespaces should align with your API domain model. A graph could also align with a formal data taxonomy, but in general an API domain model based on Domain Driven Design creates a clearer, more succinct graph. Exposing the internals of your data taxonomy can lead to overexposing data and underexposing capabilities -- think [anemic domain models](https://martinfowler.com/bliki/AnemicDomainModel.html)

Note that you can go overboard here. The exact number of namespaces and mutations per namespace will vary by your domain but in general the goal is improving readability and discoverability. I'd say you should aim to have no more than a couple hundred mutations or queries in each namespace to achieve that goal. Don't create namespaces so coarse that you still struggle to find the needle in the haystack and avoid a namespace granularity so fine that it begins to negatively impact readability and the size of your graph -- that is no good either. Think of namespaces like genres in a bookstore: they help you get to a set of related books quickly, and then you can navigate alphabetically to find the book you're looking for. In our case, namespaces should help you quickly jump to a specific domain. From there, you should rely on standard naming conventions to find the specific capability you're after.

So what does this look like in practice?

## Namespacing Mutations in a Stand-Alone Graph

When federation is not a concern, i.e., when we are working on a stand-alone graph, namespacing is straightforward. We simply create types (`CustomerMutation`, `ProfileMutation`, and `LoyaltyProgramsMutation` in the example below) to act as namespaces and add our mutations as fields to these types.

### For Example

```graphql
type Mutation {
  customer: CustomerMutation!
}

type CustomerMutation {
  profile: ProfileMutation!
  programs: LoyaltyProgramsMutation!
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

type LoyaltyProgramsMutation {
  updateFoodLoyaltyPreferences(
    input: FoodLoyaltyUpdatePreferencesInput!
  ): FoodLoyaltyUpdatePreferencesPayload!
  updatePersonalCareRewardsPreferences(
    input: PersonalCareRewardsUpdatePreferencesInput!
  ): PersonalCareRewardsUpdatePreferencesPayload!
  updateHomeAndCleaningPreferences(
    input: HomeAndCleaningUpdatePreferencesInput!
  ): HomeAndCleaningUpdatePreferencesPayload!
}
```

**<i class="fas fa-circle-info"></i>** Note that I am using the payload response format described by [Marc Andre](https://magiroux.com) in [Production Ready GraphQL](https://productionreadygraphql.com/2020-08-01-guide-to-graphql-errors), but I have left the actual object definitions out of the examples for brevity.
{: .notice--info}

## Namespacing Mutations in Federated Subgraphs

Namespacing mutations in a federated graph is similar to namespacing them in a stand-alone graph; we'll still use types to logically organize our mutations. However, in a federated graph, we will likely want multiple subgraphs to contribute to a single mutation namespace. To do this, we must turn the namespace into an entity by adding the `@key` directive. Different subgraphs will contribute fields to the _same_ namespace, so those fields must be shareable using the `@shareable` directive.

**For Example**

```graphql
# Subgraph A
type Mutation {
  customer: CustomerMutation! @shareable
}

type CustomerMutation @key(fields: "id") {
  id: ID! @shareable
  profile: ProfileMutation! @shareable
  programs: LoyaltyProgramsMutation! @shareable
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

type LoyaltyProgramsMutation @key(fields: "id") {
  id: ID! @shareable
  updateFoodLoyaltyPreferences(
    input: FoodLoyaltyUpdatePreferencesInput!
  ): FoodLoyaltyUpdatePreferencesPayload!
  updatePersonalCareRewardsPreferences(
    input: PersonalCareRewardsUpdatePreferencesInput!
  ): PersonalCareRewardsUpdatePreferencesPayload!
  updateHomeAndCleaningPreferences(
    input: HomeAndCleaningUpdatePreferencesInput!
  ): HomeAndCleaningUpdatePreferencesPayload!
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
  programs: LoyaltyProgramsMutation! @shareable
}

type ProfileMutation @key(fields: "id") {
  id: ID! @shareable
  updateEmailAddress: CustomerProfileUpdateEmailAddressPayload!
}

type LoyaltyProgramsMutation @key(fields: "id") {
  id: ID! @shareable
  foodLoyalty(
    input: FoodLoyaltyUpdatePreferencesInput!
  ): FoodLoyaltyUpdatePreferencesPayload!
  updatePersonalCareRewardsPreferences(
    input: PersonalCareRewardsUpdatePreferencesInput!
  ): PersonalCareRewardsUpdatePreferencesPayload!
  updateHomeAndCleaningPreferences(
    input: HomeAndCleaningUpdatePreferencesInput!
  ): HomeAndCleaningUpdatePreferencesPayload!
}
```

So now we know what the schema looks like, but what does a subgraph actually provide as the value of a mutation namespace's `id` field? Before we can answer that, it's important to understand the purpose of the `id` field. Apollo Router uses it to merge data in responses from multiple subgraphs that are related to the same entity. This is easier to understand with a _query_ example: a customer's loyalty information, personal care information, and profile data come from separate subgraphs, and the router returns a single customer object.

**For example, consider the query**

```graphql
query customerView {
  customer {
    profile {
      # ...
    }
    programs {
      foodLoyaltyProgram {
        # ...
      }
      personalCareRewards {
        # ...
      }
    }
  }
}
```

Each subgraph returns its piece(s) of the namespace(s) and the final response is merged and returned to the caller as shown below.

<pre class="mermaid">
%%{init: {'themeVariables': { 'edgeLabelBackground': '#fff'}}}%%
flowchart LR
  classDef default fill:none,stroke-width:2px
  classDef grouping fill:none,stroke:#999,stroke-width:2px
  classDef user fill:none,stroke:#333,stroke-width:3px
  classDef response font-family:monospace,text-align:left

  food[Food Loyalty Subgraph]
  personal[Personal Care Subgraph]
  profile[Customer Profile Subgraph]

  profileResponse["customer:{<br/>&nbsp;&nbsp;id: 123456<br/>&nbsp;&nbsp;profile: {...}<br/>}"]
  foodResponse["customer:{<br/>&nbsp;&nbsp;id: 123456<br/>&nbsp;&nbsp;programs:{<br/>&nbsp;&nbsp;&nbsp;&nbsp;id:123456<br/>&nbsp;&nbsp;&nbsp;&nbsp;foodLoyalty: {...}<br/>&nbsp;&nbsp;}<br/>}"]
  personalResponse["customer:{<br/>&nbsp;&nbsp;id: 123456<br/>&nbsp;&nbsp;programs:{<br/>&nbsp;&nbsp;&nbsp;&nbsp;id:123456<br/>&nbsp;&nbsp;&nbsp;&nbsp;personalCareRewards: {...}<br/>&nbsp;&nbsp;}<br/>}"]

  router((Apollo Router))
  merged["customer:{<br/>&nbsp;&nbsp;id: 123456<br/>&nbsp;&nbsp;profile: {...}<br/>&nbsp;&nbsp;programs:{<br/>&nbsp;&nbsp;&nbsp;&nbsp;id:123456<br/>&nbsp;&nbsp;&nbsp;&nbsp;foodLoyalty: {...}<br/>&nbsp;&nbsp;&nbsp;&nbsp;personalCareRewards: {...}<br/>&nbsp;&nbsp;}<br/>}"]

  profile --> profileResponse --> router
  food --> foodResponse --> router
  personal --> personalResponse --> router
  router --> merged

  class food,personal,profile,foodResponse,personalResponse,profileResponse,merged grouping
  class router user
  class foodResponse response
  class personalResponse response
  class profileResponse response
  class merged response
</pre>

**<i class="fas fa-triangle-exclamation"></i>** Note that I am using a query to illustrate the merging behavior above. It is typically frowned upon to run multiple mutations in a single request as there are no transactional guarantees in a GraphQL API.
{: .notice--warning}

With query namespaces, we are grouping response data returned from disparate subgraphs related to an entity; therefore the namespace IDs must relate to the underlying entity, in this case the customer. However, in our mutations, our namespace types are just logical groupings of mutations themselves -- not of response data related to an underlying entity. In fact, with our mutation namespaces there is only a single instance. _Because of this, the value of the ID of a mutation entity in a federated graph doesn't matter as long as all subgraphs agree on what it is._

In the past, I have used the convention that the value of the `id` field is simply a static string that matches the name of the type. For example, the ID of the `LoyaltyProgramsMutation` namespace is `"LoyaltyProgramsMutation"`, the ID of the `ProfileMutation` namespace is `"ProfileMutation"`, and so on. This creates a stable ID that is only an agreed identity token for the logical namespace, not a customer or resource identifier.

Clients should not send multiple mutations in one request because GraphQL does not provide transactional guarantees across mutations. In the usual case, a namespaced mutation is resolved by a single subgraph, which returns the complete response; there is no query-style join to perform.

The namespace still needs an `@key` field and a stable ID. Federation requires that identity so multiple subgraphs can contribute to the same namespace type, even when the router does not need to merge multiple mutation responses at query execution time.

**For Example, consider the mutation**

```graphql
mutation UpdatePreferredName($input: CustomerProfileUpdatePreferredNameInput!) {
  customer {
    profile {
      updatePreferredName(input: $input) {
        # ...
      }
    }
  }
}
```

The mutation is handled by a single subgraph and router returns the response to the caller. No merge is necessary.

<pre class="mermaid">
%%{init: {'themeVariables': { 'edgeLabelBackground': '#fff'}}}%%
flowchart LR
  classDef default fill:none,stroke-width:2px
  classDef grouping fill:none,stroke:#999,stroke-width:2px
  classDef user fill:none,stroke:#333,stroke-width:3px
  classDef response font-family:monospace,text-align:left

  router((Apollo Router))
  profile[Customer Profile Subgraph]
  profileResponse["customer:{<br/>&nbsp;&nbsp;id: 123456<br/>&nbsp;&nbsp;profile: {<br/>&nbsp;&nbsp;&nbsp;&nbsp;id: ProfileMutation<br/>&nbsp;&nbsp;&nbsp;&nbsp;updatePreferredName {<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;...<br/>&nbsp;&nbsp;&nbsp;&nbsp;}<br/>&nbsp;&nbsp;}<br/>}"]
  merged["customer:{<br/>&nbsp;&nbsp;id: 123456<br/>&nbsp;&nbsp;profile: {<br/>&nbsp;&nbsp;&nbsp;&nbsp;id: ProfileMutation<br/>&nbsp;&nbsp;&nbsp;&nbsp;updatePreferredName {<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;...<br/>&nbsp;&nbsp;&nbsp;&nbsp;}<br/>&nbsp;&nbsp;}<br/>}"]

  profile --> profileResponse --> router
  router --> merged

  class food,personal,profile,foodResponse,personalResponse,profileResponse,merged grouping
  class router user
  class foodResponse response
  class personalResponse response
  class profileResponse response
  class merged response
</pre>

## Closing Thoughts

A mutation namespace is more than a container for fields. It is a map for navigating a large graph. Hierarchical graphs communicate that certain queries and mutations belong to different parts of your business domain before a consumer ever reads a resolver or opens a documentation page.

That structure becomes especially valuable as a graph grows and more teams contribute to it. A well-designed namespace gives people and agents a reliable place to look, while Federation lets multiple subgraphs extend that place without forcing the graph back into a flat list. The goal is to build a graph whose shape explains the domain, makes capabilities discoverable, and stays readable as the organization behind it changes.
