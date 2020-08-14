+++
title = "4.2 Create a ComponentResource"
chapter = false
weight = 10
+++

To create our component, first create a new file `servicedeployment.ts`, with the following contents:

```ts
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

export interface ServiceDeploymentArgs {
    namespace?: pulumi.Input<string>;
    replicas?: pulumi.Input<number>;
    image: pulumi.Input<string>;
    port: k8s.types.input.core.v1.ServicePort;
}

export class ServiceDeployment extends pulumi.ComponentResource {
    deployment: k8s.apps.v1.Deployment;
    service: k8s.core.v1.Service;
    url: pulumi.Output<string>;

    constructor(name: string, args: ServiceDeploymentArgs, opts?: pulumi.ComponentResourceOptions) {
        super("my:kubernetes:ServiceDeployment", name, args, opts);
        
        // TODO
    }
}
```

The `ServiceDeploymentArgs` interface is the arguments we'll allow to be passed to our component.

The `ServiceDeployment` class is a `ComponentResource`, meaning its a kind of Pulumi resource that can manage other resources.  In our case, it will create and manage a `Service` and a `Deployment`.  It will have three output properties, for the `Service`, the `Deployment` and the `url` of the service as a convenience.

First we'll add the deployment. Add the following to the `// TODO` section.

```ts
        const appLabels = { app: name };
        this.deployment = new k8s.apps.v1.Deployment("eks-demos-app", {
            metadata: { namespace: args.namespace },
            spec: {
                selector: { matchLabels: appLabels },
                replicas: args.replicas || 1,
                template: {
                    metadata: { labels: appLabels },
                    spec: {
                        containers: [{
                            name: "eks-demo-app",
                            image: args.image,
                        }]
                    }
                },
            },
        }, { parent: this });
```

This code is nearly the same as what we have in our `index.ts` - with three differences:

* We use the `name` of the component as the `app` label.
* We use the provided `replicas` or else default to `1`.
* We pass `{parent: this}` to indicate that the resource is a child of the `ServiceDeployment` component.

Next, we do the same for our Service.  Add the following code after the `Deployment`.

```ts
        this.service = new k8s.core.v1.Service("app-svc", {
            metadata: { namespace: args.namespace },
            spec: {
                selector: appLabels,
                ports: [args.port],
                type: "LoadBalancer",
            } 
        }, { parent: this });
     
        const address = this.service.status.loadBalancer.ingress[0].hostname;
        const port = this.service.spec.ports[0].port;
        this.url = pulumi.interpolate`http://${address}:${port}`;
```

Again, this is very similar to what we have in `index.ts`.  We decide to be a little more opinionated here (we can choose how flexible we want our new component to be depending on the needs of our project and team - just like when we design APIs in our software projects) and require that we expose just a single port and that there is an HTTP service exposed on that port.  We could of course customize this further is needed.

{{% notice info %}}
The `servicedeployment.ts` file should now have the following contents:
{{% /notice %}}
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

export interface ServiceDeploymentArgs {
    namespace?: pulumi.Input<string>;
    replicas?: pulumi.Input<number>;
    image: pulumi.Input<string>;
    port: k8s.types.input.core.v1.ServicePort;
}

export class ServiceDeployment extends pulumi.ComponentResource {
    deployment: k8s.apps.v1.Deployment;
    service: k8s.core.v1.Service;
    url: pulumi.Output<string>;
    constructor(name: string, args: ServiceDeploymentArgs, opts?: pulumi.ComponentResourceOptions) {
        super("my:kubernetes:ServiceDeployment", name, args, opts);

        const appLabels = { app: name };
        this.deployment = new k8s.apps.v1.Deployment("eks-demos-app", {
            metadata: { namespace: args.namespace },
            spec: {
                selector: { matchLabels: appLabels },
                replicas: args.replicas || 1,
                template: {
                    metadata: { labels: appLabels },
                    spec: {
                        containers: [{
                            name: "eks-demo-app",
                            image: args.image,
                        }]
                    }
                },
            },
        }, { parent: this });
        
        this.service = new k8s.core.v1.Service("app-svc", {
            metadata: { namespace: args.namespace },
            spec: {
                selector: appLabels,
                ports: [args.port],
                type: "LoadBalancer",
            } 
        }, { parent: this });
     
        const address = this.service.status.loadBalancer.ingress[0].hostname;
        const port = this.service.spec.ports[0].port;
        this.url = pulumi.interpolate`http://${address}:${port}`;
        
    }
}
```
