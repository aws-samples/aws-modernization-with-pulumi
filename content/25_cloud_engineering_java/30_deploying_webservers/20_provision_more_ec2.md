+++
title = "2.3 Add more EC2 Instances"
chapter = false
weight = 20
+++

## Step 1 – Add more EC2 instances

Now you will create multiple EC2 instances, each running the same webserver, across all AWS availability zones in
your region. Replace the part of your code that creates the webserver and exports the resulting IP address and hostname with the following:

```java
var instances = AwsFunctions.getAvailabilityZones().thenApply(response ->
        response.zoneIds().stream().map(availabilityZone ->
                new Instance("web-server-%s".formatted(availabilityZone),
                        InstanceArgs.builder()
                                .instanceType("t2.micro")
                                .vpcSecurityGroupIds(Output.all(securityGroup.getId()))
                                .ami(Output.of(latestAmi))
                                .userData("""
                                        #!/bin/bash
                                        echo "Hello, World -- from %s!" > index.html
                                        nohup python -m SimpleHTTPServer 80 &""".formatted(availabilityZone))
                                .tags(Map.of("Name", "web-server-%s".formatted(availabilityZone)))
                                .build()
                )).toList()
);

var instancesOutput = Output.of(instances);
var ipAddresses = instancesOutput.apply(ec2s -> Output.all(ec2s.stream().map(Instance::publicIp).toList()));
var hostnames = instancesOutput.apply(ec2s -> Output.all(ec2s.stream().map(Instance::publicDns).toList()));

ctx.export("ips", ipAddresses);
ctx.export("hostnames", hostnames);
```

> :white_check_mark: After this change, your `App.java` should look like this:

```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.AwsFunctions;
import com.pulumi.aws.ec2.*;
import com.pulumi.aws.ec2.inputs.GetAmiArgs;
import com.pulumi.aws.ec2.inputs.GetAmiFilter;
import com.pulumi.aws.ec2.inputs.GetVpcArgs;
import com.pulumi.aws.ec2.inputs.SecurityGroupIngressArgs;
import com.pulumi.aws.ec2.outputs.GetAmiResult;
import com.pulumi.aws.ec2.outputs.GetVpcResult;
import com.pulumi.core.Output;

import java.util.Map;

public class App3 {

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


            var instances = AwsFunctions.getAvailabilityZones().thenApply(response ->
                    response.zoneIds().stream().map(availabilityZone ->
                            new Instance("web-server-%s".formatted(availabilityZone),
                                    InstanceArgs.builder()
                                            .instanceType("t2.micro")
                                            .vpcSecurityGroupIds(Output.all(securityGroup.getId()))
                                            .ami(Output.of(latestAmi))
                                            .userData("""
                                                    #!/bin/bash
                                                    echo "Hello, World -- from %s!" > index.html
                                                    nohup python -m SimpleHTTPServer 80 &""".formatted(availabilityZone))
                                            .tags(Map.of("Name", "web-server-%s".formatted(availabilityZone)))
                                            .build()
                            )).toList()
            );

            var instancesOutput = Output.of(instances);
            var ipAddresses = instancesOutput.apply(ec2s -> Output.all(ec2s.stream().map(Instance::publicIp).toList()));
            var hostnames = instancesOutput.apply(ec2s -> Output.all(ec2s.stream().map(Instance::publicDns).toList()));

            ctx.export("ips", ipAddresses);
            ctx.export("hostnames", hostnames);
        });
    }
}

```

Now run a command to update your stack with the new resource definitions:

```bash
pulumi up
```

You will see output like the following:

```
Updating (dev)

     Type                 Name                 Status      
     pulumi:pulumi:Stack  iac-workshop-webservers-dev                         
 +   ├─ aws:ec2:Instance  web-server-euw1-az3  created     
 +   ├─ aws:ec2:Instance  web-server-euw1-az1  created     
 +   ├─ aws:ec2:Instance  web-server-euw1-az2  created     
 -   └─ aws:ec2:Instance  web-server           deleted     
 
Outputs:
  - hostname : "ec2-54-229-215-7.eu-west-1.compute.amazonaws.com"
  + hostnames: [
  +     [0]: "ec2-34-253-216-194.eu-west-1.compute.amazonaws.com"
  +     [1]: "ec2-18-203-162-39.eu-west-1.compute.amazonaws.com"
  +     [2]: "ec2-54-155-163-18.eu-west-1.compute.amazonaws.com"
    ]
  - ip       : "54.229.215.7"
  + ips      : [
  +     [0]: "34.253.216.194"
  +     [1]: "18.203.162.39"
  +     [2]: "54.155.163.18"
    ]

Resources:
    + 3 created
    - 1 deleted
    4 changes. 2 unchanged

Duration: 3m21s

```

Notice that your original server was deleted and new ones created in its place, because its name changed.

To test the changes, curl any of the resulting IP addresses or hostnames. If you have the `jq` command installed, run the following command:

```bash
for i in {0..2}; do curl $(pulumi stack output hostnames | jq -r ".[${i}]"); done
```

If you do not have `jq` installed, we recommend it. You can [follow the installation instructions to get `jq`](https://stedolan.github.io/jq/download/) before continuing.

> The count of servers depends on the number of AZs in your region. Adjust the `{0..2}` accordingly.

> The `pulumi stack output` command emits JSON serialized data &mdash; hence the use of the `jq` tool to extract a specific index. If you don't have `jq`, don't worry; simply copy-and-paste the hostname or IP address from the console output and `curl` that.

Note that the webserver number is included in its response:

```
Hello, World -- from euw1-az3!
Hello, World -- from euw1-az1!
Hello, World -- from euw1-az2!
```

