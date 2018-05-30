So, we stored some data—let's look at a trivial application for keep track of some images.

It's definitely going to be contrived. But, the point is to get your comfortable with the API without having to do a bunch of extra work.

We'll do an `awsmobile init`—as we tend to do.

Let's turn on Storage.

```
awsmobile user-files enable
```

We don't have any files yet, but let's have the application component list out all of the files. Just because.

```js
// index.js
import Amplify from 'aws-amplify';
import configuration from './aws-exports';

Amplify.configure(configuration);
Amplify.Analytics.disable();
```

```js
// Application.js

componentDidMount() {
  Storage.list('').then(files => {
    console.log({ files });
  });
}
```

This is going to fail. Why?

Oh. Whoa. There is a file in there. It's an example image. Thanks Amazon!

Cool, but there is a little bit of a problem—having the file name is not enough. We can't really do anything with that.

Just shoving it into the source of an image tag will likely not be enough.

We need to get the `URL` of the file.

```js
async componentDidMount() {
  const files = await Storage.list('');
  const urls = await Promise.all(files.map(async (file) => await Storage.get(file.key)));
  console.log({ urls });
}
```

Because that's not confusing at all.

But before we come up with a better abstraction. Let's talk about what's happening.

We can get a list of the keys, but then we need to have `Storage` generate custom URLs for the ones that we want.

We could do the following:

```js
async componentDidMount() {
  const files = await Storage.list('');
  const urls = await Promise.all(files.map(async (file) => await Storage.get(file.key)));
  this.setState({ files: urls });
}
```

```js
<section className="Application-images">
  { this.state.files.map(file => (
    <article key={file}>
      <img src={file} />
    </article>
  )) }
</section>
```

Meh.

Let's start by at least pulling that out.

```js
class S3Image extends Component {
  render() {
    const file = this.props.file;
    return (
      <article key={file}>
        <img src={file} />
      </article>
    )
  }
}
```

```js
<section className="Application-images">
 { this.state.files.map(file => (
   <S3Image file={file} key={file} />
 )) }
</section>
```

So, now `aws-amplify-react` has something similar. But it works a little differently. You give it the key and then it asynchronously loads it. I'm not a giant fan of the one in the library because it's pretty hard to customize. It's also fairly easy to write out own.

```js
class S3Image extends Component {
  state = { src: null };

  async componentDidMount() {
    const { s3key } = this.props;
    const src = await Storage.get(s3key);
    this.setState({ src });
  }

  render() {
    const { src } = this.state;
    if (!src) return null;
    return (
      <article>
        <img src={src} />
      </article>
    )
  }
}
```

Ideally, we could do a bunch of fancier stuff here, but I'll leave that as an exercise to the reader.

Simplify:

```js
async componentDidMount() {
  const files = await Storage.list('');
  this.setState({ files });
}
```

Finally:

```js
<section className="Application-images">
  { this.state.files.map(file => (
    <S3Image s3key={file.key} key={file.key} />
  )) }
</section>
```

Cool. We're back where we started from, but it's no longer the parent component's job to load the image.

```js
handleSubmit = event => {
  event.preventDefault();

  const file = this.fileInput.files[0];
  const { name } = file;

  Storage.put(name, file).then(response => {
    console.log('Storage.put', { response });
    this.setState({ files: [...this.state.files, response ] })
  })
};
```

Not bad, right?

Okay, let's get rid of some pictures.

```js
class S3Image extends Component {
  state = { src: null };

  async componentDidMount() {
    const { s3key } = this.props;
    const src = await Storage.get(s3key);
    this.setState({ src });
  }

  handleRemove = () => {
    this.props.onRemove && this.props.onRemove(this.props.s3key);
  };

  render() {
    const { src } = this.state;
    if (!src) return null;
    return (
      <article>
        <img src={src} />
        <button onClick={this.handleRemove}>Remove</button>
      </article>
    )
  }
}
```

```js
handleRemove = key => {
  const files = this.state.files.filter(file => file.key !== key);
  this.setState({ files });
}
```

```js
<section className="Application-images">
  { this.state.files.map(file => (
    <S3Image s3key={file.key} key={file.key} onRemove={this.handleRemove} />
  )) }
</section>
```

Super cool, but it would be nice if it actually did anything, right?

**Your Turn**: Can you find a method that will remove the images and implement that instead?

```js
// In the S3Image component
handleRemove = () => {
  this.props.onRemove &&
  Storage.remove(this.props.s3key).then(() => this.props.onRemove(this.props.s3key));
};
```

Okay, cool. You could do it either component really.

**Your Next Task**: Add authentication to your application. Configure the Storage API to store images private to a given user.

```
awsmobile user-signin enable
```

```js
import { withAuthenticator } from 'aws-amplify-react';
```

```js
export default withAuthenticator(Application);
```

Review:

- Either use `Storage.configure({ level: 'private' })` or put a `{ level: private }` on each API call.
- Show off the `expires` option that can invalidate links after a certain period of time.