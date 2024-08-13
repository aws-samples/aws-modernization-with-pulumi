+++
title = "1.4 Updating Infrastructure"
chapter = false
weight = 30
+++

We just saw how to create new infrastructure. Next, let's make a few updates.

This exercise demonstrates how declarative infrastructure as code tools can be used not just for initial provisioning, but can also be used for subsequent changes to existing resources.

## Step 1 &mdash; Enable Public Read Access

First, we need to enable publicly readability for our bucket because AWS disables IAM policies and ACLs that grant public readability on buckets and their objects by default. Add the following to your `index.ts` file:

```typescript
const publicAccessBlock = new aws.s3.BucketPublicAccessBlock("public-access-block", {
  bucket: bucket.bucket,
  blockPublicPolicy: false,
});
```

Then, deploy your infrastructure:

```text
$ pulumi up

Previewing update (dev)

View in Browser (Ctrl+O): https://app.pulumi.com/...

     Type                               Name                 Plan       
     pulumi:pulumi:Stack                fix-s3-acls-dev                 
 +   └─ aws:s3:BucketPublicAccessBlock  public-access-block  create     

Resources:
    + 1 to create
    2 unchanged

Do you want to perform this update?  [Use arrows to move, type to filter]
  yes
> no
  details
```

Notice how the preview indicates that we will create our `aws.s3.PublicAccessBlock`, but that 2 resources (our Pulumi stack and our S3 bucket which we created earlier) will remain unchanged? This is because Pulumi is a _declarative_ tool: We do not need to specify _how_ things should be done (delete this, only create this if it hasn't be created, etc.). We only need to specify what the desired end state of our infrastructure is, and Pulumi will figure out which steps need to be performed to get from the current state of our resources to our declared desired state.

Select `yes` to continue. Optionally, if you want to explore the declarative nature of Pulumi, try changing some properties on the `aws.s3.PublicAccessBlock` (or comment it out entirely) and run `pulumi up`. See what happens!

Now we are ready to add a bucket policy to grant public access.

## Step 2 &mdash; Add a Bucket Policy

Declare a bucket policy to grant public read access to all objects in your bucket:

```typescript
new aws.s3.BucketPolicy("bucket-policy", {
  bucket: bucket.bucket,
  policy: pulumi.jsonStringify({
    Version: "2012-10-17",
    Statement: [{
      Effect: "Allow",
      Principal: "*",
      Action: ["s3:GetObject"],
      Resource: [
        pulumi.interpolate`${bucket.arn}/*`,
      ],
    }]
  })
}, {
  dependsOn: [publicAccessBlock],
});
```

Before we continue with our Pulumi program, it's worth taking a few minutes to analyze this resource.

### Aside: About Pulumi Inputs, Outputs, and Resource Options

There a couple of new things in our `aws.s3.BucketPolicy` resource that are worth diving into:

First, in the code above, notice that we use `pulumi.jsonStringify` instead of `JSON.stringify`. Because the ARN of the S3 bucket is not known until after it's created, we can't use a plain string to represent its value. Instead, `bucket.arn` is a Pulumi Output. Pulumi Inputs and Outputs are similar to promises, but they also include information that helps track the order of dependencies in Pulumi resources. This is how Pulumi works in a declarative fashion: by keeping track of dependencies in inputs and outputs. In the example above, we use the ARN of the bucket to formulate the bucket policy which allows all objects in the bucket to be read by anyone. For more information, see [Inputs and Outputs](https://www.pulumi.com/docs/intro/concepts/inputs-outputs/).

Second, in order to successfully provision our `aws.s3.BucketPolicy`, we have to ensure that our `aws.s3.PublicAccessBlock` is provisioned first or AWS will throw an error because public access policies are disabled by default on S3 buckets. In most cases Pulumi will know the order in which resources must be created, deleted, or replaced because of implicit dependencies: one resource's Output is another resource's Input. However, in this case, we need to specify an _explicit dependency_ between our `aws.s3.PublicAccessBlock` and our `aws.s3.BucketPolicy`.

We can specify explicit dependencies between Pulumi resources by specifying the `dependsOn` property of a third argument which is common to all Pulumi resources called `ResourceOptions`. Any implicit dependencies will still be enforced _in addition to_ any explicit dependencies we specify with `dependsOn`. For more information on `dependsOn` and a complete listing of all available options, see [Resource Options](https://www.pulumi.com/docs/concepts/options/).

Now we are ready to add our website files.

## Step 3 &mdash; Create Website Files

Create a directory `www/` and add a new `index.html` file with the following contents:

```html
<html>

<head>
  <meta charset="UTF-8">
  <title>Hello, Pulumi!</title>
</head>

<body>
  <p>Hello, S3!</p>
  <p>Made with ❤️ with <a href="https://pulumi.com">Pulumi</a></p>
  <p>Hosted with ❤️ by AWS!</p>
</body>

</html>
```

Now that we have our website file, we're ready to upload these files to our S3 bucket website.

## Step 4 &mdash; Upload the Bucket Contents and Create Stack Outputs

Back in our Pulumi program, let's add a resource for our index file for our static site. Add the following code to the bottom of your `index.ts` file:

```typescript
new aws.s3.BucketObject("index.html", {
  bucket: bucket,
  source: new pulumi.asset.FileAsset("www/index.html"),
  contentType: "text/html",
});
```

Note that we create a `FileAsset` to easily load the content of our `index.html` from disk. For more information, see [Assets & archives](https://www.pulumi.com/docs/concepts/inputs-outputs/assets-archives/).

We also need to get the URL of the S3 bucket so we can visit our website. In order to make values in a Pulumi program visible to the outside world, we create a Stack Output. Stack Output values can be retrieved in two ways:

1. From the command line, by using the `pulumi stack output` command. This is the method we will use in this tutorial.
1. From other Pulumi programs, by using the Stack References. For more details on how to share values between Pulumi programs, see [Stack Outputs and References](https://www.pulumi.com/docs/using-pulumi/stack-outputs-and-references/).

Add the following to your `index.ts`:

```typescript
export const url = pulumi.interpolate`http://${bucket.websiteEndpoint}`;
```

Note that the `pulumi.interpolate` method is used here to concatenate a plain string (`http://`) and a Pulumi Output (`bucket.websiteEndpoint`).

Deploy the changes:

```text
pulumi up
```

This exports the website endpoint so we can view the contents of our bucket.

Your `index.ts` should look like the following:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const bucket = new aws.s3.Bucket("my-website-bucket", {
  website: {
    indexDocument: "index.html",
  },
});

const publicAccessBlock = new aws.s3.BucketPublicAccessBlock("public-access-block", {
  bucket: bucket.bucket,
  blockPublicPolicy: false,
});

new aws.s3.BucketPolicy("bucket-policy", {
  bucket: bucket.bucket,
  policy: pulumi.jsonStringify({
    Version: "2012-10-17",
    Statement: [{
      Effect: "Allow",
      Principal: "*",
      Action: ["s3:GetObject"],
      Resource: [
        pulumi.interpolate`${bucket.arn}/*`,
      ],
    }]
  })
}, {
  dependsOn: publicAccessBlock,
});

new aws.s3.BucketObject("index.html", {
  bucket: bucket.bucket,
  source: new pulumi.asset.FileAsset("www/index.html"),
  contentType: "text/html",
});

export const url = pulumi.interpolate`http://${bucket.websiteEndpoint}`;
```

Deploy the changes:

```bash
pulumi up
```

Now you can view the contents of your website!

```bash
$ curl $(pulumi stack output url)

<html>

<head>
  <meta charset="UTF-8">
  <title>Hello, Pulumi!</title>
</head>

<body>
  <p>Hello, S3!</p>
  <p>Made with ❤️ with <a href="https://pulumi.com">Pulumi</a></p>
  <p>Hosted with ❤️ by AWS!</p>
</body>

</html>%
```

## Step 5 &mdash; Destroy your Infrastructure

We're done with this module of the workshop! Let's tear everything down using the `pulumi destroy` command:

```bash
pulumi destroy
```

Pulumi will give you a warning. Select `yes` to continue and delete all resources in the stack.

Finally, you can remove the stack completely:

```bash
pulumi stack rm dev
```

That's it!
