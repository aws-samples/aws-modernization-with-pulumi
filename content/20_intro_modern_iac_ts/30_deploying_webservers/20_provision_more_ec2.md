+++
title = "2.3 Add more EC2 Instances"
chapter = false
weight = 20
+++

## Step 1 – Add more EC2 instances

Now you will create multiple EC2 instances, each running the same Python webserver, across all AWS availability zones in
your region. Replace the part of your code that creates the webserver and exports the resulting IP address and hostname with the following:

```python
...
ips = []
hostnames = []
for az in aws.get_availability_zones().names:
    server = aws.ec2.Instance(f'web-server-{az}',
      instance_type="t2.micro",
      vpc_security_group_ids=[group.id],
      ami=ami.id,
      availability_zone=az,
      user_data="""#!/bin/bash
echo \"Hello, World -- from {}!\" > index.html
nohup python -m SimpleHTTPServer 80 &
""".format(az),
      tags={
          "Name": "web-server",
      },
    )
    ips.append(server.public_ip)
    hostnames.append(server.public_dns)

pulumi.export("ips", ips)
pulumi.export("hostnames", hostnames)
```

> :white_check_mark: After this change, your `__main__.py` should look like this:

```python
"""An AWS Python Pulumi program"""

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
    vpc_id=vpc.id,
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


ips = []
hostnames = []

for az in aws.get_availability_zones().names:
    server = aws.ec2.Instance(
        f"web-server-{az}",
        instance_type="t2.micro",
        vpc_security_group_ids=[group.id],
        ami=ami.id,
        user_data="""#!/bin/bash
echo \"Hello, World -- from {}!\" > index.html
nohup python -m SimpleHTTPServer 80 &
""".format(
            az
        ),
        tags={
            "Name": "web-server",
        },
    )
    ips.append(server.public_ip)
    hostnames.append(server.public_dns)

pulumi.export("ips", ips)
pulumi.export("hostnames", hostnames)
```

Now run a command to update your stack with the new resource definitions:

```bash
pulumi up
```

You will see output like the following:

```
Updating (dev):


     Type                 Name                         Plan       
     pulumi:pulumi:Stack  iac-workshop-webservers-dev             
 +   ├─ aws:ec2:Instance  web-server-us-west-2a        create     
 +   ├─ aws:ec2:Instance  web-server-us-west-2b        create     
 +   ├─ aws:ec2:Instance  web-server-us-west-2c        create     
 +   ├─ aws:ec2:Instance  web-server-us-west-2d        create     
 -   └─ aws:ec2:Instance  web-server                   delete   

Outputs:
  - hostname : "ec2-18-236-225-249.us-west-2.compute.amazonaws.com"
  + hostnames: [
  +     [0]: "ec2-34-221-69-215.us-west-2.compute.amazonaws.com"
  +     [1]: "ec2-54-202-104-111.us-west-2.compute.amazonaws.com"
  +     [2]: "ec2-54-244-18-49.us-west-2.compute.amazonaws.com"
  +     [3]: "ec2-18-236-225-249.us-west-2.compute.amazonaws.com"
    ]
  - ip       : "18.236.225.249"
  + ips      : [
  +     [0]: "34.221.69.215"
  +     [1]: "54.202.104.111"
  +     [2]: "54.244.18.49"
  +     [3]: "18.236.225.249"
    ]
Resources:
    + 4 created
    - 1 deleted
    5 changes. 2 unchanged

Duration: 1m2s

Permalink: https://app.pulumi.com/jaxxstorm/iac-workshop/dev/updates/2
```

Notice that your original server was deleted and new ones created in its place, because its name changed.

To test the changes, curl any of the resulting IP addresses or hostnames:

```bash
for i in {0..2}; do curl $(pulumi stack output hostnames | jq -r ".[${i}]"); done
```

> The count of servers depends on the number of AZs in your region. Adjust the `{0..2}` accordingly.

> The `pulumi stack output` command emits JSON serialized data &mdash; hence the use of the `jq` tool to extract a specific index. If you don't have `jq`, don't worry; simply copy-and-paste the hostname or IP address from the console output and `curl` that.

Note that the webserver number is included in its response:

```
Hello, World -- from us-west-2a!
Hello, World -- from us-west-2b!
Hello, World -- from us-west-2c!
```

