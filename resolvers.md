# GraphQL Resolvers

Even though we have built our GraphQL Schema, we still cannot run queries or mutations against our GraphQL API.

**Resolvers** provide the instructions for turning a GraphQL operation, a **Query**, a **Mutation**, or a **Subscription** (we will focus on `Query` and `Mutation` only) into data. They either return the same type of data we specify in our schema or a promise for that data.

A **resolver** is a function that returns data. It accepts four arguments in this order, `parent`, `args`, `context`, and `info`.
```javascript
function resolver(parent, args, context, info) {
  return data
}
```
- `parent`: An object that contains the **result returned from the resolver on the parent type**
- `args`: An object that contains the arguments passed into the field
- `context`: An object shared by all resolvers in a GraphQL operation. We usually use the HTTP request as our context and put per-request state such as authentication information
- `info`: (RARELY USED) Information about the execution state of the operation which should only be used in advanced cases

We will be using the type definitions for the online book lending schema example from the previous reading:
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
type Query {
  books: [Book]
  book(_id: ID!): Book
  me: User
}
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

## `Query` resolvers

`Query` resolvers are functions run when the client executes a specific query. There should be one resolver for every `Query` type fields. They should be defined in the `Query` key in your resolvers.

For example, the resolvers for `Query` type in the online book lending example would look something like: 
```javascript
const resolvers = {
  Query: {
    books: (_, __) {
      return Book.find({});
    },
    book: (_, { _id }) {
      return Book.findById(_id);
    },
    me: (_, __, context) {
      // context.user is the logged-in user
      return context.user;
    }
  }
}
```
The first argument to our **top-level resolvers**, the `parent`, is always blank because it refers to the root of our graph, and, therefore, has no `parent`s. Any `Query` or `Mutation` field resolver is a top-level resolver.

The second argument refers to any `arguments` passed into our query, which is defined by our type definition. 

The third argument is the `context` which holds the HTTP request information and authentication information.

The return value of our resolvers is information extracted from our MongoDB.

## `Mutation` resolvers

Writing `Mutation` resolvers is similar to the `Query` resolvers. They should be defined on the `Mutation` key of your resolvers.

For example, the resolvers for `Mutation` type in the online book lending example would look something like: 
```javascript
const resolvers = {
  // ... Query resolvers
  Mutation: {
    borrowBooks: async (_, { bookIds }, context) => {
      const loggedInUser = await context.user;
      const books = [];
      const uncheckedBookIds = [];
      bookIds.forEach(async (bookId) => {
        const book = await Book.findById(bookId);
        if (book.isBooked = false) {
          book.isBooked = true;
          loggedInUser.books.addToSet(bookId);
          books.push(await book.save());
        } else {
          uncheckedBookIds.push(bookId);
        }
      });
      await loggedInUser.save();
      const success = books.length === bookIds.length;
      const message = success 
        ? 'books checked out successfully' 
        : `the following books could not be checked out: ${uncheckedBookIds}`;
      return { 
        success,
        message,
        books
      };
    },
    returnBook: async (_, { bookId }, context) => {
      const loggedInUser = await context.user;
      const book = await Book.findById(bookId);
      loggedInUser.books.remove(bookId);
      await loggedInUser.save();
      book.isBooked = false;
      await book.save();
      return { 
        success: true,
        message: 'book returned',
        books: [await book.save()]
      };
    },
    login: (_, { email, password }) => {
      return User.login(email, password);
    }
  }
}
```
We recommend keeping your resolvers "thin" as a best practice. Any logic should be defined as a function on a document or model in mongoose. So the `borrowBooks` and `returnBook` resolvers' logic should have been defined on the Book schema, not in the resolvers.

## Object Type resolvers

We can write resolvers on any type in GraphQL, not just for queries and mutations. 

We usually write resolvers on