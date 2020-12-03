+++
title = "2.3 API Gateway"
chapter = false
weight = 20
+++

Now that you have a project configured to use AWS, you'll create some basic infrastructure in it. Let's use AWS crosswalk to define a lambda function.

## Step 1 &mdash; Declare a New Lambda Function

Add the following to your `index.ts` file:

```typescript
const api = new awsx.apigateway.API("hello-world", {
    routes: [{
        path: "/",
        method: "GET",
        eventHandler: async (event) => {
            return {
                statusCode: 200,
                body: "Hello, world!",
            };
        },
    }],
})
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

const api = new awsx.apigateway.API("hello-world", {
    routes: [{
        path: "/",
        method: "GET",
        eventHandler: async (event) => {
            return {
                statusCode: 200,
                body: "Hello, world!",
            };
        },
    }],
})
```

## Step 2 &mdash; Preview Your Changes

Now preview your changes:

```
pulumi up
```

This command evaluates your program, determines the resource updates to make, and shows you an outline of these changes:

```
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/api-gateway/dev/previews/01278a6e-b06c-4301-bff5-ba145f69b68d

     Type                                Name                          Plan
 +   pulumi:pulumi:Stack                 api-gateway-dev               create
 +   └─ aws:apigateway:x:API             hello-world                   create
 +      ├─ aws:iam:Role                  hello-world4c238266           create
 +      ├─ aws:lambda:Function           hello-world4c238266           create
 +      ├─ aws:iam:RolePolicyAttachment  hello-world4c238266-32be53a2  create
 +      ├─ aws:apigateway:RestApi        hello-world                   create
 +      ├─ aws:apigateway:Deployment     hello-world                   create
 +      ├─ aws:lambda:Permission         hello-world-fa520765          create
 +      └─ aws:apigateway:Stage          hello-world                   create
```

This is a summary view. Select `details` to view the full set of properties:

```
+ pulumi:pulumi:Stack: (create)
    [urn=urn:pulumi:dev::api-gateway::pulumi:pulumi:Stack::api-gateway-dev]
    + aws:apigateway:x:API: (create)
        [urn=urn:pulumi:dev::api-gateway::aws:apigateway:x:API::hello-world]
        + aws:iam/role:Role: (create)
            [urn=urn:pulumi:dev::api-gateway::aws:apigateway:x:API$aws:iam/role:Role::hello-world4c238266]
            [provider=urn:pulumi:dev::api-gateway::pulumi:providers:aws::default_3_17_0::04da6b54-80e4-46f7-96ec-b56ff0331ba9]
            assumeRolePolicy   : "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Action\":\"sts:AssumeRole\",\"Principal\":{\"Service\":\"lambda.amazonaws.com\"},\"Effect\":\"Allow\",\"Sid\":\"\"}]}"
            forceDetachPolicies: false
            maxSessionDuration : 3600
            name               : "hello-world4c238266-7ca9af6"
            path               : "/"
...
```

The stack resource is a synthetic resource that all resources your program creates are parented to.

You'll notice here that despite only defining a single resource, the Pulumi program has many resources inside it. This is because we're using [Pulumi Crosswalk](https://www.pulumi.com/docs/guides/crosswalk/aws/) to define our function, which is an abstraction around the basic resources. It defines best practices for you out of the box.

## Step 3 &mdash; Deploy Your Changes

Now that we've seen the full set of changes, let's deploy them. Select `yes`:

```
Updating (dev)

View Live: https://app.pulumi.com/jaxxstorm/api-gateway/dev/updates/1

     Type                                Name                          Status
 +   pulumi:pulumi:Stack                 api-gateway-dev               created
 +   └─ aws:apigateway:x:API             hello-world                   created
 +      ├─ aws:iam:Role                  hello-world4c238266           created
 +      ├─ aws:lambda:Function           hello-world4c238266           created
 +      ├─ aws:iam:RolePolicyAttachment  hello-world4c238266-32be53a2  created
 +      ├─ aws:apigateway:RestApi        hello-world                   created
 +      ├─ aws:apigateway:Deployment     hello-world                   created
 +      ├─ aws:lambda:Permission         hello-world-fa520765          created
 +      └─ aws:apigateway:Stage          hello-world                   created

```

Now our lambda function has been created in our AWS account. Feel free to click the Permalink URL and explore; this will take you to the [Pulumi Console](https://www.pulumi.com/docs/intro/console/), which records your deployment history.

## Step 4 &mdash; Export Your API Gateway URL

To inspect your new hello world function, you will need the url from the created API gateway. Pulumi records a logical name, `hello-world` for the resources in the stack, however the resulting AWS API gateway URL is generated on the server side by AWS..

Programs can export variables which will be shown in the CLI and recorded for each deployment. Export your API gateway's URL name by adding this line to `index.ts`:

```typescript
export const url = api.url;
```
> The difference between logical and physical names is in part due to "auto-naming" which Pulumi does to ensure side-by-side projects and zero-downtime upgrades work seamlessly. 
>It can be disabled if you wish; [read more about auto-naming here](https://www.pulumi.com/docs/intro/concepts/programming-model/#autonaming).

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as awsx from "@pulumi/awsx";

// Define a new GET endpoint that just returns a 200 and "hello" in the body.
const api = new awsx.apigateway.API("hello-world", {
    routes: [{
        path: "/",
        method: "GET",
        eventHandler: async (event) => {
            // This code runs in an AWS Lambda anytime `/` is hit.
            return {
                statusCode: 200,
                body: "Hello, world!",
            };
        },
    }],
})

// Export the auto-generated API Gateway base URL.
export const url = api.url;
```

Now deploy the changes:

```bash
pulumi up --yes
```

Notice a new `Outputs` section is included in the output containing the API Gateway URL:

```
Outputs:
  + url: "https://8i41k0ifjd.execute-api.us-east-1.amazonaws.com/stage/"
```

## Step 5 &mdash; Inspect Your API Gateway contents

We can get a response from our lambda function using `curl`:


```
curl $(pulumi stack output url)
```
