+++
title = "Creating an AWS EKS Cluster"
chapter = false
weight = 10
+++

Start by adding the Pulumi EKS import to your `index.ts` file:

```typescript
import * as eks from "@pulumi/eks";
```

Then add the following to your `index.ts` to create the EKS cluster using the
default VPC and default node group.

See for more details and examples:
- [API Docs](https://www.pulumi.com/docs/reference/pkg/nodejs/pulumi/eks/index.html)
- [Examples](https://www.pulumi.com/docs/reference/pkg/nodejs/pulumi/eks/index.html)

```typescript
const cluster = new eks.Cluster("eks", {
    deployDashboard: false,
});
```

We'll need to the cluster's kubeconfig to interact with the cluster.

We can retrieve it from the cluster by adding the following to your `index.ts`:

```typescript
export const kubeconfig = cluster.kubeconfig;
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as eks from "@pulumi/eks"

const cluster = new eks.Cluster("eks", {
    deployDashboard: false,
});

export const kubeconfig = cluster.kubeconfig;
```

This will create an AWS EKS cluster situated in the default VPC for the region and a default
node group that has two t2.medium instances and export the kubeconfig for us to use to interact
with the cluster

To provision the EKS Cluster, run:

```bash
pulumi up
```

After confirming, you will see output like the following:

```
Updating (dev):
     Type                                   Name                                   Status
 +   pulumi:pulumi:Stack                    eks-workshop-dev                       created
 +   └─ eks:index:Cluster                   eks                                    created
 +      ├─ eks:index:ServiceRole            eks-instanceRole                       created
 +      │  ├─ aws:iam:Role                  eks-instanceRole-role                  created
 +      │  ├─ aws:iam:RolePolicyAttachment  eks-instanceRole-03516f97              created
 +      │  ├─ aws:iam:RolePolicyAttachment  eks-instanceRole-3eb088f2              created
 +      │  └─ aws:iam:RolePolicyAttachment  eks-instanceRole-e1b295bd              created
 +      ├─ eks:index:ServiceRole            eks-eksRole                            created
 +      │  ├─ aws:iam:Role                  eks-eksRole-role                       created
 +      │  ├─ aws:iam:RolePolicyAttachment  eks-eksRole-90eb1c99                   created
 +      │  └─ aws:iam:RolePolicyAttachment  eks-eksRole-4b490823                   created
 +      ├─ pulumi-nodejs:dynamic:Resource   eks-cfnStackName                       created
 +      ├─ aws:ec2:SecurityGroup            eks-eksClusterSecurityGroup            created
 +      ├─ aws:iam:InstanceProfile          eks-instanceProfile                    created
 +      ├─ aws:eks:Cluster                  eks-eksCluster                         created
 +      ├─ aws:ec2:SecurityGroupRule        eks-eksClusterInternetEgressRule       created
 +      ├─ aws:ec2:SecurityGroup            eks-nodeSecurityGroup                  created
 +      ├─ aws:ec2:SecurityGroupRule        eks-eksNodeClusterIngressRule          created
 +      ├─ aws:ec2:SecurityGroupRule        eks-eksNodeIngressRule                 created
 +      ├─ aws:ec2:SecurityGroupRule        eks-eksClusterIngressRule              created
 +      ├─ aws:ec2:SecurityGroupRule        eks-eksExtApiServerClusterIngressRule  created
 +      ├─ aws:ec2:SecurityGroupRule        eks-eksNodeInternetEgressRule          created
 +      ├─ aws:ec2:LaunchConfiguration      eks-nodeLaunchConfiguration            created
 +      ├─ pulumi:providers:kubernetes      eks-eks-k8s                            created
 +      ├─ pulumi-nodejs:dynamic:Resource   eks-vpc-cni                            created
 +      ├─ kubernetes:core:ConfigMap        eks-nodeAccess                         created
 +      ├─ aws:cloudformation:Stack         eks-nodes                              created
 +      └─ pulumi:providers:kubernetes      eks-provider                           created

Outputs:
    kubeconfig: "{\"apiVersion\": \"v1\", \"clusters\": ...}" 

Resources:
    + 28 created

Duration: 14m40s

Permalink: https://app.pulumi.com/workshops/eks-workshop/dev/updates/1
```
