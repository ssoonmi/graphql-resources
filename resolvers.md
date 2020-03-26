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
  username: String!
  books: [Book]
}
type Query {
  books: [Book]
  book(_id: ID!): Book
  me: User
}
type Mutation {
  borrowBooks(bookIds: [ID]!): BookUpdateResponse!
  returnBook(bookId: ID!): BookUpdateResponse!
  login(username: String!, password: String!): User
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
    books(_, __) {
      return Book.find({});
    },
    book(_, { _id }) {
      return Book.findById(_id);
    },
    me(_, __, context) {
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
    borrowBooks(_, { bookIds }, context) {
      const loggedInUser = context.user;
      if (!loggedInUser) return {
        success: false,
        message: 'Need to log in to borrow books',
        books: []
      }
      return Book.borrowBooks(bookIds, loggedInUser);
    },
    returnBook: async (_, { bookId }, context) => {
      const loggedInUser = context.user;
      if (!loggedInUser) return {
        success: false,
        message: 'Need to log in to return books',
        books: []
      }
      const book = await Book.findById(bookId);
      return book.returnBook(loggedInUser);
    },
    login(_, { username, password }) {
      // login method used in MERN project
      return User.login(username, password);
    }
  }
}
```

We recommend **keeping your resolvers "thin"** as a best practice. Any logic should be defined as a function on a document or model in mongoose. So the `borrowBooks` and `returnBook` resolvers' logic should be defined on the Book schema (see below), not in the resolvers.

```javascript
// in BookSchema definition for mongoose

// statics on a schema are functions that be called on the model itself (e.g. Book.borrowBooks(...))
BookSchema.statics.borrowBooks = function (bookIds, loggedInUser) {
  const Book = this; // this is the Book model
  return (async () => {
    const books = [];
    const alreadyBookedBookIds = [];
    // for each book id, find the book
    // if the booked is not checked out, mark it as booked and add it to the user's list of books
    // if it is checked out, user cannot check it out
    for(let i = 0; i < bookIds.length; i++) {
      const bookId = bookIds[i];
      const book = await Book.findById(bookId);
      if (book.isBooked === false) {
        book.isBooked = true;
        loggedInUser.books.addToSet(bookId);
        books.push(await book.save());
      } else {
        alreadyBookedBookIds.push(bookId);
      }
    }
    await loggedInUser.save();
    const success = books.length === bookIds.length;
    const message = success
      ? 'books checked out successfully'
      : `the following books could not be checked out: ${alreadyBookedBookIds}`;
    return {
      success,
      message,
      books
    };
  })();
}
// methods on a schema are functions that can be called on a document of the model (e.g. book.returnBook(...))
BookSchema.methods.returnBook = function (loggedInUser) {
  const book = this; // "this" is the book document
  return (async () => {
    if (loggedInUser && loggedInUser.books.includes(this._id)) {
      loggedInUser.books.remove(this._id);
      await loggedInUser.save();
      book.isBooked = false;
      return {
        success: true,
        message: 'book returned',
        books: [await book.save()]
      };
    } else {
      return {
        success: false,
        message: `book with id ${this._id} could not be returned`,
        books: null
      }
    }
  })();
}
```

## Object Type resolvers

We can write resolvers on any type in GraphQL, not just for queries and mutations. 

We usually write resolvers on an Object Type when the field needs to be displayed in a way that is not readily available from the data that is returned from the resolvers for that type.

For example, the resolvers for `User` type in the online book lending example would look something like: 

```javascript
const resolvers = {
  // ... Query resolvers
  // ... Mutation resolvers
  User: {
    books: async (parentValue, _, context) => {
      const queriedUser = parentValue;
      const loggedInUser = context.user;
      // only return the borrowed books of a user if the queried user is the logged in user
      if (loggedInUser && queriedUser._id === loggedInUser._id) {
        await loggedInUser.populate('books').execPopulate();
        return loggedInUser.books;
      }
      return null;
    }
  }
};
```

We write a resolver for `books` on the `User` type because the `books` are stored as an array of ObjectID's on our MongoDB, not as the array of `Book`s themselves. We use the `books` resolver to convert the array of ID's on the key of `books` of a `User` into an array of `Book`s;

The first argument, the parentValue, is the user that is being queried for. The `loggedInUser` is the authorized user saved on the `context`. We will only return the borrowed books of a `queriedUser` if that user is the `loggedInUser`. 

Let's now consider the type of `Author` with the field of `books`. Let's say that we don't even store the `books` as an array of ID's on an `author` in our database. We can still create a resolver for the key of `books` on the type of `Author` like so:

```javascript
const resolvers = {
  // ... Query, Mutation, and User resolvers
  Author: {
    books(parentValue, _) {
      const author = parentValue;
      // find all the books who have the queried author as their author
      return Book.find({ author: author._id });
    }
  }
};
```

Here, we are returning all the books who have the queried `author`'s `_id` (the `parentValue`) as the key of `author`.

## Conclusion

Previously, we learned how to create type definitions for our GraphQL schema. 

Here, we learned how to create resolvers for each of the `Query` and `Mutation` fields as well as specific fields on Object Types. These **resolvers enable GraphQL queries and mutations to access the database**. 

Next, we will be learning **how to formulate and send queries and mutations from our client to our server**. 

Next resource: [Formulating GraphQL Queries and Mutations]

[Formulating GraphQL Queries and Mutations]: /formulating_queries_and_mutations.md