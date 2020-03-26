# Apollo GraphQL example with MERN

## Server Set Up
Create a folder in your project called `server`. `cd` into the `server` folder.

Run `npm init --y` to create the `package.json` file in your `server` folder.

### Packages to Install

`npm install` the following dependencies: 

- express
- mongoose
- passport
- passport-jwt
- jsonwebtoken
- body-parser
- bcryptjs
- validator
- express-graphql
- graphql
- graphql-tools
- graphql-playground-middleware-express
- morgan

`npm install -D` the following development dependencies:

- nodemon
- cors

## Client Set Up
Run `create-react-app client` and `cd` into the newly created `client` folder.

### Packages to Install

`npm install` the following dependencies in the `client` folder: 

- apollo-client
- apollo-cache-inmemory
- apollo-link-http
- @apollo/react-hooks
- react-apollo
- graphql
- graphql-tag
- react-router-dom

### GraphQL IDE for better development
GraphQL Playground is a GraphQL IDE that we will implement using `graphql-playground-middleware-express`.

Better than the default GraphQL IDE because you can define authorization headers. (Plus it has prettier colors!)

Link to reference: [https://github.com/prisma-labs/graphql-playground]:(https://github.com/prisma-labs/graphql-playground)


### Modularizing GraphQL Schema
Using `makeExecutableSchema` with `graphql-tools`.

Link to reference: [https://blog.apollographql.com/modularizing-your-graphql-schema-code-d7f71d5ed5f2](https://blog.apollographql.com/modularizing-your-graphql-schema-code-d7f71d5ed5f2)