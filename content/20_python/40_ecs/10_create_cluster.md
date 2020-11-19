+++
title = "3.2 Create an ECS Cluster & LoadBalancer"
chapter = false
weight = 10
+++

## Step 1 &mdash; Create an ECS Cluster

Remove all the boilerplate from the project bootstrap.

Import the AWS and Pulumi packages in an empty `__main__.py` file:

```python
import pulumi
import pulumi_aws as aws
```

And now create a new ECS cluster. You will use the default values, so doing so is very concise:

```python
...
cluster = aws.ecs.Cluster("cluster")
```

> :white_check_mark: After this change, your `__main__.py` should look like this:

```python
import pulumi
import pulumi_aws as aws

cluster = aws.ecs.Cluster("cluster")
```

## Step 2 &mdash; Create a Load-Balanced Container Service

Next, allocate the application load balancer (ALB) and listen for HTTP traffic port 80. In order to do this, we need to find the
default VPC and the subnet groups for it:

```python
...
vpc = aws.ec2.get_vpc(default=True)
vpc_subnets = aws.ec2.get_subnet_ids(vpc_id=vpc.id)

group = aws.ec2.SecurityGroup(
    "web-secgrp",
    vpc_id=default_vpc.id,
    description='Enable HTTP access',
    ingress=[
        { 'protocol': 'icmp', 'from_port': 8, 'to_port': 0, 'cidr_blocks': ['0.0.0.0/0'] },
        { 'protocol': 'tcp', 'from_port': 80, 'to_port': 80, 'cidr_blocks': ['0.0.0.0/0'] }
    ],
    egress=[
        { 'protocol': "-1", 'from_port': 0, 'to_port': 0, 'cidr_blocks': ['0.0.0.0/0'] }
    ])

alb = aws.lb.LoadBalancer(
    "app-lb",
    internal="false",
    security_groups=[group.id],
    subnets=vpc_subnets.ids,
    load_balancer_type="application",
)

atg = aws.lb.TargetGroup(
    "app-tg",
    port=80,
    deregistration_delay=0,
    protocol="HTTP",
    target_type="ip",
    vpc_id=vpc.id,
)

wl = aws.lb.Listener(
    "web",
    load_balancer_arn=alb.arn,
    port=80,
    default_actions=[{"type": "forward", "target_group_arn": atg.arn}],
)
```
> :white_check_mark: After this change, your `__main__.py` should look like this:

```python
import pulumi
import pulumi_aws as aws

cluster = aws.ecs.Cluster("cluster")

vpc = aws.ec2.get_vpc(default=True)
vpc_subnets = aws.ec2.get_subnet_ids(vpc_id=vpc.id)

group = aws.ec2.SecurityGroup(
    "web-secgrp",
    vpc_id=vpc.id,
    description="Enable HTTP access",
    ingress=[
        {
            "protocol": "icmp",
            "from_port": 8,
            "to_port": 0,
            "cidr_blocks": ["0.0.0.0/0"],
        },
        {
            "protocol": "tcp",
            "from_port": 80,
            "to_port": 80,
            "cidr_blocks": ["0.0.0.0/0"],
        },
    ],
    egress=[
        {
            "protocol": "-1",
            "from_port": 0,
            "to_port": 0,
            "cidr_blocks": ["0.0.0.0/0"],
        }
    ],
)

alb = aws.lb.LoadBalancer(
    "app-lb",
    internal="false",
    security_groups=[group.id],
    subnets=vpc_subnets.ids,
    load_balancer_type="application",
)

atg = aws.lb.TargetGroup(
    "app-tg",
    port=80,
    deregistration_delay=0,
    protocol="HTTP",
    target_type="ip",
    vpc_id=vpc.id,
)

wl = aws.lb.Listener(
    "web",
    load_balancer_arn=alb.arn,
    port=80,
    default_actions=[{"type": "forward", "target_group_arn": atg.arn}],
)
```

Run your program with `pulumi up`:

```
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/iac-workshops-ecs/dev/previews/15da708c-a82d-4d97-bdf6-a982901d9fda

     Type                      Name                   Plan       
 +   pulumi:pulumi:Stack       iac-workshops-ecs-dev  create     
 +   ├─ aws:ecs:Cluster        cluster                create     
 +   ├─ aws:ec2:SecurityGroup  web-secgrp             create     
 +   ├─ aws:lb:TargetGroup     app-tg                 create     
 +   ├─ aws:lb:LoadBalancer    app-lb                 create     
 +   └─ aws:lb:Listener        web                    create     
```

We've fleshed out our infrastructure and added a LoadBalancer we can add infrastructure to, in the next steps we'll run a container.
