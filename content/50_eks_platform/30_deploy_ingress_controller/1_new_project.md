+++
title = "2.1 Creating a New Project"
chapter = false
weight = 10
+++


Infrastructure in Pulumi is organized into projects. Each project is a single program that, when run, declares the desired infrastructure for Pulumi to manage.

## Step 1 &mdash; Create a Directory

Each Pulumi project lives in its own directory. Create one now and change into it:

```bash
mkdir aws-load-balancer-controller
cd aws-load-balancer-controller
```

> Pulumi will use the directory name as your project name by default. To create an independent project, simply name the directory differently.

## Step 2 &mdash; Initialize Your Project

A Pulumi project is just a directory with some files in it. It's possible for you to create a new one by hand. The `pulumi new` command, however, automates the process:

```bash
pulumi new aws-python -y
```

This will print output similar to the following with a bit more information and status as it goes:

```
Created project 'aws-load-balancer-controller'

Please enter your desired stack name.
To create a stack in an organization, use the format <org-name>/<stack-name> (e.g. `acmecorp/dev`).
Created stack 'dev'

Saved config

Creating virtual environment...
Finished creating virtual environment
Updating pip, setuptools, and wheel in virtual environment...
Collecting pip
```

This command has created all the files we need, initialized a new stack named `dev` (an instance of our project), and installed the needed package dependencies from NPM.

## Step 3 &mdash; Inspect Your New Project

Our project is comprised of multiple files:

* **`__main__.py`**: your program's main entrypoint file
* **`requirements.txt`**: your project's Python dependency information
* **`Pulumi.yaml`**: your project's metadata, containing its name and language
* **`venv`**: a [virtualenv](https://pypi.org/project/virtualenv/) for your project

Run `cat __main__.py` to see the contents of your project's empty program:

```python
"""An AWS Python Pulumi program"""

import pulumi
from pulumi_aws import s3

# Create an AWS resource (S3 Bucket)
bucket = s3.Bucket('my-bucket')

# Export the name of the bucket
pulumi.export('bucket_name', bucket.id)
```

Feel free to explore the other files, although we won't be editing any of them by hand.

## Step 3 &mdash; Add the Kubernetes Provider

We'll be provisioning Kubernetes resources alongside AWS resources in this Pulumi project, so we'll add the Kubernetes provider to our program.

First, we need to install it. We'll activate our virtual env, 

```
source venv/bin/activate
```

and the install the provider inside that:

```
pip3 install pulumi-kubernetes
```

Next, add the provider to our Pulumi program. Clear our any boilerplate configuration in your Pulumi program and add the following:

```python
import pulumi_kubernetes as k8s
import pulumi_aws as aws
```

{{% notice info %}}
The `__main__.py` file should now have the following contents:
{{% /notice %}}
```typescript
import pulumi
import pulumi_kubernetes as k8s
import pulumi_aws as aws
```

Now we're ready to provision the dependent AWS resources we need for the LoadBalancer
