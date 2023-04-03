+++
title = "2.4 Add a LoadBalancer"
chapter = false
weight = 30
+++

Needing to loop over the webservers isn't very realistic. You will now create a load balancer over them to distribute load evenly.

## Step 1 &mdash; Update our Security Group 

We need to add an egress rule to our security group. Whenever you add a listener to your load balancer or update the health check port for a
target group used by the load balancer to route requests, you must verify that the security groups associated with the load balancer allow 
traffic on the new port in both directions.


```java
var securityGroup = new SecurityGroup("web-secgrp", SecurityGroupArgs
        .builder()
        .description("Enable HTTP Access")
        .vpcId(vpcIdOutput)
        .ingress(
                SecurityGroupIngressArgs.builder()
                        .protocol("tcp")
                        .fromPort(80)
                        .toPort(80)
                        .cidrBlocks("0.0.0.0/0")
                        .build())
        .egress(
                SecurityGroupEgressArgs.builder()
                        .protocol("tcp")
                        .fromPort(80)
                        .toPort(80)
                        .cidrBlocks("0.0.0.0/0")
                        .build())
        .build());
```

This is required to ensure the security group ingress rules don't conflict with the load balancer's.

## Step 2 &mdash; Define the ALB

Now right after the security group creation, and before the EC2 creation block, add the load balancer creation steps:

```java
var vpcSubnetsIds = defaultVpcId
        .thenCompose(vpcId -> Ec2Functions.getSubnetIds(GetSubnetIdsArgs.builder().vpcId(vpcId).build()))
        .thenApply(GetSubnetIdsResult::ids);

var loadBalancer = new LoadBalancer("loadbalancer",
        LoadBalancerArgs.builder()
                .internal(false)
                .securityGroups(Output.all(securityGroup.getId()))
                .subnets(Output.of(vpcSubnetsIds))
                .loadBalancerType("application")
                .build()
);

var targetGroup = new TargetGroup("target-group",
        TargetGroupArgs.builder()
                .port(80)
                .protocol("HTTP")
                .targetType("ip")
                .vpcId(Output.of(defaultVpcId))
                .build()
);

var albListener = new Listener("listener",
        ListenerArgs.builder()
                .loadBalancerArn(loadBalancer.arn())
                .port(80)
                .defaultActions(
                        ListenerDefaultActionArgs.builder()
                                .type("forward")
                                .targetGroupArn(targetGroup.arn())
                                .build())
                .build()
);

```

Here, we've defined the ALB, its TargetGroup and some Listeners, but we haven't actually added the EC2 instances to the ALB. 

## Step 3 &mdash; Add the Instances to the ALB

Replace the EC2 creation block with the following:

```java
var instances = AwsFunctions.getAvailabilityZones().thenApply(response ->
        response.zoneIds().stream().map(availabilityZone -> {
            var ec2Instance = new Instance("web-server-%s".formatted(availabilityZone),
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
            );

            new TargetGroupAttachment("web-server-%s".formatted(availabilityZone),
                    TargetGroupAttachmentArgs.builder()
                            .targetGroupArn(targetGroup.arn())
                            .targetId(ec2Instance.privateIp())
                            .build()
            );

            return ec2Instance;
        }).toList()
);

var instancesOutput = Output.of(instances);
var ipAddresses = instancesOutput.apply(ec2s -> Output.all(ec2s.stream().map(Instance::publicIp).toList()));
var hostnames = instancesOutput.apply(ec2s -> Output.all(ec2s.stream().map(Instance::publicDns).toList()));

ctx.export("ips", ipAddresses);
ctx.export("hostnames", hostnames);
ctx.export("url", loadBalancer.dnsName());
```

> :white_check_mark: After this change, your `App.java` should look like this:

```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.AwsFunctions;
import com.pulumi.aws.alb.*;
import com.pulumi.aws.alb.inputs.ListenerDefaultActionArgs;
import com.pulumi.aws.ec2.*;
import com.pulumi.aws.ec2.inputs.*;
import com.pulumi.aws.ec2.outputs.GetAmiResult;
import com.pulumi.aws.ec2.outputs.GetSubnetIdsResult;
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
                    .egress(
                            SecurityGroupEgressArgs.builder()
                                    .protocol("tcp")
                                    .fromPort(80)
                                    .toPort(80)
                                    .cidrBlocks("0.0.0.0/0")
                                    .build())
                    .build());

            var vpcSubnetsIds = defaultVpcId
                    .thenCompose(vpcId -> Ec2Functions.getSubnetIds(GetSubnetIdsArgs.builder().vpcId(vpcId).build()))
                    .thenApply(GetSubnetIdsResult::ids);

            var loadBalancer = new LoadBalancer("loadbalancer",
                    LoadBalancerArgs.builder()
                            .internal(false)
                            .securityGroups(Output.all(securityGroup.getId()))
                            .subnets(Output.of(vpcSubnetsIds))
                            .loadBalancerType("application")
                            .build()
            );

            var targetGroup = new TargetGroup("target-group",
                    TargetGroupArgs.builder()
                            .port(80)
                            .protocol("HTTP")
                            .targetType("ip")
                            .vpcId(Output.of(defaultVpcId))
                            .build()
            );

            var albListener = new Listener("listener",
                    ListenerArgs.builder()
                            .loadBalancerArn(loadBalancer.arn())
                            .port(80)
                            .defaultActions(
                                    ListenerDefaultActionArgs.builder()
                                            .type("forward")
                                            .targetGroupArn(targetGroup.arn())
                                            .build())
                            .build()
            );

            var instances = AwsFunctions.getAvailabilityZones().thenApply(response ->
                    response.zoneIds().stream().map(availabilityZone -> {
                        var ec2Instance = new Instance("web-server-%s".formatted(availabilityZone),
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
                        );

                        new TargetGroupAttachment("web-server-%s".formatted(availabilityZone),
                                TargetGroupAttachmentArgs.builder()
                                        .targetGroupArn(targetGroup.arn())
                                        .targetId(ec2Instance.privateIp())
                                        .build()
                        );

                        return ec2Instance;
                    }).toList()
            );

            var instancesOutput = Output.of(instances);
            var ipAddresses = instancesOutput.apply(ec2s -> Output.all(ec2s.stream().map(Instance::publicIp).toList()));
            var hostnames = instancesOutput.apply(ec2s -> Output.all(ec2s.stream().map(Instance::publicDns).toList()));

            ctx.export("ips", ipAddresses);
            ctx.export("hostnames", hostnames);
            ctx.export("url", loadBalancer.dnsName());
        });
    }
}
```

This is all the infrastructure we need for our load balanced webserver. Let's apply it.

## Step 4 &mdash; Deploy your Changes

Deploy these updates:

```bash
pulumi up
```

This should result in a fairly large update and, if all goes well, the load balancer's resulting endpoint URL:

```
Updating (dev)

     Type                              Name                 Status      Info
     pulumi:pulumi:Stack               java-dev                         1 warning
 ~   ├─ aws:ec2:SecurityGroup          web-secgrp           updated     [diff: ~egress]
 +   ├─ aws:alb:TargetGroup            target-group         created     
 +   ├─ aws:alb:LoadBalancer           loadbalancer         created     
 +   ├─ aws:alb:TargetGroupAttachment  web-server-euw1-az2  created     
 +   ├─ aws:alb:TargetGroupAttachment  web-server-euw1-az1  created     
 +   ├─ aws:alb:TargetGroupAttachment  web-server-euw1-az3  created     
 +   └─ aws:alb:Listener               listener             created     
 
Outputs:
    hostnames: [
        [0]: "ec2-34-253-216-194.eu-west-1.compute.amazonaws.com"
        [1]: "ec2-18-203-162-39.eu-west-1.compute.amazonaws.com"
        [2]: "ec2-54-155-163-18.eu-west-1.compute.amazonaws.com"
    ]
    ips      : [
        [0]: "34.253.216.194"
        [1]: "18.203.162.39"
        [2]: "54.155.163.18"
    ]
  + url      : "loadbalancer-a1a71b8-1205493908.eu-west-1.elb.amazonaws.com"

Resources:
    + 6 created
    ~ 1 updated
    7 changes. 4 unchanged

Duration: 2m10s

```

## Step 5 &mdash; Verify 

Now we can curl the load balancer:

```bash
for i in {0..10}; do curl $(pulumi stack output url); done
```

Observe that the resulting text changes based on where the request is routed:

```
Hello, World -- from euw1-az1!
Hello, World -- from euw1-az2!
Hello, World -- from euw1-az1!
Hello, World -- from euw1-az1!
Hello, World -- from euw1-az3!
Hello, World -- from euw1-az3!
Hello, World -- from euw1-az2!
Hello, World -- from euw1-az1!
Hello, World -- from euw1-az2!
Hello, World -- from euw1-az1!
Hello, World -- from euw1-az3!
```

## Step 6 &mdash; Destroy Everything

Finally, destroy the resources and the stack itself:

```
pulumi destroy
pulumi stack rm
```
