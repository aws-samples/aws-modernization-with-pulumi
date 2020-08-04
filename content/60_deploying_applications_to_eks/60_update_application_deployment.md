+++
title = "Update Application Version"
chapter = false
weight = 60
+++

Next, you'll make two changes to the application:

* Scale out to 3 replicas, instead of just 1.
* Update the version of your application by changing its container image tag

Note that the application says `Hello Kubernetes bootcamp! | Running on: app-dep-9p399mj2-6c7cdd7d79-7w7vj | v=1`. After deploying, this will change.

First update your deployment's configuration's replica count:

```
  replicas=3,
```

And then update its image to:

```
  image="jocatalin/kubernetes-bootcamp:v2",
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript

```

Deploy the changes:

```bash
pulumi up
```

This will show you a preview and, after selecting `yes`, the application will be deployed:

```
Updating (dev):

     Type                             Name                                    Status
 +   pulumi:pulumi:Stack              eks-deployment-dev                      created
 ~   └─ kubernetes:apps:Deployment  app-dep             updated     [diff: ~spec]

Outputs:
    url: "http://ae7c37b7c510511eab4540a6f2211784-521581596.us-west-2.elb.amazonaws.com:80"

Resources:
    ~ 1 updated
    4 unchanged

Duration: 29s

Permalink: https://app.pulumi.com/workshops/eks-deloyment/dev/updates/2
```

Query the pods again using your chosen namespace from earlier:

```bash
kubectl get pods --namespace eks-workshop
```

Check that there are now three:

```
NAME                                READY   STATUS    RESTARTS   AGE
app-dep-9p399mj2-69c7fd4657-79wcp   1/1     Running   0          3m19s
app-dep-9p399mj2-69c7fd4657-dlx4p   1/1     Running   0          3m16s
app-dep-9p399mj2-69c7fd4657-mrvtt   1/1     Running   0          3m30s
```

Finally, curl the endpoint again:

```bash
curl $(pulumi stack output url)
```

And verify that the output now ends in `v=2`, instead of `v=1` (the result of the new container image):

```
Hello Kubernetes bootcamp! | Running on: app-dep-8r1febnu-6cd57d964-c76rx | v=2
```

If you'd like, do it a few more times, and observe that traffic will be load balanced across the three pods:

```bash
for i in {0..10}; do curl $(pulumi stack output url); done
```
