+++
title = "2.3 Deploy Ingress Controller"
chapter = false
weight = 40
+++

We have an IAM role defined, we can deploy the AWS Load Balancer controller using the Helm Chart

## Step 1 &mdash; Create a Provider

Before we create resources in our Kubernetes cluster, we need to provide a valid Kubernetes endpoint for Pulumi to talk to.

Usually, we would use `KUBECONFIG` file for interacting with Kubernetes, but we can also explicitly set a provider on each Kubernetes resource.

Let's do this now. Define a `provider` resource in your `index.ts`. It gets populated from the `kubeconfig` output from the cluster.

```python
kubeconfig = cluster.get_output("kubeconfig")

provider = k8s.Provider("provider", kubeconfig=kubeconfig)
```

This provider can then be passed to all the resources we're going to create with Pulumi.

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

kubeconfig = cluster.get_output("kubeconfig")
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

provider = k8s.Provider("provider", kubeconfig=kubeconfig)
```

## Step 2 &mdash; Create a Namespace

Before we deploy the helm chart, we need to ensure we have a namespace in our Kubernetes cluster for it to be deployed to.

Earlier we defined a constant called `ns` to ensure we got a consistent namespace name. Let's use that to create our namespace:

Add the following to your `__main__.py` file:

```python
namespace = k8s.core.v1.Namespace(
    f"{ns}-ns",
    metadata={
        "name": ns,
        "labels": {
            "app.kubernetes.io/name": "aws-load-balancer-controller",
        }
    },
    opts=pulumi.ResourceOptions(
        provider=provider,
        parent=provider,
    )
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

kubeconfig = cluster.get_output("kubeconfig")
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

provider = k8s.Provider("provider", kubeconfig=kubeconfig)

namespace = k8s.core.v1.Namespace(
    f"{ns}-ns",
    metadata={
        "name": ns,
        "labels": {
            "app.kubernetes.io/name": "aws-load-balancer-controller",
        }
    },
    opts=pulumi.ResourceOptions(
        provider=provider,
        parent=provider,
    )
)

```

## Step 2 &mdash; Retrieve needed outputs

The next thing we need to do is retrieve some values from our cluster stack so that we can pass them to our helm chart. The AWS Load Balancer Controller first needs to know which cluster to target when it starts.

We can retrieve the cluster name from the other stack - add the following to your `__main__.py` file near the top, where you retrieved the other outputs:

```python
cluster_name = cluster.get_output("clusterName")
vpc_id = cluster.get_output("vpcId")
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

kubeconfig = cluster.get_output("kubeconfig")
oidc_arn = cluster.get_output("clusterOidcProviderArn")
oidc_url = cluster.get_output("clusterOidcProvider")
cluster_name = cluster.get_output("clusterName")
vpc_id = cluster.get_output("vpcId")

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

provider = k8s.Provider("provider", kubeconfig=kubeconfig)

namespace = k8s.core.v1.Namespace(
    f"{ns}-ns",
    metadata={
        "name": ns,
        "labels": {
            "app.kubernetes.io/name": "aws-load-balancer-controller",
        }
    },
    opts=pulumi.ResourceOptions(
        provider=provider,
        parent=provider,
    )
)
```

## Step 3 &mdash; Define your Helm Chart

Now that we've got all our dependencies from other stacks, we can deploy our Helm Chart. We'll pass the retrieved values from the cluster stack to it.

In your `__main__.py` file, add the following:

```python
k8s.helm.v3.Chart(
    "lb", k8s.helm.v3.ChartOpts(
        chart="aws-load-balancer-controller",
        fetch_opts=k8s.helm.v3.FetchOpts(
            repo="https://aws.github.io/eks-charts"
        ),
        namespace=namespace.metadata["name"],
        values={
            "region": "us-west-1",
            "serviceAccount": {
                "name": "aws-lb-controller-serviceaccount"
            },
            "vpcId": vpc_id,
            "clusterName": cluster_name,
            "podLabels": {
                "stack": stack,
                "app": "aws-lb-controller"
            }
        }
    ), pulumi.ResourceOptions(
        provider=provider, parent=namespace
    )
)
``` 

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

kubeconfig = cluster.get_output("kubeconfig")
oidc_arn = cluster.get_output("clusterOidcProviderArn")
oidc_url = cluster.get_output("clusterOidcProvider")
cluster_name = cluster.get_output("clusterName")
vpc_id = cluster.get_output("vpcId")

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

provider = k8s.Provider("provider", kubeconfig=kubeconfig)

namespace = k8s.core.v1.Namespace(
    f"{ns}-ns",
    metadata={
        "name": ns,
        "labels": {
            "app.kubernetes.io/name": "aws-load-balancer-controller",
        }
    },
    opts=pulumi.ResourceOptions(
        provider=provider,
        parent=provider,
    )
)

k8s.helm.v3.Chart(
    "lb", k8s.helm.v3.ChartOpts(
        chart="aws-load-balancer-controller",
        fetch_opts=k8s.helm.v3.FetchOpts(
            repo="https://aws.github.io/eks-charts"
        ),
        namespace=namespace.metadata["name"],
        values={
            "region": "us-west-1",
            "serviceAccount": {
                "name": "aws-lb-controller-serviceaccount"
            },
            "vpcId": vpc_id,
            "clusterName": cluster_name,
            "podLabels": {
                "stack": stack,
                "app": "aws-lb-controller"
            }
        }
    ), pulumi.ResourceOptions(
        provider=provider, parent=namespace
    )
)
```

## Step 4 &mdash; Patch the Helm Chart

Pulumi's Kubernetes provider can manipulate values in Helm charts using transformations.

The Helm chart we're installing has a field which causes issues when installing, so we're going to remove it from the chart.

Add a function to your `__main__.py` which will remove the broken fields:

```python
def remove_status(obj, opts):
    if obj["kind"] == "CustomResourceDefinition":
        del obj["status"]
```

Then update your helm chart to include the transformation:

```python
k8s.helm.v3.Chart(
    "lb", k8s.helm.v3.ChartOpts(
        chart="aws-load-balancer-controller",
        fetch_opts=k8s.helm.v3.FetchOpts(
            repo="https://aws.github.io/eks-charts"
        ),
        namespace=namespace.metadata["name"],
        values={
            "region": "us-west-1",
            "serviceAccount": {
                "name": "aws-lb-controller-serviceaccount"
            },
            "vpcId": vpc_id,
            "clusterName": cluster_name,
            "podLabels": {
                "stack": stack,
                "app": "aws-lb-controller"
            }
        },
        transformations=[remove_status]
    ), pulumi.ResourceOptions(
        provider=provider, parent=namespace
    )
)
```

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

kubeconfig = cluster.get_output("kubeconfig")
oidc_arn = cluster.get_output("clusterOidcProviderArn")
oidc_url = cluster.get_output("clusterOidcProvider")
cluster_name = cluster.get_output("clusterName")
vpc_id = cluster.get_output("vpcId")

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

provider = k8s.Provider("provider", kubeconfig=kubeconfig)

namespace = k8s.core.v1.Namespace(
    f"{ns}-ns",
    metadata={
        "name": ns,
        "labels": {
            "app.kubernetes.io/name": "aws-load-balancer-controller",
        }
    },
    opts=pulumi.ResourceOptions(
        provider=provider,
        parent=provider,
    )
)

def remove_status(obj, opts):
    if obj["kind"] == "CustomResourceDefinition":
        del obj["status"]

k8s.helm.v3.Chart(
    "lb", k8s.helm.v3.ChartOpts(
        chart="aws-load-balancer-controller",
        fetch_opts=k8s.helm.v3.FetchOpts(
            repo="https://aws.github.io/eks-charts"
        ),
        namespace=namespace.metadata["name"],
        values={
            "region": "us-west-2",
            "serviceAccount": {
                "name": "aws-lb-controller-serviceaccount"
            },
            "vpcId": vpc_id,
            "clusterName": cluster_name,
            "podLabels": {
                "stack": stack,
                "app": "aws-lb-controller"
            }
        },
        transformations=[remove_status]
    ), pulumi.ResourceOptions(
        provider=provider, parent=namespace
    )
)
```

## Step 5 &mdash; Provision your Infrastructure

We're now ready to deploy our loadbalancer. Run `pulumi up`:

```
pulumi up
```

and validate the changes:

```
pulumi up
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/aws-load-balancer-controller/dev/previews/b4c0a60e-938a-4e06-8838-e974f9ea46b3

     Type                                                                                  Name                                                                                 Plan
 +   pulumi:pulumi:Stack                                                                   aws-load-balancer-controller-dev                                                     create
 +   ├─ kubernetes:helm.sh/v3:Chart                                                        lb-chart                                                                             create
 +   │  ├─ kubernetes:rbac.authorization.k8s.io/v1:ClusterRoleBinding                      lb-chart-aws-load-balancer-controller-rolebinding                                    create
 +   │  ├─ kubernetes:core/v1:Service                                                      aws-lb-controller/aws-load-balancer-webhook-service                                  create
 +   │  ├─ kubernetes:admissionregistration.k8s.io/v1beta1:ValidatingWebhookConfiguration  aws-load-balancer-webhook                                                            create
 +   │  ├─ kubernetes:core/v1:ServiceAccount                                               aws-lb-controller/aws-lb-controller-serviceaccount                                   create
 +   │  ├─ kubernetes:core/v1:Secret                                                       aws-lb-controller/aws-load-balancer-tls                                              create
 +   │  ├─ kubernetes:rbac.authorization.k8s.io/v1:RoleBinding                             aws-lb-controller/lb-chart-aws-load-balancer-controller-leader-election-rolebinding  create
 +   │  ├─ kubernetes:rbac.authorization.k8s.io/v1:Role                                    aws-lb-controller/lb-chart-aws-load-balancer-controller-leader-election-role         create
 +   │  ├─ kubernetes:rbac.authorization.k8s.io/v1:ClusterRole                             lb-chart-aws-load-balancer-controller-role                                           create
 +   │  ├─ kubernetes:admissionregistration.k8s.io/v1beta1:MutatingWebhookConfiguration    aws-load-balancer-webhook                                                            create
 +   │  ├─ kubernetes:apps/v1:Deployment                                                   aws-lb-controller/lb-chart-aws-load-balancer-controller                              create
 +   │  └─ kubernetes:apiextensions.k8s.io/v1beta1:CustomResourceDefinition                targetgroupbindings.elbv2.k8s.aws                                                    create
 +   ├─ pulumi:providers:kubernetes                                                        provider                                                                             create
 +   ├─ aws:iam:Role                                                                       aws-loadbalancer-controller-role                                                     create
 +   │  ├─ aws:iam:Policy                                                                  aws-loadbalancer-controller-policy                                                   create
 +   │  └─ aws:iam:PolicyAttachment                                                        aws-loadbalancer-controller-policy-attachment                                        create
 +   └─ kubernetes:core/v1:Namespace                                                       aws-lb-controller-ns                                                                 create

Resources:
    + 18 to create

Do you want to perform this update?  [Use arrows to move, enter to select, type to filter]
  yes
> no
  details
```

## Step 6 &mdash; Validate the changes

We can now verify our AWS Load Balancer Controller started correctly. Let's check the logs of the pod.

We'll use `kubectl` to gather the logs and target the running pod using the labels we specified.

Run the following command to retrieve the pod logs:

```
kubectl logs -n aws-lb-controller -l app=aws-lb-controller -l stack=dev
```

You should see some output from the running pod, like so:

```
{"level":"info","ts":1607475108.9502738,"logger":"controller-runtime.certwatcher","msg":"Starting certificate watcher"}
{"level":"info","ts":1607475108.9502833,"logger":"controller","msg":"Starting EventSource","controller":"ingress","source":"kind source: /, Kind="}
{"level":"info","ts":1607475108.9503238,"logger":"controller","msg":"Starting EventSource","controller":"service","source":"kind source: /, Kind="}
{"level":"info","ts":1607475108.9503775,"logger":"controller","msg":"Starting Controller","controller":"service"}
{"level":"info","ts":1607475109.0503407,"logger":"controller","msg":"Starting EventSource","reconcilerGroup":"elbv2.k8s.aws","reconcilerKind":"TargetGroupBinding","controller":"targetGroupBinding","source":"kind source: /, Kind="}
{"level":"info","ts":1607475109.0504348,"logger":"controller","msg":"Starting workers","controller":"service","worker count":3}
{"level":"info","ts":1607475109.0504942,"logger":"controller","msg":"Starting Controller","controller":"ingress"}
{"level":"info","ts":1607475109.150941,"logger":"controller","msg":"Starting Controller","reconcilerGroup":"elbv2.k8s.aws","reconcilerKind":"TargetGroupBinding","controller":"targetGroupBinding"}
{"level":"info","ts":1607475109.1510022,"logger":"controller","msg":"Starting workers","reconcilerGroup":"elbv2.k8s.aws","reconcilerKind":"TargetGroupBinding","controller":"targetGroupBinding","worker count":3}
{"level":"info","ts":1607475109.1510797,"logger":"controller","msg":"Starting workers","controller":"ingress","worker count":3}
```

We're now ready to deploy an application to our cluster.


