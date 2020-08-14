+++
title = "4.3 Use our Component"
chapter = false
weight = 20
+++

Next, we'll use our new `ServiceDeployment` component.

Start by deleting all of the code in `index.ts` after the namespace so that you left with just:

```ts
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";
import { ServiceDeployment } from "./servicedeployment";

const config = new pulumi.Config();
const clusterStackRef = new pulumi.StackReference(config.require("clusterStackRef"));

const kubeconfig = clusterStackRef.getOutput("kubeconfig").apply(JSON.stringify);
const provider = new k8s.Provider("k8sProvider", { kubeconfig });

const ns = new k8s.core.v1.Namespace("app-ns", {
    metadata: { name: "eks-demo-apps"},
}, { provider });
```

Then add a new `import` at the top of the file:

```ts
import { ServiceDeployment } from "./servicedeployment";
```

This imports the component we just defined.  Now we'll create an instance of this component that matches what we had previously.

Add the following at the end of the file:

```ts
const bootcamp = new ServiceDeployment("eks-demo-app", {
    image: "jocatalin/kubernetes-bootcamp:v2",
    port: { port:3000, targetPort: 8080 },
    namespace: ns.metadata.name,
}, { provider});

export const url = bootcamp.url;
```

{{% notice info %}}
The `index.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";
import { ServiceDeployment } from "./servicedeployment";

const config = new pulumi.Config();
const clusterStackRef = new pulumi.StackReference(config.require("clusterStackRef"));

const kubeconfig = clusterStackRef.getOutput("kubeconfig").apply(JSON.stringify);
const provider = new k8s.Provider("k8sProvider", { kubeconfig });

const ns = new k8s.core.v1.Namespace("app-ns", {
    metadata: { name: "eks-demo-apps"},
}, { provider });

const bootcamp = new ServiceDeployment("eks-demo-app", {
    image: "jocatalin/kubernetes-bootcamp:v2",
    port: { port:3000, targetPort: 8080 },
    namespace: ns.metadata.name,
}, { provider});

export const url = bootcamp.url;
```
