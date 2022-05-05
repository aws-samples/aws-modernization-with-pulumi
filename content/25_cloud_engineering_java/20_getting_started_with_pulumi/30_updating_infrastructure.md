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

Create a directory `src/main/resources/www` and add a new `index.html` file with the following contents:

```html
<html>
    <head><meta charset="UTF-8">
    <title>Hello, Pulumi!</title></head>
<body>
    <p>Hello, S3!</p>
    <p>Made with ❤️ with <a href="https://pulumi.com">Pulumi</a> and Java</p>
    <img src="java.png" />
</body></html>
```

Next, download a java image to the `src/main/resources/www` directory:

```
curl https://upload.wikimedia.org/wikipedia/en/thumb/3/30/Java_programming_language_logo.svg/1200px-Java_programming_language_logo.svg.png -o src/main/resources/www/java.png
```

Now we've created our example website, we're ready to upload these files to our s3 bucket website.

## Step 2 &mdash; Upload the Bucket Contents

Back inside our Pulumi program, we now need to upload these contents to our bucket. 
We'll list the contents of the `www` directory and for each file, we'll add a new Pulumi resource called `BucketObject`.

```java
readFilesFromDirectory("www").forEach(path ->
        new BucketObject(path.getFileName().toString(),
            BucketObjectArgs.builder()
                .bucket(bucket.getId())
                .source(new FileAsset(path.toString()))
                .contentType(guessContentType(path))
                .build(),

        CustomResourceOptions.builder()
            .dependsOn(bucket)
            .build()
));

private static Stream<Path> readFilesFromDirectory(String classPathDir) {
    try {
        var normalizedPath = classPathDir.startsWith("/") ? classPathDir : "/" + classPathDir;
        var directoryPath = Path.of(Objects.requireNonNull(App.class.getResource(normalizedPath)).toURI());
        return Files.walk(directoryPath).filter(Files::isRegularFile);
    } catch (IOException | URISyntaxException error) {
        throw new RuntimeException(error);
    }
}
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
 +      ├─ aws:s3:BucketObject  java.png         create     
 +      └─ aws:s3:BucketObject  index.html         create     
 
Resources:
    + 2 to create
    2 unchanged
```

We can now list the contents of our bucket again and see the files have been uploaded:

```bash
$  aws s3 ls $(pulumi stack output bucket_name)
2022-05-04 16:51:58        218 index.html
2022-05-04 16:52:00     118587 java.png
```

## Step 3 &mdash; Add a Bucket Policy

Now we have an S3 bucket and some objects in it, we need to make the bucket accessible so you can see it. Currently, the objects in the bucket are private.

There are a few methods to manage this, but we're going to do it by adding a bucket policy to the bucket that allows objects to be read.

We're going to be dealing with a JSON string here, so let's add a new import so we can deal structured json:

Create a new bucket policy object in your Pulumi program like so:
```java
var bucketPolicy = new BucketPolicy("my-website-bucket-policy",
        BucketPolicyArgs.builder()
            .bucket(bucket.getId())
            .policy(bucket.arn().applyValue(arn -> """
                    {
                        "Version": "2012-10-17",
                        "Statement": [
                            {
                                "Effect": "Allow",
                                "Principal": "*",
                                "Action": ["s3:GetObject"],
                                "Resource": ["%s/*"]
                             }
                        ]
                    }""".formatted(arn)))
        .build()
);
```

Make a note of the policy object here. For the 'policy' argument, you can either pass a raw java String or an Output<<String>String>. 
The latter pattern lets you reference things that will be only available once the entire stack is run - in this particular example, we construct the policy based on bucket.arn(), which is not yet known at the time we write the code (and thus, it's return type is Output<<String>String>).

{{% notice info %}}
Your workshop host will explain Outputs here. If you're following along on your own, please see [this video](https://www.youtube.com/watch?v=lybOxul2otM) for an explanation
{{% /notice %}}

> :white_check_mark: After this change, your `App.java` should look like this:
```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.asset.FileAsset;
import com.pulumi.aws.s3.*;
import com.pulumi.aws.s3.inputs.BucketWebsiteArgs;
import com.pulumi.resources.CustomResourceOptions;

import java.io.IOException;
import java.net.URISyntaxException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Objects;
import java.util.stream.Stream;

public class App {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var bucket = new Bucket("my-website-bucket",
                    BucketArgs.builder()
                            .website(BucketWebsiteArgs.builder().indexDocument("index.html").build())
                            .build()
            );

            var bucketPolicy = new BucketPolicy("my-website-bucket-policy",
                    BucketPolicyArgs.builder()
                            .bucket(bucket.getId())
                            .policy(bucket.arn().applyValue(arn -> """
                                    {
                                        "Version": "2012-10-17",
                                        "Statement": [
                                            {
                                                "Effect": "Allow",
                                                "Principal": "*",
                                                "Action": ["s3:GetObject"],
                                                "Resource": ["%s/*"]
                                             }
                                        ]
                                    }""".formatted(arn)))
                            .build()
            );

            readFilesFromDirectory("www").forEach(path ->
                    new BucketObject(path.getFileName().toString(),
                            BucketObjectArgs.builder()
                                    .bucket(bucket.getId())
                                    .source(new FileAsset(path.toString()))
                                    .contentType(guessContentType(path))
                                    .build(),

                            CustomResourceOptions.builder()
                                    .dependsOn(bucket)
                                    .build()
                    ));

            ctx.export("bucket_name", bucket.bucket());
        });
    }

    private static Stream<Path> readFilesFromDirectory(String classPathDir) {
        try {
            var normalizedPath = classPathDir.startsWith("/") ? classPathDir : "/" + classPathDir;
            var directoryPath = Path.of(Objects.requireNonNull(App.class.getResource(normalizedPath)).toURI());
            return Files.walk(directoryPath).filter(Files::isRegularFile);
        } catch (IOException | URISyntaxException error) {
            throw new RuntimeException(error);
        }
    }

    private static String guessContentType(Path file) {
        try {
            return Files.probeContentType(file);
        } catch (IOException error) {
            throw new RuntimeException(error);
        }
    }
}
```

Before we update our pulumi program, let's add one final line of code.

```java
ctx.export("website_url", bucket.websiteEndpoint());
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
 -      ├─ aws:s3:BucketObject  java.png                  delete     
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
