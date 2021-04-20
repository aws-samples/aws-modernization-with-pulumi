+++
title = "3.4 Define Lambda Container Function"
chapter = false
weight = 20
+++

In the previous step, we built and pushed a Lambda container to an ECR repository. Now let's define a lambda function whichs runs this container

## Step 1 &mdash; Declare an S3 Bucket and IAM Role

Add the following to your `index.ts` file:

```typescript
// A bucket to store videos and thumbnails.
const bucket = new aws.s3.Bucket("thumbnailer",{
    forceDestroy: true,
});


const role = new aws.iam.Role("thumbnailerRole", {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({ Service: "lambda.amazonaws.com" }),
});
const lambdaFullAccess =  new aws.iam.RolePolicyAttachment("lambdaFullAccess", {
    role: role.name,
    policyArn: aws.iam.ManagedPolicy.LambdaFullAccess,
});
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi"
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

// A bucket to store videos and thumbnails.
const bucket = new aws.s3.Bucket("thumbnailer", {
    forceDestroy: true,
});

const image = awsx.ecr.buildAndPushImage("thumbnailer", {
    context: "./app",
});
const role = new aws.iam.Role("thumbnailerRole", {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({ Service: "lambda.amazonaws.com" }),
});
const lambdaFullAccess =  new aws.iam.RolePolicyAttachment("lambdaFullAccess", {
    role: role.name,
    policyArn: aws.iam.ManagedPolicy.LambdaFullAccess,
});
```

## Step 2 &mdash; Add the Lambda Function

We've created the role we need, let's now define our Lambda function.

Add the following to your `index.ts` file:

```typescript
const thumbnailer = new aws.lambda.Function("thumbnailer", {
    packageType: "Image",
    imageUri: image.imageValue,
    role: role.arn,
    timeout: 900,
});
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi"
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

// A bucket to store videos and thumbnails.
const bucket = new aws.s3.Bucket("thumbnailer", {
    forceDestroy: true,
});

const image = awsx.ecr.buildAndPushImage("thumbnailer", {
    context: "./app",
});
const role = new aws.iam.Role("thumbnailerRole", {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({ Service: "lambda.amazonaws.com" }),
});
const lambdaFullAccess =  new aws.iam.RolePolicyAttachment("lambdaFullAccess", {
    role: role.name,
    policyArn: aws.iam.ManagedPolicy.LambdaFullAccess,
});

const thumbnailer = new aws.lambda.Function("thumbnailer", {
    packageType: "Image",
    imageUri: image.imageValue,
    role: role.arn,
    timeout: 900,
});
```

Notice we're referencing the image we built earlier and passing it to the function.

## Step 3 &mdash; Add the Lambda Event

We want our Lambda function to trigger when we upload files to it. We can do this inline within our Pulumi program.

Add the following to your `index.ts` file:

```typescript
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
    ],
}), { filterSuffix: ".jpg" });
```
{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi"
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

// A bucket to store videos and thumbnails.
const bucket = new aws.s3.Bucket("thumbnailer", {
    forceDestroy: true,
});

const image = awsx.ecr.buildAndPushImage("thumbnailer", {
    context: "./app",
});
const role = new aws.iam.Role("thumbnailerRole", {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({ Service: "lambda.amazonaws.com" }),
});
const lambdaFullAccess =  new aws.iam.RolePolicyAttachment("lambdaFullAccess", {
    role: role.name,
    policyArn: aws.iam.ManagedPolicy.LambdaFullAccess,
});

const thumbnailer = new aws.lambda.Function("thumbnailer", {
    packageType: "Image",
    imageUri: image.imageValue,
    role: role.arn,
    timeout: 900,
});

bucket.onObjectCreated("onNewVideo", thumbnailer, { filterSuffix: ".mp4" });

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
        aws.iam.ManagedPolicy.LambdaFullAccess,
    ],
}), { filterSuffix: ".jpg" });
```

We've added two events here. One of the events waits for videos to be uploaded to our S3 bucket with the suffix `.mp4` and if that happenes, it triggers the thumbnailer lambda container.

The second function waits for a thumbnail to be written and defines an inline Lambda callback function which logs a message for us to read.

## Step 4 &mdash; Export the Bucket name

We need to know where to upload our videos to, so let's export our bucket name from our Pulumi program. Add the following line to the end of your `index.ts

```typescript
// Export the bucket name.
export const bucketName = bucket.id;
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi"
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

// A bucket to store videos and thumbnails.
const bucket = new aws.s3.Bucket("thumbnailer", {
    forceDestroy: true,
});

const image = awsx.ecr.buildAndPushImage("thumbnailer", {
    context: "./app",
});
const role = new aws.iam.Role("thumbnailerRole", {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({ Service: "lambda.amazonaws.com" }),
});
const lambdaFullAccess =  new aws.iam.RolePolicyAttachment("lambdaFullAccess", {
    role: role.name,
    policyArn: aws.iam.ManagedPolicy.LambdaFullAccess,
});

const thumbnailer = new aws.lambda.Function("thumbnailer", {
    packageType: "Image",
    imageUri: image.imageValue,
    role: role.arn,
    timeout: 900,
});

bucket.onObjectCreated("onNewVideo", thumbnailer, { filterSuffix: ".mp4" });

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
        aws.iam.ManagedPolicy.LambdaFullAccess,
    ],
}), { filterSuffix: ".jpg" });

export const bucketName = bucket.id;
```

## Step 4 &mdash; Provision your function

At this stage we're ready to provision our infrastructrue again. Run `pulumi up` and observe the resources we're going to provision:

```
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/thumbnailer/dev/previews/40e76859-5ef4-4a79-a7cf-abc39108c0cb

     Type                                  Name                     Plan       Info
     pulumi:pulumi:Stack                   thumbnailer-dev
 +   ├─ aws:s3:Bucket                      thumbnailer              create
 +   │  ├─ aws:s3:BucketEventSubscription  onNewThumbnail           create
 +   │  │  └─ aws:lambda:Permission        onNewThumbnail           create
 +   │  ├─ aws:s3:BucketEventSubscription  onNewVideo               create
 +   │  │  └─ aws:lambda:Permission        onNewVideo               create
 +   │  └─ aws:s3:BucketNotification       onNewVideo               create
 +   ├─ aws:iam:Role                       onNewThumbnail           create
 +   ├─ aws:iam:Role                       thumbnailerRole          create
 +   ├─ aws:iam:RolePolicyAttachment       lambdaFullAccess         create
 +   ├─ aws:iam:RolePolicyAttachment       onNewThumbnail-32be53a2  create
     ├─ awsx:ecr:Repository                thumbnailer                         1 warning
 +   ├─ aws:lambda:Function                onNewThumbnail           create
 +   └─ aws:lambda:Function                thumbnailer              create
```

Hit `yes` and provision your infrastructure. You should see your bucket name displayed as an Output:

```
Outputs:
  + bucketName: "thumbnailer-f91a64e"

Resources:
    + 12 created
    4 unchanged
```

## Step 5 &mdash; Upload an example video

{{% notice info %}}
If you need an exampe video, try the one in our examples repo:
https://github.com/pulumi/examples/tree/master/aws-ts-lambda-thumbnailer/sample
{{% /notice %}}

Let's trigger our function by uploading an `.mp4` video to the bucket:

```
aws s3 cp ./sample/cat.mp4 s3://$(pulumi stack output bucketName)/cat_00-01.mp4                                                                                                                    
```

You should see the file get uploaded to s3:

```
upload: sample/cat.mp4 to s3://thumbnailer-f91a64e/cat_00-01.mp4
```

You can view the logs from your function to see if the functions were triggered:


```
2020-12-02T14:38:18.423-08:00[           thumbnailer-8d60c20] START RequestId: 07106ad0-f7dd-440e-99dd-984d90b50f3c Version: $LATEST
 2020-12-02T14:38:18.428-08:00[           thumbnailer-8d60c20] 2020-12-02T22:38:18.425Z	07106ad0-f7dd-440e-99dd-984d90b50f3c	INFO	Video handler called
 2020-12-02T14:38:18.449-08:00[           thumbnailer-8d60c20] 2020-12-02T22:38:18.449Z	07106ad0-f7dd-440e-99dd-984d90b50f3c	INFO	aws s3 cp s3://thumbnailer-f91a64e/cat_00-01.mp4 /tmp/cat_00-01.mp4
download: s3://thumbnailer-f91a64e/cat_00-01.mp4 to ../../tmp/cat_00-01.mp46.0 KiB/666.5 KiB (1.1 MiB/s) with 1 file(s) remaining
 2020-12-02T14:38:34.789-08:00[           thumbnailer-8d60c20] 2020-12-02T22:38:34.788Z	07106ad0-f7dd-440e-99dd-984d90b50f3c	INFO	ffmpeg -v error -i /tmp/cat_00-01.mp4 -ss 00:01 -vframes 1 -f image2 -an -y /tmp/cat.jpg
 2020-12-02T14:38:45.198-08:00[           thumbnailer-8d60c20] 2020-12-02T22:38:45.197Z	07106ad0-f7dd-440e-99dd-984d90b50f3c	INFO	aws s3 cp /tmp/cat.jpg s3://thumbnailer-f91a64e/cat.jpg
upload: ../../tmp/cat.jpg to s3://thumbnailer-f91a64e/cat.jpg     pleted 86.6 KiB/86.6 KiB (280.9 KiB/s) with 1 file(s) remaining
 2020-12-02T14:38:55.910-08:00[           thumbnailer-8d60c20] 2020-12-02T22:38:55.910Z	07106ad0-f7dd-440e-99dd-984d90b50f3c	INFO	*** New thumbnail: file cat_00-01.mp4 was saved at 2020-12-02T22:38:11.842Z.
 2020-12-02T14:38:55.939-08:00[           thumbnailer-8d60c20] END RequestId: 07106ad0-f7dd-440e-99dd-984d90b50f3c
 2020-12-02T14:38:55.939-08:00[           thumbnailer-8d60c20] REPORT RequestId: 07106ad0-f7dd-440e-99dd-984d90b50f3c	Duration: 37508.62 ms	Billed Duration: 38316 ms	Memory Size: 128 MB	Max Memory Used: 128 MB	Init Duration: 807.08 ms
 2020-12-02T14:38:56.144-08:00[        onNewThumbnail-1cb1a7d] START RequestId: 6a5ead91-f030-4b3b-a22a-c3cc16ce7864 Version: $LATEST
 2020-12-02T14:38:56.158-08:00[        onNewThumbnail-1cb1a7d] 2020-12-02T22:38:56.158Z	6a5ead91-f030-4b3b-a22a-c3cc16ce7864	INFO	onNewThumbnail called
 2020-12-02T14:38:56.158-08:00[        onNewThumbnail-1cb1a7d] 2020-12-02T22:38:56.158Z	6a5ead91-f030-4b3b-a22a-c3cc16ce7864	INFO	*** New thumbnail: file cat.jpg was saved at 2020-12-02T22:38:49.839Z.
 2020-12-02T14:38:56.175-08:00[        onNewThumbnail-1cb1a7d] END RequestId: 6a5ead91-f030-4b3b-a22a-c3cc16ce7864
 2020-12-02T14:38:56.175-08:00[        onNewThumbnail-1cb1a7d] REPORT RequestId: 6a5ead91-f030-4b3b-a22a-c3cc16ce7864	Duration: 30.58 ms	Billed Duration: 31 ms	Memory Size: 128 MB	Max Memory Used: 64 MB	Init Duration: 160.74 ms
```

You can see that the thumbnailer generated a `cat.jpg` for our video! Let's download it:

```
aws s3 cp s3://$(pulumi stack output bucketName)/cat.jpg .
```
