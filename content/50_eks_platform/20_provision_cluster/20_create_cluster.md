+++
title = "1.3 Create an EKS Cluster"
chapter = false
weight = 20
+++

Now that you have a project configured to use AWS, you'll need an AWS VPC configured for EKS. Let's create that using [Pulumi Crosswalk for AWS](https://www.pulumi.com/docs/guides/crosswalk/aws/).

## Step 1 &mdash; Define the VPC

Define your VPC. We're going to create a VPC with public and private subnets like so:

```typescript
const name = 'lbriggs-workshop' // replace this with your name!
const clusName = `${name}-cluster`
const clusterTag = `kubernetes.io/cluster/${clusName}`

// this defines a valid VPC that can be used for EKS
const vpc = new awsx.ec2.Vpc(`vpc-${name}`, {
    cidrBlock: "172.16.0.0/24",
    subnets: [
        {
            type: "private",
            tags: {
                [clusterTag]: "owned",
                "kubernetes.io/role/internal-elb": "1",
            }
        },
        {
            type: "public",
            tags: {
                [clusterTag]: "owned",
                "kubernetes.io/role/elb": "1",
            }
        }],
    tags: {
        Name: `${name}-vpc`,
    }
});
```
{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

const name = `lbriggs-workshop`
const clusName = `${name}-cluster`
const clusterTag = `kubernetes.io/cluster/${clusName}`

// this defines a valid VPC that can be used for EKS
const vpc = new awsx.ec2.Vpc(`vpc-${name}`, {
    cidrBlock: "172.16.0.0/24",
    subnets: [
        {
            type: "private",
            tags: {
                [clusterTag]: "owned",
                "kubernetes.io/role/internal-elb": "1",
            }
        },
        {
            type: "public",
            tags: {
                [clusterTag]: "owned",
                "kubernetes.io/role/elb": "1",
            }
        }],
    tags: {
        Name: `${name}-vpc`,
    }
});
```

There are a couple things to break down here. The first is that we are specifying some [important tags on the subnets inside the VPC](https://aws.amazon.com/premiumsupport/knowledge-center/eks-vpc-subnet-discovery/), which are necessarily for some later components.

The second thing to note is that we've defined some thing as variables, so that we don't make mistakes later. 

## Step 2 &mdash; Define the Amazon EKS Cluster

We'll now add an EKS cluster in this project. In order to do this, we'll use the [Pulumi EKS package](https://www.pulumi.com/docs/guides/crosswalk/aws/eks/). Let's install it first:

```
npm install @pulumi/eks
```

And then we can add an EKS cluster to our project:

```typescript
import * as eks from "@pulumi/eks";

const cluster = new eks.Cluster(name, {
    name: clusName,
    vpcId: vpc.id,
    privateSubnetIds: vpc.privateSubnetIds,
    publicSubnetIds: vpc.publicSubnetIds,
    instanceType: "t2.medium",
    desiredCapacity: 2,
    minSize: 1,
    maxSize: 2,
    createOidcProvider: true,
});
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
import * as eks from "@pulumi/eks";

const name = `lbriggs-workshop`
const clusName = `${name}-cluster`
const clusterTag = `kubernetes.io/cluster/${clusName}`

// this defines a valid VPC that can be used for EKS
const vpc = new awsx.ec2.Vpc(`vpc-${name}`, {
    cidrBlock: "172.16.0.0/24",
    subnets: [
        {
            type: "private",
            tags: {
                [clusterTag]: "owned",
                "kubernetes.io/role/internal-elb": "1",
            }
        },
        {
            type: "public",
            tags: {
                [clusterTag]: "owned",
                "kubernetes.io/role/elb": "1",
            }
        }],
    tags: {
        Name: `${name}-vpc`,
    }
});

const cluster = new eks.Cluster(name, {
    name: clusName,
    vpcId: vpc.id,
    privateSubnetIds: vpc.privateSubnetIds,
    publicSubnetIds: vpc.publicSubnetIds,
    instanceType: "t2.medium",
    desiredCapacity: 2,
    minSize: 1,
    maxSize: 2,
    createOidcProvider: true,
});
```

Again, notice how we're use the variables we've defined to reduce possible mistakes. In addition to this, we're passing outputs from our VPC to our EKS cluster.

## Step 3 &mdash; Preview Your Changes

Now preview your changes:

```
pulumi up
```

This command evaluates your program, determines the resource updates to make, and shows you an outline of these changes:

```

pulumi up
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/workshop-cluster/dev/previews/a23fbd93-81da-49b5-bb5b-22beeb69f487

     Type                                    Name                                           Plan
 +   pulumi:pulumi:Stack                     workshop-cluster-dev                           create
 +   ├─ awsx:x:ec2:Vpc                       vpc-lbriggs-workshop                                create
 +   │  ├─ awsx:x:ec2:Subnet                 vpc-lbriggs-workshop-public-1                       create
 +   │  │  ├─ aws:ec2:RouteTable             vpc-lbriggs-workshop-public-1                       create
 +   │  │  ├─ aws:ec2:Subnet                 vpc-lbriggs-workshop-public-1                       create
 +   │  │  ├─ aws:ec2:Route                  vpc-lbriggs-workshop-public-1-ig                    create
 +   │  │  └─ aws:ec2:RouteTableAssociation  vpc-lbriggs-workshop-public-1                       create
 +   │  ├─ awsx:x:ec2:NatGateway             vpc-lbriggs-workshop-0                              create
 +   │  │  ├─ aws:ec2:Eip                    vpc-lbriggs-workshop-0                              create

.... redacted ...

Resources:
    + 59 to create

Do you want to perform this update?  [Use arrows to move, enter to select, type to filter]
  yes
> no
  details
```

This is a summary view and has been redacted for its length. In less than 50 lines of code, we're defining a best practice VPC and an Amazon EKS cluster.

## Step 4 &mdash; Deploy Your Changes

Now that we've seen the full set of changes, let's deploy them. Select `yes`:

{{% notice info %}}
This creation process will take a little while, please be patient.
{{% /notice %}}

```
Do you want to perform this update? yes
Updating (dev)

View Live: https://app.pulumi.com/jaxxstorm/workshop-cluster/dev/updates/3

     Type                                    Name                                           Status       Info
 +   pulumi:pulumi:Stack                     workshop-cluster-dev                           creating...
 +   ├─ eks:index:Cluster                    lbriggs-workshop                                    creating.
 +   │  ├─ eks:index:ServiceRole             lbriggs-workshop-instanceRole                       created
 +   │  │  ├─ aws:iam:Role                   lbriggs-workshop-instanceRole-role                  created
 +   │  │  ├─ aws:iam:RolePolicyAttachment   lbriggs-workshop-instanceRole-e1b295bd              created
 +   │  │  ├─ aws:iam:RolePolicyAttachment   lbriggs-workshop-instanceRole-3eb088f2              created
 +   ├─ eks:index:Cluster                    lbriggs-workshop                                    creating...
 +   ├─ eks:index:Cluster                    lbriggs-workshop                                    creating.
 +   ├─ eks:index:Cluster                    lbriggs-workshop                                    creating..
......

 +      ├─ awsx:x:ec2:NatGateway             vpc-lbriggs-workshop-1                              created
 +      │  ├─ aws:ec2:Eip                    vpc-lbriggs-workshop-1                              created
 +      │  └─ aws:ec2:NatGateway             vpc-lbriggs-workshop-1                              created
 +      └─ aws:ec2:Vpc                       vpc-lbriggs-workshop                                created

Resources:
    + 59 created

Duration: 14m33s
```

At this stage, you should have a working EKS cluster.

## Step 5 &mdash; Access your Cluster

Now that we have a cluster provisioned, we need to export our `KUBECONFIG` from the cluster so we can interact with it. 

Add the following to your `index.ts` at the end of the file:

```typescript
export const kubeconfig = cluster.kubeconfig
export const clusterName = clusName
export const vpcId = vpc.id
export const clusterOidcProvider = cluster.core.oidcProvider?.url
export const clusterOidcProviderArn = cluster.core.oidcProvider?.arn
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
import * as eks from "@pulumi/eks";

const name = `lbriggs-workshop`
const clusName = `${name}-cluster`
const clusterTag = `kubernetes.io/cluster/${clusName}`

// this defines a valid VPC that can be used for EKS
const vpc = new awsx.ec2.Vpc(`vpc-${name}`, {
    cidrBlock: "172.16.0.0/24",
    subnets: [
        {
            type: "private",
            tags: {
                [clusterTag]: "owned",
                "kubernetes.io/role/internal-elb": "1",
            }
        },
        {
            type: "public",
            tags: {
                [clusterTag]: "owned",
                "kubernetes.io/role/elb": "1",
            }
        }],
    tags: {
        Name: `${name}-vpc`,
    }
});

const cluster = new eks.Cluster(name, {
    name: clusName,
    vpcId: vpc.id,
    privateSubnetIds: vpc.privateSubnetIds,
    publicSubnetIds: vpc.publicSubnetIds,
    instanceType: "t2.medium",
    desiredCapacity: 2,
    minSize: 1,
    maxSize: 2,
    createOidcProvider: true,
});

export const kubeconfig = cluster.kubeconfig
export const clusterName = clusName
export const vpcId = vpc.id
export const clusterOidcProvider = cluster.core.oidcProvider?.url
export const clusterOidcProviderArn = cluster.core.oidcProvider?.arn
```

This creates some outputs, one of which is the `KUBECONFIG` (the other two will be used later).

Re-run your `pulumi up` command and hit yes, which will create some `Outputs` you can use:

```
Do you want to perform this update? yes
Updating (dev)

View Live: https://app.pulumi.com/jaxxstorm/workshop-cluster/dev/updates/4

     Type                   Name                         Status
     pulumi:pulumi:Stack    workshop-cluster-dev
     └─ eks:index:Cluster   lbriggs-workshop
        └─ aws:eks:Cluster  lbriggs-workshop-eksCluster

Outputs:
  + clusterOidcProvider   : "oidc.eks.us-west-1.amazonaws.com/id/6C8ED2C48B8B2BD3022877F93BF16E7D"
  + clusterOidcProviderArn: "arn:aws:iam::616138583583:oidc-provider/oidc.eks.us-west-1.amazonaws.com/id/6C8ED2C48B8B2BD3022877F93BF16E7D"
  + kubeconfig            : {
      + apiVersion     : "v1"

...
```

Once the outputs have been created, we can output the `KUBECONFIG` to a file using the `pulumi stack output` command:

```
pulumi stack output kubeconfig | tee ~/.kube/config
```

This will give us the ability to use the `kubectl` command with our cluster. Let's try that now:

```
kubectl get nodes
NAME                                         STATUS   ROLES    AGE     VERSION
ip-172-16-0-101.us-west-1.compute.internal   Ready    <none>   9m10s   v1.18.9-eks-d1db3c
ip-172-16-0-28.us-west-1.compute.internal    Ready    <none>   9m18s   v1.18.9-eks-d1db3c
```

We're now ready to deploy resources to our EKS cluster.

