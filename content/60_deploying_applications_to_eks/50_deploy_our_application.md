+++
title = "Deploying our Application Stack"
chapter = false
weight = 50
+++

First, add the `StackReference` to the cluster stack, which is used to get the kubeconfig
from its stack output. This is a reference to the project created in the [previous lab]().

```bash
pulumi config set eks-deployment:clusterStackRef workshops/eks-workshop/dev
```

Deploy Everything:

```bash
pulumi up
```

This will show you a preview and, after selecting `yes`, the application will be deployed:

```
Updating (dev):

     Type                             Name                                    Status
 +   pulumi:pulumi:Stack              eks-deployment-dev                      created
 +   ├─ pulumi:providers:kubernetes   k8s-provider                            created
 +   ├─ kubernetes:core:Namespace     app-ns                                  created
 +   ├─ kubernetes:core:Service       app-service                             created
 +   └─ kubernetes:apps:Deployment    app-dep                                 created

Outputs:
    url: "http://ae7c37b7c510511eab4540a6f2211784-521581596.us-west-2.elb.amazonaws.com:80"

Resources:
    + 5 created

Duration: 32s

Permalink: https://app.pulumi.com/workshops/eks-deloyment/dev/updates/1
```

List the pods in your namespace, again replacing `eks-workshop` with the namespace you chose earlier:

```bash
kubectl get pods --namespace eks-workshop
```

And you should see a single replica:

```
NAME                                READY   STATUS    RESTARTS   AGE
app-dep-9p399mj2-6c7cdd7d79-7w7vj   1/1     Running   0          0m15s
```
