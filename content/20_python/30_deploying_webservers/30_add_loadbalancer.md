+++
title = "1.4. Add a LoadBalancer"
chapter = false
weight = 30
+++

Needing to loop over the webservers isn't very realistic. You will now create a load balancer over them to distribute load evenly.

## Step 1 &mdash; Update our Security Group 

We need to add an egress rule to our security group. Whenever you add a listener to your load balancer or update the health check port for a
target group used by the load balancer to route requests, you must verify that the security groups associated with the load balancer allow 
traffic on the new port in both directions.

```python
...
group = aws.ec2.SecurityGroup(
    "web-secgrp",
    ingress=[
        { 'protocol': 'icmp', 'from_port': 8, 'to_port': 0, 'cidr_blocks': ['0.0.0.0/0'] },
        { 'protocol': 'tcp', 'from_port': 80, 'to_port': 80, 'cidr_blocks': ['0.0.0.0/0'] },
    ],
    egress=[
        { 'protocol': 'tcp', 'from_port': 80, 'to_port': 80, 'cidr_blocks': ['0.0.0.0/0'] },
    ]
)
...
```

This is required to ensure the security group ingress rules don't conflict with the load balancer's.

## Step 2 &mdash; Define the ALB

Now right after the security group creation, and before the EC2 creation block, add the load balancer creation steps:

```python
...
default_vpc = aws.ec2.get_vpc(default="true")
default_vpc_subnets = aws.ec2.get_subnet_ids(vpc_id=default_vpc.id)

lb = aws.lb.LoadBalancer("external-loadbalancer",
    internal="false",
    security_groups=[group.id],
    subnets=default_vpc_subnets.ids,
    load_balancer_type="application",
)

target_group = aws.lb.TargetGroup("target-group",
    port=80,
    protocol="HTTP",
    target_type="ip",
    vpc_id=default_vpc.id
)

listener = aws.lb.Listener("listener",
   load_balancer_arn=lb.arn,
   port=80,
   default_actions=[{
       "type": "forward",
       "target_group_arn": target_group.arn
   }]
)
...
```

Here, we've defined the ALB, its TargetGroup and some Listeners, but we haven't actually added the EC2 instances to the ALB. 

## Step 3 &mdash; Add the Instances to the ALB

Replace the VM creation block with the following:

```python
...
ips = []
hostnames = []
for az in aws.get_availability_zones().names:
    server = aws.ec2.Instance(f'web-server-{az}',
      instance_type="t2.micro",
      security_groups=[group.name],
      ami=ami.id,
      user_data="""#!/bin/bash
echo \"Hello, World -- from {}!\" > index.html
nohup python -m SimpleHTTPServer 80 &
""".format(az),
      availability_zone=az,
      tags={
          "Name": "web-server",
      },
    )
    ips.append(server.public_ip)
    hostnames.append(server.public_dns)

    attachment = aws.lb.TargetGroupAttachment(f'web-server-{az}',
        target_group_arn=target_group.arn,
        target_id=server.private_ip,
        port=80,
    )

export('ips', ips)
export('hostnames', hostnames)
export("url", lb.dns_name)
```

> :white_check_mark: After this change, your `__main__.py` should look like this:

```python
from pulumi import export
import pulumi_aws as aws

ami = aws.get_ami(
    most_recent="true",
    owners=["137112412989"],
    filters=[{"name":"name","values":["amzn-ami-hvm-*-x86_64-ebs"]}])

group = aws.ec2.SecurityGroup(
    "web-secgrp",
    ingress=[
        { 'protocol': 'icmp', 'from_port': 8, 'to_port': 0, 'cidr_blocks': ['0.0.0.0/0'] },
        { 'protocol': 'tcp', 'from_port': 80, 'to_port': 80, 'cidr_blocks': ['0.0.0.0/0'] },
    ],
    egress=[
        { 'protocol': 'tcp', 'from_port': 80, 'to_port': 80, 'cidr_blocks': ['0.0.0.0/0'] },
    ]
)

default_vpc = aws.ec2.get_vpc(default="true")
default_vpc_subnets = aws.ec2.get_subnet_ids(vpc_id=default_vpc.id)

lb = aws.lb.LoadBalancer("external-loadbalancer",
    internal="false",
    security_groups=[group.id],
    subnets=default_vpc_subnets.ids,
    load_balancer_type="application",
)

target_group = aws.lb.TargetGroup("target-group",
    port=80,
    protocol="HTTP",
    target_type="ip",
    vpc_id=default_vpc.id
)

listener = aws.lb.Listener("listener",
   load_balancer_arn=lb.arn,
   port=80,
   default_actions=[{
       "type": "forward",
       "target_group_arn": target_group.arn
   }]
)

ips = []
hostnames = []
for az in aws.get_availability_zones().names:
    server = aws.ec2.Instance(f'web-server-{az}',
      instance_type="t2.micro",
      security_groups=[group.name],
      ami=ami.id,
      user_data="""#!/bin/bash
echo \"Hello, World -- from {}!\" > index.html
nohup python -m SimpleHTTPServer 80 &
""".format(az),
      availability_zone=az,
      tags={
          "Name": "web-server",
      },
    )
    ips.append(server.public_ip)
    hostnames.append(server.public_dns)

    attachment = aws.lb.TargetGroupAttachment(f'web-server-{az}',
        target_group_arn=target_group.arn,
        target_id=server.private_ip,
        port=80,
    )

export('ips', ips)
export('hostnames', hostnames)
export("url", lb.dns_name)
```

This is all the infrastructure we need for our load balanced webserver. Let's apply it.

## Step 4 &mdash; Deploy your Changes

Deploy these updates:

```bash
pulumi up
```

This should result in a fairly large update and, if all goes well, the load balancer's resulting endpoint URL:

```
Updating (dev):

     Type                             Name                      Status
     pulumi:pulumi:Stack              python-testing-dev        created
~    ├─ aws:ec2:SecurityGroup         web-secgrp                updated     [diff: ~ingress, ~egress]
 +   ├─ aws:lb:TargetGroup            target-group              created
 +   ├─ aws:lb:LoadBalancer           external-loadbalancer     created
 +   ├─ aws:lb:TargetGroupAttachment  web-server-eu-central-1b  created
 +   ├─ aws:lb:TargetGroupAttachment  web-server-eu-central-1c  created
 +   ├─ aws:lb:TargetGroupAttachment  web-server-eu-central-1a  created
 +   └─ aws:lb:Listener               listener                  created

Outputs:
    hostnames: [
        [0]: "ec2-18-197-184-46.eu-central-1.compute.amazonaws.com"
        [1]: "ec2-18-196-225-191.eu-central-1.compute.amazonaws.com"
        [2]: "ec2-35-158-83-62.eu-central-1.compute.amazonaws.com"
    ]
    ips      : [
        [0]: "18.197.184.46"
        [1]: "18.196.225.191"
        [2]: "35.158.83.62"
  + url      : "web-traffic-09348bc-723263075.eu-central-1.elb.amazonaws.com"

Resources:
    + 6 created
    ~ 1 updated
    7 changes. 4 unchanged

Duration: 2m33s

Permalink: https://app.pulumi.com/jaxxstorm/iac-workshop/dev/updates/3
```

## Step 5 &mdash; Verify 

Now we can curl the load balancer:

```bash
for i in {0..10}; do curl $(pulumi stack output url); done
```

Observe that the resulting text changes based on where the request is routed:

```
Hello, World -- from eu-central-1b!
Hello, World -- from eu-central-1c!
Hello, World -- from eu-central-1a!
Hello, World -- from eu-central-1b!
Hello, World -- from eu-central-1b!
Hello, World -- from eu-central-1a!
Hello, World -- from eu-central-1c!
Hello, World -- from eu-central-1a!
Hello, World -- from eu-central-1c!
Hello, World -- from eu-central-1b!
```

## Step 6 &mdash; Destroy Everything

Finally, destroy the resources and the stack itself:

```
pulumi destroy
pulumi stack rm

