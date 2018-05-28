
# Graph SQL Cursor pagination helper

## Motivation
This library is extension of awesome [graphql-relay-js](https://github.com/graphql/graphql-relay-js) which solve
problem how simply implement `cursor pagination` to your SQL's dbs query without special magic.

If you're creating GraphQl schema which is compatible with relay cursor pagination
([cursor pagination spec](https://www.google.cz/search?q=cursor+pagination+spec&oq=cursor+pagination+spec&aqs=chrome..69i57.2610j0j4&sourceid=chrome&ie=UTF-8) ).
You probably use [graphql-relay-js](https://github.com/graphql/graphql-relay-js) library for helper functions to define your schema.

Problem with `graphql-relay-js` is that functions like `connectionFromArray(...)` works on application javascript layer. That means that process of pagination is inefficient and it takes too much RAM & CPU power.

This library solve how to recalculate Pagination args (`first` & `after` & `last` & `before`) to SQL's `OFFSET` `LIMIT` params. 

### Another benefics
We added `totalCount` field to graphQl schema. At the moment `graphql-relay-js` does not implement it yet.
We decided that `totalCount` is productive for and you have option to set it.

[Why is not totalCount in graphql-relay-js (issue) ](https://github.com/graphql/graphql-relay-js/pull/205)

If you want to enable `totalCount` you have to set it to your `connectionDefinition` like in example:

```javascript 
connectionDefinitions({
  name: 'User',
  nodeType: ChatMessageType,
  // adding connectionFields to type
  // is necessary for quering totalCount in graphQl
  connectionFields: {
    totalCount: { type: new GraphQLNonNull(GraphQLInt) }
  }
}).connectionType,
```
*note: connectionField param of connectionDefinition is not documented in readme yet. But you can find implementation here: [implemenation of connectionFields in relay-graphql-js ](https://github.com/graphql/graphql-relay-js/blob/4fdadd3bbf3d5aaf66f1799be3e4eb010c115a4a/src/connection/connection.js#L89)*
## Instalation 
```bash
yarn add graph-sql-cursor-helper
```
or
```bash
npm install graph-sql-cursor-helper --save
```

## API
library provide this function:
- `connectionFromPromisedSqlResult`

### `connectionFromPromisedSqlResult(...)`
#### Params
 - *totalCount* - count of line for whole SQL SELECT 
 - *paginationArgs* - provided from `connectionArgs` fn from `graphql-relay-js`
 - *callbackWithValues* - callback return object with `offset` and `limit` computed values

#### return
 - Connection with sql results & totalCount

#### Example
##### Raw function API example

```javascript
// graphql resolver
// ...
return connectionFromPromisedSqlResult(
  // this return totalCount for current select
  await someOrmSqlModel.getTotalCount(), 
  paginationArgs,
  // get computed values for sql pagination
  ({ offset, limit }) => (
    // applying it to sql select for native sql pagination
    someOrmSqlModel.getMessages({ offset, limit })
  )
)
```

##### Example with graphQl resolver for users

```javascript
const Admin = new GraphQLObjectType({
  name: 'Admin',
  description: '...',
  interfaces: [nodeInterface],
  isTypeOf: obj => instanceof sqlModelAdmin,
  fields: () => ({
    /*
           _          _                          _          _  
     >(')____,  >(')____,   ... some code ...  >(')____,  >(') ___, 
       (` =~~/    (` =~~/   ... some code ...   (` =~~/    (` =~~/ 
    ~^~~^~^`---'~^~^~^`---'~^~^~^`---'~^~^~^`---'~^~^~^`---'~^~^~ 
    */
    users: {
      type: connectionDefinitions({
        name: 'User',
        nodeType: UserType,
        // adding connectionFields to type
        // is necessary for quering totalCount in graphQl
        connectionFields: {
          totalCount: { type: new GraphQLNonNull(GraphQLInt) }
        }
      }).connectionType,
      args: connectionArgs,
      description: `...`,
      resolve: async (parent, paginationArgs) => {

        // call `graphql-sql-cursor-helper` function for calculating real OFFSET & LIMIT
        return connectionFromPromisedSqlResult(
          await someOrmSqlModel.getTotalCount(), // <- this 
          paginationArgs,
          // get computed values for sql pagination
          ({ offset, limit }) => (
            // applying it to sql select for native sql pagination
            someOrmSqlModel.getMessages({ offset, limit })
          )
        )
      }
    }
  })
})
```