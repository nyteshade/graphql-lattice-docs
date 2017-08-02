---
title: API GQL Lattice
language_tabs:
  - javascript
toc_footers:
  - '<a href=''mailto:admin@graphql-lattice.com''>Contact Author</a>'
  - '<a href=''https://github.com/tripit/slate''>Documentation Powered by Slate</a>'
includes:
  - errors
search: true
---

# Introduction

Welcome to the GraphQL Lattice API Docs!

GraphQL Lattice is designed to work with NodeJS and Express 4.x. Although it may be made to work in more places later if there is sufficient demand.

<aside class="warning">Lattice is a work in progress and if you start to consume it now, you should be aware that it will likely change rapidly over the near future.</aside>

# Attaching to Express

> When adding the endpoint, for now, you will need to import all the classes that extend `GQLBase` as well as `graphql-lattice` itself.

> A typical express route file looks like this

```javascript
const Express = require('express')
const { GQLExpressMiddleware } = require('graphql-lattice')
const SampleGQLBaseObj = require('./graph/SampleGQLBaseObj')

// Create an Express object to add to your app via app.use()
const router = Express();

// Instantiate a new instance of GQLExpressMiddleware. Currently it takes
// the class objects you've defined that extend GQLBase. 
// NOTE That it does not take an instancce of these classes but the class 
// or function itself.
const lattice = new GQLExpressMiddleware([
  SampleGQLBaseObj
]);

// Add the route using the simplest version
router.use('/graphql', lattice.middleware);
```

> In the future a version or function on the middleware will allow you to simply specify a directory in your project's filesystem and any and all GQLBase extended exports will be read automatically once the server starts.

You will likely be importing graphql-lattice in two types of places. The first will be wherever you are adding the express endpoint for graphql. The second place will be wherever you create your GraphQL objects.

# Creating your GraphQL Objects

> When writing a class that extends GQLBase, you can do so like the following

```javascript
const { GQLBase } = require('graphql-lattice');

class SampleGQLBaseObj extends GQLBase {
  // Guts of the GQLBase extension class (see below)
}

module.exports = SampleGQLBaseObj;
```

# GQLExpressMiddleware

## Create a new instance

> Create a new instance of GQLExpressMiddleware

```javascript
const { GQLExpressMiddleware } = require('graphql-lattice')

// Import/require all the GQLBase classes you've written

const lattice = new GQLExpressMiddleware([
  // Comma separate all the GQLBase'd classes that 
  // make up your graph.
]);
```

A handler that exposes an express middleware function that mounts a GraphQL I/O endpoint. For now, takes an Array of classes extending from GQLBase. These are parsed and a combined schema of all their individual schemas is generated via the use of ASTs. This is passed off to express-graphql.

As mentioned above, this approach is temporary and will quickly become a less desirable option once the module parser is complete. It will, however, remain for legacy and one-off usages

## Obtain the middleware used with Express

> As of this writing, GQL Lattice supports a few different ways to obtain the Express compatible middleware.

```javascript
// The most common
app.use('/graphql', lattice.middleware);
```

> Some folks will want a version that doesn't mount the GraphiQL explorer on POST

```javascript
// The second most common. 
app.get('/graphql', lattice.middleware);
app.post('/graphql', lattice.middlewareWithoutGraphiQL)
```

> Alternatively, if there are specific `express-graphql` options you wish to submit, you can create a custom middleware

```javascript
// Rare
// See https://github.com/graphql/express-graphql#options for more detail
const opts = { /* graphqlHTTP options */ }
app.use('/graphql', lattice.customMiddleware(opts))
```

> Finally, if you need to programmatically do something specific with the options AFTER GraphQL Lattice is done with them, you can specify a function to do so

```javascript
// Almost never happens
// See https://github.com/graphql/express-graphql#options for more detail
const opts = { /* graphqlHTTP options */ }
app.use('/graphql', lattice.customMiddleware(opts, function(opts) {
  // do something to the completed opts object
  // THEN return it when you're done making changes
  return opts;
}));
```

### middleware (getter)

This is by far the most common. It is identical to calling `customMiddleware({graphiql: true})`

### middlewareWithoutGraphiQL (getter)

This is used for those that wish to mount their `GET` and `POST` endpoints separately and do not want to allow the GraphiQL browser on POST.

### customMiddleware

If your needs require you to specify different values to graphqlHTTP, part of the express-graphql package, you can use the customMiddleware function to do so.

The first parameter is an object that should contain valid graphqlHTTP options. See <https://github.com/graphql/express-graphql#options> for more details. Validation is NOT performed.

The second parameter is a function that will be called after any options have been applied from the first parameter and the rest of the middleware has been performed. This, if not modified, will be the final options passed into graphqlHTTP. In your callback, it is expected that the supplied object is to be modified and THEN RETURNED. Whatever is returned will be used or passed on. If nothing is returned, the options supplied to the function will be used instead.

# GQLBase

> All your base GraphQL object types should extend from GQLBase. It provides a lot of consistency and allows the type to be automatically added to the generated schema.

```javascript
import { GQLBase } from 'graphql-lattice'
// or
const { GQLBase } = require('graphql-lattice')

export default class YourGQLType extends GQLBase {
  // Class contents
}

// Or alternatively you can use `module.exports = YourGQLType`
```

The `GQLBase` class should be the root of your business logic's GraphQL Types if you wish them to be used with GraphQL Lattice. The base class offers us several small advantages that we can take advantage of.

- It has a built in mechanism, standardized, to store our data model after fetching from whichever data source it lives in.
- It provides us a couple consistent ways to store and present our GraphQL schema for our objects
- It provides us a way to either import or define our query resolvers and mutation resolvers specific to our type

### Schemas

> Example showing a definition of a schema

```javascript
import { GQLBase } from 'graphql-lattice'

export class Sample extends GQLBase { 
  static get SCHEMA() { 
    return `
      type Sample { 
        id: ID! 
        name: String 
      }
    ` 
  } 
}
```

Every GraphQL object definition needs some associated GraphQL schema code. Lattice suggests the usage of the GraphQL SDL Schema Language to build your schema up.

### Resolvers

> Handling basic query resolvers

```javascript
import { GQLBase } from 'graphql-lattice'

export class Sample extends GQLBase {
  static get SCHEMA() {
    return `
      type Sample {
        id: ID!
        name: String 
      }

      type Query {
        exampleSample: Sample
      }
    `
  }

  static async RESOLVERS({req, res, gql}) {
    return {
      exampleSample: async () => {
        const model = await GetOurExampleModelData();
        return new Sample(model)
      }
    }
  }
}
```

> Example using a getter/function/async function and differences between that and something returned in RESOLVERS()

```javascript
// a comment
```

Typically when writing new GraphQL types using `GQLBase`, you will need to define a few things. One, you must define the schema for your type as well as any resolvers required for actions for your type. Resolvers are broken into two locations

1. In the object returned by the static function `RESOLVERS()`
2. As a `getter`, `function` or `async function` on the GraphQL type class itself.
