+++
title = "1.3 Provisioning a Lambda Function"
chapter = false
weight = 20
+++

Now that you have a project configured to use AWS, you'll create some basic infrastructure in it. Let's create a simple lambda function.

## Step 1 &mdash; Create a Lambda Execution Role

Add a role to your Pulumi project so that your Lambda function can execute.

```typescript
const role = new aws.iam.Role('my-function-role', {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({
        Service: "lambda.amazonaws.com"
    })
})
```
{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const role = new aws.iam.Role('my-function-role', {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({
        Service: "lambda.amazonaws.com"
    })
})
```

## Step 2 &mdash; Declare a New Lambda Function

We've defined an execution role, let's define a Lambda function that uses this execution role.

Add the following to your `index.ts` file:

```typescript
const lambdaFunction = new aws.lambda.Function('my-function', {
    role: role.arn,
    handler: "index.handler",
    runtime: aws.lambda.NodeJS12dXRuntime,
    code: new pulumi.asset.AssetArchive({
        "index.js": new pulumi.asset.StringAsset(
            "exports.handler = (e, c, cb) => cb(null, {statusCode: 200, body: 'Hello, world!'});",
        ),
    }),
})
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const role = new aws.iam.Role('my-function-role', {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({
        Service: "lambda.amazonaws.com"
    })
})

const lambdaFunction = new aws.lambda.Function('my-function', {
    role: role.arn,
    handler: "index.handler",
    runtime: aws.lambda.NodeJS12dXRuntime,
    code: new pulumi.asset.AssetArchive({
        "index.js": new pulumi.asset.StringAsset(
            "exports.handler = (e, c, cb) => cb(null, {statusCode: 200, body: 'Hello, world!'});",
        ),
    }),
})
```

## Step 3 &mdash; Preview Your Changes

Now preview your changes:

```
pulumi up
```

This command evaluates your program, determines the resource updates to make, and shows you an outline of these changes:

```
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/hello-world/dev/previews/855bb68e-7f45-4740-a7c5-e1acb10aeb04

     Type                    Name              Plan
 +   pulumi:pulumi:Stack     hello-world-dev   create
 +   ├─ aws:iam:Role         my-function-role  create
 +   └─ aws:lambda:Function  my-function       create

Resources:
    + 3 to create
```

This is a summary view. Select `details` to view the full set of properties:

```
Do you want to perform this update? details
+ pulumi:pulumi:Stack: (create)
    [urn=urn:pulumi:dev::hello-world::pulumi:pulumi:Stack::hello-world-dev]
    + aws:iam/role:Role: (create)
        [urn=urn:pulumi:dev::hello-world::aws:iam/role:Role::my-function-role]
        [provider=urn:pulumi:dev::hello-world::pulumi:providers:aws::default_3_16_0::04da6b54-80e4-46f7-96ec-b56ff0331ba9]
        assumeRolePolicy   : "{\"Statement\":[{\"Action\":\"sts:AssumeRole\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"lambda.amazonaws.com\"},\"Sid\":\"AllowAssumeRole\"}],\"Version\":\"2012-10-17\"}"
        forceDetachPolicies: false
        maxSessionDuration : 3600
        name               : "my-function-role-13ec370"
        path               : "/"
    + aws:lambda/function:Function: (create)
        [urn=urn:pulumi:dev::hello-world::aws:lambda/function:Function::my-function]
        [provider=urn:pulumi:dev::hello-world::pulumi:providers:aws::default_3_16_0::04da6b54-80e4-46f7-96ec-b56ff0331ba9]
        code                        : archive(assets:b898ebe) {
            "index.js": asset(text:fd60d66) {
                <stripped>
            }
        }
        handler                     : "index.handler"
        memorySize                  : 128
        name                        : "my-function-142ac16"
        publish                     : false
        reservedConcurrentExecutions: -1
        role                        : output<string>
        runtime                     : "nodejs12.x"
        timeout                     : 3
```

The stack resource is a synthetic resource that all resources your program creates are parented to.

## Step 4 &mdash; Deploy Your Changes

Now that we've seen the full set of changes, let's deploy them. Select `yes`:

```
Updating (dev)

View Live: https://app.pulumi.com/jaxxstorm/hello-world/dev/updates/4

     Type                    Name              Status
 +   pulumi:pulumi:Stack     hello-world-dev   created
 +   ├─ aws:iam:Role         my-function-role  created
 +   └─ aws:lambda:Function  my-function       created

Resources:
    + 3 created

Duration: 27s
```

Now our lambda function has been created in our AWS account. Feel free to click the Permalink URL and explore; this will take you to the [Pulumi Console](https://www.pulumi.com/docs/intro/console/), which records your deployment history.

## Step 4 &mdash; Export Your Function Name

To inspect your new Lambda function, you will need its name. Pulumi records a logical name, `my-function`, however the resulting AWS lambda function resource in AWS will be autogenerated.

Programs can export variables which will be shown in the CLI and recorded for each deployment. Export your lambda function's name by adding this line to `index.ts`:

```typescript
export const functionName = lambdaFunction.name
```

> The difference between logical and physical names is in part due to "auto-naming" which Pulumi does to ensure side-by-side projects and zero-downtime upgrades work seamlessly. 
>It can be disabled if you wish; [read more about auto-naming here](https://www.pulumi.com/docs/intro/concepts/programming-model/#autonaming).


{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const role = new aws.iam.Role('my-function-role', {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({
        Service: "lambda.amazonaws.com"
    })
})

const lambdaFunction = new aws.lambda.Function('my-function', {
    role: role.arn,
    handler: "index.handler",
    runtime: aws.lambda.NodeJS12dXRuntime,
    code: new pulumi.asset.AssetArchive({
        "index.js": new pulumi.asset.StringAsset(
            "exports.handler = (e, c, cb) => cb(null, {statusCode: 200, body: 'Hello, world!'});",
        ),
    }),
})

export const functionName = lambdaFunction.name
```

Now deploy the changes:

```bash
pulumi up --yes
```

Notice a new `Outputs` section is included in the output containing the lambda function's name:

```
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/hello-world/dev/previews/ae4d29f1-e2f2-4ee4-89a2-f7e769ecd63e

     Type                 Name             Plan
     pulumi:pulumi:Stack  hello-world-dev

Outputs:
  + functionName: "my-function-d60edb4"

Resources:
    3 unchanged

Updating (dev)

View Live: https://app.pulumi.com/jaxxstorm/hello-world/dev/updates/5

     Type                 Name             Status
     pulumi:pulumi:Stack  hello-world-dev

Outputs:
  + functionName: "my-function-d60edb4"

Resources:
    3 unchanged

Duration: 2s
```

## Step 5 &mdash; Trigger your function

We can now trigger out function using the AWS CLI

```
aws lambda invoke --function-name $(pulumi stack output functionName) /tmp/out
```

This should tell us that the function executed successfully. We could go and look at the cloudwatch log group, but there's an easier way we'll be able to see in the next step.

## Step 6 &mdash; Destroy Everything

Finally, destroy the resources and the stack itself:

```
pulumi destroy
pulumi stack rm
```
