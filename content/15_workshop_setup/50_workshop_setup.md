+++
title = "Workshop Setup"
chapter = false
weight = 40
+++

These hands-on labs will walk you through various cloud infrastructure tasks. The prerequisites listed below are required to successfully complete them.

## Pulumi

Before you start this workshop you will need to [sign in](https://app.pulumi.com/signin?reason=401) to your Pulumi account.

If you don't have an account yet, make sure you sign up [here](https://app.pulumi.com/signup)

You will use Pulumi to depoy infrastructure changes using code. 
To install Pulumi run the following command:

```bash
$ curl -fsSL https://get.pulumi.com | sh

```
Now restart your Cloud 9 terminal

After installing the CLI, verify that it is working:
 
```bash
$ pulumi version
v2.8.0
```
If the Pulumi command isn't recognized yet, You should close and relaunch the terminal window first.

The Pulumi CLI will ask you to login to your Pulumi account as needed. If you prefer to signup now, [go to the signup page](http://app.pulumi.com/signup). Multiple identity provider options are available &mdash; email, GitHub, GitLab, or Atlassian &mdash; and each of them will work equally well for these labs.

## Node.js

If you are not running this workshop from Cloud9 you will need to install Node.js. 

You will need Node.js version 10 or later to run Pulumi programs written in [TypeScript](https://www.typescriptlang.org/).
Install your desired LTS version from [the Node.js download page](https://nodejs.org/en/download/) or
[using a package manager](https://nodejs.org/en/download/package-manager/).

After installing, verify that Node.js is working:

```bash
$ node --version
v12.10.0
```

Also verify that the Node Package Manager (NPM) is working:

```bash
$ npm --version
6.10.3
```

## Docker

If you are not running this workshop from Cloud9 you will need to install Docker. 

If you will be completing the container labs, [install Docker Community Edition](https://docs.docker.com/install). After doing so, verify that the docker CLI is operational:

```bash
$ docker --version
Docker version 19.03.1, build 74b1e89
```

## Kubectl

If you will be completing the Kubernetes labs, [install the kubectl CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/). It isn't necessary to configure it to speak to a cluster &mdash; you will do that during the appropriate labs that require it.

Download the latest release:

```bash
$ curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"

```

Make the kubetcl binary executable:

```bash
$ chmod +x ./kubectl

```

Move the binary into your path:

```bash
$ sudo mv ./kubectl /usr/local/bin/kubectl

```
Test to ensure the version you installed is up to date:

```bash
$ kubectl version --client


```


## AWS Subscription and CLI

If you will be completing the AWS labs, you will need an AWS account. If you don't already have one, you can [sign up for the free tier here](https://portal.aws.amazon.com/billing/signup). 
The labs have been designed to use the free tier as much as possible, so that the total cost of all resources should be very close to $0. 
If in doubt, please [go here](https://aws.amazon.com/free) to see what services and resource types are available in the free tier.

At various points, you will use the AWS CLI to interact with infrastructure you've provisioned. Installation instructions are 
[available here](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html). As explained further on that page, the 
CLI requires Python.

> If you have multiple AWS accounts, you'll need to configure a profile for the account you're using in these labs. That process is 
>[described here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html). All Pulumi operations will respect your profile settings.

To verify that everything is working, run:

```bash
$ aws sts get-caller-identity

```
```bash

{
    "UserId": "ABDAII73ZGOGZ5V4QSTWY",
    "Account": "161298451113",
    "Arn": "arn:aws:iam::161298451113:user/joe@pulumi.com"
}
```
{{% children showhidden="false" %}}
