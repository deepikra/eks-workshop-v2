---
title: Enabling ISTIO, Cert Manager and AWS Private CA Issuer using EKS Blueprints
sidebar_position: 30
weight: 5
---

Before proceeding with the integration steps, you must first install ISTIO and cert-manager on your EKS cluster. The following eks-cluster-with-new-vpc example deploys a new EKS cluster with a managed node group. It will also bootstrap the cluster with vpc-cni, coredns, kube-proxy, istio, cert-manager, and aws_privateca_issuer add-ons. Indicating that an add-on should be installed in an EKS cluster is as simple as setting a boolean value to true.

Go edit the main.tf file located in the root path of the example used. Add the following lines under the module eks_blueprints_kubernetes_addons:

Note: In addition to tetrate istio and cert manager, we enable the AWS private CA Issuer module in the above code. The AWS PrivateCA Issuer plugin acts as an addon (see external cert configuration) to cert-manager that signs certificate requests using ACM Private CA. 

To enable the AWS Private CA Issuer, you must provide the ARN of the AWS ACM Private CA (the Root CA), so that it can create an IAM role with the least privileges that will be associated with the private ca issuer pods via IRSA. And as you can see, we obtain the ARN of this Root CA by referring to the ARN attribute of the terraform resource aws acmpca certificate authority.istio that will be created in the following step.
