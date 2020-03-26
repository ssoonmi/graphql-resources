# GraphQL Schema

The Schema determines what types of data included in your data graph. This schema is not the schema of your database, but how you want your database to be mapped into a graph. 

We will be seeing how we can make define those types of data in our schema with **type definitions**.

For example, the schema for an online book lending store might define the following types:
```graphql
type Book {
  title: String
  isBooked: Boolean
  author: Author
}
type Author {
  name: String
  books: [Book]
}
```
If the client asked for information on a `Book`, the information can include the `title` as a `String`, and the `author` which can include information about their `name` and their `books` as a type of `Book`.

Example of data returned for information on a single `Book`:
```javascript
{
  title: 'Hello',
  isBooked: true,
  author: {
    name: 'Mike',
    books: [{
      title: 'Hello'
    }, {
      title: 'World!'
    }]
  }
}
```

## Object Type

Object Types have a collection of **fields** and each field has a type of their own. 

A field can either be another **Object Type** or can be a **Scalar Type**.

A **Scalar Type** is a primitive (`ID`, `String`, `Boolean`, `Int`) that always resolves to a single value. Check out [GraphQL's built-in scalar types] here. 

You can also build your own custom Scalar Types. An example of a common custom Scalar Type would be a `Date` type. Check out [How to Define Custom Scalar Types] here.

Let's use the online book lending store example again: 
```graphql
type Book {
  _id: ID!
  title: String
  isBooked: Boolean!
  author: Author
}
type Author {
  _id: ID!
  name: String
  books: [Book]
}
type User {
  _id: ID!
  email: String!
  books: [Book]!
}
```

The type `Book` is an Object Type that has fields, `_id`, `title`, `isBooked`, with Scalar Types `ID`, `String`, `Boolean`, and a field, `author`, of Object Type `Author`. The Object Type of `Author` is also defined in the schema. 

The `!` after some of the fields indicate that this field's value can **never be null**.

The `[]` around a field's value means that the field should return an array of the type it surrounds.

## the `Query` type

Object Types define the objects that exist in our data graph, but the client doesn't yet have a way to **fetch** those objects. We need to define **queries** that clients can execute against the data graph.

You can define these queries on a special type called the `Query` type.

Using the online book lending store example again:
```graphql
type Query {
  books: [Book]
  book(_id: ID!): Book
  me: User
}
```
This `Query` type defines three available queries for clients to execute: `books`, `book`, and `me`.
- The `books` query will return an array of all `Book`s.
- The `book` query will return a single `Book` that corresponds to the `_id` argument provided by the query.
- The `me` query will return details for the `User` that's currently logged in.

## the `Mutation` type

Queries enable clients to fetch data, but not to **modify** data. To enable clients to modify data, we need to define some **mutations** on the **`Mutation`** type

The `Mutation` type is a special type that's similar in structure to the `Query` type.

Using the online book lending store example again:
```graphql
type Mutation {
  borrowBooks(bookIds: [ID]!): BookUpdateResponse!
  returnBook(bookId: ID!): BookUpdateResponse!
  login(email: String!, password: String!): User
}

type BookUpdateResponse {
  success: Boolean!
  message: String
  books: [Book]
}
```
This `Mutation` type defines three available mutations for clients to execute: `borrowBooks`, `returnBook`, and `login`.
- The `borrowBooks` mutation enables a logged-in user to borrow one or more `Book`s specified by an array of `bookIds`.
- The `returnBook` mutation enables a logged-in user to return a book that they previously borrowed specified by a `bookId`.
- The `login` mutation enables a user to log in by providing their email address and password.

The `BookUpdateResponse` is an Object Type that contains a `success` status, a corresponding `message`, and an array of `Book`s that were modified by the mutation. It's good practice for a mutation to return whatever objects it modifies so the requesting client can update its cache and UI without needing to make a following query.

The GraphQL Schema tells our server which GraphQL **types and operations** it supports, but it **doesn't know where to obtain the data to respond to those operations**. For that, we need our **GraphQL Resolvers**. 

[GraphQL's built-in scalar types]: https://graphql.org/learn/schema/
[How to Define Custom Scalar Types]: https://itnext.io/custom-scalars-in-graphql-9c26f43133f3