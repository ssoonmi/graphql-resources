# Formulating GraphQL Queries and Mutations

Now that we have learned how to create our GraphQL schema with type definitions and resolvers, we will learn how to create a `Query` and a `Mutation`.

## Set Up

[GraphQL Playground] is an amazing GraphQL IDE that is much better than the built-in GraphQL IDE in the `express-graphql` package that we will be using to run GraphQL on our server. To follow along with this reading and play around and make your own queries and mutations, please do the following instructions:

1. Create a MongoDB cluster or use a past cluster to create a MongoDB URI connection string
2. Download this repository [Book Lending Server Example]
3. Run `npm install`
4. Run `MONGO_URI="<your MongoDB URI connection string>" npm run seeds`
5. Run `MONGO_URI="<your MongoDB URI connection string>" npm start`
6. Go to [localhost:5000/playground](localhost:5000/playground)

In GraphQL Playground, you can see the type definitions on the "SCHEMA" tab to the right of the IDE. You can also see all the queries and mutations you can run against the server in the "DOCS" tab.

## Making a `Query`

GraphQL queries are used to access information from our server. 

To create a query, we need to define what `Query` we want to use, any arguments the query requires, and select the data we want returned.

Take a look at the "DOCS" tab on the right of Playground to see all the `Queries` we can try.

If we want to get all the books we can write our query like so:

```graphql
{
  books {
    _id
    title
  }
}
```

Paste this query into the window on the left side of Playground and press the play button to run the query.

Since the `books` query doesn't need any arguments, we don't need to pass it in. We specified that we only want to get back the `_id`'s and `title`s of all the books.

What if we want to know who the `author` of the book is? We can't just do this:

```graphql
# This won't work
{
  books {
    _id
    title
    author
  }
}
```

Try this and you will receive an error. 

Reference the "SCHEMA" tab on the right of Playground that show all the type definitions when considering the following.

The `books` query returns an array of the `Book` Object Type. `author` is a field on the `Book` whose value is an `Author` Object Type. An `Author` Object type must return an object with it's own fields and values.

The following query properly extract's the information about the `author` of a `Book`.

```graphql
# This will work
{
  books {
    _id
    title
    author {
      # fields defined on the `Author` type
      _id
      name
    }
  }
}
```

We need to define the fields we want to extract from the `Author` Object Type when asking for the `author` field on a `Book` type.

Fields that have Scalar Types as values (e.g. `ID`, `String`, `Boolean`) are finite and can't have additional fields extracted from them.

Fields that have Object Types as values (e.g. `Book`, `Author`, `User`) are not finite so you need to define the subfields that you want from them.

Try the `book` query out! The `book` query expects an argument of `_id`. Use an `_id` from one of the books in the `books` query. The return fields for a `book` query is the same for the `books` query.

```graphql
{
  book(_id: "<input book id>") {
    _id
    title
  }
}
```

There is also another way we could have defined this query. And that's using `Query Variables`. You can define `Query Variables` in Playground by clicking on the bottom left tab. Let's define a variable called `bookId` pointing to an `_id` from one of the books in the `books` query.

```json
{
  "bookId": "<input book id>"
}
```

Then, let's define the query to accept the `Query Variables` that it's expecting, like so: 
```graphql
query SingleBook($bookId: ID!) {
  book(_id: $bookId) {
    _id
    title
  }
}
```

First we need to give the `query` a name. We called it `SingleBook`. We need to pass in the names of the `Query Variables` we want to use into that named `query`. We want to use the `bookId` variable, and we need to make sure we explicitly define what type we are expecting that variable to be, in our case it's a non-null `ID`. In the `query`, variables are defined as `$variableName`. Then we set the argument to our `book` query to that variable of `$bookId`. Give it a go!

Yay! We learned how to do a **named `query`** and how to pass in variables into a named `query`. 

## Performing a `Mutation`

GraphQL mutations enables us to manipulate the data in our backend.

Defining a mutation is similar to defining a query. We just need to put the `mutation` keyword at the very beginning of the definition, like so: 

```graphql
mutation {
  ...
}
```

By default, all GraphQL requests are queries unless specified, which is why we need to specify that it is a mutation. We can also be explicity and define queries like:

```graphql
query {
  ...
}
```

Take a look at the "DOCS" tab on the right of Playground to see all the `Mutations` we can try.

You'll notice that the `login` mutation takes in an `username` String input and a `password` String input as arguments. We define arguments as key-value pairs. Also, notice the value of the `login` mutation is `UserCredentials` instead of `User`. There are two extra fields, `token` and `loggedIn`, that we expect a `UserCredentials` type to have over a `User` type. 

Let's try logging in the demo user with a `username` of `"demo"` and a `password` of `"password"` and returning `_id`, `username`, `token`, and `loggedIn` on the `UserCredentials` type:

```graphql
mutation {
  login(username: "demo", password: "password") {
    _id
    username
    token
    loggedIn
  }
}
```

`token` should be a String that starts with `"Bearer "` and ends with a `jsonwebtoken`. We will be setting that `token` to the `"authorization"` header in all our future GraphQL requests requests so the server knows that the `loggedInUser` is the demo user. The second extra field, `loggedIn`, will not be used by us much right now, but will eventually be saved to our client to indicate on the client if a user is logged in or not.

Before we explore the other two mutations, let's try and see if our `token` works. Click on the `"HTTP HEADERS"` tab on the bottom left of Playground. Enter the following:

```json
{
  "authorization": "<the token you got back from logging in>"
}
```

Then, let's use the `me` query to see information about the `loggedInUser` and what books they have checked out. 

```graphql
{
  me {
    _id
    username
    books {
      _id
      title
    }
  }
}
```

The `me` query returns the `loggedInUser` that is identified through the `authorization` token. 

Now, let's check out some more books using the `borrowBooks` mutation. Take a look at the `"DOCS"` tab for information on that. The mutation takes in an array of `bookIds`. Find some books you want to check out in the `books` query and put their `_id`'s in a variable called `"bookIds"` in the `Query Variables`:

```json
{
  "bookIds": ["<book id 1", "<book id 2>"]
}
```

Next, let's define a **named `mutation`** called `CheckOutBooks` that takes in the `$bookIds` variable and expect it to be an array of type `ID`'s. The `borrowBooks` mutation returns fields of `success` of type `Boolean`, `message` of `String` and `books` of type `Book`. Let's pass in the variable `$bookIds` into the input `bookIds` argument for `borrowBooks`.

```graphql
mutation CheckOutBooks($bookIds: [ID]) {
  borrowBooks(bookIds: $bookIds) {
    success
    message
    books {
      _id
      title
    }
  }
}
```

Make sure to have the demo user's `token` in the `authorization` header when making this query.

When we define mutations, we usually define if the mutation was successful or not, an optional message, and the array of objects that have been mutated. 

Try doing the `returnBook` mutation on your own based on the above example (note: it only takes in a single `bookId`).

Awesome! We just finished going through all the mutations!

## Additional Notes

As you may have noticed and may have already tried to do, our schema architecture allows us to do crazy circular queries, like:

```graphql
{
  books {
    _id
    author {
      _id
      books {
        _id
        author {
          books {
            _id
          }
        }
      }
    }
  }
}
```

I would not suggest running this, but I'm sure some of you have already tried it. If you did run it, you'll notice that it takes a while for the query to run. These types of queries are expensive and are a real concern for production level GraphQL architectures. There are tons of resources out there describing this problem and what to do about it. If you are interested, you can take a look at [Securing Your GraphQL API from Malicious Queries].

## Conclusion

Queries and mutations give a lot of power to our client in determining what data we want to be sent from our server to our client. In future readings, we will be learning how to connect our client to our server using these GraphQL queries and mutations.

[GraphQL Playground]: https://github.com/prisma-labs/graphql-playground
[Book Lending Server Example]: https://github.com/ssoonmi/book-lending-server-example
[Securing Your GraphQL API from Malicious Queries]: https://blog.apollographql.com/securing-your-graphql-api-from-malicious-queries-16130a324a6b