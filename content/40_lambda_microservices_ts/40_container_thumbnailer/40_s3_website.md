+++
title = "3.5 Make a thumbnail website"
chapter = false
weight = 50
+++

The final step is to make a website to display our images. We can use S3's static content host for this.

## Step 1 &mdash; Add a bucket policy

Add the following to your `index.ts` file:

```typescript
const bucketPolicy = new aws.s3.BucketPolicy("thumbnailer", {
    bucket: bucket.id,
    policy: bucket.arn.apply(arn => JSON.stringify({
        "Version": "2012-10-17",
            "Statement": [{
                "Effect": "Allow",
                "Principal": "*",
                "Action": [
                    "s3:*"
                ],
                "Resource": [
                    `${arn}/*`,
                    `${arn}`
                ]
            }]
    }))
})
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

const image = awsx.ecr.buildAndPushImage("thumbnailer", {
    context: "./app",
});

// A bucket to store videos and thumbnails.
const bucket = new aws.s3.Bucket("thumbnailer", {
    forceDestroy: true,
});

const bucketPolicy = new aws.s3.BucketPolicy("thumbnailer", {
    bucket: bucket.id,
    policy: bucket.arn.apply(arn => JSON.stringify({
        "Version": "2012-10-17",
            "Statement": [{
                "Effect": "Allow",
                "Principal": "*",
                "Action": [
                    "s3:*"
                ],
                "Resource": [
                    `${arn}/*`,
                    `${arn}`
                ]
            }]
    }))
})

const role = new aws.iam.Role("thumbnailerRole", {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({ Service: "lambda.amazonaws.com" }),
});
const lambdaFullAccess =  new aws.iam.RolePolicyAttachment("lambdaFullAccess", {
    role: role.name,
    policyArn: aws.iam.ManagedPolicy.LambdaFullAccess,
});

const lambdaBasicExecutionRole = new aws.iam.RolePolicyAttachment("basicExecutionRole", {
    role: role.name,
    policyArn: aws.iam.ManagedPolicy.AWSLambdaBasicExecutionRole,
});

const thumbnailer = new aws.lambda.Function("thumbnailer", {
    packageType: "Image",
    imageUri: image.imageValue,
    role: role.arn,
    timeout: 900,
});

// When a new video is uploaded, run the FFMPEG task on the video file.
// Use the time index specified in the filename (e.g. cat_00-01.mp4 uses timestamp 00:01)
bucket.onObjectCreated("onNewVideo", thumbnailer, { filterSuffix: ".mp4" });

// When a new thumbnail is created, log a message.
bucket.onObjectCreated("onNewThumbnail", new aws.lambda.CallbackFunction<aws.s3.BucketEvent, void>("onNewThumbnail", {
    callback: async bucketArgs => {
        console.log("onNewThumbnail called");
        if (!bucketArgs.Records) {
            return;
        }

        for (const record of bucketArgs.Records) {
            console.log(`*** New thumbnail: file ${record.s3.object.key} was saved at ${record.eventTime}.`);
        }
    },
    policies: [
        aws.iam.ManagedPolicy.LambdaFullAccess,                 // Provides wide access to "serverless" services (Dynamo, S3, etc.)
        aws.iam.ManagedPolicy.AWSLambdaBasicExecutionRole,
    ],
}), { filterSuffix: ".jpg" });

export const bucketName = bucket.id;
```

## Step 2 &mdash; Make the S3 Bucket a Static Website

Now we need to update our bucket to add a parameter

Update your bucket resource with the following in your `index.ts` file:

```typescript
const bucket = new aws.s3.Bucket("thumbnailer", {
    forceDestroy: true,
    website: {
        indexDocument: "cat.jpg",
    }
});
```

Also, make sure you export the website url:

```typescript
export const url = bucket.websiteEndpoint
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

const image = awsx.ecr.buildAndPushImage("thumbnailer", {
    context: "./app",
});

// A bucket to store videos and thumbnails.
const bucket = new aws.s3.Bucket("thumbnailer", {
    forceDestroy: true,
    website: {
        indexDocument: "cat.jpg",
    }
});

const bucketPolicy = new aws.s3.BucketPolicy("thumbnailer", {
    bucket: bucket.id,
    policy: bucket.arn.apply(arn => JSON.stringify({
        "Version": "2012-10-17",
            "Statement": [{
                "Effect": "Allow",
                "Principal": "*",
                "Action": [
                    "s3:*"
                ],
                "Resource": [
                    `${arn}/*`,
                    `${arn}`
                ]
            }]
    }))
})

const role = new aws.iam.Role("thumbnailerRole", {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({ Service: "lambda.amazonaws.com" }),
});
const lambdaFullAccess =  new aws.iam.RolePolicyAttachment("lambdaFullAccess", {
    role: role.name,
    policyArn: aws.iam.ManagedPolicy.LambdaFullAccess,
});
const lambdaBasicExecutionRole = new aws.iam.RolePolicyAttachment("basicExecutionRole", {
    role: role.name,
    policyArn: aws.iam.ManagedPolicy.AWSLambdaBasicExecutionRole,
});

const thumbnailer = new aws.lambda.Function("thumbnailer", {
    packageType: "Image",
    imageUri: image.imageValue,
    role: role.arn,
    timeout: 900,
});

// When a new video is uploaded, run the FFMPEG task on the video file.
// Use the time index specified in the filename (e.g. cat_00-01.mp4 uses timestamp 00:01)
bucket.onObjectCreated("onNewVideo", thumbnailer, { filterSuffix: ".mp4" });

// When a new thumbnail is created, log a message.
bucket.onObjectCreated("onNewThumbnail", new aws.lambda.CallbackFunction<aws.s3.BucketEvent, void>("onNewThumbnail", {
    callback: async bucketArgs => {
        console.log("onNewThumbnail called");
        if (!bucketArgs.Records) {
            return;
        }

        for (const record of bucketArgs.Records) {
            console.log(`*** New thumbnail: file ${record.s3.object.key} was saved at ${record.eventTime}.`);
        }
    },
    policies: [
        aws.iam.ManagedPolicy.LambdaFullAccess,                 // Provides wide access to "serverless" services (Dynamo, S3, etc.)
        aws.iam.ManagedPolicy.AWSLambdaBasicExecutionRole,
    ],
}), { filterSuffix: ".jpg" });

export const bucketName = bucket.id;
export const url = bucket.websiteEndpoint

```

## Step 3 &mdash; Provision your website

At this stage we're ready to provision our infrastructrue again. Run `pulumi up` and observe the resources we're going to provision:

```
Updating (dev)

View Live: https://app.pulumi.com/jaxxstorm/lambda-thumbnailer/dev/updates/9

     Type                    Name                    Status      Info
     pulumi:pulumi:Stack     lambda-thumbnailer-dev              
     ├─ awsx:ecr:Repository  thumbnailer                         1 warning
 +   └─ aws:s3:BucketPolicy  thumbnailer             created     
 
Diagnostics:
  awsx:ecr:Repository (thumbnailer):
    warning: WARNING! Your password will be stored unencrypted in /home/ec2-user/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store
 
Outputs:
    bucketName: "thumbnailer-8414171"
    url       : "thumbnailer-8414171.s3-website-us-west-2.amazonaws.com"

Resources:
    + 1 created
    16 unchanged
```

Hit `yes` and provision your infrastructure. You can now visit the chosen URL to see a lovely image of a cat.

## Step 4 &mdash; Destroy Everything

Finally, destroy the resources and the stack itself:

```
pulumi destroy
pulumi stack rm
```







