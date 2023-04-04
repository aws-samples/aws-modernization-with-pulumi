+++
title = "1.2 Configuring AWS"
chapter = false
weight = 10
+++

Now that we have a basic project, let's add the Pulumi AWS provider and configure our credentials.

## Step 1 &mdash; Install the AWS Provider

Run the following command:

```bash
npm i @pulumi/aws
```

This will install the Pulumi AWS node SDK and add it to your `package.json` file. This is the library that will allow us to manage AWS assets with Pulumi. Pulumi also supports a wide range of other providers. For a complete list of all supported providers, see the [Pulumi Registry](https://www.pulumi.com/registry).

## Step 2 &mdash; Import the AWS Provider

Now that the AWS package is installed, we need to import it as part of our project.

Add the following to the top of your `index.ts`:

```typescript
import * as aws from "@pulumi/aws";
```

After this change, your `index.ts` should look like this:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
```

## Step 3 &mdash; Configure an AWS Region

Configure the AWS region you would like to deploy to, replacing `us-east-1` with your AWS region of choice:

```bash
pulumi config set aws:region us-east-1
```

{{% notice info %}}
Setting the AWS region is not strictly necessary, but we do it here to demonstrate how to set stack configuration values. If you do not set the region, Pulumi will use the default region for your AWS profile.
{{% /notice %}}

Note that the previous command will create the file `Pulumi.dev.yaml` which contains the configuration for our `dev` stack. (Stacks are logical groupings of Pulumi resources.) We will be working with a single Pulumi stack in this tutorial, but we could define additional stacks to deploy our infrastructure to different regions/accounts with different parameters. To learn more about Pulumi stacks, see [Stacks](https://www.pulumi.com/docs/intro/concepts/stack/) in the Pulumi docs.

## (Optional) Step 4 &mdash; Configure an AWS Profile

If you are using an AWS profile other than the default, you can tell Pulumi which to use in one of two ways:

* Using an environment variable: `export AWS_PROFILE=<profile name>`
* Using configuration: `pulumi config set aws:profile <profile name>`
