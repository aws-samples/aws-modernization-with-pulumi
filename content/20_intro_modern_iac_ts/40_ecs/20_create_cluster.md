+++
title = "3.3 Create a VPC, ECS Cluster, & Load Balancer"
chapter = false
weight = 10
+++

## Step 1 &mdash; Create a VPC and ECS Cluster

First, we'll create a VPC to host our ECS cluster. We'll use the AWSx provider to do this. The AWSx provider contains higher-level constructs called [component resources](https://www.pulumi.com/docs/intro/concepts/resources/components/) that allow us to create production-ready infrastructure without needing to declare every individual resource. We'll also declare our ECS cluster.

Add the following to your `index.ts`:

```typescript
const vpc = new awsx.ec2.Vpc("vpc", {
  natGateways: {
    strategy: awsx.ec2.NatGatewayStrategy.Single,
  }
});

const cluster = new aws.ecs.Cluster("cluster");
```

To see how Pulumi components help us ship infrastructure faster with fewer lines of code, run `pulumi preview` to see all the resources that will be created. Notice how, with a single line of code and the default options, we have a complete, functional VPC, deployed across 3 AZs, with public and private subnets. To explore the options available in the VPC component, as well as the other components available in AWSx, see the [AWSx API docs](https://www.pulumi.com/registry/packages/awsx/api-docs/) in the Pulumi Registry.

## Step 2 &mdash; Add Load Balancer Resources

Next, we will add an application load balancer (ALB) to the public subnet of our VPC and listen for HTTP traffic port 80, plus some associated resources.

Add the following to your `index.ts`:

```typescript
const albSecGroup = new aws.ec2.SecurityGroup("alb-security-group", {
  vpcId: vpc.vpcId,
  description: "ALB",
  ingress: [{
    description: "Allow HTTP from anywhere",
    protocol: "tcp",
    fromPort: 80,
    toPort: 80,
    cidrBlocks: ["0.0.0.0/0"],
  }],
  egress: [{
    description: "Allow HTTP to VPC",
    protocol: "tcp",
    fromPort: 80,
    toPort: 80,
    cidrBlocks: [vpc.vpc.cidrBlock],
  }],
  tags: {
    Name: "ALB"
  }
});

const alb = new aws.lb.LoadBalancer("app-lb", {
  securityGroups: [albSecGroup.id],
  subnets: vpc.publicSubnetIds,
});

const targetGroup = new aws.lb.TargetGroup("app-tg", {
  port: 80,
  protocol: "HTTP",
  targetType: "ip",
  vpcId: vpc.vpcId,
});

new aws.lb.Listener("http-listener", {
  loadBalancerArn: alb.arn,
  port: 80,
  defaultActions: [{
    type: "forward",
    targetGroupArn: targetGroup.arn,
  }],
});
```

At this point, your `index.ts` should look like this:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

const vpc = new awsx.ec2.Vpc("vpc", {
  natGateways: {
    strategy: awsx.ec2.NatGatewayStrategy.Single,
  }
});

const cluster = new aws.ecs.Cluster("cluster");

const albSecGroup = new aws.ec2.SecurityGroup("alb-security-group", {
  vpcId: vpc.vpcId,
  description: "ALB",
  ingress: [{
    description: "Allow HTTP from anywhere",
    protocol: "tcp",
    fromPort: 80,
    toPort: 80,
    cidrBlocks: ["0.0.0.0/0"],
  }],
  egress: [{
    description: "Allow HTTP to VPC",
    protocol: "tcp",
    fromPort: 80,
    toPort: 80,
    cidrBlocks: [vpc.vpc.cidrBlock],
  }],
  tags: {
    Name: "ALB"
  }
});

const alb = new aws.lb.LoadBalancer("app-lb", {
  securityGroups: [albSecGroup.id],
  subnets: vpc.publicSubnetIds,
});

const targetGroup = new aws.lb.TargetGroup("app-tg", {
  port: 80,
  protocol: "HTTP",
  targetType: "ip",
  vpcId: vpc.vpcId,
});

new aws.lb.Listener("http-listener", {
  loadBalancerArn: alb.arn,
  port: 80,
  defaultActions: [{
    type: "forward",
    targetGroupArn: targetGroup.arn,
  }],
});
```

Run your program to deploy the infrastructure we've declared so far:

```bash
pulumi up
```

In the next step, we'll define an ECS on Fargate service and deploy a container.
