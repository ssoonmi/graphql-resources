# Apollo Client

Apollo Client gives us a very easy way to write and receive GraphQL requests and responses. 

Apollo Client works with many different frontend frameworks including React. 

With Apollo Client, there is no need for Redux because there is a built-in cache that Apollo will use to store the information from GraphQL queries. There is also no more need for Redux container components. Instead, all of the information from our server that we will be using inside of components will be coming from our Apollo cache.

![Image of GraphQL](https://ssoonmi.github.com/mern-graphql-curriculum/assets/graphql-with-store.png)

## Making a Query

Let's use our book lending store example from earlier which have the following GraphQL schema type definitions.

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

Let's say that we wanted to have a `BookIndex` page where we list all the books in our store. To do that, we would have to query our server with the `books` query. We would get back a list of books which we can then map to a list of books to display in React.

We will be using a React hook called `useQuery` from the `@apollo/react-hooks` package. This takes in a template literal wrapped in `graphlql-tag`, another package we will be using.

```javascript
// Book List component
import { useQuery } from '@apollo/react-hooks';
import gql from 'graphql-tag';

export default function BookList() {
  const { data, loading, error } = useQuery(gql`
    query GET_BOOKS {
      books {
        _id
        title
        isBooked
        author {
          _id
          name
        }
      }
    }
  `);
  // ...
}
```

`useQuery` takes in the GraphQL query and returns an object which have keys of `data`, `loading` and `error`. `loading` when the query is being run but hasn't come back yet. `error` returns an error if there was an error in the server or the network. `data` is what our query returns if it is successful. In this case, it will be an array of `books`.

When the query is loading or when there is an error, we can render something like:

```javascript
// Book List component
export default function BookList() {
  const { data, loading, error } = useQuery(GET_BOOKS);
  if (loading) return <p>Loading...</p>
  if (error) return <p>Error</p>
  // ...
}
```

However, when our `data` finally comes back from our server, we can successfully render our list of books:

```javascript
export default function BookList() {
  const { data, loading, error } = useQuery(GET_BOOKS);
  if (loading) return <p>Loading...</p>
  if (error) return <p>Error</p>
  return (
    <ul>
      {data && data.books && data.books.map(book => (
        <li key={book._id}>
          {book.title} by: {book.author.name}
        </li>
      ))}
    </ul>
  )
}
```

If there was another component that used this same query, then that component's data will come from the Apollo cache because the data already exists in our cache. If we want to always fetch from the network instead of the cache, we have to specify that as the second argument to our `useQuery` hook:

```javascript
const { data, loading, error } = useQuery(GET_BOOKS, { fetchPolicy: 'network-only' });
```

The default is `'cache-first'`, which means if the query exists in the cache, fetch it from the cache, if not, the network/server. 

If you want to know about other `fetchPolicy` options, check out the [React Apollo docs] and this article, [Apollo Client fetchPolicies, React, and Pre-Rendering].

## Mutations

Just like the hook, `useQuery`, to fetch queries from our server, there is a hook called `useMutation` which allows us to run mutations against our server in a component.

`useMutation` hook takes in the GraphQL mutation string and an optional second argument which is an options object. The hook returns an array. The first element in the array is a function, that when called, will run the mutation. The second element in the array is an object that contains the keys, `loading` and `error`, similar to `useQuery`'s `loading` and `error`, to indicate if the mutation is loading or if there was an error during it.

```javascript
// Return Book button component
function ReturnBook({ book }) {
  const [returnBook, { loading, error }] = useMutation(
    RETURN_BOOK,
    {
      variables: { bookId: book._id },
      refetchQueries: [
        {
          query: GET_BOOK,
          variables: { bookId: book._id }
        }
      ]
    }
  )
  return (
    <button disabled={loading} onClick={returnBook}>
      Return Book
    <button>
  )
}


// where RETURN_BOOK IS:
const RETURN_BOOK = gql`
  mutation ReturnBook($bookId: ID!) {
    returnBook(bookId: $bookId) {
      _id
      title
      isBooked
    }
  }
`;
```

There's a lot going on here. Let's start with the options object passed as the first argument the `useMutation`. The `RETURN_BOOK` variable is the mutation, `returnBook` which takes in an argument of `bookId` to indicate which book the user wants to return. 

The second argument to `useMutation` is an options object which has two keys, `variables` and `refetchQueries`. `variables` has an object as it's value whose key-value pairs define the variables for the mutation you want to run. `refetchQueries` array defines which queries you want to run again after the mutation is complete. In this case, it's refetching just the `query`, `GET_BOOK` for the information about the book being returned (notice how it has it's own `variables` key).

This is one way to pass `variables` mutation. The other way to pass in variables is by passing it into the mutation function, `returnBook` itself:

```javascript
// Return Book button component
function ReturnBook({ book }) {
  const [returnBook, { loading, error }] = useMutation(
    RETURN_BOOK,
    {
      refetchQueries: [
        {
          query: GET_BOOK,
          variables: { bookId: book._id }
        }
      ]
    }
  )
  // ...
  return (
    <button disabled={loading} onClick={() => returnBook({ variables: { bookId: book._id } })}>
      Return Book
    <button>
  )
}
```

`onClick` of the button to `Return Book`, we are passing a callback function that when called, will call the `returnBook` mutation function passing in the options object with the `variables`.

## Updating Our Cache

Great! So you might be wondering, how do we update the cache properly especially after a mutation? Well, there are several ways for several different situations.

### 1. After a Mutation Updating a Single Entity

If you are updating a single entity and are returning the `_id` properly from the mutation, then all our instances for that entity in our cache should be updated. There's no need to `refetchQueries`.

### 2. After a Mutation Updating Multiple Entities

If you are updating multiple entities at once, it doesn't matter if you are returning the `_id` property properly from all the entities, the cache for those entities will not reflect the update.

We need to `refetchQueries` which will have those entities in our cache.

Let's say we are running the mutation `borrowBooks` which is updating the `isBooked` key for multiple books. We need to refetch the query for `getBooks` to make sure our list of books gets updated with the right information. 

### 3. After a Mutation Creating or Deleting Entities

If we are creating or deleting single or multiples entities, then we need to write to the cache directly. The cache stores its data based on queries. We can grab information from our cache by `read`ing a query from the cache:

```javascript
const { books } = cache.readQuery({ query: GET_BOOKS });
```

To write to the cache, we have to also specify which query we are writing to:

```javascript
cache.writeQuery({
  query: GET_BOOKS,
  data: { books: newBooks }
});
```

Let's say we have an `ADD_BOOK` mutation and after we run it, we want to add that book to our `GET_BOOKS` query. We can use the `update` function in the options object to `useMutation` to define what we want to do after the mutation is run. 

```javascript
// AddBook Component
function AddBook() {
  const [title, setTitle] = useState('');
  const [addBook, { loading, error }] = useMutation(
    ADD_BOOK,
    {
      variables: { title },
      update: function(cache, { data: { addBook } }) {
        const { books } = cache.readQuery({ query: GET_BOOKS });
        cache.writeQuery({
          query: GET_BOOKS,
          data: { books: books.concat([addbook]) },
        });
      }
    }
  );
  return (
    <form onSubmit={addBook}>
      <input type="title" value="title" onChange={e => setTitle(e.target.value)}>
      <input type="Submit" disabled="loading" value="Create Book"/>
    </form>
  )
}
```

The `update` function takes in the `cache` as its first argument and the return value to the `addBook` mutation as its second argument. This return value has a `data` key with an object as its value. This `data` object has a key which is the name of the `mutation` run against the server which has the data returned from that mutation.

First, we `read` the `GET_BOOKS` query from our cache, which is an object with the data from that query stored in the name of the query, `books`. We write to the cache with that same query, but this time overwriting the `books` key to be a new array of `books` in our data that includes the old books and the new book we just created.

For deleting a book, we can define the update method like so:

```javascript
const [removeBook, { loading, error }] = useMutation(
  REMOVE_BOOK,
  {
    variables: { bookId: book._id },
    update: function(cache, { data: { removeBook } }) {
      const { books } = cache.readQuery({ query: GET_BOOKS });
      cache.writeQuery({
        query: GET_BOOKS,
        data: { books: books.filter((book) => book._id != removeBook._id ) },
      });
    }
  }
);
```

We use the filter method to return a new array of books that do not have the book that was removed.

### 4. Updating the Cache without a Mutation

To update the cache without having to define a mutation, we can use another hook called `useApolloClient`. `useApolloClient` returns the `client`. You can also access the `cache` on the `client` using `client.cache`, but this is pretty unnecessary as `client` has almost the same methods as `cache` with a little more functionality. We can call `readQuery` and `writeQuery` directly on the `client`. 

In this example, we will be using `writeData` instead. This `write`s directly to the `cache`. 

```javascript
import { useApolloClient } from '@apollo/react-hooks';
const LogoutButton = () => {
  const client = useApolloClient();
  return (
    <button
      onClick={() => {
        client.writeData({ data: { isLoggedIn: false } });
        localStorage.clear();
      }}
    >
      Logout
    </button>
  );
}

export default LogoutButton;
```

`client.writeData` will only manipulate the keys in the `cache` that is specified in the `data` key.

Those are all the different basic ways in which we will be interacting with the `cache`. There are many different reasons why you would want to interact with the `cache` directly. If you'd like to check out other ways to interact with it, check out the docs here, [Interacting with cached data].

There is some setup and finetuning involved when configuring the `client` and `cache` that we will get to in the projects.

[React Apollo docs]: https://www.apollographql.com/docs/react/api/react-apollo/#optionsfetchpolicy
[Apollo Client fetchPolicies, React, and Pre-Rendering]: https://dev.to/hotgazpacho/apollo-client-fetchpolicies-react-and-pre-rendering-1i77
[Interacting with cached data]: https://www.apollographql.com/docs/react/caching/cache-interaction/#cache-redirects-with-cacheredirects
