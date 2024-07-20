+++
title = "3.3 Create a VPC, ECS Cluster, & Load Balancer"
chapter = false
weight = 10
+++

## Step 1 &mdash; Create a VPC

First, we'll create a VPC to host our ECS cluster. We'll use the AWSx provider to do this. The AWSx provider contains higher-level constructs called [component resources](https://www.pulumi.com/docs/intro/concepts/resources/components/) that allow us to create production-ready infrastructure without needing to declare every individual resource.

Add the following to your `__main__.py`:

```python
import pulumi as pulumi
import pulumi_aws as aws
import pulumi_awsx as awsx

vpc = awsx.ec2.Vpc(
    "vpc",
    nat_gateways=awsx.ec2.NatGatewayConfigurationArgs(
        strategy=awsx.ec2.NatGatewayStrategy.SINGLE
    )
)
```

To see how Pulumi components help us ship infrastructure faster with fewer lines of code, run `pulumi preview` to see all the resources that will be created. Notice how, with a single line of code and the default options, we have a complete, functional VPC, deployed across 3 AZs, with public and private subnets. To explore the options available in the VPC component, as well as the other components available in AWSx, see the [AWSx API docs](https://www.pulumi.com/registry/packages/awsx/api-docs/) in the Pulumi Registry.

## Step 2 &mdash; Create an ECS Cluster

Now we will add the ECS cluster itself.

Add the following to your `__main__.py`:

```python
cluster = aws.ecs.Cluster("cluster")
```

> :white_check_mark: After this change, your `__main__.py` should look like this:

```python
import pulumi as pulumi
import pulumi_aws as aws
import pulumi_awsx as awsx

vpc = awsx.ec2.Vpc(
    "vpc",
    nat_gateways=awsx.ec2.NatGatewayConfigurationArgs(
        strategy=awsx.ec2.NatGatewayStrategy.SINGLE
    )
)

cluster = aws.ecs.Cluster("cluster")
```

## Step 3 &mdash; Add Load Balancer Resources

Next, we will add an application load balancer (ALB) to the public subnet of our VPC and listen for HTTP traffic port 80, plus some associated resources.

Add the following to your `__main__.py`:

```python
alb_security_group = aws.ec2.SecurityGroup(
    "alb-security-group",
    vpc_id=vpc.vpc_id,
    description="ALB",
    ingress=[aws.ec2.SecurityGroupIngressArgs(
        description="Allow HTTP from anywhere",
        protocol="tcp",
        from_port=80,
        to_port=80,
        cidr_blocks=["0.0.0.0/0"],
    )],
    egress=[aws.ec2.SecurityGroupEgressArgs(
        description="Allow HTTP to VPC",
        protocol="tcp",
        from_port=80,
        to_port=80,
        cidr_blocks=[vpc.vpc.cidr_block],
    )],
    tags={
        "Name": "ALB"
    },
)

alb = aws.lb.LoadBalancer(
    "app-lb",
    security_groups=[alb_security_group.id],
    subnets=vpc.public_subnet_ids,
)

target_group = aws.lb.TargetGroup(
    "app-tg",
    port=80,
    protocol="HTTP",
    target_type="ip",
    vpc_id=vpc.vpc_id,
)

http_listener = aws.lb.Listener(
    "http-listener",
    load_balancer_arn=alb.arn,
    port=80,
    default_actions=[{
        "type": "forward",
        "target_group_arn": target_group.arn
    }]
)
```

> :white_check_mark: After this change, your `__main__.py` should look like this:

```python
import pulumi
import pulumi_aws as aws
import pulumi_awsx as awsx

vpc = awsx.ec2.Vpc(
    "vpc",
    nat_gateways=awsx.ec2.NatGatewayConfigurationArgs(
        strategy=awsx.ec2.NatGatewayStrategy.SINGLE
    )
)

cluster = aws.ecs.Cluster("cluster")

alb_security_group = aws.ec2.SecurityGroup(
    "alb-security-group",
    vpc_id=vpc.vpc_id,
    description="ALB",
    ingress=[aws.ec2.SecurityGroupIngressArgs(
        description="Allow HTTP from anywhere",
        protocol="tcp",
        from_port=80,
        to_port=80,
        cidr_blocks=["0.0.0.0/0"],
    )],
    egress=[aws.ec2.SecurityGroupEgressArgs(
        description="Allow HTTP to VPC",
        protocol="tcp",
        from_port=80,
        to_port=80,
        cidr_blocks=[vpc.vpc.cidr_block],
    )],
    tags={
        "Name": "ALB"
    },
)

alb = aws.lb.LoadBalancer(
    "app-lb",
    security_groups=[alb_security_group.id],
    subnets=vpc.public_subnet_ids,
)

target_group = aws.lb.TargetGroup(
    "app-tg",
    port=80,
    protocol="HTTP",
    target_type="ip",
    vpc_id=vpc.vpc_id,
)

http_listener = aws.lb.Listener(
    "http-listener",
    load_balancer_arn=alb.arn,
    port=80,
    default_actions=[{
        "type": "forward",
        "target_group_arn": target_group.arn
    }]
)
```

Run your program to deploy the infrastructure we've declared so far:

```bash
pulumi up
```

In the next step, we'll define a Fargate service and deploy a container.
