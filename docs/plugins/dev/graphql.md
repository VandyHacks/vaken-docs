#GraphQL

GraphQL is the framework that Vaken uses for its API. To add new endpoints to the API, there are multiple that must be changed.
In this tutorial, we'll go over creating a new API endpoint in the GraphQL framework via Vaken plugins. All of the files mentioned here are found in the plugins folder.


##Schema.graphql.ts

This is where all of the GraphQL objects (types, queries, mutations) are initially defined. Follow the GraphQL conventions (linked below) to ensure that your object types and properties are correct. A user defined type is a type with properties that are defined by the user. This is an example of creating a user defined type:

```typescript
type _Plugin__EventCheckIn  @entity {
    id: ID! @id @column
    user: String! @column
    timestamp: Int! @column(overrideType: "Date")
}
```

There are 3 properties for this type, which are int, user, and timestamp. These properties have their own types associated with them, which are ID, String, and Int respectively. You can also have a property which has a type of another user defined type. This allows for nesting of types and can be very useful. 

The next type of schema you may define is a query. This looks like:

```typescript
_Plugin__eventCheckIn(id: ID!): _Plugin__EventCheckIn! 
_Plugin__eventCheckIns(sortDirection: SortDirection): [_Plugin__EventCheckIn!]!
```

A query takes input and queries the internal MongoDB collection using the input as a filter. Each user defined type has its own MongoDB collection, which is automatically generated with the type definition. There are two queries here, which are slightly different. This first query takes an ID and returns a single object (with the type we just defined). The second query has an ‘s’ at the end of the name, and takes a sort direction instead. This query returns an array with ALL instances of the object, sorted by the specified sort direction. 

Another commonly used schema is an input. An input defines the parameters and types of a user input for your API. An input looks like:

```typescript
input _Plugin__EventCheckInInput {
    user: ID!
    event: ID!
}
```

This input takes two parameters, both of which are type ID. These inputs are used for the Mutations. A Mutation takes an input and performs an action using the parameters. An example of this is:

```typescript
extend type Mutation {
    _Plugin__checkInUserToEvent(input: _Plugin__EventCheckInInput!): User!
}
```

This mutation uses our previously defined input as the parameter for this function. The function definition is mentioned later, in the server.ts file. 


To learn more about what some of these types and annotations mean, https://graphql.org/learn/schema/  and https://graphql-code-generator.com/docs/plugins/typescript-mongodb are great resources. 


##Client.ts

This dictates what the users see when they visit the vaken website. To integrate a new component (React page), follow this example:

```typescript
export class NFCPlugin {
  get routeInfo() {
    return {
      displayText: "Scan NFC (Plugin Version)",
      path: "/nfc",
      authLevel: [UserType.Organizer]
    }
  }

  async component() {
    return await import('./components/Nfc')
  }
}

export default {
  NFCPlugin
}
```
Let’s break this down. The routeInfo function returns the route that links a user to your component. This path is seen from the path parameter. The auth level specifies which level of access is needed to view this component. In this instance, a user needs to be an organizer to access this route. The component() function imports the user-defined React component and returns it. The export at the bottom of the file is just allowing other files to import this NFCPlugin class. 

###Server.ts

This file is where you put all of the GraphQL resolvers for your plugin. A resolver is essentially a function that executes the GraphQL request sent to the server. This is where most of the logic for the plugin will go. 

The first type of resolver is related to the GraphQL schema for the user-defined types. You can think of these as getters.

```typescript
_Plugin__EventCheckIn: {
        id: async eventCheckIn => (await eventCheckIn)._id.toHexString(),
        timestamp: async eventCheckIn => (await eventCheckIn).timestamp.getTime(),
        user: async eventCheckIn => (await eventCheckIn).user,
}
```

For example, this resolver fetches the attributes associated with the object. The logic itself isn’t very complex but this is still very important. For those unfamiliar, async-await is a way of fetching properties and waiting until the fetch is complete before continuing with execution. This is especially useful when making API calls because it's pointless to try and use the API return value before the call is complete.

The next type of resolver is a query. Queries query the database, in this case MongoDB, using the parameters passed to the GraphQL query. This is similar to a GET request in REST.

```typescript
_Plugin__eventCheckIns: async (root, args, ctx) => {
          checkIsAuthorizedArray([UserType.Organizer], ctx.user);
          return ctx.db.collection<_Plugin__EventCheckInDbObject>('_Plugin__eventCheckIns').find().toArray();
}
```

In this function, the first thing we do is authenticate that the user has the correct permissions, in this case an organizer. It’s VERY important to do this type of validation so that unauthorized users are prevented from querying the database freely. This function returns the result of a query of the database associated with the GraphQL type. Whenever we define a type in the GraphQL schema, GraphQL generates a MongoDB collection for that type using the type’s properties as keys. This collection can be accessed via the name ‘_Plugin__<Type Name>DbObject’. 

The final relevant type of resolver is a Mutation. A Mutation is a way to modify the data contained in the databases. This is similar to a POST request in GET.

```typescript
Mutation: {
        _Plugin__checkInUserToEvent: async (root, { input }, { models, user }) => {
          checkIsAuthorizedArray([UserType.Organizer, UserType.Volunteer, UserType.Sponsor], user);
          return checkInUserToEvent(input.user, input.event, models);
        },
}
```

This mutation uses the input (from the schema) as parameters for the checkInUserToEvent. In this case, both the user and event properties from the input are needed. 

With all of this code written, we should now be able to access our API using GraphQL notation.
For example, to check a user *123* into event *456* and then display the type of the user, the GraphQL notation would be:

```
mutation {
     _Plugin__checkInUserToEvent(input: {
       user: 123
       event: 345
     }) {
       userType
     }
}
```

An example query we might run is:
```
query {
  _Plugin__eventCheckIns {
    user
  }
}
```

This makes a call to the query we defined before that returns all of the users in the collection associated with the eventCheckIn type.
