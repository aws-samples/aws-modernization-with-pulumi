+++
title = "1.3. Add more EC2 Instances"
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
      security_groups=[group.name],
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
```

{{% notice info %}}
The `__main__.py` file should now have the following contents:
{{% /notice %}}
```python
from pulumi import export
import pulumi_aws as aws

ami = aws.get_ami(
    most_recent="true",
    owners=["137112412989"],
    filters=[{"name":"name","values":["amzn-ami-hvm-*-x86_64-ebs"]}])

group = aws.ec2.SecurityGroup(
    "web-secgrp",
    description='Enable HTTP access',
    ingress=[
        { 'protocol': 'icmp', 'from_port': 8, 'to_port': 0, 'cidr_blocks': ['0.0.0.0/0'] },
        { 'protocol': 'tcp', 'from_port': 80, 'to_port': 80, 'cidr_blocks': ['0.0.0.0/0'] }
    ])

ips = []
hostnames = []
for az in aws.get_availability_zones().names:
    server = aws.ec2.Instance(f'web-server-{az}',
      instance_type="t2.micro",
      security_groups=[group.name],
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

export('ips', ips)
export('hostnames', hostnames)

```

Now run a command to update your stack with the new resource definitions:

```bash
pulumi up
```

You will see output like the following:

```
Updating (dev):

     Type                 Name                      Status
     pulumi:pulumi:Stack  iac-workshop-dev
 +   ├─ aws:ec2:Instance  web-server-eu-central-1a  created
 +   ├─ aws:ec2:Instance  web-server-eu-central-1b  created
 +   ├─ aws:ec2:Instance  web-server-eu-central-1c  created
 -   └─ aws:ec2:Instance  web-server                deleted

Outputs:
  + hostnames     : [
  +     [0]: "ec2-18-197-184-46.eu-central-1.compute.amazonaws.com"
  +     [1]: "ec2-18-196-225-191.eu-central-1.compute.amazonaws.com"
  +     [2]: "ec2-35-158-83-62.eu-central-1.compute.amazonaws.com"
    ]
  + ips           : [
  +     [0]: "18.197.184.46"
  +     [1]: "18.196.225.191"
  +     [2]: "35.158.83.62"
    ]
  - hostname: "ec2-52-57-250-206.eu-central-1.compute.amazonaws.com"
  - ip      : "52.57.250.206"

Resources:
    + 3 created
    - 1 deleted
    4 changes. 2 unchanged

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
Hello, World -- from eu-central-1a!
Hello, World -- from eu-central-1b!
Hello, World -- from eu-central-1c!
```

