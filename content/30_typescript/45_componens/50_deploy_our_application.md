+++
title = "4.4 Deploying our Application Stack"
chapter = false
weight = 50
+++

We are ready to deploy Everything:

```bash
pulumi up
```

This will show you a preview and, after selecting `yes`, the application will be deployed:

```
Updating (dev2):
     Type                                Name           Status      
 +   pulumi:pulumi:Stack                 eks-apps-dev2  created     
 +   ├─ my:kubernetes:ServiceDeployment  eks-demo-app   created     
 +   │  ├─ kubernetes:core:Service       app-svc        created     
 +   │  └─ kubernetes:apps:Deployment    eks-demos-app  created     
 +   ├─ pulumi:providers:kubernetes      k8sProvider    created     
 +   └─ kubernetes:core:Namespace        app-ns         created     
 
Outputs:
    url: "http://a8bf84659dc8f4dd1b266e9711de3c2c-1478981199.us-west-2.elb.amazonaws.com:3000"

Resources:
    + 6 created

Duration: 15s

Permalink: https://app.pulumi.com/workshops/eks-apps/dev2/updates/1
```

It will take a few seconds for the Load Balancer to be ready, but once it is, you can curl the `url` to see the same application running again:

```bash
$ curl $(pulumi stack output url)
Hello Kubernetes bootcamp! | Running on: eks-demos-app-07it3okr-7df9cddf49-974xn | v=2
```

We now have a reusable component we can use to deploy any Docker image we want in just a couple lines of code!