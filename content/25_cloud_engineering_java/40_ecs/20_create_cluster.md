+++
title = "3.2 Create an ECS Cluster & LoadBalancer"
chapter = false
weight = 10
+++

## Step 1 &mdash; Create an ECS Cluster

Remove all the boilerplate from the project bootstrap.

As a first step, we're going to create an ECS cluster:
```java
var cluster = new Cluster("cluster");
```

> :white_check_mark: After this change, your `App.java` should look like this:

```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.ecs.Cluster;

public class App {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var cluster = new Cluster("cluster");
        });
    }
}
```

## Step 2 &mdash; Create a Load-Balanced Container Service

Next, allocate the application load balancer (ALB) and listen for HTTP traffic port 80. In order to do this, we need to find the
default VPC and the subnet groups for it:

```java
var defaultVpcId = Ec2Functions
        .getVpc(GetVpcArgs.builder().default_(true).build())
        .thenApply(GetVpcResult::id);

var vpcSubnetsIds = defaultVpcId
        .thenCompose(vpcId -> Ec2Functions.getSubnetIds(GetSubnetIdsArgs.builder().vpcId(vpcId).build()))
        .thenApply(GetSubnetIdsResult::ids);

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
                        .protocol("-1")
                        .fromPort(0)
                        .toPort(0)
                        .cidrBlocks("0.0.0.0/0")
                        .build()
        )
        .build());

var loadBalancer = new LoadBalancer("app-lb",
        LoadBalancerArgs.builder()
                .internal(false)
                .securityGroups(Output.all(securityGroup.getId()))
                .subnets(Output.of(vpcSubnetsIds))
                .loadBalancerType("application")
                .build()
);

var targetGroup = new TargetGroup("app-tg",
        TargetGroupArgs.builder()
                .port(80)
                .protocol("HTTP")
                .targetType("ip")
                .vpcId(Output.of(defaultVpcId))
                .build()
);

var albListener = new Listener("web",
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
> :white_check_mark: After this change, your `App.java` should look like this:

```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.alb.*;
import com.pulumi.aws.alb.inputs.ListenerDefaultActionArgs;
import com.pulumi.aws.ec2.Ec2Functions;
import com.pulumi.aws.ec2.SecurityGroup;
import com.pulumi.aws.ec2.SecurityGroupArgs;
import com.pulumi.aws.ec2.inputs.GetSubnetIdsArgs;
import com.pulumi.aws.ec2.inputs.GetVpcArgs;
import com.pulumi.aws.ec2.inputs.SecurityGroupEgressArgs;
import com.pulumi.aws.ec2.inputs.SecurityGroupIngressArgs;
import com.pulumi.aws.ec2.outputs.GetSubnetIdsResult;
import com.pulumi.aws.ec2.outputs.GetVpcResult;
import com.pulumi.aws.ecs.Cluster;
import com.pulumi.core.Output;

public class App {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var cluster = new Cluster("cluster");

            var defaultVpcId = Ec2Functions
                    .getVpc(GetVpcArgs.builder().default_(true).build())
                    .thenApply(GetVpcResult::id);

            var vpcSubnetsIds = defaultVpcId
                    .thenCompose(vpcId -> Ec2Functions.getSubnetIds(GetSubnetIdsArgs.builder().vpcId(vpcId).build()))
                    .thenApply(GetSubnetIdsResult::ids);

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
                                    .protocol("-1")
                                    .fromPort(0)
                                    .toPort(0)
                                    .cidrBlocks("0.0.0.0/0")
                                    .build()
                    )
                    .build());

            var loadBalancer = new LoadBalancer("app-lb",
                    LoadBalancerArgs.builder()
                            .internal(false)
                            .securityGroups(Output.all(securityGroup.getId()))
                            .subnets(Output.of(vpcSubnetsIds))
                            .loadBalancerType("application")
                            .build()
            );

            var targetGroup = new TargetGroup("app-tg",
                    TargetGroupArgs.builder()
                            .port(80)
                            .protocol("HTTP")
                            .targetType("ip")
                            .vpcId(Output.of(defaultVpcId))
                            .build()
            );

            var albListener = new Listener("web",
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
        });
    }
}
```

Run your program with `pulumi up`:

```
Updating (dev)

     Type                      Name        Status
 +   pulumi:pulumi:Stack       java-dev    created
 +   ├─ aws:ecs:Cluster        cluster     created     
 +   ├─ aws:ec2:SecurityGroup  web-secgrp  created     
 +   ├─ aws:alb:TargetGroup    app-tg      created     
 +   ├─ aws:alb:LoadBalancer   app-lb      created     
 +   └─ aws:alb:Listener       web         created     
 
Resources:
    + 6 created

Duration: 2m13s

```

We've fleshed out our infrastructure and added a LoadBalancer we can add infrastructure to, in the next steps we'll run a container.
