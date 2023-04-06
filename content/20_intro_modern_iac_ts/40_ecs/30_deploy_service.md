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

Now we define a task definition for our ECS service and add the DNS name of the ALB we defined earlier so we can get the public URL for our service.

Add the following to your `index.ts`:

```typescript
const taskDefinition = new aws.ecs.TaskDefinition("app-task", {
  family: "fargate-task-definition",
  cpu: "256",
  memory: "512",
  networkMode: "awsvpc",
  requiresCompatibilities: ["FARGATE"],
  executionRoleArn: role.arn,
  containerDefinitions: JSON.stringify([{
    name: "my-app",
    image: "nginx:latest",
    portMappings: [{
      containerPort: 80,
      hostPort: 80,
      protocol: "tcp",
    }],
  }]),
});


const service = new aws.ecs.Service("app-svc", {
  cluster: cluster.arn,
  desiredCount: 1,
  launchType: "FARGATE",
  taskDefinition: taskDefinition.arn,
  networkConfiguration: {
    assignPublicIp: true,
    subnets: vpc.privateSubnetIds,
    securityGroups: [group.id],
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

const group = new aws.ec2.SecurityGroup("web-secgrp", {
  vpcId: vpc.vpcId,
  description: "Enable HTTP access",
  ingress: [{
    protocol: "tcp",
    fromPort: 80,
    toPort: 80,
    cidrBlocks: ["0.0.0.0/0"],
  }],
  egress: [{
    protocol: "-1",
    fromPort: 0,
    toPort: 0,
    cidrBlocks: ["0.0.0.0/0"],
  }],
});

const alb = new aws.lb.LoadBalancer("app-lb", {
  securityGroups: [group.id],
  subnets: vpc.publicSubnetIds,
});

const targetGroup = new aws.lb.TargetGroup("app-tg", {
  port: 80,
  protocol: "HTTP",
  targetType: "ip",
  vpcId: vpc.vpcId,
});

const listener = new aws.lb.Listener("web", {
  loadBalancerArn: alb.arn,
  port: 80,
  defaultActions: [{
    type: "forward",
    targetGroupArn: targetGroup.arn,
  }],
});

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
  family: "fargate-task-definition",
  cpu: "256",
  memory: "512",
  networkMode: "awsvpc",
  requiresCompatibilities: ["FARGATE"],
  executionRoleArn: role.arn,
  containerDefinitions: JSON.stringify([{
    name: "my-app",
    image: "nginx:latest",
    portMappings: [{
      containerPort: 80,
      hostPort: 80,
      protocol: "tcp",
    }],
  }]),
});

const service = new aws.ecs.Service("app-svc", {
  cluster: cluster.arn,
  desiredCount: 1,
  launchType: "FARGATE",
  taskDefinition: taskDefinition.arn,
  networkConfiguration: {
    assignPublicIp: true,
    subnets: vpc.privateSubnetIds,
    securityGroups: [group.id],
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
