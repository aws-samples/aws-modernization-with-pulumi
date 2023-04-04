+++
title = "3.3 Deploy a Docker Image"
chapter = false
weight = 20
+++

## Step 1 &mdash; Create ECS FargateService

In order to create a Fargate service, we need to add an IAM Role and a Task Definition and Service. the ECS Cluster will run
the `"nginx"` image from the Docker Hub.

Let's define our IAM Role and attach a policy. You should define this at the end of your `App.java`:

```java
var role = new Role("task-exec-role",
        RoleArgs.builder()
                .assumeRolePolicy("""
                        {
                            "Version": "2008-10-17",
                            "Statement": [
                                {
                                    "Sid": "",
                                    "Effect": "Allow",
                                    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
                                    "Action": "sts:AssumeRole"
                                }
                            ]
                        }""")
                .build()
);

var policyAttachment = new RolePolicyAttachment("task-exec-policy",
        RolePolicyAttachmentArgs.builder()
                .role(role.name())
                .policyArn("arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy")
                .build()
);
```

Then we can define a task definition for our ECS service and the service itself:

```java
var taskDefinition = new TaskDefinition("app-task",
                    TaskDefinitionArgs.builder()
                            .family("fargate-task-definition")
                            .cpu("256")
                            .memory("512")
                            .networkMode("awsvpc")
                            .requiresCompatibilities("FARGATE")
                            .executionRoleArn(role.arn())
                            .containerDefinitions("""
                                    [
                                        {
                                            "name": "my-app",
                                            "image": "nginx",
                                            "portMappings": [{"containerPort": 80, "hostPort": 80, "protocol": "tcp"}]
                                        }
                                    ]""")
                            .build()
            );

var service = new Service("app-svc",
        ServiceArgs.builder()
                .cluster(cluster.arn())
                .desiredCount(1)
                .launchType("FARGATE")
                .taskDefinition(taskDefinition.arn())
                .networkConfiguration(
                        ServiceNetworkConfigurationArgs.builder()
                                .assignPublicIp(true)
                                .subnets(Output.of(vpcSubnetsIds))
                                .securityGroups(Output.all(securityGroup.getId()))
                                .build()
                )
                .loadBalancers(
                        ServiceLoadBalancerArgs.builder()
                                .targetGroupArn(targetGroup.arn())
                                .containerName("my-app")
                                .containerPort(80)
                                .build()
                )
                .build(),

        CustomResourceOptions.builder()
                .dependsOn(albListener)
                .build()
);


ctx.export("url", loadBalancer.dnsName());
```

> :white_check_mark: After these changes, your `App.java` should look like this

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
import com.pulumi.aws.ecs.*;
import com.pulumi.aws.ecs.inputs.ServiceLoadBalancerArgs;
import com.pulumi.aws.ecs.inputs.ServiceNetworkConfigurationArgs;
import com.pulumi.aws.iam.Role;
import com.pulumi.aws.iam.RoleArgs;
import com.pulumi.aws.iam.RolePolicyAttachment;
import com.pulumi.aws.iam.RolePolicyAttachmentArgs;
import com.pulumi.core.Output;
import com.pulumi.resources.CustomResourceOptions;

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

            var role = new Role("task-exec-role",
                    RoleArgs.builder()
                            .assumeRolePolicy("""
                                    {
                                        "Version": "2008-10-17",
                                        "Statement": [
                                            {
                                                "Sid": "",
                                                "Effect": "Allow",
                                                "Principal": {"Service": "ecs-tasks.amazonaws.com"},
                                                "Action": "sts:AssumeRole"
                                            }
                                        ]
                                    }""")
                            .build()
            );

            var policyAttachment = new RolePolicyAttachment("task-exec-policy",
                    RolePolicyAttachmentArgs.builder()
                            .role(role.name())
                            .policyArn("arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy")
                            .build()
            );

            var taskDefinition = new TaskDefinition("app-task",
                    TaskDefinitionArgs.builder()
                            .family("fargate-task-definition")
                            .cpu("256")
                            .memory("512")
                            .networkMode("awsvpc")
                            .requiresCompatibilities("FARGATE")
                            .executionRoleArn(role.arn())
                            .containerDefinitions("""
                                    [
                                        {
                                            "name": "my-app",
                                            "image": "nginx",
                                            "portMappings": [{"containerPort": 80, "hostPort": 80, "protocol": "tcp"}]
                                        }
                                    ]""")
                            .build()
            );

            var service = new Service("app-svc",
                    ServiceArgs.builder()
                            .cluster(cluster.arn())
                            .desiredCount(1)
                            .launchType("FARGATE")
                            .taskDefinition(taskDefinition.arn())
                            .networkConfiguration(
                                    ServiceNetworkConfigurationArgs.builder()
                                            .assignPublicIp(true)
                                            .subnets(Output.of(vpcSubnetsIds))
                                            .securityGroups(Output.all(securityGroup.getId()))
                                            .build()
                            )
                            .loadBalancers(
                                    ServiceLoadBalancerArgs.builder()
                                            .targetGroupArn(targetGroup.arn())
                                            .containerName("my-app")
                                            .containerPort(80)
                                            .build()
                            )
                            .build(),

                    CustomResourceOptions.builder()
                            .dependsOn(albListener)
                            .build()
            );


            ctx.export("url", loadBalancer.dnsName());
        });
    }
}
```

## Step 2 &mdash; Provision the Cluster and Service

Deploy the program to stand up your initial cluster and service:

```bash
pulumi up
```

This will output the status and resulting load balancer URL:

```
Updating (dev)

     Type                             Name              Status
     pulumi:pulumi:Stack              java-dev                 
 +   ├─ aws:iam:Role                  task-exec-role    created     
 +   ├─ aws:ecs:TaskDefinition        app-task          created     
 +   ├─ aws:iam:RolePolicyAttachment  task-exec-policy  created     
 +   └─ aws:ecs:Service               app-svc           created     
 
Outputs:
  + url: "app-lb-598a127-1152695482.eu-west-1.elb.amazonaws.com"

Resources:
    + 4 created
    6 unchanged

Duration: 9s

```

You can now curl the resulting endpoint:

```bash
curl $(pulumi stack output url)
```

And you'll see the Nginx default homepage:

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

## Step 3 &mdash; Update the Service

Now, let's also update the desired container count from `1` to `3`.

To do this, we need to adjust the ServiceArgsBuilder

```java
...
    ServiceArgs.builder()
        .desiredCount(3)
...
```

Next update the stack:

```bash
pulumi up
```

The output should look something like this:

```
Updating (dev)

     Type                 Name      Status      Info
     pulumi:pulumi:Stack  java-dev              
 ~   └─ aws:ecs:Service   app-svc   updated     [diff: ~desiredCount]
 
Outputs:
    url: "app-lb-598a127-1152695482.eu-west-1.elb.amazonaws.com"

Resources:
    ~ 1 updated
    9 unchanged

Duration: 6s

```

## Step 4 &mdash; Destroy Everything

Finally, destroy the resources and the stack itself:

```
pulumi destroy
pulumi stack rm
```

