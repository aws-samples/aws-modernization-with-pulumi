+++
title = "3.2 Installing and Configuring the AWS and AWSX Providers"
chapter = false
weight = 10
+++

Now that we have our project boilerplate, we will add 2 [Pulumi providers](https://www.pulumi.com/docs/intro/concepts/resources/providers/):

1. [AWS Classic](https://www.pulumi.com/registry/packages/aws/), which gives us all the fundamental AWS resources, like VPC subnets.
1. [AWSx](https://www.pulumi.com/registry/packages/awsx/), which contains higher level [Pulumi components](https://www.pulumi.com/docs/intro/concepts/resources/components/), like a full, production-ready VPC that includes subnets, NAT gateways, routing tables, and so on.

## Step 1 &mdash; Install the AWS and AWSx Packages

Run the following commands to install the AWS Classic and AWSX packages:

```bash
npm i @pulumi/aws @pulumi/awsx
```

## Step 2 &mdash; Import the AWS Package

Now that our packages are installed, we need to import them as part of our project.

Add the following to your `index.ts`:

```typescript
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
```

After this change, your `__main__.py` should look like this:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
```

## (Optional) Step 3 &mdash; Configure an AWS Region

Configure the AWS region you would like to deploy to, replacing `us-east-1` with your AWS region of choice:

```bash
pulumi config set aws:region us-east-1
```

Note that the previous command will create the file `Pulumi.dev.yaml` which contains the configuration for our `dev` stack. (Stacks are logical groupings of Pulumi resources.) We will be working with a single Pulumi stack in this tutorial, but we could define additional stacks to deploy our infrastructure to different regions/accounts with different parameters. To learn more about Pulumi stacks, see [Stacks](https://www.pulumi.com/docs/intro/concepts/stack/) in the Pulumi docs.

## (Optional) Step 4 &mdash; Configure an AWS Profile

If you are using an alternative AWS profile, you can tell Pulumi which to use in one of two ways:

* Using an environment variable: `export AWS_PROFILE=<profile name>`
* Using configuration: `pulumi config set aws:profile <profile name>`
