+++
title = "1.2 Configuring AWS"
chapter = false
weight = 10
+++

Now that you have a basic project, let's configure AWS support for it.

## Step 1 &mdash; Install the AWS Package

Pulumi created a `virtualenv` for you when you created your `iac-workshop` project. We'll need to activate it to install dependencies:

```bash
source venv/bin/activate

```

Run the following command to install the AWS package:

```bash
pip install pulumi_aws
```

The package will be added to `requirements.txt`.

## Step 2 &mdash; Import the AWS Package

Now that the AWS package is installed, we need to import it as part of our project:

```python
import pulumi_aws as aws
```

{{% notice info %}}
The `__main__.py` file should now have the following contents:
{{% /notice %}}
```python
import pulumi
import pulumi_aws as aws
```

## Step 3 &mdash; Configure an AWS Region

Configure the AWS region you would like to deploy to:

```bash
pulumi config set aws:region us-west-2
```

Feel free to choose any AWS region that supports the services used in these labs ([see this table](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-available-regions) for a list of available regions).

## (Optional) Step 4 &mdash; Configure an AWS Profile

If you're using an alternative AWS profile, you can tell Pulumi which to use in one of two ways:

* Using an environment variable: `export AWS_PROFILE=<profile name>`
* Using configuration: `pulumi config set aws:profile <profile name>`
