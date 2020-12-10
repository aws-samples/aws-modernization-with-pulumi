+++
title = "2.2 Create an IAM Role"
chapter = false
weight = 10
+++

We'll be deploying the AWS Load Balancer Controller into our EKS Cluster, but it will need to make API calls to AWS for some operations.

We can pass an IAM role to our Kubernetes deployment using [IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) which we configured on our cluster

Let's define the IAM role we need to pass to our Kubernetes resources

## Step 1 &mdash; Retrieve Stack References

Before we define any resources, we need to retrieve some outputs from our cluster stack.

Add a StackReference call to your `__main__.py`:

```python
stack = pulumi.get_stack()
cluster = pulumi.StackReference(f"jaxxstorm/workshop-cluster/{stack}")
```
{{% notice info %}}
The `__main__.py` file should now have the following contents:
{{% /notice %}}
```python
import pulumi
import pulumi_aws as aws
import pulumi_kubernetes as k8s

stack = pulumi.get_stack()
cluster = pulumi.StackReference(f"jaxxstorm/workshop-cluster/{stack}")

oidc_arn = cluster.get_output('clusterOidcProviderArn')
oidc_url = cluster.get_output('clusterOidcProvider')
```

## Step 2 &mdash; Define your IAM Role

Now we've retrieved these important values from our cluster stack, we can start to build our AWS resources. Define your IAM role and use your retrieved values inside your IAM policy:

```python
import json

ns = "aws-lb-controller"
service_account_name = f"system:serviceaccount:{ns}:aws-lb-controller-serviceaccount"

iam_role = aws.iam.Role(
    "aws-loadbalancer-controller-role",
    assume_role_policy=pulumi.Output.all(oidc_arn, oidc_url).apply(
        lambda args: json.dumps(
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Federated": args[0],
                        },
                        "Action": "sts:AssumeRoleWithWebIdentity",
                        "Condition": {
                            "StringEquals": {f"{args[1]}:sub": service_account_name},
                        },
                    }
                ],
            }
        )
    ),
)

```

We use two important pulumi methods here,  an `all` and an `apply`. This allows us to concatenate a standard `string` with a `pulumi.Output<string>`.

We've also defined a namespace variable, so we can ensure we don't make mistakes later.

{{% notice info %}}
The `__main__.py` file should now have the following contents:
{{% /notice %}}
```python
import pulumi
import pulumi_aws as aws
import json
import pulumi_kubernetes as k8s

stack = pulumi.get_stack()
cluster = pulumi.StackReference(f"jaxxstorm/workshop-cluster/{stack}")

oidc_arn = cluster.get_output('clusterOidcProviderArn')
oidc_url = cluster.get_output('clusterOidcProvider')

ns = "aws-lb-controller"
service_account_name = f"system:serviceaccount:{ns}:aws-lb-controller-serviceaccount"

iam_role = aws.iam.Role(
    "aws-loadbalancer-controller-role",
    assume_role_policy=pulumi.Output.all(oidc_arn, oidc_url).apply(
        lambda args: json.dumps(
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Federated": args[0],
                        },
                        "Action": "sts:AssumeRoleWithWebIdentity",
                        "Condition": {
                            "StringEquals": {f"{args[1]}:sub": service_account_name},
                        },
                    }
                ],
            }
        )
    ),
)
```

## Step 3 &mdash; Define your IAM Policy

Now we have an IAM role, we need to add the policy to this role that allows the AWS Load Balancer Controller to call the AWS APIs it needs to do its job.

This policy is very long, so in order to save some lines of code, we'll download the policy and read it from a file.

Create a directory to store the policy file, and then download the policy from the AWS Load Balancer controller GitHub:

```
mkdir files
curl -L https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json -o files/iam_policy.json
```

Then, add the IAM policy resource

```python
with open("files/iam_policy.json") as policy_file:
    policy_doc = policy_file.read()

iam_policy = aws.iam.Policy(
    "aws-loadbalancer-controller-policy",
    policy=policy_doc,
    opts=pulumi.ResourceOptions(parent=iam_role),
)
```

{{% notice info %}}
The `__main__.py` file should now have the following contents:
{{% /notice %}}
```python
"""An AWS Python Pulumi program"""

import pulumi
import pulumi_aws as aws
import json
import pulumi_kubernetes as k8s

stack = pulumi.get_stack()
cluster = pulumi.StackReference(f"jaxxstorm/workshop-cluster/{stack}")

oidc_arn = cluster.get_output("clusterOidcProviderArn")
oidc_url = cluster.get_output("clusterOidcProvider")

ns = "aws-lb-controller"
service_account_name = f"system:serviceaccount:{ns}:aws-lb-controller-serviceaccount"

iam_role = aws.iam.Role(
    "aws-loadbalancer-controller-role",
    assume_role_policy=pulumi.Output.all(oidc_arn, oidc_url).apply(
        lambda args: json.dumps(
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Federated": args[0],
                        },
                        "Action": "sts:AssumeRoleWithWebIdentity",
                        "Condition": {
                            "StringEquals": {f"{args[1]}:sub": service_account_name},
                        },
                    }
                ],
            }
        )
    ),
)

with open("files/iam_policy.json") as policy_file:
    policy_doc = policy_file.read()

iam_policy = aws.iam.Policy(
    "aws-loadbalancer-controller-policy",
    policy=policy_doc,
    opts=pulumi.ResourceOptions(parent=iam_role),
)

```

## Step 4 &mdash; Attach the IAM Policy to the Role

Now we have an IAM role for our Kubernetes pod to use, and a policy, but they don't yet know about each other.

To rectify this, we can use `IamPolicyAttachment` resource.

Add the following to your Pulumi program to link the two resources together

```python
aws.iam.PolicyAttachment(
    "aws-loadbalancer-controller-attachment",
    policy_arn=iam_policy.arn,
    roles=[iam_role.name],
    opts=pulumi.ResourceOptions(parent=iam_role),
)
```

{{% notice info %}}
The `__main__.py` file should now have the following contents:
{{% /notice %}}
```python
"""An AWS Python Pulumi program"""

import pulumi
import pulumi_aws as aws
import json
import pulumi_kubernetes as k8s

stack = pulumi.get_stack()
cluster = pulumi.StackReference(f"jaxxstorm/workshop-cluster/{stack}")

oidc_arn = cluster.get_output("clusterOidcProviderArn")
oidc_url = cluster.get_output("clusterOidcProvider")

ns = "aws-lb-controller"
service_account_name = f"system:serviceaccount:{ns}:aws-lb-controller-serviceaccount"

iam_role = aws.iam.Role(
    "aws-loadbalancer-controller-role",
    assume_role_policy=pulumi.Output.all(oidc_arn, oidc_url).apply(
        lambda args: json.dumps(
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Federated": args[0],
                        },
                        "Action": "sts:AssumeRoleWithWebIdentity",
                        "Condition": {
                            "StringEquals": {f"{args[1]}:sub": service_account_name},
                        },
                    }
                ],
            }
        )
    ),
)

with open("files/iam_policy.json") as policy_file:
    policy_doc = policy_file.read()

iam_policy = aws.iam.Policy(
    "aws-loadbalancer-controller-policy",
    policy=policy_doc,
    opts=pulumi.ResourceOptions(parent=iam_role),
)

aws.iam.PolicyAttachment(
    "aws-loadbalancer-controller-attachment",
    policy_arn=iam_policy.arn,
    roles=[iam_role.name],
    opts=pulumi.ResourceOptions(parent=iam_role),
)

```

## Step 5 &mdash; Provision your Infrastructure

Now that we have our role fleshed out, we can use Pulumi to provision it. 

Let's preview your changes:

```
pulumi up
```

This command evaluates your program, determines the resource updates to make, and shows you an outline of these changes:

```
pulumi up
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/aws-load-balancer-controller/dev/previews/a4fe98ec-cc73-43d8-af04-9e2c0ac2aeb9

     Type                            Name                                           Plan
 +   pulumi:pulumi:Stack             aws-load-balancer-controller-dev               create
 +   └─ aws:iam:Role                 aws-loadbalancer-controller-role               create
 +      ├─ aws:iam:Policy            aws-loadbalancer-controller-policy             create
 +      └─ aws:iam:PolicyAttachment  aws-loadbalancer-controller-policy-attachment  create

Resources:
    + 4 to create

Do you want to perform this update?  [Use arrows to move, enter to select, type to filter]
  yes
> no
  details
```

Hit `yes` to create your resources



