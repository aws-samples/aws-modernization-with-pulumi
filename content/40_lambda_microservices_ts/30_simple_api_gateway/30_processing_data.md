+++
title = "2.4 Processing Input Data"
chapter = false
weight = 30
+++

We just saw how Pulumi Crosswalk allows you to quickly define serverless API endpoints and created a simple get request.

Now, let's take things a little further.

## Step 1 &mdash; Add a new API Gateway Route

Create a new route in your API Gateway that accepts post requests. The route should like this this:

```typescript
{
    path: "encode",
    method: "POST",
    eventHandler: async (event) => {
        console.log("request: " + JSON.stringify(event));
        let body: string
        let responseCode: number

        if (event.body != null) {
            if (event.isBase64Encoded) {
                body = event.body // API gateway will base64 encode for us
                responseCode = 200
            } else {
                body = Buffer.from(event.body).toString('base64')
                responseCode = 200
            }
        } else {
            body = "No data submitted"
            responseCode = 400
        }

        return {
            statusCode: responseCode,
            body: body,
        }
    },
}
```

This new `echo` endpoint simply takes some HTTP body data and base64 encodes it for us. Let's run this program and add the new route to our API gateway.


{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
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
    }, {
        path: "encode",
        method: "POST",
        eventHandler: async (event) => {
            console.log("request: " + JSON.stringify(event));
            let body: string
            let responseCode: number

            if (event.body != null) {
                if (event.isBase64Encoded) {
                    body = event.body // API gateway will base64 encode for us
                    responseCode = 200
                } else {
                    body = Buffer.from(event.body).toString('base64')
                    responseCode = 200
                }
            } else {
                body = "No data submitted"
                responseCode = 400
            }

            return {
                statusCode: responseCode,
                body: body,
            }
        },
    }],
})

export const url = api.url;

```

Deploy the changes:

```bash
pulumi up
```

This will give you a preview and selecting `yes` will apply the changes:

```
Previewing update (dev)

View Live: https://app.pulumi.com/jaxxstorm/api-gateway/dev/previews/657bac12-9b9e-4ca8-861e-a014b8e69f79

     Type                                Name                          Plan        Info
     pulumi:pulumi:Stack                 api-gateway-dev
     └─ aws:apigateway:x:API             hello-world
 +      ├─ aws:iam:Role                  hello-world02cbc3ee           create
 +      ├─ aws:iam:RolePolicyAttachment  hello-world02cbc3ee-32be53a2  create
 +      ├─ aws:lambda:Function           hello-world02cbc3ee           create
 ~      ├─ aws:apigateway:RestApi        hello-world                   update      [diff: ~body]
 +-     ├─ aws:apigateway:Deployment     hello-world                   replace     [diff: ~variables]
 +-     ├─ aws:lambda:Permission         hello-world-fa520765          replace     [diff: ~sourceArn]
 +      ├─ aws:lambda:Permission         hello-world-9451c357          create
 ~      └─ aws:apigateway:Stage          hello-world                   update      [diff: ~deployment]

Outputs:
  ~ url: "https://14oplegt8g.execute-api.us-east-1.amazonaws.com/stage/" => output<string>

Resources:
    + 4 to create
    ~ 2 to update
    +-2 to replace
    8 changes. 5 unchanged
```

## Step 2 &mdash; Send some data to the echo endpoint

We can post some data to our base64 encoding service using `cURL`. First, let's send a simple string:

```bash
curl -X POST $(pulumi stack output url)/encode -d foo
```

{{% notice info %}}
The API gateway may take a few moments to provision the endpoint
{{% /notice %}}

This should return a small base64 encoded string:

```
Zm9v
```

We can also send some JSON data:

```bash
curl -X POST $(pulumi stack output url)/encode -d '{"foo": "bar"}'
```

## Step 3 &mdash; View the function logs

A nice feature of Pulumi when developing serverless applications is the ability to quickly view the logs from the function within the Pulumi application. Run `pulumi logs` from without your stack:

```
pulumi logs
```

You'll get the lambda function output:

```
 2020-12-01T22:15:00.668-08:00[   hello-world02cbc3ee-f622aef] START RequestId: 4d57a569-cb00-494f-8613-875d6038a979 Version: $LATEST
 2020-12-01T22:15:00.670-08:00[   hello-world02cbc3ee-f622aef] 2020-12-02T06:15:00.670Z	4d57a569-cb00-494f-8613-875d6038a979	INFO	request: {"resource":"/encode","path":"/encode","httpMethod":"POST","headers":{"Accept":"*/*","CloudFront-Forwarded-Proto":"https","CloudFront-Is-Desktop-Viewer":"true","CloudFront-Is-Mobile-Viewer":"false","CloudFront-Is-SmartTV-Viewer":"false","CloudFront-Is-Tablet-Viewer":"false","CloudFront-Viewer-Country":"US","content-type":"application/x-www-form-urlencoded","Host":"8i41k0ifjd.execute-api.us-east-1.amazonaws.com","User-Agent":"curl/7.64.1","Via":"2.0 3dcf7c8001b07734617b28e9bacc90ad.cloudfront.net (CloudFront)","X-Amz-Cf-Id":"Xmm43gQdDAsIGfQrrePMTkNfakJild8CwLi5Ueq0CArYFWwlqB42hw==","X-Amzn-Trace-Id":"Root=1-5fc730e4-5a1e0d750d6f40c80527173e","X-Forwarded-For":"24.56.241.187, 130.176.132.123","X-Forwarded-Port":"443","X-Forwarded-Proto":"https"},"multiValueHeaders":{"Accept":["*/*"],"CloudFront-Forwarded-Proto":["https"],"CloudFront-Is-Desktop-Viewer":["true"],"CloudFront-Is-Mobile-Viewer":["false"],"CloudFront-Is-SmartTV-Viewer":["false"],"CloudFront-Is-Tablet-Viewer":["false"],"CloudFront-Viewer-Country":["US"],"content-type":["application/x-www-form-urlencoded"],"Host":["8i41k0ifjd.execute-api.us-east-1.amazonaws.com"],"User-Agent":["curl/7.64.1"],"Via":["2.0 3dcf7c8001b07734617b28e9bacc90ad.cloudfront.net (CloudFront)"],"X-Amz-Cf-Id":["Xmm43gQdDAsIGfQrrePMTkNfakJild8CwLi5Ueq0CArYFWwlqB42hw=="],"X-Amzn-Trace-Id":["Root=1-5fc730e4-5a1e0d750d6f40c80527173e"],"X-Forwarded-For":["24.56.241.187, 130.176.132.123"],"X-Forwarded-Port":["443"],"X-Forwarded-Proto":["https"]},"queryStringParameters":null,"multiValueQueryStringParameters":null,"pathParameters":null,"stageVariables":null,"requestContext":{"resourceId":"10gsf9","resourcePath":"/encode","httpMethod":"POST","extendedRequestId":"W6STvFSJoAMFeYA=","requestTime":"02/Dec/2020:06:15:00 +0000","path":"/stage//encode","accountId":"616138583583","protocol":"HTTP/1.1","stage":"stage","domainPrefix":"8i41k0ifjd","requestTimeEpoch":1606889700655,"requestId":"e62303e4-684b-4f96-a319-f143cdaa07aa","identity":{"cognitoIdentityPoolId":null,"accountId":null,"cognitoIdentityId":null,"caller":null,"sourceIp":"24.56.241.187","principalOrgId":null,"accessKey":null,"cognitoAuthenticationType":null,"cognitoAuthenticationProvider":null,"userArn":null,"userAgent":"curl/7.64.1","user":null},"domainName":"8i41k0ifjd.execute-api.us-east-1.amazonaws.com","apiId":"8i41k0ifjd"},"body":"eyJmb28iOiAiYmFyIn0=","isBase64Encoded":true}
 2020-12-01T22:15:00.671-08:00[   hello-world02cbc3ee-f622aef] END RequestId: 4d57a569-cb00-494f-8613-875d6038a979
 2020-12-01T22:15:00.671-08:00[   hello-world02cbc3ee-f622aef] REPORT RequestId: 4d57a569-cb00-494f-8613-875d6038a979	Duration: 1.24 ms	Billed Duration: 2 ms	Memory Size: 128 MB	Max Memory Used: 66 MB
```

This is being returned from the `console.log` statement in our function. 

## Step 4 &mdash; Destroy Everything

Finally, destroy the resources and the stack itself:

```
pulumi destroy
pulumi stack rm
```
