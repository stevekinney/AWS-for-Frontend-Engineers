# AWS for Front End Engineers

## New Shit That has Come to Light

- Show off the CloudFront metrics
- Run device tests in AWS Mobile Hub
- CloudFront compression: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html
- Read up on DynamoDB primary keys: http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html#HowItWorks.CoreComponents.PrimaryKey
- Read up on secondary indices: http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html#HowItWorks.CoreComponents.SecondaryIndexes

## Prerequisities

AWS: https://gist.github.com/stevekinney/941e815e3f2ae824529cc4470e45794c

Nodebots: https://gist.github.com/stevekinney/bcd0fef3fb7cd9ce5af8ffba08480b02

- Create an AWS account.
- Have a valid credit card.
- Install multi-factor authentication app (e.g. Authy, Google Authenticator, Duo)
- Install the AWS CLI tools.
- Install the AWS SDK for Node.
- Install the Travis CLI command-line tools
- Have a domain name that you control the DNS records for _or_ be willing to purchase one during the workshop.

## Game Plan

- Introduction
- Take a quick tour of the free tier.
- Set up a billing notification if anything is going to happen outside of the free tier.
- Set up an IAM user.
	- Give the IAM user an administrator policy.
	- Talk a little bit about how AWS policies work.
- Explain how regions and availability zones work.
- Turn on MFA for the IAM user.
- Buy a domain name through Route 53.
- Set up an S3 bucket.
  - Enable web access.
  - Set up cross-region replication. (TODO)
- Use webpack’s S3 plugin to automatically deploy to S3 on build.
- Get Route 53 set up to the S3 bucket.
- Create a Cloudfront distribution.
- Set up CI/CD to automatically handle deploying when merging to master.
- Set up Route 53 to point to the Cloudfront bucket.
- Set up SSL and get a certificate with ACM.
- Use Lambda@Edge to get our response HTTP status codes correct.
- Other Lambda@Edge fun times

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

### TODO: Give billing access to your non-root user

https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_billing.html?icmpid=docs_iam_console#tutorial-billing-step1

### Configuring the CLI

```
AWS Access Key ID [None]: YOUR_ACCESS_KEY_HERE
AWS Secret Access Key [None]: YOUR_SECRET_KEY_HERE
Default region name [None]: us-east-1
Default output format [None]: json
```

### Logging into the Console

- Go to the IAM console.
- Grab the "IAM users sign-in link"
  - In my case, this is: https://stevekinney.signin.aws.amazon.com/console
- Head over to that link and log in
- You're now no longer your root user

### TODO: A Brief Discussion of Regions and Availability Zones

### What is S3?

Fun facts about S3 (TODO: Add more from your notes)

- TL;DR: It's a place to store your files in the cloud
- Files are stored in buckets and buckets are kind of like folders
- S3 bucket names are globally unique
- Files can be up to 5GB in size

Durability and Availability

- Three 9s for availability (built for four 9s)
- 99.999999999% durability (11 nines)
- Lifecycle management
- Versioning
- Encryption
- Secure your data with ACLs and bucket policies

Storage tiers:

- Regular S3
- Infrequently accessed (lower fee for storage but you’re charged a retrieval fee)
- Reduced redundancy storage (four 9s)
- Glacier is very cheap and it takes 3-5 hours to retrieve

More about how S3 works:

- It’s a simple key/value store
- Key: file name of the object
  - Things are sorted by key name
- Value: The file/data
- There is a version ID
- Metadata about the file
- Subresources
  - Access Control List
  - Torrent (TODO: what did I mean by this?)

Data Consistency Model

- Read after write consistency for PUTS of *new* objects
  - You get it immediately
  - You get a HTTP 200 response if it worked
- Eventual consistency for overwrite PUTS and DELETES
  - This means they can type some time to propagate

Charges (e.g. 💸):

- Charged for storage
- Requests
- Storage management pricing
- Data transfer
	- Getting it into Amazon is free
- Transfer acceleration—kind of like a CDN

### Setting Up Our Web Application

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
- TODO: Learn about S3 redirection rules

Alright, let's push it up to the Internet

```
aws s3 cp dist/ s3://myfundemoapp.net/ --recursive
```

Even better: make this an npm script so you can do something `npm run deploy`.

Boom! Now it should be deployed on the Internet.

```
http://myfundemoapp.net.s3-website-us-east-1.amazonaws.com/
```

How does this work? It's fairly simple:

```
<bucket-name>.s3-website-<AWS-region>.amazonaws.com
```

Very cool, let's hook up our domain name.

#### Setting up an A record

- Add a new record set
- Select "A" for the type
- Leave the subdomain blank
- Check the radio to "Yes"
- You should see your thing in the dropdown
- If not, go with `s3-website-us-east-1.amazonaws.com`
- Leave the rest of the defaults

Your website should now be in the right place

### Configuring Route 53

- Go to Route 53 and click on domains
- Click on your new domain
- Click on the "Manage DNS" button
- Click on "Go to Record Sets"

#### Create an Alias Bucket

Here's the thing, if we go to a `www.` whatever your domain is, it no longer works.

1. Make a second bucket with the a `www` prefix.
2. Turn on static website hosting.
3. Have all requests redirect to your original bucket

#### Route 53 to your other bucket

1. Set up a CNAME for `www`
2. Set the value to www.myfundemoapp.net.s3-website-us-east-1.amazonaws.com
3. Profit

TODO: This wasn't working previously. Round back and see what you need to do.

### Cloudfront Distribution

So, it's up and running, but our site is located in North Virginia.

This means that all of our users have to head over to the east coast to get the content.

Due to the speed of light, this means different things for different people.

We can check this here:

https://www.dnsperf.com/dns-speed-benchmark

#### Setting Up a Cloudfront Distribution

- Go to Cloudfront and select "Create Distribution"
- For Origin Domain Name, we're going to go with our S3 bucket name. It _should_ be in the dropdown list.
- Set "Restrict Bucket Access" to `true`
- Turn on "Grant Read Permissions"
- Make sure the "Default Root Object" is `/index.html`
- Now wait a while.

##### Invalidating the Cache on Build

```
aws cloudfront create-invalidation --distribution-id E3V9NTKJ5YSLRU --paths /index.html
```

### An Aside: Setting up CI/CD

TODO: Let's set up Travis (or something else) in order

### Set Up Route 53 and an SSL Certificate

- Point the Alias at the Cloudfront distribution
- Point the CNAME at the URI for the Cloudfront distribution
- Go to the Certificate Manager
- Create a new certificate
- Put in the first domain name
- Put in the additional domain name
- Select email for verification

### An Aside: Set up cross region replication

TODO: Set up a second bucket with cross-region replication
TODO: See if you can write a Lambda@Edge function that helps solve these issues
