+++
title = "3.2 Build Platform Binary"
chapter = false
weight = 20
+++

We've created an application - how do we get it into our Kubernetes cluster?

Developers often like to use command line tools to deploy applications. We'll build a command line tool locally.

## Step 1 &mdash; Clone the Repo

Our first step is to clone the Git repository containing the app locally.

```
git clone https://github.com/jaxxstorm/ploy.git
```

Change into this directory now ready to build our application:

```
cd ploy
```

## Step 2 &mdash; Examine the Repo

This repo contains Pulumi code which uses the Automation API. A lot of the code is used to build a CLI application using popular the Go package [Cobra](https://github.com/spf13/cobra)

However, if you examine the code in `pkg/program/program.go` you'll notice a Go function `NewPloyDeployment`.

This is a Pulumi [component](https://www.pulumi.com/docs/intro/concepts/programming-model/#components) which encapulates a set of Pulumi resources.

In order to deploy our application to our Kubernetes cluster, we'll need:

- [an ECR repository](https://github.com/jaxxstorm/ploy/blob/main/pkg/pulumi/program.go#L40)
- [a Docker Image](https://github.com/jaxxstorm/ploy/blob/main/pkg/pulumi/program.go#L71-L81)
- [Kubernetes resources](https://github.com/jaxxstorm/ploy/blob/main/pkg/pulumi/program.go#L103-L156)

All of this is done via the command line tool.

We could also write a Pulumi program that uses the same logic, but this CLI tool can be used with no underlying knowledge of Pulumi.

## Step 3 &mdash; Build the binary

{{% notice info %}}
It might be the case that you're unable to build the binary on the Cloud9 instance. If you're struggling to build it, you can download a pre-built binary from here:
https://github.com/jaxxstorm/ploy/releases/download/v0.0.2-alpha/ploy-v0.0.2-alpha-linux-amd64.tar.gz
{{% /notice %}}

We'll need to build the binary so we can use it. You can do this using the `go build` command:

```
go build -o ploy ./cmd/ploy/main.go
```

{{% notice info %}}
The EC2 Cloud9 instance comes with several Docker Images on it that might mean you run out of space. If this happens, clear out these images - you won't need them:
```
docker rmi $(docker images -q)
```
{{% /notice %}}

You should be able to run the `ploy` command now locally:

```
./ploy
Deploy your applications

Usage:
  ploy [command]

Available Commands:
  destroy     Remove your application
  get         Get all ploy deployed applications
  help        Help about any command
  up          Deploy your application

Flags:
      --debug           Enable debug logging
  -h, --help            help for ploy
  -o, --org string      Pulumi org to use for your stack
  -r, --region string   AWS Region to use (default "us-west-2")

Use "ploy [command] --help" for more information about a command.
```

