+++
title = "2.2 Provision an EC2 Instances"
chapter = false
weight = 10
+++

## Step 1 &mdash;  Declare the EC2 Instance

Remove any existing code here from the bootstrapping of your project. 
We're going to use AWS API to query the latest Amazon Linux machine image. Doing this in code avoids needing to hard-code the machine image (a.k.a., its AMI):

```java
var latestAmi = Ec2Functions
        .getAmi(
                GetAmiArgs.builder()
                        .mostRecent(true)
                        .owners("137112412989")
                        .filters(GetAmiFilter.builder().name("name").values("amzn-ami-hvm-*-x86_64-ebs").build())
                        .build()
        ).thenApply(GetAmiResult::id);
```

We also need to grab the default vpc that is available in our AWS account:

```java
var defaultVpcId = Ec2Functions
        .getVpc(GetVpcArgs.builder().default_(true).build())
        .thenApply(GetVpcResult::id);
```

Next, create an AWS security group. This enables HTTP traffic on port 80:

```java
var securityGroup = new SecurityGroup("web-secgrp", SecurityGroupArgs
        .builder()
        .description("Enable HTTP Access")
        .vpcId(Output.of(defaultVpcId))
        .ingress(
                SecurityGroupIngressArgs.builder()
                        .protocol("tcp")
                        .fromPort(80)
                        .toPort(80)
                        .cidrBlocks("0.0.0.0/0")
                        .build())
        .build());
```

Create the server. Notice it has a startup script that spins up a simple Python webserver:

```java
var ec2Instance = new Instance("web-server",
        InstanceArgs.builder()
                .instanceType("t2.micro")
                .vpcSecurityGroupIds(Output.all(securityGroup.getId()))
                .ami(Output.of(latestAmi))
                .userData("""
                        #!/bin/bash
                        echo "Hello, World!" > index.html
                        nohup python -m SimpleHTTPServer 80 &""")
                .tags(Map.of("Name", "web-server"))
                .build()
);
```

> For most real-world applications, you would want to create a dedicated image for your application, rather than embedding the script in your code like this.

Finally, export the EC2 instances' resulting IP address and hostname:

```java
ctx.export("ip", ec2Instance.publicIp());
ctx.export("hostname", ec2Instance.publicDns());
```

> :white_check_mark: After this change, your `App.java` should look like this:

```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.ec2.*;
import com.pulumi.aws.ec2.inputs.GetAmiArgs;
import com.pulumi.aws.ec2.inputs.GetAmiFilter;
import com.pulumi.aws.ec2.inputs.GetVpcArgs;
import com.pulumi.aws.ec2.inputs.SecurityGroupIngressArgs;
import com.pulumi.aws.ec2.outputs.GetAmiResult;
import com.pulumi.aws.ec2.outputs.GetVpcResult;
import com.pulumi.core.Output;

import java.util.Map;

public class App {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var latestAmi = Ec2Functions
                    .getAmi(
                            GetAmiArgs.builder()
                                    .mostRecent(true)
                                    .owners("137112412989")
                                    .filters(GetAmiFilter.builder().name("name").values("amzn-ami-hvm-*-x86_64-ebs").build())
                                    .build()
                    ).thenApply(GetAmiResult::id);

            var defaultVpcId = Ec2Functions
                    .getVpc(GetVpcArgs.builder().default_(true).build())
                    .thenApply(GetVpcResult::id);

            var securityGroup = new SecurityGroup("web-secgrp", SecurityGroupArgs
                    .builder()
                    .description("Enable HTTP Access")
                    .vpcId(Output.of(defaultVpcId))
                    .ingress(
                            SecurityGroupIngressArgs.builder()
                                    .protocol("tcp")
                                    .fromPort(80)
                                    .toPort(80)
                                    .cidrBlocks("0.0.0.0/0")
                                    .build())
                    .build());


            var ec2Instance = new Instance("web-server",
                    InstanceArgs.builder()
                            .instanceType("t2.micro")
                            .vpcSecurityGroupIds(Output.all(securityGroup.getId()))
                            .ami(Output.of(latestAmi))
                            .userData("""
                                    #!/bin/bash
                                    echo "Hello, World!" > index.html
                                    nohup python -m SimpleHTTPServer 80 &""")
                            .tags(Map.of("Name", "web-server"))
                            .build()
            );

            ctx.export("ip", ec2Instance.publicIp());
            ctx.export("hostname", ec2Instance.publicDns());
        });
    }
}
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

Either way you should see a response from the webserver:

```
Hello, World!
```
