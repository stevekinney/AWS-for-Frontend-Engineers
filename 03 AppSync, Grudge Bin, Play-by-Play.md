Okay, we're going to start by branching off of `master` again.

We're going to do a `awsmobile init` again.

But this time, we're going to take things a bit differently.

Yes, `awsmobile` supports AppSync. But, it gives you a default schema.

We're going to take a look at this default schema, but we're not going to us it.

Instead, we're going to mess around with my Grudge List again.

Let's make a new branch and we can fire up `awsmobile init` again.

Cool. Now, we'll flip over to the AWS Console.

https://console.aws.amazon.com/appsync/home

You may or may not see some APIs here already—you probably don't. Let's go ahead and create an API.

Let's start by looking at the example schema. We're _not_ going to create it.

We're just going to look at it.

Cool—flip it back to "Custom Schema."

We kind of already know the shape of a grudge. We've worked with one before, right?

Once this is set up—let's go to schema.

We can start with something pretty simple for now.

```gql
type Grudge {
  id: ID!
  person: String!
  deed: String!
  avenged: Boolean
}

type Query {
  fetchGrudges: [Grudge]!
}
```

This isn't disimilar from how we set up that DynamoDB earlier.

(Fun fact: our GraphQL API is going to be backed by DynamoDB.)

Hit "Save."

Boom! Our schema is in place.

Earlier we had the problem where we had a database and no API. Now, we have an API and no database.

Let's rectify that. Hit "Create Resources."

Not only will this create a DynamoDB table on our behalf, but it's definitely going to flesh out our schema for us as well.

Hit "Create." This page is very serious about us not refreshing. It will only take a minute or two though.

We just have one thing we need to adjust. Amazon made a much better way to get grudges: `GrudgeConnection`.

This is basically what we did, but it supports pagination. This is a good thing if our list of grudges grows really long—and let's face it…

Let's hop over to the Queries tab. (Is it technically a tab? I don't know.)

Let's try some stuff out—shall we?

Again, we have that problem where there is nothing in our database. So, let's start with a mutation.

```gql
mutation CreateGrudge {
  createGrudge(input: {
    person: "Tanner",
    deed: "Making me use my old MacBook",
    avenged: false
  }) {
    id
    person
    deed
    avenged
  }
}
```

This will create a new grudge and then basically give me it back from the database.

Let's query for it, just to make sure I'm not some kind of charlatan.

```gql
query GetGrudge {
  getGrudge(id: "123") {
    id
    person
    deed
    avenged
  }
}
```

Now, let's slowly start taking away properties and prove that it all works.

Cool, cool. **Your turn**: Make a few (at least two) more grudges.

Welcome back. Now I'll make two more real quick.

Let's look at that `GrudgeConnection`, shall we?

We'll start simple:

```gql
query GetAllOfTheGrudges {
  listGrudges{
    items {
      person
    }
  }
}
```

So, we got everyone. But what about that `nextToken` thing we saw before? Why didn't we get it?

We didn't get it because we didn't ask for it—silly.

```gql
query GetAllOfTheGrudges {
  listGrudges {
    items {
      person
    }
    nextToken
  }
}
```

Okay, so we got it—but why is it `null`?

Simple answer: we're at the end of the list.

Let's give it an argument.

```gql
query GetAllOfTheGrudges {
  listGrudges(limit: 2) {
    items {
      person
    }
    nextToken
  }
}
```

Cool, we got the first two. And if we use that token? We'll pick up where we left off.

```gql
query GetAllOfTheGrudges {
  listGrudges(limit: 2, nextToken: "eyJ2ZXJzaW9uIjoxLCJ0b2tlbiI6IkFRSUNBSGg5OUIvN3BjWU41eE96NDZJMW5GeGM4WUNGeG1acmFOMUpqajZLWkFDQ25BSEhIdVpyZlJGVlVQT3FxMHB2UXRPd0FBQUJ2akNDQWJvR0NTcUdTSWIzRFFFSEJxQ0NBYXN3Z2dHbkFnRUFNSUlCb0FZSktvWklodmNOQVFjQk1CNEdDV0NHU0FGbEF3UUJMakFSQkF3NnJRTjJIc1V5RXgybWVPQUNBUkNBZ2dGeHRkbmxrOTdrTnVEUzJNcldoekU1alpaMnY5Vy9xWEEydzkrbUw3QjRkU0ZjWUFVeWZ1aVJZTHc5TFNqbkZXa3BhS3N1US9sZTh6ZUs1RTEwdzRKQmtJMGNwLzRvNWFKbVZxUTlvRlVHRjJXRTg2LzkzTVFVVEJLbFBOdEZ3SWN2MUlaUk00Nk1kclBrNDI0WmR6eVg3Qjh3a2JBRUlwdTdXUTl2TDAxa1N5UDZkLzBDSTk2SWh5MUNoZ0JtWjhOenZvZWtJTjdFQ2c0bGRPZ1QrR0xWdzFwSWl1Vmw4MGQ3cFpHa1RjVEdaNnZNRENLbXFyaURwWVNIeHA3c0lnVklKNUVyS2s3bWx2ZENSVXBqOStoRWQweVNLMVowTVU1OERtK2E3L1p3SWRuSElkcVZWZGlmLzBpQ0dtdWNWc1hTRFE5SGQ3ZDBRbHFrT3BBbnNoLzlnSHMxSW1oU1d5S0lqamM1Y0Y4TEIrY2ppOHVPVFZ1U3ZJck83MHMzeEZtUHRhdFZYM2x1d2x2eGhnL3hhYXg5bTdUNGVvUCt2c2toMlFHc3NxQXd5RnBEOStiQkZDN2ZFVVdIRWdlcnE0TVlQWDVxUUFhaWZUT0NwTFViQzVJVk9iekNzdTJtWXlRRUxTMEZ1cUZwdUl0bCJ9") {
    items {
      person
    }
    nextToken
  }
}
```

This is all pretty impressive, but I think it's more impressive to point out that we didn't really do any work to make any of this happen. Crazy, right?

Anyway—we were working on an application, right? Let's flip back to that.

Let's make a new file called `graphql.js`. This is going to be where we put our queries. If this application was more complicated, then we'd need to figure out a more complicated folder setup, but it's not—so, here we go.

I tricked y'all into putting some into the database, so—this time we can start by listing all of them out.

```js
export const ListGrudges = `
  query ListGrudges {
    listGrudges {
      items {
        id
        person
        deed
        avenged
      }
    }
  }
`;
```

Cool. So, that's just a string. We need to do some plumbing with Amplify and AWS Mobile Hub to get this all working.

Click on the name of your API and scroll to the bottom. We're web developers, so go there and download the configuration file.

It should look something like this:

```js
export default {
	"graphqlEndpoint": "https://lrthbjmnbvgivd7telzgoasod4.appsync-api.us-east-1.amazonaws.com/graphql",
	"region": "us-east-1",
	"authenticationType": "API_KEY",
	"apiKey": "da2-em5xdilte5farhzs63grtr2u2m"
}
```

(Don't troll me.)

In order to make it clear to Amplify what we're talking about—we need to modify this a little bit.

```js
const AppSyncConfiguration = {
  "aws_appsync_graphqlEndpoint": "https://lrthbjmnbvgivd7telzgoasod4.appsync-api.us-east-1.amazonaws.com/graphql",
  "aws_appsync_region": "us-east-1",
  "aws_appsync_authenticationType": "API_KEY",
  "aws_appsync_apiKey": "da2-em5xdilte5farhzs63grtr2u2m"
}
```

We're effectively just prefixing everything with `aws_appsync_`. You can put it in it's own file if you want, but I'm not going to.

I'm just going to pop it into `index.js`,

```js
import Amplify, { Analytics } from 'aws-amplify';
import configuration from './aws-exports';

const AppSyncConfiguration = {
  "aws_appsync_graphqlEndpoint": "https://lrthbjmnbvgivd7telzgoasod4.appsync-api.us-east-1.amazonaws.com/graphql",
  "aws_appsync_region": "us-east-1",
  "aws_appsync_authenticationType": "API_KEY",
  "aws_appsync_apiKey": "da2-em5xdilte5farhzs63grtr2u2m"
};

Amplify.configure({ ...configuration, ...AppSyncConfiguration });
Analytics.disable();
```

Cool. We have no idea if it works yet, but we did make that nifty query earlier.

We're going to need that API helper along with a helper that turns our string of GraphQL into a real query.

```js
import { API, graphqlOperation } from 'aws-amplify';
import { ListGrudges } from './graphql';
```

Let's start by fetching them and then seeing what our data look like.

```js
componentDidMount() {
  API.graphql(graphqlOperation(ListGrudges)).then(response => {
    console.log({ response });
  });
}
```

It looks like the real meat is at `response.data.listGrudges.items`.

This isn't so crazy, because it's exactly what we saw before.

```js
componentDidMount() {
  API.graphql(graphqlOperation(ListGrudges)).then(response => {
    this.setState({ grudges: response.data.listGrudges.items });
  });
}
```

Sweet—let's add some grudges!

This is going to be the first time we're trying to actually be a bit dynamic. We cheated when we were working on this earlier by hard coding in everything.

Let's use one of our previous mutations and make it a bit more flexible.

```js
export const CreateGrudge = `
  mutation CreateGrudge(
    $person: String!
    $deed: String!
    $avenged: Boolean!
  ) {
    createGrudge(input: {
      person: $person,
      deed: $deed,
      avenged: $avenged
    }) {
      id
      person
      deed
      avenged
    }
  }
`;
```

What does that look like in our React application?

```js
addGrudge = grudge => {
  API.graphql(graphqlOperation(CreateGrudge, grudge)).then(response => {
    const newGrudge = response.data.createGrudge;
    this.setState({ grudges: [newGrudge, ...this.state.grudges] });
  });
};
```

Take it for a spin.

**Your Turn**: You should have seen this coming. Your job is to implement updating and deletion of grudges.

Tasting notes:

- Look at the schema before you throw code at the wall.
- You'll need to make two new mutations.

```js
export const deleteGrudge = `
  mutation deleteGrudge($id: ID!) {
    deleteGrudge(input: {
      id: $id,
    }) {
      id
    }
  }
`;

export const updateGrudge = `
  mutation updateGrudge($id: ID!, $person: String!, $deed: String!, $avenged: Boolean!) {
    updateGrudge(input: {
      id: $id,
      person: $person,
      deed: $deed
      avenged: $avenged,
    }) {
      id
      person
      deed
      avenged
    }
  }
`;
```

And the React?

```js
removeGrudge = (grudge) => {
  API.graphql(graphqlOperation(DeleteGrudge, grudge))
    .then(response => {
      const id = response.data.deleteGrudge.id;
      this.setState({
        grudges: this.state.grudges.filter(other => other.id !== id),
      });
    })
    .catch(console.error);
};

toggle = (grudge) => {
  const updatedGrudge = { ...grudge, avenged: !grudge.avenged };
  API.graphql(graphqlOperation(UpdateGrudge, updatedGrudge))
    .then(response => {
      this.setState({
        grudges: this.state.grudges.map(g => {
          if (g.id !== response.data.updateGrudge.id) return g;
          return response.data.updateGrudge;
        })
      })
    })
    .catch(console.error);
};
```

Cool! We've got some CRUD.

But we saw something about subscriptions before and I made some promises about "real-time".

Let's look at them in our schema.

(Welcome back.)

Let's write a subscription!

```js
export const SubscribeToNewGrudges = `
  subscription onCreateGrudge {
    onCreateGrudge {
      id
      person
      deed
      avenged
    }
  }
`;
```

Whenever a client anywhere makes a new grudge—we want to be notified.

(We're all under one big happy account anyway, so whatever.)

Cool, let's set that up for when a component mounts.

```js
componentDidMount() {
  API.graphql(graphqlOperation(ListGrudges)).then(grudges => {
    this.setState({ grudges: grudges.data.listGrudges.items });
  });

  API.graphql(graphqlOperation(SubscribeToNewGrudges)).subscribe({
    next: (response) => {
      const grudge = response.value.data.onCreateGrudge;
      this.setState({ grudges: [...this.state.grudges, grudge] });
    },
  });
}
```

We have a minor bug. We get everything twice. Now that our subscription works.

Let's _not_ do it manually.

```js
addGrudge = grudge => {
  API.graphql(graphqlOperation(CreateGrudge, grudge));
};
```

(Open multiple browsers and verify that this works.)

**Your Turn**: What do you think I'm going to ask you to do?

Yea, that's right—implement update and delete.

```js
export const SubscribeToUpdatedGrudges = `
  subscription onUpdateGrudge {
    onUpdateGrudge {
      id
      person
      deed
      avenged
    }
  }
`;

export const SubscribeToDeletedGrudges = `
  subscription onDeleteGrudge {
    onDeleteGrudge {
      id
      person
      deed
      avenged
    }
  }
`;
```

And in React?

```js
API.graphql(graphqlOperation(SubscribeToUpdatedGrudges)).subscribe({
  next: (response) => {
    const updatedGrudge = response.value.data.onUpdateGrudge;
    const grudges = this.state.grudges.map((grudge) => {
      if (grudge.id !== updatedGrudge.id) return grudge;
      return updatedGrudge;
    });
    this.setState({ grudges });
  },
});

API.graphql(graphqlOperation(SubscribeToDeletedGrudges)).subscribe({
  next: (response) => {
    const deletedGrudge = response.value.data.onDeleteGrudge;
    const grudges = this.state.grudges.filter(grudge => grudge.id !== deletedGrudge.id);
    this.setState({ grudges });
  },
});
```

Neat—we did it.

You know I have a pet peeve for generating my own IDs on the client. You'd hate it even more if you saw how I was doing it.

Let's mess around a little bit with resolvers.

Let's get an automatically generared ID.

Here is the default for `Mutation.createGrudge`.

```json
{
  "version": "2017-02-28",
  "operation": "PutItem",
  "key": {
    "id": { "S" : "${util.autoId()}" },
  },
  "attributeValues": $util.dynamodb.toMapValuesJson($ctx.args.input),
  "condition": {
    "expression": "attribute_not_exists(#id)",
    "expressionNames": {
      "#id": "id",
    },
  },
}
```

I'm a bad person and I was sorting on the client by the `id`. So, let's get a date too.

```js
{
  "version": "2017-02-28",
  "operation": "PutItem",
  "key": {
    "id": { "S" : "${utils.time.nowEpochMilliSeconds()}-${util.autoId()}" },
  },
  "attributeValues": $util.dynamodb.toMapValuesJson($ctx.args.input),
  "condition": {
    "expression": "attribute_not_exists(#id)",
    "expressionNames": {
      "#id": "id",
    },
  },
}
```

What about something like default values?

```js
#set($attributeValues = $util.dynamodb.toMapValues($ctx.args.input))
#set($attributeValues.avenged = { "BOOL" : false })

{
  "version": "2017-02-28",
  "operation": "PutItem",
  "key": {
    "id": { "S" : "${utils.time.nowEpochMilliSeconds()}-${util.autoId()}" },
  },
  "attributeValues": $util.toJson($attributeValues),
  "condition": {
    "expression": "attribute_not_exists(#id)",
    "expressionNames": {
      "#id": "id",
    },
  },
}
```

We're not going to get a chance to go fully into here, but there are some interesting things you can do with other things passed in on the `$ctx` object.

For example: `$ctx.indentity` is the information from Cognito. This allows you to do cool things like authorization.

You have the username on the way in and you can query for objects with that username on the way out too.
