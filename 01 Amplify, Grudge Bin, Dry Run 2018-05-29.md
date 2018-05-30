Run `awsmobile init` and answer all of the default questions.

Show everyone that there are a few new files. Notable, `src/aws-exports.js` and the top-level `awsmobilejs` directory.

You can also give `awsmobile` the following options:

- `-y, --yes     suppress console input and use default in the init procedure`
- `-r, --remove  remove awsmobilejs from the current project`

Let's take a look at `aws-exports.js`. It's an auto-generated file filled with configuration and API keys and whatnot. It also has some explicit instructions not to edit it.

If we fire up the application, not much has changed.

This makes sense, our application doesn't know anything about AWS yet.

We have a bunch of configuration, but we're not using it.

Let's rectify that.

```js
// index.js
import Amplify from 'aws-amplify';
import configuration from './aws-exports';

Amplify.configure(configuration);
```

Still, not a lot has changed. (Secretly, stuff has changed. We're not tracking our users. Creepy—right?)

So, the first thing we should add is authentication, right?

Let's go ahead and do that.

We can get that rolling with `awsmobile user-signin enable`. The defaults are—umm—opinionated to say the least, but let's roll with them for now.

We've made a change on our machine, but we need to let AWS know about these changes as well.

`awsmobile push`

So, now—it's time for the yeoman's work of creating and wiring up a sign-up form right?

I mean, you certainly could do that. But if we're just looking to get rocking and rolling. Then we can use a HOC that has been provided for us.

```js
// Application.js
import { withAuthenticator } from 'aws-amplify-react';

class Application extends Component { … }

export default withAuthenticator(Application);
```

Lo and behold! We have a sign in and sign up form. Let's go for it.

Now, we'll go through the login dance. I'll use Mailinator because Jon doesn't like it when we have our cell radios on while he's recording. I respect Jon.

It's be cool to have a sign-out button, I guess.

```js
export default withAuthenticator(Application, { includeGreetings: true });
```

Okay. Now we're in! But it's the same app as it was before.

I feel like we're going to need a database. So, let's spin one up.

`awsmobile database enable --prompt`

We'll use the `--prompt` flag to have it ask us what to do instead of just rolling with the defaults.

First question: should it be open or restricted by user?

Grudges are personal—let's go with a "Restricted" database for now.

> Note: This will create a column called "userId"

We already have sign-in—so, we know who the user is right? AWS will figure all of this out for us. It will use the `userId` as a partition key in the database.

What's our table name—how about "grudges"? That seems good, right?

DynamoDB is a NoSQL database, but let's humor it and tell it about the columns that we're expecting and their types.

We'll set up the following columns.

- `id` → `string`
- `person` → `string`
- `deed` → `string`
- `avenged` → `boolean`

To be clear—we're generating the `id` in our client-side code for now.

Cool—so, we have a database. But, how are we going interact with it from our client-side application?

(Pause for dramatic effect.)

Oh, yea—we need an API.

So, let's make one of those as well.

`awsmobile cloud-api enable --prompt`

There are now too options.

```
? Select from one of the choices below. (Use arrow keys)
❯ Create a new API
  Create CRUD API for an existing Amazon DynamoDB table
```

That second one seems *very* compelling right now.

Oh, look—it knows about the table we just made.

For now, it makes sense to restrict it to signed-in users because that's what we did with the database.

It's now adding some AWS Lambda functions for us in `awsmobilejs/backend/cloud-api/grudges/`.

Very cool. Let's go take a look at it.

It's interesting to notice that they have set up `put` to just use the `/grudges` endpoint but what do I know. It is interesting that you _could_ just change this code if you had strong feels about any of these conventions. Code you didn't write isn't the easiest to grok, but you've got this.

Okay, I'm going to wire up posting new grudges and getting a list of all grudges.

Our database is empty, so it makes sense to start by adding one.

We're going to be lazy on purpose. We might want to do a bunch of crazy stuff with optimistic updating, but I want to see that stuff works and prove to you all that I'm not a giant charlatan.

```js
API.post('grudgesCRUD', '/grudges', { body: grudge })
  .then(response => {
    console.log({ response });
    this.setState({ grudges: [grudge, ...this.state.grudges] });
  })
  .catch(console.error);
```

It blows up! Any guesses why? Oh yea, I never pushed up this backend.

(I need to fill some time while this all goes up the cloud.)

Look it works now! Now, we're not actually loading these when the application starts up, but let's look inside of a real deal DynamoDB database to verify that it's there.

`awsmobile console`

Here we can see a very "safe" version of the database. But, like I said in the introduction. We're using real services under the hood. So, let's pull up Dynamo's console.

(Go pull up Dynamo.)

Here we can see our fresh piece of data.

Let's pull everything from the database.

```js
class Application extends Component {
  // …

  componentDidMount() {
    API.get('grudgesCRUD', '/grudges')
      .then(grudges => {
        this.setState({ grudges });
      })
      .catch(console.error);
  }

  // …
}
```

**(BACK TO THE SLIDES!)**

**Your Turn**: Can you implement update and remove?

(Give participants like 20 minutes—15 if you're tight on time.)

```js
removeGrudge = grudge => {
  API.del('grudgesCRUD', `/grudges/object/${grudge.id}`)
    .then(response => {
      console.log({ response });
      this.setState({
        grudges: this.state.grudges.filter(other => other.id !== grudge.id),
      });
    })
    .catch(console.error);
};
```

```js
toggle = grudge => {
  const updatedGrudge = { ...grudge, avenged: !grudge.avenged };
  API.put('grudgesCRUD', '/grudges', { body: updatedGrudge }).then(response => {
    console.log({ response });
    const othergrudges = this.state.grudges.filter(
      other => other.id !== grudge.id,
    );
    this.setState({ grudges: [updatedGrudge, ...othergrudges] });
  }).catch(console.error);
};
```

Very cool!

Now, there are a few things I don't like about this API.

- I don't love that I have to make the `id`s client-side.
- I'd love to get back the `id` from the server on `PUT` and `DELETE`.

Let's head over to: `cd awsmobilejs/backend/cloud-api/grudges/`.

We'll do an `npm install uuid` to get the same library we're using currently on the client.

Require it in `app.js`.

Add the following to the `app.post` route:

```js
req.body.id = `${Date.now()}-${uuid()}`;
```

Finally, let's add `id: req.params.id` to the `POST`, `PUT`, and `DELETE` requests.

- While we wait: Let's also remove the `id` from `NewGrudge.js`.
- We can also probably just remove `uuid` from the client-side library as well.

Let's update the code in `Application.js` while that cooks.

```js
addGrudge = grudge => {
  API.post('grudgesCRUD', '/grudges', { body: grudge })
    .then(response => {
      console.log({ response });
      grudge.id = response.id;
      this.setState({ grudges: [grudge, ...this.state.grudges] });
    })
    .catch(console.error);
};

removeGrudge = grudge => {
  API.del('grudgesCRUD', `/grudges/object/${grudge.id}`)
    .then(response => {
      console.log({ response });
      this.setState({
        grudges: this.state.grudges.filter(other => other.id !== response.id),
      });
    })
    .catch(console.error);
};

toggle = grudge => {
  const updatedGrudge = { ...grudge, avenged: !grudge.avenged };
  API.put('grudgesCRUD', '/grudges', { body: updatedGrudge }).then(response => {
    console.log({ response });
    const othergrudges = this.state.grudges.filter(
      other => other.id !== response.id,
    );
    this.setState({ grudges: [updatedGrudge, ...othergrudges] });
  }).catch(console.error);
};
```

Neat. So it works. Sure, we could get more robust and change the default API to our heart's content. But let's leave it here. Besides—we're going to look at another—entirely different approach.

Our application has one other fatal flaw: it's running on localhost.

`awsmobile publish`

(Switch over to the slides. You've got some time to kill.)
