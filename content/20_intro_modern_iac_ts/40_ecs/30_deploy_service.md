+++
title = "3.4 Deploy a Fargate Service"
chapter = false
weight = 20
+++

In order to create a Fargate service, we need to add an IAM Role and a Task Definition and Service. the ECS Cluster will run
the [`nginx` image](https://hub.docker.com/_/nginx) from Docker Hub.

## Step 1 &mdash; Create an ECS Task Execution Role

Now let's define our IAM execution role and attach a policy. Add the following to your `index.ts`:

```typescript
const role = new aws.iam.Role("task-exec-role", {
  assumeRolePolicy: JSON.stringify({
    Version: "2008-10-17",
    Statement: [{
      Action: "sts:AssumeRole",
      Principal: {
        Service: "ecs-tasks.amazonaws.com",
      },
      Effect: "Allow",
      Sid: "",
    }],
  }),
});

new aws.iam.RolePolicyAttachment("task-exec-policy", {
  role: role.name,
  policyArn: aws.iam.ManagedPolicy.AmazonECSTaskExecutionRolePolicy,
});
```

## Step 2 &mdash; Create an ECS Task Definition

Now we need to define some additional resources for our ECS service:

- A task definition
- A security group used by the ECS service
- An ECS service

We'll also add a stack output: the DNS name of the ALB we defined earlier so we can get the public URL for our service.

Add the following to your `index.ts`:

```typescript
const taskDefinition = new aws.ecs.TaskDefinition("app-task", {
  family: "nginx",
  cpu: "256",
  memory: "512",
  networkMode: "awsvpc",
  requiresCompatibilities: ["FARGATE"],
  executionRoleArn: role.arn,
  containerDefinitions: JSON.stringify([{
    name: "my-app",
    image: "nginx:latest",
    portMappings: [{ containerPort: 80 }],
  }]),
});

const serviceSecGroup = new aws.ec2.SecurityGroup("service-sec-grp", {
  vpcId: vpc.vpcId,
  description: "NGINX service",
  ingress: [{
    description: "Allow HTTP from within the VPC",
    protocol: "tcp",
    fromPort: 80,
    toPort: 80,
    cidrBlocks: [vpc.vpc.cidrBlock],
  }],
  egress: [{
    description: "Allow HTTPS to anywhere (to pull container images)",
    protocol: "tcp",
    fromPort: 443,
    toPort: 443,
    cidrBlocks: ["0.0.0.0/0"],
  }],
});

new aws.ecs.Service("app-svc", {
  name: "NGINX",
  cluster: cluster.arn,
  desiredCount: 1,
  launchType: "FARGATE",
  taskDefinition: taskDefinition.arn,
  networkConfiguration: {
    assignPublicIp: true,
    subnets: vpc.privateSubnetIds,
    securityGroups: [serviceSecGroup.id],
  },
  loadBalancers: [{
    targetGroupArn: targetGroup.arn,
    containerName: "my-app",
    containerPort: 80,
  }],
});

export const url = pulumi.interpolate`http://${alb.dnsName}`;
```

After these changes, your `index.ts` should look like this:

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
}, { aliases: [{ name: "web" }] });

const role = new aws.iam.Role("task-exec-role", {
  assumeRolePolicy: JSON.stringify({
    Version: "2008-10-17",
    Statement: [{
      Action: "sts:AssumeRole",
      Principal: {
        Service: "ecs-tasks.amazonaws.com",
      },
      Effect: "Allow",
      Sid: "",
    }],
  }),
});

new aws.iam.RolePolicyAttachment("task-exec-policy", {
  role: role.name,
  policyArn: aws.iam.ManagedPolicy.AmazonECSTaskExecutionRolePolicy,
});

const taskDefinition = new aws.ecs.TaskDefinition("app-task", {
  family: "nginx",
  cpu: "256",
  memory: "512",
  networkMode: "awsvpc",
  requiresCompatibilities: ["FARGATE"],
  executionRoleArn: role.arn,
  containerDefinitions: JSON.stringify([{
    name: "my-app",
    image: "nginx:latest",
    portMappings: [{ containerPort: 80 }],
  }]),
});

const serviceSecGroup = new aws.ec2.SecurityGroup("service-sec-grp", {
  vpcId: vpc.vpcId,
  description: "NGINX service",
  ingress: [{
    description: "Allow HTTP from within the VPC",
    protocol: "tcp",
    fromPort: 80,
    toPort: 80,
    cidrBlocks: [vpc.vpc.cidrBlock],
  }],
  egress: [{
    description: "Allow HTTPS to anywhere (to pull container images)",
    protocol: "tcp",
    fromPort: 443,
    toPort: 443,
    cidrBlocks: ["0.0.0.0/0"],
  }],
});

new aws.ecs.Service("app-svc", {
  name: "NGINX",
  cluster: cluster.arn,
  desiredCount: 1,
  launchType: "FARGATE",
  taskDefinition: taskDefinition.arn,
  networkConfiguration: {
    assignPublicIp: true,
    subnets: vpc.privateSubnetIds,
    securityGroups: [serviceSecGroup.id],
  },
  loadBalancers: [{
    targetGroupArn: targetGroup.arn,
    containerName: "my-app",
    containerPort: 80,
  }],
});

export const url = pulumi.interpolate`http://${alb.dnsName}`;
```

## Step 3 &mdash; Provision the Cluster and Service

Deploy the program to stand up your initial cluster and service:

```bash
pulumi up
```

You can now curl the resulting endpoint:

```bash
curl $(pulumi stack output url)
```

And you'll see the Nginx default homepage:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

## Step 4 &mdash; Update the Service

Now, let update the desired container count from `1` to `3`:

```typescript
...
    desiredCount: 3,
...
```

Next update the stack:

```bash
pulumi up
```

After the command completes, you should be able to view the NGINX default index page by running the following command:

```bash
curl $(pulumi stack output url)
```

## Step 4 &mdash; Cleaning Up

Now that we're done we can destroy the resources we created and the stack itself:

```bash
pulumi destroy
pulumi stack rm dev
```
