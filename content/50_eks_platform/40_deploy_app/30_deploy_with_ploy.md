+++
title = "3.3 Deploy with Ploy"
chapter = false
weight = 30
+++

## Step 1 &mdash; Run Ploy

We can now use `ploy` to deploy our application. First let's make sure it's in our `$PATH`:

```
sudo mv ploy /usr/local/bin/ploy
```

Then switch back to our `platform-app` directory:

```
cd ~/environment/platform-app
```

Now, run `ploy up` from within this directory and specify an organization to use - this should match the name you signed up to the Pulumi SaaS with:

```
ploy up -o jaxxstorm --verbose
```

You should see some output from the command line application:

```
INFO[0003] Creating application: yearly-choice-tadpole  
INFO[0003] Creating ECR repository                      
INFO[0003] Creating local docker image                  
INFO[0003] Creating Kubernetes namespace                
INFO[0003] Creating Kubernetes deployment               
INFO[0003] Creating Kubernetes service                  
INFO[0004] Repository created: 886783038127.dkr.ecr.us-east-1.amazonaws.com/yearly-choice-tadpole-d9d3ae4 

```

{{% notice info %}}
Pass the `--verbose` flag if you'd like to see what is happening with Pulumi behind the scenes.
{{% /notice %}}

Eventually, you'll see your service has been deployed, like tos:

```
INFO[0055] Your service is available at: a89a4e0e31b194a02bcf0207253ccd24-798957911.us-west-2.elb.amazonaws.com 
```

## Step 2 &mdash; Examine our deployed app

We can now look at our deployed application. Let's use `curl` to do this:

```
curl a89a4e0e31b194a02bcf0207253ccd24-798957911.us-west-2.elb.amazonaws.com
Hello, world!
```

The application we created inside the Docker file was deployed via Pulumi, without having to write a single line of infrastructure code.

## Step 3 &mdash; Examine Pulumi console

To examine the Pulumi program created from the automation API application, visit the Pulumi SaaS, replacing `jaxxstorm` with your org id:

https://app.pulumi.com/jaxxstorm/ploy

You should see your created application visible in the console!

## Step 4 &mdash; Destroy everything

Now we need to destroy our `ploy` application. Use the randomly generated name and run `ploy destroy` on it:

```
ploy -o jaxxstorm destroy firstly-discrete-panther 
```

Ploy will prompt you for confirmation:

```
This will delete the application yearly-choice-tadpole. Are you sure you wish to continue?: y
INFO[0001] Deleting application: yearly-choice-tadpole
```

Now, cd into your load balancer controller and destroy that Pulumi application:

```
cd ~/environment/aws-load-balancer-controller/
pulumi destroy --yes
```

Pulumi will delete all the resources from both your Kubernetes cluster, and the AWS account:

```
Previewing destroy (dev)

View Live: https://app.pulumi.com/jaxxstorm/aws-load-balancer-controller/dev/previews/04df4af6-48be-4711-a550-8f51fd23161d

     Type                                                                                        Name                                                                           Plan       
 -   pulumi:pulumi:Stack                                                                         aws-load-balancer-controller-dev                                               delete     
 -   ├─ pulumi:providers:kubernetes                                                              provider                                                                       delete     
 -   │  └─ kubernetes:core/v1:Namespace                                                          aws-lb-controller-ns                                                           delete     
 -   │     └─ kubernetes:helm.sh/v3:Chart                                                        lb                                                                             delete     
 -   │        ├─ kubernetes:apps/v1:Deployment                                                   aws-lb-controller/lb-aws-load-balancer-controller                              delete     
 -   │        ├─ kubernetes:admissionregistration.k8s.io/v1beta1:MutatingWebhookConfiguration    aws-load-balancer-webhook                                                      delete     
 -   │        ├─ kubernetes:rbac.authorization.k8s.io/v1:ClusterRoleBinding                      lb-aws-load-balancer-controller-rolebinding                                    delete     
 -   │        ├─ kubernetes:rbac.authorization.k8s.io/v1:RoleBinding                             aws-lb-controller/lb-aws-load-balancer-controller-leader-election-rolebinding  delete     
 -   │        ├─ kubernetes:admissionregistration.k8s.io/v1beta1:ValidatingWebhookConfiguration  aws-load-balancer-webhook                                                      delete     
 -   │        ├─ kubernetes:rbac.authorization.k8s.io/v1:ClusterRole                             lb-aws-load-balancer-controller-role                                           delete     
 -   │        ├─ kubernetes:rbac.authorization.k8s.io/v1:Role                                    aws-lb-controller/lb-aws-load-balancer-controller-leader-election-role         delete     
 -   │        ├─ kubernetes:core/v1:Secret                                                       aws-lb-controller/aws-load-balancer-tls                                        delete     
 -   │        ├─ kubernetes:core/v1:Service                                                      aws-lb-controller/aws-load-balancer-webhook-service                            delete     
 -   │        ├─ kubernetes:apiextensions.k8s.io/v1beta1:CustomResourceDefinition                targetgroupbindings.elbv2.k8s.aws                                              delete     
 -   │        └─ kubernetes:core/v1:ServiceAccount                                               aws-lb-controller/aws-lb-controller-serviceaccount                             delete     
 -   └─ aws:iam:Role                                                                             aws-loadbalancer-controller-role                                               delete     
 -      ├─ aws:iam:PolicyAttachment                                                              aws-loadbalancer-controller-attachment                                         delete     
 -      └─ aws:iam:Policy                                                                        aws-loadbalancer-controller-policy                                             delete     
 
Resources:
    - 18 to delete

Destroying (dev)
```

Finally, destroy your workshop EKS cluster:

```
cd ~/environment/workshop-cluster/
pulumi destroy --yes
```

Which will tear down the EKS cluster:

```
Previewing destroy (dev)

View Live: https://app.pulumi.com/jaxxstorm/workshop-cluster/dev/previews/ffc2865e-f3c2-4d4c-bfcc-eb632a6d5263

     Type                                Name                                                Plan       
     pulumi:pulumi:Stack                 workshop-cluster-dev                                           
 -   ├─ pulumi:providers:kubernetes      lbriggs-workshop-provider                           delete     
 -   ├─ aws:cloudformation:Stack         lbriggs-workshop-nodes                              delete     
 -   ├─ aws:ec2:LaunchConfiguration      lbriggs-workshop-nodeLaunchConfiguration            delete     
 -   ├─ aws:ec2:SecurityGroupRule        lbriggs-workshop-eksExtApiServerClusterIngressRule  delete     
 -   ├─ aws:ec2:SecurityGroupRule        lbriggs-workshop-eksNodeIngressRule                 delete     
 -   ├─ aws:ec2:SecurityGroupRule         lbriggs-workshop-eksNodeClusterIngressRule          delete     
 -   ├─ aws:ec2:SecurityGroupRule         lbriggs-workshop-eksNodeInternetEgressRule          delete     
 -   ├─ kubernetes:core/v1:ConfigMap      lbriggs-workshop-nodeAccess                         delete     
 -   ├─ aws:ec2:SecurityGroupRule         lbriggs-workshop-eksClusterIngressRule              delete     
 -   ├─ pulumi-nodejs:dynamic:Resource       lbriggs-workshop-vpc-cni                            delete     
 -   │  ├─ awsx:x:ec2:Subnet                 vpc-lbriggs-workshop-public-0                       delete     
 -   pulumi:pulumi:Stack                     workshop-cluster-dev                                delete     
 -   │  │  ├─ aws:ec2:Route                  vpc-lbriggs-workshop-public-0-ig                    delete     
 -   │  │  ├─ aws:ec2:Subnet                 vpc-lbriggs-workshop-public-0                       delete     
 ...
```
