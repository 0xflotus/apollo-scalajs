---
title: Queries
order: 2
---

Now that you have your client set up, let's dive in deeper and see how to use Query components to load data into your application.

## The Query component
Using the Query component from Scala.js is just like using it with the original JavaScript API. Simply pass in a parsed GraphQL query object (from `gql`) and a function that decides what to render given the state of the query. Because the Query component tracks all steps of making a query, you can easily render loading and error states by reading properties of the query status given to you.

When you use the Query component, Apollo Client also performs caching to minimize server roundtrips. If the data requested is already in the cache, your Query component will immediately render with the cached data. If it isn't, the result of the query will be normalized and placed into the cache as soon as the data arrives.

Let's see this in action!

```scala
import com.apollographql.scalajs.react.Query

Query[js.Object](gql(
  """{
    |  rates(currency: "USD") {
    |    currency
    |  }
    |}""".stripMargin
)) { result =>
  if (result.loading) {
    h1("Loading!")
  } else {
    div(result.data.get.toString)
  }
}
```

Here, we specified the type of the result to be `js.Object`, essentially making it untyped. This data isn't super friendly to work with, though, so let's see how we can type our data.

## Typing Query Responses
Apollo Scala.js allows you to specify models to parse the query results into. These models are usually case classes and can be handwritten or generated by Apollo Codegen (see [automatic query types](#Automatic-Query-Types) for how to set this up).

Let's say we have a query
```graphql
{
  dogs {
    id
    breed
  }
}
```

The corresponding type for this would look something like this:
```scala
case class Dog(id: String, breed: String)
case class QueryData(dogs: Seq[Dog])
```

Once we have this type written, all we have to do is replace `js.Object` with `QueryData` to get typed responses! Apollo Scala.js handles the process of parsing the result into Scala objects, so you can focus on using the data.

```scala
Query[QueryData](gql(
  """{
    |  dogs {
    |    id
    |    breed
    |  }
    |}""".stripMargin
)) { queryStatus =>
  if (queryStatus.loading) "Loading..."
  else if (queryStatus.error) s"Error! ${queryStatus.error.message}"
  else {
    div(
      queryStatus.data.get.dogs.map { dog =>
        h1(key := dog.id)(dog.breed)
      }
    )
  }
}
```

## Querying with Variables
When performing a GraphQL query with a Query component, you can also pass in variables to be used by the query as an additional parameter. Just like the result, variables can be typed with an additional type parameter, or left untyped by using `js.Object` as the type.

To perform a query with untyped variables, you can use `js.Dynamic.literal` to construct the variables object.
```scala
case class Dog(id: String, breed: String)
case class QueryData(dog: Dog)

Query[QueryData, js.Object](
  gql(
    """query dog($breed: String!) {
      |  dog(breed: $breed) {
      |    id
      |    displayImage
      |  }
      |}""".stripMargin
  ),
  js.Dynamic.literal(
    breed = "bulldog"
  )
) { queryStatus =>
  if (queryStatus.loading) "Loading..."
  else if (queryStatus.error) s"Error! ${queryStatus.error.message}"
  else {
    img(src := queryStatus.data.get.dog.displayImage)
  }
}
```

If we want to type the variables being passed into the query, we can define a case class to replace `js.Object` in the type parameters.
```scala
case class Variables(breed: String)

Query[QueryData, Variables](
  gql(
    """query dog($breed: String!) {
      |  dog(breed: $breed) {
      |    id
      |    displayImage
      |  }
      |}""".stripMargin
  ),
  Variables(breed = "bulldog")
) { queryStatus =>
  if (queryStatus.loading) "Loading..."
  else if (queryStatus.error) s"Error! ${queryStatus.error.message}"
  else {
    img(src := queryStatus.data.get.dog.displayImage)
  }
}
```

## Automatic Query Types
With `apollo-codegen`, we can automatically generate query objects that tie together GraphQL queries with response types based on the schema definition. First, make sure you have followed the [Apollo Codegen Installation](./installation.html#Apollo-Codegen) steps and downloaded a schema by following the [instructions](https://github.com/apollographql/apollo-codegen#introspect-schema). For this example, we will be using the GraphQL server at `https://w5xlvm3vzz.lp.gql.zone/graphql`.

With Apollo Codegen installed, we can define our first query! Under `src/main/graphql`, you can define static queries that will be converted to Scala code by Apollo Codegen. For example, we could define a query `usdrates.graphql`:
```graphql
query USDRates {
  rates(currency: "USD") {
    currency
  }
}
```

Which results in an object `UsdRatesQuery` being generated. To use this in a query component, we can simply pass the generated object in instead of a query string and Apollo Scala.js will automatically gather the result type based on the generated code.

```scala
Query(UsdRatesQuery)  { queryStatus =>
  if (queryStatus.loading) "Loading..."
  else if (queryStatus.error) s"Error! ${queryStatus.error.message}"
  else {
    div(queryStatus.data.get.rates.mkString(", "))
  }
}
```

Apollo Codegen also supports generating variables types, which are emitted as case classes inside the query object. To pass in variables to a query that requires some, simply pass in an instance of the variables class as a parameter after the query object.

For a query:
```graphql
query CurrencyRates($cur: String!) {
  rates(currency: $cur) {
    currency
  }
}
```

We can pass in variables in a query component as:
```scala
Query(CurrencyRatesQuery, CurrencyRatesQuery.Variables("USD"))  { queryStatus =>
  if (queryStatus.loading) "Loading..."
  else if (queryStatus.error) s"Error! ${queryStatus.error.message}"
  else {
    div(queryStatus.data.get.rates.mkString(", "))
  }
}
```

## Extra Query Options
If you want to pass in additional query options, such as the fetch policy, you can provide an instance of `ExtraQueryOptions` as an additional parameter. For example, to force the query to only load from the cache you can use:

```scala
Query(CurrencyRatesQuery, CurrencyRatesQuery.Variables("USD"), ExtraQueryOptions(
  fetchPolicy = "cache-only"
)) { ... }
```

## Next steps
Now that you've learned how to get data from your GraphQL server, it's time to learn to update that data with [Mutations](./mutations.html)!