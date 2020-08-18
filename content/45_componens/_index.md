+++
title = "Module 04"
chapter = true
weight = 45
+++

# Creating Infrastructure Components - Kubernetes

{{% notice warning %}}<p> You are responsible for the cost of the AWS services used while running this workshop in your AWS account.</p> {{% /notice %}}

In this lab, you will extract a common Kubernetes infrastructure of creating both a Deployment and a Service pattern into a reusable ServiceDeployment component.

This lab builds on the final state of the [deploying_applications_to_eks](../40_deploying_applications_to_eks.html) lab, so you are encouraged to do that lab first.

This lab assumes you have a Kubernetes cluster up and running. If you don't have a cluster yet, please
complete [deploying_eks_cluster](./30_deploying_eks_cluster.html) lab first.

{{% children showhidden="false" %}}
