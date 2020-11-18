+++
title = "1.1. Provision an EC2 Instances"
chapter = false
weight = 10
+++

## Step 1 &mdash;  Declare the EC2 Instance

Remove any existing code here from the bootstrapping of your project. Then, import the AWS package in an empty `__main__.py` file:

```python
from pulumi import export
import pulumi_aws as aws
```

Now dynamically query the Amazon Linux machine image. Doing this in code avoids needing to hard-code the machine image (a.k.a., its AMI):

```python
ami = aws.get_ami(
    most_recent="true",
    owners=["137112412989"],
    filters=[{"name":"name","values":["amzn-ami-hvm-*-x86_64-ebs"]}])
```

We also need to grab the default vpc that is available in our AWS account:

```python
vpc = aws.ec2.get_vpc(default=True)
```

Next, create an AWS security group. This enables `ping` over ICMP and HTTP traffic on port 80:

```python
group = aws.ec2.SecurityGroup(
    "web-secgrp",
    description='Enable HTTP access',
    vpc_id=vpc.id,
    ingress=[
        { 'protocol': 'icmp', 'from_port': 8, 'to_port': 0, 'cidr_blocks': ['0.0.0.0/0'] },
        { 'protocol': 'tcp', 'from_port': 80, 'to_port': 80, 'cidr_blocks': ['0.0.0.0/0'] }
])
```

Create the server. Notice it has a startup script that spins up a simple Python webserver:

```python
server = aws.ec2.Instance(
    'web-server',
    instance_type="t2.micro",
    vpc_security_group_ids=[group.id],
    ami=ami.id,
    user_data="""
#!/bin/bash
echo "Hello, World!" > index.html
nohup python -m SimpleHTTPServer 80 &
    """,
    tags={
        "Name": "web-server",
    },
)
```

> For most real-world applications, you would want to create a dedicated image for your application, rather than embedding the script in your code like this.

Finally export the EC2 instances's resulting IP address and hostname:

```python
pulumi.export('ip', server.public_ip)
pulumi.export('hostname', server.public_dns)
```

> :white_check_mark: After this change, your `__main__.py` should look like this:

```python
import pulumi
import pulumi_aws as aws


ami = aws.get_ami(
    most_recent="true",
    owners=["137112412989"],
    filters=[{"name": "name", "values": ["amzn-ami-hvm-*-x86_64-ebs"]}],
)

vpc = aws.ec2.get_vpc(default=True)

group = aws.ec2.SecurityGroup(
    "web-secgrp",
    description="Enable HTTP Access",
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
)

server = aws.ec2.Instance(
    "web=server",
    instance_type="t2.micro",
    vpc_security_group_ids=[group.name],
    ami=ami.id,
    user_data="""
#!/bin/bash
echo "Hello, World!" > index.html
nohup python -m SimpleHTTPServer 80 &
    """,
    tags={
        "Name": "web-server",
    },
)

pulumi.export("ip", server.public_ip)
pulumi.export("hostname", server.public_dns)
```

## Step 2 &mdash; Provision the EC2 Instance and Access It

To provision the VM, run:

```bash
pulumi up
```

After confirming, you will see output like the following:

```
Updating (dev):

     Type                      Name              Status
 +   pulumi:pulumi:Stack       iac-workshop-dev  created
 +   ├─ aws:ec2:SecurityGroup  web-secgrp        created
 +   └─ aws:ec2:Instance       web-server        created

Outputs:
    hostname: "ec2-52-57-250-206.eu-central-1.compute.amazonaws.com"
    ip      : "52.57.250.206"

Resources:
    + 3 created

Duration: 40s

Permalink: https://app.pulumi.com/jaxxstorm/iac-workshop/dev/updates/1
```

To verify that our server is accepting requests properly, curl either the hostname or IP address:

```bash
curl $(pulumi stack output hostname)
```

Either way you should see a response from the Python webserver:

```
Hello, World!
```
