+++
title = "Declare an Application Service Object"
chapter = false
weight = 40
+++

Next, you'll declare a [service object](https://kubernetes.io/docs/concepts/services-networking/service/), which enables networking and load balancing across your deployment replicas.

Append this to your `index.ts` file:

```typescript
const service = new k8s.core.v1.Service("app-svc", {
    metadata: { namespace: ns.metadata.name },
    spec: {
        selector: appLabels,
        ports: [{ port: 80, targetPort: 8080 }],
        type: "LoadBalancer",
    },
}, {provider});
```

Afterwards, add these lines to export the resulting, dynamically assigned endpoint for the resulting load balancer:

```typescript
const address = service.status.loadBalancer.ingress[0].hostname;
const port = service.spec.ports[0].port;
export const url = pulumi.interpolate`http://${address}:${port}`;
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

const pulumiConfig = new pulumi.Config();

// Existing Pulumi stack reference in the format:
// <organization>/<project>/<stack> e.g. "myUser/myProject/dev"
const clusterStackRef = new pulumi.StackReference(pulumiConfig.require("clusterStackRef"));

// Get the kubeconfig from the cluster stack output.
const kubeconfig = clusterStackRef.getOutput("kubeconfig").apply(JSON.stringify);

// Create the k8s provider with the kubeconfig.
const provider = new k8s.Provider("k8sProvider", {kubeconfig});

const ns = new k8s.core.v1.Namespace("app-ns", {
    metadata: { name: "eks-workshop" },
}, {provider});

const appLabels = { app: "eks-workshop" };
const deployment = new k8s.apps.v1.Deployment("app-dep", {
    metadata: { namespace: ns.metadata.name },
    spec: {
        selector: { matchLabels: appLabels },
        replicas: 1,
        template: {
            metadata: { labels: appLabels },
            spec: {
                containers: [{
                    name: "eks-workshop",
                    image: "gcr.io/google-samples/kubernetes-bootcamp:v1",
                }],
            },
        },
    },
}, {provider});

const service = new k8s.core.v1.Service("app-svc", {
    metadata: { namespace: ns.metadata.name },
    spec: {
        selector: appLabels,
        ports: [{ port: 80, targetPort: 8080 }],
        type: "LoadBalancer",
    },
}, {provider});

const address = service.status.loadBalancer.ingress[0].hostname;
const port = service.spec.ports[0].port;
export const url = pulumi.interpolate`http://${address}:${port}`;
```
