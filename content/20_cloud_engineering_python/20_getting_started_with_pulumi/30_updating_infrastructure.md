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
  <p>Made with ❤️ with <a href="https://pulumi.com">Pulumi</a> and Python</p>
  <p>Hosted with ❤️ by AWS!</p>
</body>

</html>
```

Next, download a python image to the `www` directory:

```bash
curl https://raw.githubusercontent.com/pulumi/examples/ec43670866809bfd64d3a39f68451f957d3c1e1d/aws-py-s3-folder/www/python.png -o www/python.png
```

Now that we have our website files, we're ready to upload these files to our S3 bucket website.

## Step 2 &mdash; Upload the Bucket Contents

Back in our Pulumi program, let's add two new imports near the top of our `__main__.py` with the rest of our imports:

```python
import mimetypes
import os
```

Next, we'll list the files in the `www` directory and add a new `s3.BucketObject` resource for each file. Because Pulumi allows us to use real programming languages to define our infrastructure, we can use regular Python libraries like `mimetypes` and `os` along with language constructs like `for` loops.

Add the following code to the bottom of your `__main__.py` file:

```python
content_dir = "www"
for file in os.listdir(content_dir):
    filepath = os.path.join(content_dir, file)
    mime_type, _ = mimetypes.guess_type(filepath)
    obj = aws.s3.BucketObject(
        file,
        bucket=bucket.id,
        source=pulumi.FileAsset(filepath),
        content_type=mime_type,
    )
```

We also would like the name of our bucket to be visible outside of our Pulumi program so that we can list the contents of the bucket to verify that our local files have indeed been written to the bucket. (We do this for the purposes of demonstration in this tutorial, but once you're comfortable using Pulumi, verifying the creation of resources is not necessary.)

In order to make values in a Pulumi program visible to the outside world, we use the `pulumi.export` method. This creates a Stack Output and its value can be obtained via the command line by using the `pulumi stack output` command. For more information, see [Stack Outputs](https://www.pulumi.com/docs/intro/concepts/stack/#outputs) in the Pulumi docs.

Add the following to your `__main__.py`:

```python
pulumi.export("bucket_name", bucket.bucket)
```

At this point, your `__main.py__` should look like the following:

```python
"""A Python Pulumi program"""

import pulumi
import pulumi_aws as aws
import mimetypes
import os

bucket = aws.s3.Bucket(
    "my-website-bucket",
    aws.s3.BucketArgs(
        website=aws.s3.BucketWebsiteArgs(
            index_document="index.html"
        )
    )
)

content_dir = "www"
for file in os.listdir(content_dir):
    filepath = os.path.join(content_dir, file)
    mime_type, _ = mimetypes.guess_type(filepath)

    obj = aws.s3.BucketObject(
        file,
        aws.s3.BucketObjectArgs(
            bucket=bucket.id,
            source=pulumi.FileAsset(filepath),
            content_type=mime_type,
        )
    )

pulumi.export("bucket_name", bucket.bucket)
```

Deploy the changes:

```bash
pulumi up
```

This will give you a preview similar to the following:

```text
Previewing update (dev)

View Live: https://app.pulumi.com/user/iac-workshop/dev/previews/3b99429c-bcf4-4a52-a831-08173f962756

     Type                    Name              Plan       
     pulumi:pulumi:Stack     iac-workshop-dev             
 +   ├─ aws:s3:BucketObject  index.html        create     
 +   └─ aws:s3:BucketObject  python.png        create     
 
Resources:
    + 2 to create
    2 unchanged

Do you want to perform this update? yes
```

Note that our output lists 2 items to create (our bucket objects we just defined), but no changes are applied to our stack or S3 bucket. This is because Pulumi is _declarative_: we declare the desired end state of our infrastructure, and Pulumi figures out the steps necessary to get our infrastructure to that desired state. To put it another way, with Pulumi we declare _what we want_ and Pulumi will determine _how_ (i.e., the individual changes necessary) to get what we want.

Select `yes` to deploy the bucket objects.

We can now list the contents of our bucket using the AWS CLI (or via the AWS Console) and see the files have been uploaded:

```bash
aws s3 ls $(pulumi stack output bucket_name)
```

## Step 3 &mdash; Add a Bucket Policy

Now that we have an S3 bucket and some objects in it, we need to make the bucket accessible so we can see it. Currently, the objects in the bucket are private.

There are a few methods to manage this, but we're going to do it by adding a bucket policy to the bucket that allows objects to be read.

We're going to be dealing with a JSON string here, so let's add a new import so we can deal structured JSON:

```python
import json
```

Create a new bucket policy object in your Pulumi program like so:

```python
bucket_policy = aws.s3.BucketPolicy(
    "my-website-bucket-policy",
    bucket=bucket.id,
    policy=bucket.arn.apply(
        lambda arn: json.dumps({
            "Version": "2012-10-17",
            "Statement": [{
                "Effect": "Allow",
                "Principal": "*",
                "Action": [
                    "s3:GetObject"
                ],
                "Resource": [
                    f"{arn}/*"
                ]
            }]
        })),
)
```

In the code above, we need to make use of a special Pulumi method called `apply`. Because the ARN of the S3 bucket is not known until after it's created, we can't use a plain string to represent its value. Instead, `bucket.arn` is a Pulumi Output. (If you're familiar with JavaScript, Pulumi Outputs are similar to promises.) All Pulumi Outputs have a special function called `apply` which allows us to use the value of the output once it's known. In the example above, we use the ARN of the bucket to formulate the bucket policy which allows all objects in the bucket to be read by anyone.

Note that we're also using the `bucket.id` Output as an Input to the bucket policy. In most cases, we can seamlessly use an Output of one resource as the Input of another resource, but because we have to embed the value of the ARN in the bucket policy, we need to use `apply` when creating the policy. (The most common need for `apply` in AWS is when we're creating policies, like S3 bucket policies or IAM polices.) For additional information on Pulumi Inputs and Outputs, see [Inputs and Outputs](https://www.pulumi.com/docs/intro/concepts/inputs-outputs/) in the Pulumi docs.

Before we update our Pulumi program, let's add one final line of code to give us the URL of our static site. Add the following at the end of your `__main__.py`:

```python
pulumi.export("website_url", pulumi.Output.concat(
    "http://", bucket.website_endpoint))
```

This exports the website endpoint so we can view the contents of our bucket. Note that `pulumi.Output.concat` is a helper function that allows us to create a regular string from a Pulumi Output (a value which is not known until after a resource is created). It's a convenient helper function that works very similarly to `apply`.

Your `__main__.py` should look like the following:

> :white_check_mark: After this change, your `__main__.py` should look like this:

```python
"""A Python Pulumi program"""

import pulumi
import pulumi_aws as aws
import mimetypes
import os
import json

bucket = aws.s3.Bucket(
    "my-website-bucket",
    aws.s3.BucketArgs(
        acl="public-read",
        website=aws.s3.BucketWebsiteArgs(
            index_document="index.html"
        )
    )
)

content_dir = "www"
for file in os.listdir(content_dir):
    filepath = os.path.join(content_dir, file)
    mime_type, _ = mimetypes.guess_type(filepath)

    obj = aws.s3.BucketObject(
        file,
        aws.s3.BucketObjectArgs(
            bucket=bucket.id,
            source=pulumi.FileAsset(filepath),
            content_type=mime_type,
        )
    )

pulumi.export("bucket_name", bucket.bucket)

aws.s3.BucketPolicy(
    "my-website-bucket-policy",
    aws.s3.BucketPolicyArgs(
        bucket=bucket.id,
        policy=bucket.arn.apply(
            lambda arn: json.dumps({
                "Version": "2012-10-17",
                "Statement": [{
                    "Effect": "Allow",
                    "Principal": "*",
                    "Action": [
                        "s3:GetObject"
                    ],
                    "Resource": [
                        f"{arn}/*"
                    ]
                }]
            })),
    )
)

pulumi.export("website_url", pulumi.Output.concat(
    "http://", bucket.website_endpoint))
```

Deploy the changes:

```bash
pulumi up
```

You'll see the BucketPolicy get added. You'll also get a URL as an `output`. We can now view the contents of that URL using `curl`:

```bash
curl $(pulumi stack output website_url)
```

{{% notice info %}}
You should also be able to view the contents in your browser, take a look!
{{% /notice %}}

## Step 4 &mdash; Destroy your Infrastructure

We're done with this section of the workshop! Let's tear everything down.

```bash
pulumi destroy
```

Pulumi will give you a warning. Select `yes` to continue and delete all resources in the stack.

Finally, you can remove the stack:

```bash
pulumi stack rm dev
```
