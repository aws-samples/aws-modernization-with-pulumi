+++
title = "1.4 Updating Infrastructure"
chapter = false
weight = 30
+++

We just saw how to create new infrastructure from scratch. Next, let's make a few updates:

1. Add an object to your bucket.
2. Serve content from your bucket as a website.
3. Programmatically create infrastructure.

This demonstrates how declarative infrastrucutre as code tools can be used not just for initial provisioning, but also subsequent changes to existing resources.

## Step 1 &mdash; Add an Object to Your Bucket

Create a directory `www/` and add a new `index.html` file with the following contents:

```html
<html>
    <head><meta charset="UTF-8">
    <title>Hello, Pulumi!</title></head>
<body>
    <p>Hello, S3!</p>
    <p>Made with ❤️ with <a href="https://pulumi.com">Pulumi</a> and Python</p>
    <img src="python.png" />
</body></html>
```

Next, download a python image to the `www` directory:

```
curl https://raw.githubusercontent.com/pulumi/examples/ec43670866809bfd64d3a39f68451f957d3c1e1d/aws-py-s3-folder/www/python.png -o www/python.png
```

Now we've created our example website, we're ready to upload these files to our s3 bucket website.

## Step 2 &mdash; Upload the Bucket Contents

Back inside our Pulumi program, we now need to upload these contents to our bucket. 

Let's add two new imports to our program:

```python
import pulumi
import mimetypes
import os
import pulumi_aws as aws_classic
import pulumi_aws_native as aws_native
```

Next, we'll list the contents of the `www` directory and add a new Pulumi resource for each called `BucketObject`. wW can use a simple for loop over the contents of the `www` directory:

```python
content_dir = "www"
for file in os.listdir(content_dir):
    filepath = os.path.join(content_dir, file)
    mime_type, _ = mimetypes.guess_type(filepath)
    obj = aws_classic.s3.BucketObject(
        file,
        bucket=bucket.id,
        source=pulumi.FileAsset(filepath),
        content_type=mime_type,
        opts=pulumi.ResourceOptions(parent=bucket)
    )
```

Deploy the changes:

```bash
pulumi up
```

This will give you a preview and selecting `yes` will apply the changes:

```
     Type                       Name               Plan       
     pulumi:pulumi:Stack        iac-workshop-dev              
     └─ aws-native:s3:Bucket    my-website-bucket             
 +      ├─ aws:s3:BucketObject  python.png         create     
 +      └─ aws:s3:BucketObject  index.html         create     
 
Resources:
    + 2 to create
    2 unchanged
```

We can now list the contents of our bucket again and see the files have been uploaded:

```bash
$ aws s3 ls $(pulumi stack output bucket_name)
2020-11-18 19:06:10        237 index.html
2020-11-18 19:06:10        831 python.png
```

## Step 3 &mdash; Add a Bucket Policy

Now we have an S3 bucket and some objects in it, we need to make the bucket accessible so you can see it. Currently, the objects in the bucket are private.

There are a few methods to manage this, but we're going to do it by adding a bucket policy to the bucket that allows objects to be read.

We're going to be dealing with a JSON string here, so let's add a new import so we can deal structured json:

```python
import json
```

Create a new bucket policy object in your Pulumi program like so:

```python
bucket_policy = aws_classic.s3.BucketPolicy(
    "my-website-bucket-policy",
    bucket=bucket.id,
    policy=bucket.arn.apply(
        lambda arn:  json.dumps({
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
    opts=pulumi.ResourceOptions(parent=bucket)
)
```

Make a note of the policy object here. The `policy` parameter for a bucket policy takes a standard python string, and we want to inject a special Pulumi type into it called an Output

{{% notice info %}}
Your workshop host will explain Outputs here. If you're following along on your own, please see [this video](https://www.youtube.com/watch?v=lybOxul2otM) for an explanation
{{% /notice %}}

In order to use these computed output values, we use an `apply` method to inject the bucket arn into the string.

> :white_check_mark: After this change, your `__main__.py` should look like this:
```python
import pulumi
import mimetypes
import json
import os
import pulumi_aws as aws_classic
import pulumi_aws_native as aws_native

bucket = aws_native.s3.Bucket(
    "my-website-bucket",
    website_configuration=aws_native.s3.BucketWebsiteConfigurationArgs(
        index_document="index.html"
    ),
)

content_dir = "www"
for file in os.listdir(content_dir):
    filepath = os.path.join(content_dir, file)
    mime_type, _ = mimetypes.guess_type(filepath)
    obj = aws_classic.s3.BucketObject(
        file,
        bucket=bucket.id,
        source=pulumi.FileAsset(filepath),
        content_type=mime_type,
        opts=pulumi.ResourceOptions(parent=bucket)
    )

bucket_policy = aws_classic.s3.BucketPolicy(
    "my-website-bucket-policy",
    bucket=bucket.id,
    policy=bucket.arn.apply(
        lambda arn:  json.dumps({
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
    opts=pulumi.ResourceOptions(parent=bucket)

)

pulumi.export("bucket_name", bucket.bucket_name)
```

Before we update our pulumi program, let's add one final line of code.

```python
pulumi.export('website_url', bucket.website_url)
```

This exports the website endpoint so we can view the contents of our bucket.

Deploy the changes:

```bash
pulumi up
```

And you'll see the BucketPolicy get added. You'll also get a URL as an `output`. We can now view the contents of that URL using `curl`:

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

You'll see Pulumi tell you it's destroying infrastructure:


```
Previewing destroy (dev)

View Live: https://app.pulumi.com/jaxxstorm/iac-workshop/dev/previews/33ce7a73-6d53-42df-854e-296a6276231d

     Type                       Name                      Plan       
 -   pulumi:pulumi:Stack        iac-workshop-dev          delete     
 -   ├─ aws:s3:BucketPolicy     my-website-bucket-policy  delete     
 -   └─ aws-native:s3:Bucket    my-website-bucket         delete     
 -      ├─ aws:s3:BucketObject  python.png                delete     
 -      └─ aws:s3:BucketObject  index.html                delete     
 
Outputs:
  - bucket_name: "my-website-bucket-4d8d96b"
  - website_url: "my-website-bucket-4d8d96b.s3-website-us-west-2.amazonaws.com"

Resources:
    - 5 to delete
```

Hit yes, and watch your bucket disappear.

Finally, remove the stack:

```
pulumi stack rm dev
```
