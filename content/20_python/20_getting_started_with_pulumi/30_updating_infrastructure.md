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
curl -o www/python.png https://www.python.org/static/community_logos/python-logo-master-v3-TM-flattened.png
```

Now we've created our example website, we're ready to upload these files to our s3 bucket website.

## Step 2 &mdash; Upload the Bucket Contents

Back inside our Pulumi program, we now need to upload these contents to our bucket. Because we're using Python, we can use a simple for loop over the contents of the `www` directory:

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
        opts=pulumi.ResourceOptions(parent=bucket)
    )
```

Deploy the changes:

```bash
pulumi up
```

This will give you a preview and selecting `yes` will apply the changes:

```

insert output here

```

## Step 3 &mdash; Add a Bucket Policy

Now we have an S3 bucket and some objects in it, we need to make the bucket accessible so you can see it. Currently, the objects in the bucket are private.

There are a few methods to manage this, but we're going to do it by adding a bucket policy to the bucket that allows objects to be read.

Create a new bucket policy object in your Pulumi program like so:

```python
bucket_policy = aws.s3.BucketPolicy(
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
import pulumi_aws as aws

bucket = aws.s3.Bucket(
    "my-website-bucket",
    website=s3.BucketWebsiteArgs(
        index_document="index.html",
    ),
)

content_dir = "www"
for file in os.listdir(content_dir):
    filepath = os.path.join(content_dir, file)
    mime_type, _ = mimetypes.guess_type(filepath)
    obj = aws.s3.BucketObject(
        file,
        bucket=bucket.id,
        source=pulumi.FileAsset(filepath),
        content_type=mime_type,
        opts=pulumi.ResourceOptions(parent=bucket)
    )

bucket_policy = aws.s3.BucketPolicy(
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

Before we update our pulumi program, let's add one final line of code.

```python
pulumi.export('website_url', bucket.website_endpoint)
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
