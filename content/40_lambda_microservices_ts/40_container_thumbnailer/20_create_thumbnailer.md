+++
title = "3.3 Thumbnailer Container"
chapter = false
weight = 20
+++

Now that you have a project configured to use AWS, we can create the thumbnailer. We'll use the AWS Lambda Containers for this.

## Step 1 &mdash; Create a Docker Image

Create a new directory within your Pulumi project called `app`

```bash
mkdir app
```

Next, create a `Dockerfile` within the `app` directory and use it to build ffmpeg into the image:

```dockerfile
FROM amazon/aws-lambda-nodejs:12
ARG FUNCTION_DIR="/var/task"

# Install tar and xz
RUN yum install tar xz unzip -y

# Install awscli
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" -s
RUN unzip -q awscliv2.zip
RUN ./aws/install

# Install ffmpeg
RUN curl https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz -o ffmpeg.tar.xz -s
RUN tar -xf ffmpeg.tar.xz
RUN mv ffmpeg-4.4-amd64-static/ffmpeg /usr/bin

# Create function directory
RUN mkdir -p ${FUNCTION_DIR}

# Copy handler function and package.json
COPY index.js ${FUNCTION_DIR}

# Set the CMD to your handler (could also be done as a parameter override outside of the Dockerfile)
CMD [ "index.handler" ]
```

You'll notice you're specifying an `index.handler` as the command, but we haven't defined it yet. 

## Step 2 &mdash; Define your lambda script

Inside the `app/` directory, let's create `index.js` which becomes your Lambda entrypoint.

```javascript
'use strict';
const { execSync } = require('child_process');

function run(command) {
    console.log(command);
    const result = execSync(command, {stdio: 'inherit'});
    if (result != null) {
        console.log(result.toString());
    }
}

exports.handler = async (event) => {
    console.log("Video handler called");

    if (!event.Records) {
        return;
    }

    for (const record of event.Records) {
        const fileName = record.s3.object.key;
        const bucketName = record.s3.bucket.name;
        const thumbnailFile = fileName.substring(0, fileName.indexOf("_")) + ".jpg";
        const framePos = fileName.substring(fileName.indexOf("_")+1, fileName.indexOf(".")).replace("-", ":");
        
        run(`aws s3 cp s3://${bucketName}/${fileName} /tmp/${fileName}`);
        run(`ffmpeg -v error -i /tmp/${fileName} -ss ${framePos} -vframes 1 -f image2 -an -y /tmp/${thumbnailFile}`);
        run(`aws s3 cp /tmp/${thumbnailFile} s3://${bucketName}/${thumbnailFile}`);

        console.log(`*** New thumbnail: file ${fileName} was saved at ${record.eventTime}.`);
    }    
};
```

{{% notice info %}}
The `app` directory should now look like this:
{{% /notice %}}
```
.
├── Dockerfile
└── index.js
```


## Step 3 &mdash; Build Docker Image & Push to ECR

Back inside your Pulumi program (`index.ts` in the directory above the `app` directory) we now need to build this Docker image and create a place to push it.

We'll use Pulumi Crosswalk to build the image and push it to the ECR repository. Add the following to `index.ts`:

```typescript
const image = awsx.ecr.buildAndPushImage("thumbnailer", {
    context: "./app",
});
```

{{% notice info %}}
The `index.ts` file should now look like this:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

const image = awsx.ecr.buildAndPushImage("thumbnailer", {
    context: "./app",
});

```

This Pulumi component takes care of creating an ECR repository and building the local container image and pushing it.

## Step 4 &mdash; Run the Pulumi Program

Run your Pulumi program and inspect the resources it'll create:

```
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/thumbnailer/dev/previews/4a0d6e8e-1b71-4c97-8aae-5bfb5c10d419

     Type                           Name             Plan
 +   pulumi:pulumi:Stack            thumbnailer-dev  create
 +   └─ awsx:ecr:Repository         thumbnailer      create
 +      ├─ aws:ecr:Repository       thumbnailer      create
 +      └─ aws:ecr:LifecyclePolicy  thumbnailer      create

Resources:
    + 4 to create
```

Hit yes when you're ready, you should see some output while the image builds in the background this may take a few minutes:

```
Updating (dev)

View Live: https://app.pulumi.com/jaxxstorm/thumbnailer/dev/updates/1

     Type                           Name             Status       Info
 +   pulumi:pulumi:Stack            thumbnailer-dev  creating
 +   └─ awsx:ecr:Repository         thumbnailer      created      Executing ' docker build ./app -t 12fda807-container'
 +      ├─ aws:ecr:Repository       thumbnailer      created
 +      └─ aws:ecr:LifecyclePolicy  thumbnailer      created
```



