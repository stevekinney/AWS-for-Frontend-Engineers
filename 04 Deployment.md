## Play by Play

### Turn on MFA

- Go to IAM in the AWS Console
- Go to *Security Status* and expand *Activate MFA* on your root account.
- Click on the *Manage MFA* button.
- Choose to use a "virtual MFA device"
- Scan in the bar code in Authy (or whatever)
- Enter in two consecutive MFA codes
- Click "Activate"

### Create a Secondary Account

Think of your root account like the root account on your computer. With great power comes great responsibility.

**Side note**: If you have access keys for your root account, it's time to get rid of those.

- Go to "Manage Security Credentials"
- Expand the "Access keys" accordion
- Delete whatever keys exist there.

Generally speaking you don't want to use your root account for most tasks. Let's make a "subuser" that isn't _as_ powerful as our current root user.

- Head back over to IAM in the AWS Console
- Click on "Users"
- Click "Add user"
- Pop in a username that suits your fancy
- Select both programmatic access as well as AWS Management Console access
- Enter in a password for Console access
- Uncheck "Require password reset"—we don't need that now.
- Move on to Permissions.
- We're going to attach some built-in policies directly.
- We want to add the following policies.
  - Administrator Access
  - Billing
- Click "Next: Review"
- Click "Create user"

### Locks and Keys

Okay, so now we've created our new IAM user.

This is an import page. Here, you'll see you *Access key ID* and *Secret access key*.

This is the last time you'll ever see your secret access key unless you download the CSV.

The "Send email" link isn't super helpful if you're creating this account for yourself. It's just a `mailto:` link with the username.

Let's go ahead and download the CSV.

If you lose the secret key, you can generate a new one, but you'll never be able to see this one again—outside of the CSV, of course. So, keep that in mind.

If you have something like 1Password or LastPass, it's probably a good idea to put that there.

#### Setting Up MFA

- Go find the user in IAM.
- Go to the "Security credentials" tab.
- Click on the little pencil next to "Assigned MFA device"
- Set up a virtual MFA device

### Setting Up a Billing Alert

- Go to "Billing" in the AWS Console
- Go to "Preferences"
- Click on the checkbox for "Receive Free Tier Usage Alerts"
- Click the checkbox for "Receive Billing Alerts"
- Enter your email address
- "Save preferences"

#### Billing Alarms with CloudWatch

(Make sure you're in `us-east-1`—a.k.a. Northern Virginia.)

- Under "Alarms" on the left-hand side, head over to "Billing."
- When creating a new one, select "Total Estimated Charge"
- Create a new list
- Add your email to the list
- Confirm your email

<!-- ### Give your IAM user access to billing information

- Log in as your IAM user
- Go into IAM
- "Choose a service"
- "Billing"
- "All billing actions"
- Name it `BillingFullAccess`
- Click "Create"
- Go find it in the list
- Click the check-circle
- Apply it to your user -->

## Set up the CLI

AWS Access Key ID [None]: YOUR_ACCESS_KEY_HERE
AWS Secret Access Key [None]: YOUR_SECRET_KEY_HERE
Default region name [None]: us-east-1
Default output format [None]: json

### Logging into the Console

- Go to the IAM console.
- Grab the "IAM users sign-in link"
  - In my case, this is: https://stevekinney.signin.aws.amazon.com/console
- Head over to that link and log in
- You're now no longer your root user

### Policies and Roles

### A Brief Discussion of Regions and Availability Zones

### Creating a Bucket

#### A Slightly Complicated Dance Between Domains and Buckets

- Domain names need to be unique
- Bucket names need to be unique
- Bucket names need to be the same as the domain name if you want to serve assets publicly

This means that if you want to host a site, you need to make sure that both the bucket and the domain name are available.

1. Go to Route 53 and register a new domain (you're not going to pull this trigger just)
2. Once you have a domain that is available, go ahead and create the bucket
  - Hit "Create Bucket" in the S3 console, we're going to just stick with the basics for now
  - (You may also want to register the domain now since it takes a while)
3. Upload an `index.html`
4. Turn on Static Website Hosting
5. See that it doesn't work
6. Add the bucket policy below

### Push your assets to the S3 bucket

First, let's make the bucket public:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME_HERE/*"
        }
    ]
}
```

- Turn on "Static website hosting"
- Make both the index document and the error document `index.html`

**Your Turn**: Create a Second Bucket. I've left a copy of my bucket polcies in the course notes.

- Play around with versioning
- Play around with replication

Just show that this stuff exists and that we're probably not going to use it too much.

### Getting a React App Up and Running

```
yarn run build && aws s3 cp build/ s3://<YOUR_BUCKET_NAME>/ --recursive
```

Boom and now it's on the Interwebs.

Okay, let's set up some DNS in Route 53 and then we'll talk about Route 53—because DNS is not instantaneous.

- Set up the redirect bucket.
- Set up the DNS in Route 53.

#### Setting up an A record

- Add a new record set
- Select "A" for the type
- Leave the subdomain blank
- Check the radio to "Yes"
- You should see your thing in the dropdown
- If not, go with `s3-website-us-east-1.amazonaws.com`
- Leave the rest of the defaults

#### Create an Alias Bucket

Here's the thing, if we go to a `www.` whatever your domain is, it no longer works.

1. Make a second bucket with the a `www` prefix.
2. Turn on static website hosting.
3. Have all requests redirect to your original bucket

#### Route 53 to your other bucket

1. Set up a CNAME for `www`
2. Set the value to www.myfundemoapp.net.s3-website-us-east-1.amazonaws.com
3. Profit

(Show the Route 53 slides.)

Go back and verify that everything works.

## CloudFront

(Go back to the slides for like one slide or so.)

CloudFront can take a while, so let's start to get one cooking and then we'll talk about it.

- Go to Cloudfront and select "Create Distribution"
- Paste in your S3 web address.
- Select "Compression"
- Now wait a while.

## Getting a Certificate

- Go to the Certificate Manager
- Click on "Request a Certificate"
- Request a public certificate

(You might have to take a break—at max like 5 minutes.)

- Add your certificate to each of your distrubtions.

## Setting Up Travis

```yml
language: node_js
node_js:
  - '8'
cache:
  yarn: true
  directories:
    - node_modules
script:
  - yarn test
before_deploy:
  - yarn global add travis-ci-cloudfront-invalidation
  - yarn run build
deploy:
  provider: s3
  access_key_id: $AWS_ACCESS_KEY_ID
  secret_access_key: $AWS_SECRET_ACCESS_KEY
  bucket: $S3_BUCKET
  skip_cleanup: true
  local-dir: dist
  on:
    branch: master
after_deploy:
  - travis-ci-cloudfront-invalidation -a $AWS_ACCESS_KEY_ID -s $AWS_SECRET_ACCESS_KEY -c $CLOUDFRONT_ID -i '/*' -b $TRAVIS_BRANCH -p $TRAVIS_PULL_REQUEST
```

- Make a branch.
- Add Travis.
- Make some color change or something.
- Make a new PR.
- Merge it into master.

Wait for it to build.

## Lambda@Edge

### Swap Image

Viewer Request

```js
'use strict';

exports.handler = (event, context, callback) => {
  const request = event.Records[0].cf.request;

  const toReplace = '/prince-1.jpg';  
  const replacement = '/prince-2.jpg';
  
  if (request.uri !== toReplace) return callback(null, request);

  request.uri = replacement;
  
  console.log(`Request uri set to "${request.uri}"`);

  callback(null, request);
};
```

### Response change

```js
'use strict';

exports.handler = (event, context, callback) => {
    const request = event.Records[0].cf.request;
    const response = event.Records[0].cf.response;
    

    if (response.status >= 400 && response.status <= 599) {
      if (/notes\/\d(\/edit)?/.test(request.uri)) {
        response.status = 200;
        response.statusDescription = 'OK';
      }
    }

    callback(null, response);
};
```