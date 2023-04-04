+++
title = "1.4 Updating Infrastructure"
chapter = false
weight = 30
+++

We just saw how to create new infrastructure. Next, let's make a few updates.

This exercise demonstrates how declarative infrastructure as code tools can be used not just for initial provisioning, but also subsequent changes to existing resources.

## Step 1 &mdash; Create Website Files

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

## Step 2 &mdash; Upload the Bucket Contents

Back in our Pulumi program, let's add a resource for our index file for our static site. Add the following code to the bottom of your `index.ts` file:

```typescript
new aws.s3.BucketObject("index.html", {
  bucket: bucket,
  source: new pulumi.asset.FileAsset("www/index.html"),
  acl: "public-read",
  contentType: "text/html",
});
```

We also would like the name of our bucket to be visible outside of our Pulumi program so that we can list the contents of the bucket to verify that our local files have indeed been written to the bucket.

{{% notice info %}}
We verify the creation of resources here for the purpose of demonstration in this tutorial, but once you're comfortable using Pulumi, verifying the creation of resources is not necessary.
{{% /notice %}}

In order to make values in a Pulumi program visible to the outside world, we use the `pulumi.export` method. This creates a Stack Output and its value can be obtained via the command line by using the `pulumi stack output` command. For more information, see [Stack Outputs](https://www.pulumi.com/docs/intro/concepts/stack/#outputs) in the Pulumi docs.

Add the following to your `index.ts`:

```typescript
export const bucketName = bucket.bucket;
```

At this point, your `index.ts` should look like the following:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const bucket = new aws.s3.Bucket("my-website-bucket", {
  website: {
    indexDocument: "index.html",
  },
});

new aws.s3.BucketObject("index.html", {
  bucket: bucket,
  source: new pulumi.asset.FileAsset("www/index.html"),
  acl: "public-read",
  contentType: "text/html",
});

export const bucketName = bucket.bucket;
```

Deploy the changes:

```bash
pulumi up
```

This will give you a preview similar to the following:

```text
Previewing update (dev)

View in Browser (Ctrl+O): https://app.pulumi.com/jkodrofftest/iac-workshop-ts/dev/previews/03c1110d-2919-4e82-bef4-02c9db3b503f

     Type                    Name                 Plan       
     pulumi:pulumi:Stack     iac-workshop-ts-dev             
 +   └─ aws:s3:BucketObject  index.html            create     


Outputs:
  + bucketName: "my-website-bucket-5d3569c"

Resources:
    + 1 to create
    2 unchanged

Do you want to perform this update? yes
```

Note that our output lists 1 item to create (our bucket object we just defined), but no changes are applied to our stack or S3 bucket. This is because Pulumi is _declarative_: we declare the desired end state of our infrastructure, and Pulumi figures out the steps necessary to get our infrastructure to that desired state. To put it another way, with Pulumi we declare _what we want_ and Pulumi will determine _how_ (i.e., the individual changes necessary) to get what we want.

Select `yes` to deploy the bucket object.

We can now list the contents of our bucket using the AWS CLI (or via the AWS Console) and see the files have been uploaded:

```bash
$ aws s3 ls $(pulumi stack output bucketName)
2023-04-03 19:37:54        231 index.html
```

## Step 3 &mdash; Add a Bucket Policy

Now that we have an S3 bucket and an object in it, we need to make the bucket accessible so we can see it. Currently, the objects in the bucket are private.

There are a few methods to manage this, but we're going to do it by adding a bucket policy to the bucket that allows objects to be read.

Create a new bucket policy object in your Pulumi program like so:

```typescript
new aws.s3.BucketPolicy("bucket-policy", {
  bucket: bucket.id,
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
});
```

{{% notice info %}}
In the code above, notice that we use `pulumi.jsonStringify` instead of `JSON.stringify`. Because the ARN of the S3 bucket is not known until after it's created, we can't use a plain string to represent its value. Instead, `bucket.arn` is a Pulumi Output. (Pulumi Inputs and Outputs are similar to promises, but they also include information that helps track the order of dependencies in Pulumi resources. This is a large part of how Pulumi works in a declarative fashion: by keeping track of dependencies in inputs and outputs.) In the example above, we use the ARN of the bucket to formulate the bucket policy which allows all objects in the bucket to be read by anyone. For more information on this topic, see [Inputs and Outputs](https://www.pulumi.com/docs/intro/concepts/inputs-outputs/).
{{% /notice %}}

Before we update our Pulumi program, let's add one final line of code to give us the URL of our static site. Add the following at the end of your `index.ts`:

```typescript
export const bucketUrl = pulumi.interpolate`http://${bucket.websiteEndpoint}`;
```

This exports the website endpoint so we can view the contents of our bucket. Note that `pulumi.interpolate` as it allows us to create a regular string from a Pulumi Output (a value which is not known until after a resource is created).

Your `index.ts` should look like the following:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const bucket = new aws.s3.Bucket("my-website-bucket", {
  website: {
    indexDocument: "index.html",
  },
});

new aws.s3.BucketObject("index.html", {
  bucket: bucket,
  source: new pulumi.asset.FileAsset("www/index.html"),
  acl: "public-read",
  contentType: "text/html",
});

new aws.s3.BucketPolicy("bucket-policy", {
  bucket: bucket.id,
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
});

export const bucketName = bucket.bucket;
export const websiteUrl = pulumi.interpolate`http://${bucket.websiteEndpoint}`;
```

Deploy the changes:

```bash
pulumi up
```

You'll see the BucketPolicy get added. You'll also get a URL as an `output`. We can now view the contents of that URL using `curl`:

```bash
curl $(pulumi stack output websiteUrl)
```

You should also be able to view the contents in your browser, take a look!

## Step 4 &mdash; Destroy your Infrastructure

We're done with this module of the workshop! Let's tear everything down.

```bash
pulumi destroy
```

Pulumi will give you a warning. Select `yes` to continue and delete all resources in the stack.

Finally, you can optionally remove the stack:

```bash
pulumi stack rm dev
```
