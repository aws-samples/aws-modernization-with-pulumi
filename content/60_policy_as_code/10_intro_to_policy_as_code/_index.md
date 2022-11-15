+++
title = "Module 1: Intro to Policy as Code in Pulumi"
chapter = true
weight = 10
+++

## Intro to Policy as Code in Pulumi

{{% notice warning %}}<p> You are responsible for the cost of the AWS services used while running this workshop in your AWS account.</p> {{% /notice %}}

In this module, you will use Pulumi's policy as code features to ensure compliant infrastructure in AWS.

## Step 1: Create the directory structure

We will start by creating a new directory to hold our code for this workshop and changing into it:

```bash
mkdir workshop-policy-as-code
cd workshop-policy-as-code
```

Within our project, we will organize our infrastructure code (our Pulumi stack) and our policy packs into separate directories. Let's create them now:

```bash
mkdir infra policy
```

## Step 2: Initialize the infrastructure code

Let's initialize our infrastructure stack:

```bash
cd infra
pulumi new aws-python -n workshop-policy-as-code
```

Accept the defaults for all prompts except possibly the AWS Region - choose a region near to you.

Replace your `__main__.py` with the following content, which alters the S3 bucket to use the `pulumi-read-write` ACL:

```python
"""An AWS Python Pulumi program"""

import pulumi
import pulumi_aws as aws

# Create an AWS resource (S3 Bucket)
bucket = aws.s3.Bucket(
    'my-bucket',
    aws.s3.BucketArgs(
        acl="public-read-write",
    ),
)

# Export the name of the bucket
pulumi.export('bucket_name', bucket.id)
```

After Pulumi has finished initializing the project, ensure you've activated the Python virtual environment:

```bash
source venv/bin/activate
```

## Step 3: Install Pulumi policy pre-requisites

In order to run the policy pack we are going to create, we need to add the `pulumi_policy` to our dependencies, even though we will not be using it directly in our Pulumi program. (Adding `pulumi_policy` is necessary due to the way Pulumi policy packs are implemented in Python.)

Run the following command to add `pulumi_policy` as a dependency in the `infra` directory:

```bash
echo "pulumi_policy>=1.5.0,<2.0.0" >> requirements.txt
```

Now we can install all of our dependencies:

```bash
pip install -r requirements.txt
```

## Step 4: Initialize and run a Pulumi policy pack

Now we are ready to initialize a basic policy pack. Change to the `policy` directory:

```bash
cd ../policy
```

Initialize a Pulumi policy pack by running the following command:

```bash
pulumi policy new aws-python
```

This command will create a Pulumi policy pack with a single policy that ensures S3 buckets do not have the `public-read` nor the `public-read-write` ACL. Feel free to check out `policy/__main__.py` for the details, but we'll explore its contents author our own rules in a future module.

Back in the `infra/` directory, let's run the `pulumi preview` command against our policy pack by specifying the `--policy-pack` flag:

```bash
cd ../infra
pulumi preview --policy-pack ../policy
```

The command should fail. You should see output similar to the following:

```text
Policy Violations:
    [mandatory]  aws-python v0.0.1  s3-no-public-read (aws:s3/bucket:Bucket: my-bucket)
    Prohibits setting the publicRead or publicReadWrite permission on AWS S3 buckets.
    You cannot set public-read or public-read-write on an S3 bucket. Read more about ACLs here: https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html
```

A few things to note:

* The `--policy-pack` flag works with both the `pulumi preview` command and the `pulumi up` command.
* We can specify the `--policy-pack` flag multiple times to run any number of policy packs against our Pulumi program.

## Step 5: Exploring policy enforcement levels

Each Pulumi policy in a policy pack may be set to one of the following enforcement levels:

1. **Disabled:** The policy will not be run at all.
1. **Advisory:** Pulumi will print a message stating a resource is not in compliance with the policy, but Pulumi commands will not fail.
1. **Mandatory:** Pulumi will print a message stating a resource is not in compliance with the policy and Pulumi commands will fail.

Let's explore how these levels work. In `policy/__main__.py`, change the following line (toward the end of the file):

```python
enforcement_level=EnforcementLevel.MANDATORY,
```

to this:

```python
enforcement_level=EnforcementLevel.ADVISORY,
```

And let's attempt to deploy our infrastructure:

```bash
pulumi up -y --policy-pack ../policy
```

You'll see that the command succeeds and our infrastructure deploys, but we still receive a warning that our infrastructure is not in compliance:

```text
Policy Violations:
    [advisory]  aws-python v0.0.1  s3-no-public-read (aws:s3/bucket:Bucket: my-bucket)
    Prohibits setting the publicRead or publicReadWrite permission on AWS S3 buckets.
    You cannot set public-read or public-read-write on an S3 bucket. Read more about ACLs here: https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html
```

Now let's disable the rule altogether. Change this:

```python
enforcement_level=EnforcementLevel.ADVISORY,
```

To this:

```python
enforcement_level=EnforcementLevel.DISABLED,
```

And then attempt to deploy our infrastructure:

```bash
pulumi up -y --policy-pack ../policy
```

The command will succeed (with no updates since we haven't changed any of our infrastructure code - just our policy pack), but Pulumi will still indicate that our policy pack was indeed run:

```text
Policy Packs run:
    Name                    Version
    aws-python (../policy)  (local)
```

## Step 6: Policies run against existent resources

Let's demonstrate how Pulumi policy packs run against resources that are already deployed. In the previous step, we were able to deploy our non-compliant infrastructure because we set our policy enforcement level to `ADVISORY`. Now let's change it back to `MANDATORY`.

Change the following line in `policy/__main__.py` from this:

```python
enforcement_level=EnforcementLevel.DISABLED,
```

to this:

```python
enforcement_level=EnforcementLevel.MANDATORY,
```

And let's attempt to run the following command:

```bash
pulumi up -y --policy-pack ../policy
```

You'll see that even though there are no changes in our infrastructure, the command still fails:

```text
Diagnostics:
  pulumi:pulumi:Stack (workshop-policy-as-code-dev):
    error: preview failed

Policy Violations:
    [mandatory]  aws-python v0.0.1  s3-no-public-read (aws:s3/bucket:Bucket: my-bucket)
    Prohibits setting the publicRead or publicReadWrite permission on AWS S3 buckets.
    You cannot set public-read or public-read-write on an S3 bucket. Read more about ACLs here: https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html
```

Pulumi policy packs are designed to check all resources in a stack, not just those that have changed. In this way, we can catch non-compliant resources that have already been deployed once a policy becomes mandatory. In a production scenario, we might consider rolling out policies first as `ADVISORY` to allow teams some time to comply with the policies, then change them to `MANDATORY` after teams have had sufficient time to become compliant.
