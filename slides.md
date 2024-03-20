---
# try also 'default' to start simple
theme: seriph
themeConfig:
  primary: '#A66BE7'
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
#background: https://cover.sli.dev
background: https://images.unsplash.com/photo-1482747029550-096ad6aa9caf?q=80&w=2670&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
# some information about your slides, markdown enabled
title: Nullability in GraphQL
# apply any unocss classes to the current slide
class: text-center
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# https://sli.dev/guide/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/guide/syntax#mdc-syntax
mdc: true
---

# GraphQL: Nullability

Please send all complaints to /dev/null

<!-- We are going to cover what non-null is and why we should use it -->

--- 
layout: image-right
image: /Maybe.png
---

# I can haz null?
Someone has a case of the "maybes"
````md magic-move
```graphql {*|2-4|7}
type Customer {
  id: ID!
  firstName: String!
  lastName: String!
  fullName: String!
  address: Address!
  order: Order 
}
```
```graphql
type Customer {
  id: ID!
  firstName: String!
  lastName: String!
  fullName: String!
  address: Address!
  order: Order 
}

type Order {
  customer: Customer!
  id: ID!
}

type Query {
  customer(id: ID!): Customer
  customers(name: String, limit: Int): [Customer!]
  orders(customer: Customer): [Order!]!
}
```
````

<!-- 
So what do null fields look like?

Fields which have a ! next to them mean non-nullable

This means these fields will never return a null value
[click]

Here we can see that names, and address are required fields that will always return a value

[click]

whereas order could return null

[click]

In this next example we can start to model our relationships between orders and customers and how we might query them
We can see some more examples of non-nullability in action

our customer query uses an id filter which MUST be supplied
whereas the customers query has some optional fields

Also note the subtle difference between the return types of customers and orders. Can you guess how they might be different?
We will cover this later
-->

---
layout: image-right
image: https://images.unsplash.com/photo-1607799279861-4dd421887fb3?q=80&w=2670&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---

# Query results
What's in the box?

````md magic-move
```js
// ReturnType [Customer!]

// no customer found
{ customers: null } 

// some results found
{
  customers: [{
    name: "Mickey Mouse",
    firstName: "Mickey",
    lastName: "Mouse",
  }]
}
```
```js
// But these results are not possible

customers: [{
  fullName: null,
  firstName: null,
  lastName: null,
}] 

{customers: [null]} 
```
````
<!--
This means that in addition to guaranteeing the shape of the result, GraphQL can provide a guarantee for which fields must be present when queried.
-->
---
layout: image-left
image: https://images.unsplash.com/photo-1607799279861-4dd421887fb3?q=80&w=2670&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---

# Why is this useful?
Stronger client contracts

````md magic-move
```ts
const result = await client.query({ query, });

result.data.customers.forEach(customer => {
  console.log(customer.fullName);
  console.log(restaurant.address.address1);
});
```

```ts
const result = await client.query({ query: query });

if (result.data.customers) {
  result.data.customers.forEach(customer => {
    if (customer) {
      console.log(customer.fullName);
      console.log(customer.address.postalCode);
    }
  });
}
```
````
<!-- 
The usefulness of such a guarantee becomes apparent when you think about the frontend code that might consume the result of such a query. Imagine the following code in JavaScript:

At first, this might seem like a reasonable piece of code to write based on the query, since you’re expecting to get back a list of restaurants that each have a name to be displayed. 
But, once we look more closely at our schema, it turns out that there are two potential errors hidden in this code:

[click]

by enforcing our types with null and non-nulls we can rely on "null" as a code path in our clients
we can also rely on null rather than undefined. This is then further reinforced by the gql codegen tools we use.

But the great thing is that we don’t have to worry about customer.name being null, because in our schema we’ve declared that if we have a customer object at all, it’s guaranteed to have a value for the name field or non-null address fields. That’s the value of declaring a field as non-null in GraphQL.

-->

---

# What if i accidentally return a null

````md magic-move
```ts
const resolvers = {
  Query: {
    customers: (parent) => {
      // Let's say this is what our database happens
      // to return
      return [{
        firstName: "Layton",
        lastName: "Gilbraith",
        address: null,
      }]
    }
  }
};
```
```ts
const resolvers = {
  Query: {
    customers: (parent) => {
      // Let's say this is what our database happens
      // to return
      return [{
        firstName: "Layton",
        lastName: "Gilbraith",
        address: null,
      }]
    }
  }
};

{
  "data": {
    "customers": [ null ]
  },
  "errors": [{
    "message": "Cannot return null for non-nullable field Customer.address."
  }]
}
```
````
<!-- 
In this case, let’s say that we’re using a schemaless database like MongoDB and someone forgot to store an address field for one of our customers. So when we return that query result through GraphQL, we’ll be returning a null value for a non-null field. What happens?

When you try to return a null value in a resolver for a non-null field, the null result bubbles up to the nearest nullable parent.
Let's assume for this case our customers list result is nullable.

[click]

So in this case, we’d get null for the entire array item, plus an error notifying us about the null:

This ensures that our frontend code never has to check for null when accessing the address field, even if it’s at the expense of not getting any part of the customer object. So there’s a tradeoff here:

When using non-null, you need to decide if you’d rather have a partial result, or no result at all.

This also brings up another common question about nullability: How does it work with lists?
-->

---

# Lists

```graphql
type Query {
  customer(id: ID!): Customer
  customers(name: String, limit: Int): [Customer!]
  orders(customer: Customer): [Order!]!
}
```

<v-click>

```ts
// possible results for customers
[] // valid
[{...Customer}] // valid
null // valid
[null] //invalid
```

</v-click>

<v-click>

```ts
// possible results for orders
[] // valid
[{...Customer}] // valid
null // invalid
[null] //invalid
```

</v-click>

<!-- 
Ok I said I would come back to this

here we have 3 queries and the 2nd to return lists.

We have a seemingly subtle difference one ending in the ! modifier and one that doesn't

The latter is the strictest

[click]

This means that the list itself can be null, but it can’t have any null members.

[click]

The last is the most strict
you can see that in all cases an array will be returned
however it cannot contain nulls or be null itself
The client code would never have to worry about null or undefined, just the empty list case

One interesting conclusion here is that there’s no way to specify that the list can’t be empty — an empty list [] is always valid, regardless of whether the list or items are non-null.

-->

---

# Inputs

```graphql
input CustomerFilters {
    name: String,
    orders: [Int!]!
}

type Query {
  customersByOrder(query: CustomerFilters!): [Customer!]
}
```

<!-- 
it's likely many of you have been using non-null modifier all this time without really thinking about it.

Here we can use it to modify query fields in more complex filtering patterns

in this example an array of orderIDs is required, but additional filtering say by fuzzy matching on name is an optional additional filter
-->

---
layout: image-right
image: https://images.unsplash.com/photo-1517849845537-4d257902454a?q=80&w=2670&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---

# I `null` therefore I aren't

<v-clicks depth="2">

- Nulls can make it hard to evolve your schema
  - is there any plausible universe no matter how improbable where I may want this to be null?
  - Field privacy
  - Deprecation can be difficult
- Small data inconsistencies could have an outsized impact

</v-clicks>

<!-- 
So now you know what non-null fields look like and how to use them, the next question is when should you use them?

[click]

Nulls can make it harder to evolve your schema in the future

[click]

is it plausible or knowable that I will never want to return a null?

If the answer is no (or if the question is too complex to answer) then perhaps consider sticking with the GraphQL default and make the field nullable
as going from nullable -> non-nullable is backwards compatible, but the reverse is not
you can always make it non-nullable later without impacting your clients

For input arguments and fields, adding non-null is a breaking change. i.e. a search param, if you make it nullable later your client's search will still work

For output fields (results), removing non-null from a field is a breaking change. there's a new state they likely won't handle

[click]

another example could be field privacy
You might allow the client to specify a non-value for a field like age
if the field is non-nullable you have to return some value in it's place like -1, rather than just null

[click]

You might also want to deprecate a field, but because a field is non-null you must always provide a value for this field. Even for new instances of the type.

[click]

If you are returning a list of results, but one of the results has an invalid null datafield, like our mongoDB example earlier
it could cause the entire list to return as null, because it is the first nullable parent. This might rob us of an ability to partially render that list of other perfectly good results?

-->

--- 

# Best practices

Use nulls thoughtfully


<span class="flex flex-row items-center"><uim-check class="text-green text-2xl" /> use non-nullable for <uim-check class="text-green text-2xl" /></span>

<v-clicks>

- Field arguments and input objects, where the field doesn’t make any sense if that argument is not passed
- On object fields that are guaranteed to have a result
- Better error detection, errors are thrown for incomplete data

</v-clicks>

<span class="flex flex-row items-center"><uim-multiply class="text-red text-2xl" /> When you might not use it <uim-multiply class="text-red text-2xl" /></span>

<v-clicks>

- Fields that may want to enforce privacy in future
- schemas that are still in flux, as nullable fields are easier to evolve
- When you still want to be able to retrieve partial data (requires more defensive coding)

</v-clicks>

<!--
So in conclusion here are some best practices

[click]

When you know that something will always be required for filtering, or as a relationship between entities such as a 1:1 relationship.
i.e. if an order absolutely must have a customer associated. we should enforce that through typings
or when a results must return at least 1 result, make this easier 

-->

