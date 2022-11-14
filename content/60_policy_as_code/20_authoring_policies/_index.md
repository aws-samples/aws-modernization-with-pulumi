+++
title = "Module 2: Authoring Pulumi Policies"
chapter = true
weight = 10
+++

## Intro to Policy as Code in Pulumi

{{% notice warning %}}<p> You are responsible for the cost of the AWS services used while running this workshop in your AWS account.</p> {{% /notice %}}

In this module, you will use Pulumi's policy as code features to author your own policies.

## Step 1: Initialize the project

If you are arriving at this module from Module 1, you can reuse your Pulumi program and policy pack from that module and skip this step. If not, run the following commands to initialize your workspace:

```bash
mkdir workshop-policy-as-code
cd workshop-policy-as-code
mkdir infra policy
```

Then, initialize a Pulumi program in the `infra` directory:

```bash
cd infra
pulumi new aws-python -n workshop-policy-as-code
```

Choose the default for all options with the possible exception of AWS region - pick a region near to you.

Then, initialize a Pulumi policy pack:

```bash
cd ../policy
pulumi policy new aws-python
```

## Step 2: Analyzing the structure of a Pulumi policy pack

Let's look at `policy/__main__.py` and break it down by sections so we get an understanding of how the code in Pulumi policy packs work.

First, we import some required classes from `pulumi-policy`, the Python SDK for authoring policies:

```python
from pulumi_policy import (
    EnforcementLevel,
    PolicyPack,
    ReportViolation,
    ResourceValidationArgs,
    ResourceValidationPolicy,
)
```

Next we define our validator function:

```python
def s3_no_public_read_validator(args: ResourceValidationArgs, report_violation: ReportViolation):
    if args.resource_type == "aws:s3/bucket:Bucket" and "acl" in args.props:
        acl = args.props["acl"]
        if acl == "public-read" or acl == "public-read-write":
            report_violation(
                "You cannot set public-read or public-read-write on an S3 bucket. " +
                "Read more about ACLs here: https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html")
```

Validator functions for resources (there are also stack policies whose validators have a different signature and will be covered later in this module) take 2 arguments: `ResourceValidationArgs` which contains information about the resource we're checking for compliance, and `ReportViolation`, a function we will invoke with a message if the resource is not in compliance.

Next, we plug our validator function into the constructor for a `ResourceValidationPolicy`, also giving it a name and description. We can optionally add a default `enforcement_level` (whether the rule is disabled, advisory, or mandatory by default). This setting takes precedence over the default enforcement level for the entire policy pack. We can also an optional `config_schema` to allow our policy to be configurable. (We will cover configurable policies later in this module.):

```python
s3_no_public_read = ResourceValidationPolicy(
    name="s3-no-public-read",
    description="Prohibits setting the publicRead or publicReadWrite permission on AWS S3 buckets.",
    validate=s3_no_public_read_validator,
)
```

Finally, we construct our `PolicyPack` which includes a name (which displays in the output of Pulumi commands when checking against our policies), the default enforcement level for all policies in the policy pack, and the policies to include in the pack:

```python
PolicyPack(
    name="aws-python",
    enforcement_level=EnforcementLevel.MANDATORY,
    policies=[
        s3_no_public_read,
    ],
)
```

## Step 3: Authoring a resource policy

Let's add a new resource policy to our policy pack. This rule will ensure the [`force_destroy`](https://www.pulumi.com/registry/packages/aws/api-docs/s3/bucket/#force_destroy_python) option is not set for our S3 bucket. The `force_destroy` option allows us to delete a bucket even if it has objects in it, so this is a wise policy to enforce for production S3 buckets.

First, let's add the `force_destroy` attribute to our S3 bucket:

```python
bucket = aws.s3.Bucket(
    'my-bucket',
    aws.s3.BucketArgs(
        acl="public-read-write", # <-- ensure this line is present
        force_destroy=True, # <-- add this line
    ),
)
```

Then, we'll define our validator function. Add the following to `policy/__main__.py`:

```python
def s3_no_force_destroy_validator(args: ResourceValidationArgs, report_violation: ReportViolation):
    if args.resource_type != "aws:s3/bucket:Bucket":
        return

    if "forceDestroy" not in args.props:
        return

    if args.props["forceDestroy"] == True:
        report_violation("Force destroy must be disabled for S3 buckets.")
```

{{% notice note %}}
You might be wondering why we have to specify the attribute name as `forceDestroy` in our policy validator, rather than `force_destroy`, which is the name of the attribute in our Python Pulumi program where we define the bucket. Pulumi is a multi-language tool - the same provider (in this case, AWS Classic) can be consumed by Pulumi programs in multiple languages.

When writing policies, we must reference language-neutral Pulumi identifiers which are a part of the provider's schema, the magic behind the scenes which allows us to use the same provider for Pulumi programs written in different languages. Having to use these schema identifiers can be counter-intuitive at first, but by using the identifiers from the Pulumi schema we are able to use a policy written in any language (at the time of writing, Policy packs can be written in TypeScript or Python) against *any* Pulumi program, no matter what language we use to define our infrastructure!

For more details on Pulumi schemas, see [the Pulumi docs](https://www.pulumi.com/docs/guides/pulumi-packages/schema/).
{{% /notice %}}

Next, we'll create the `ResourceValidationPolicy`:

```python
s3_no_force_destroy = ResourceValidationPolicy(
    name="s3-no-force-destroy",
    description="Prohibits setting force delete option in S3 buckets.",
    validate=s3_no_force_destroy_validator,
)
```

And finally, we'll add our policy to the policy pack. Add the indicated line to the `PolicyPack` constructor:

```python
PolicyPack(
    name="aws-python",
    enforcement_level=EnforcementLevel.MANDATORY,
    policies=[
        s3_no_public_read,
        s3_no_force_destroy, # <-- add this line
    ],
)
```

Now let's run our policy pack:

```bash
pulumi preview -y --policy-pack ../policy
```

The command should fail with the following output:

```text
Policy Violations:
    [mandatory]  aws-python v0.0.1  s3-no-force-delete (aws:s3/bucket:Bucket: my-bucket)
    Prohibits setting force delete option in S3 buckets.
    Force destroy must be disabled for S3 buckets.
    
    [mandatory]  aws-python v0.0.1  s3-no-public-read (aws:s3/bucket:Bucket: my-bucket)
    Prohibits setting the publicRead or publicReadWrite permission on AWS S3 buckets.
    You cannot set public-read or public-read-write on an S3 bucket. Read more about ACLs here: https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html
```

## Step 4: Authoring stack policies


