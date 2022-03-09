---
layout: post
title: Beyond the Kubernetes security certification CKS
bigimg:
  - "/img/cks/cks.jpg": "https://unsplash.com/photos/mT7lXZPjk7U"
image: https://ahmedkhamessi.com/img/cks/cks.jpg
share-img: https://ahmedkhamessi.com/img/cks/cks.jpg
tags: [AKS, K8s, kubernetes, Security, CKS]
comments: true
time: 5
---

What is more important than actually getting the Kubernetes security certificate CKS is to have a sort of a mind map or a clear vision over the Kubernetes core security concepts and the different tools that you may use to prevent or mitigate attacks that could get through one of the 4 main attack surfaces.

![cloud native security 4 c's](https://ahmedkhamessi.com/img/cks/cloudnativesecurity-4c.png)

That's already a great starting point to this post, what are actually the layers of the cloud native security

## Cloud

If the cloud layer is vulnerable or configured in a vulnerable way there is no guarantee that the components built upon it will be secure. Thus the first checkpoint is to check out security fundamentals documented by the cloud provider of your choice, as an example this is the [Azure security overview](https://docs.microsoft.com/en-us/azure/security/fundamentals/overview) that goes over the built-in capabilities in the core functional areas

- Operations
- Applications
- Storage
- Networking
- Compute
- Identity and access management

There is also the security baseline for the Kubernetes service which applies guidance from security benchmarks and provide recommendations on how to secure the cloud solution. Check out the [Full Azure Kubernetes Service security baseline mapping](https://github.com/MicrosoftDocs/SecurityBenchmarks/raw/master/Azure%20Offer%20Security%20Baselines/1.1/azure-kubernetes-service-security-baseline-v1.1.xlsx)

## Cluster

The second layer of security concerns the cluster setup itself, There are two main areas
 
### Securing the cluster components

This is your first line of defense, as a poor configuration and a loose access to the main components of your cluster like the kubernetes API or kubelet may put in danger your setup on both malicious and accidental levels. 

Therefor an Authentication (who can access) and an Authorization (what can the User/Admin/ServiceAccount do) concept must be considered before the implementation phase. Checkout the [Access and identity options](https://docs.microsoft.com/en-us/azure/aks/concepts-identity#azure-role-based-access-control) for Azure Kubernetes Service which covers how to enhance the cluster security with the Azure AD integration and lays out the role based access control RBAC concept on both Kubernetes and Azure levels.

As shown below in the flow diagram  you can manage Azure AD-integrated Kubernetes cluster resource permissions and assignments using Azure role definition and role assignments.

![Azure RBAC for Kubernetes Authorization](https://ahmedkhamessi.com/img/cks/azure-rbac-k8s-authz-flow.png)

### Securing the apps running in the cluster

- Application secrets management
- Pod security standards
- Quality of Service for Pods
- Network Policies
- TLS for k8s Ingress

For the general recommendations in regards with the cluster overall security, please check out the [official documentation](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)


## Containers

![container security](https://ahmedkhamessi.com/img/cks/container-security.png)

If the cloud environment and the cluster get compromised, it's up to the containers running on the cluster to restrict the damage through best practices and security guidelines such as

- Container vulnerability scanning and OS dependency security
- Image signing and enforcement
- Disallow priveleged users
- Use container runtime with stronger isolation

**Code**

The application code is in general a topic doesn't concern the kubernetes environment itself, nevertheless it's on of the attack surfaces that may compromise the whole setup by exposing unnecessary ports or by not enforcing communication over TLS only.
 
- Dynamic probing attacks like SQL injection, CSRF and other common web attacks can be faced by a Web Application Firewall in front of the cluster to secure the traffic flowing to your applications, this is a [tutorial](https://docs.microsoft.com/en-us/samples/azure-samples/aks-agic/aks-agic/) on how to implement the concept with Azure Application Gateway.

![aks-agic-waf](https://ahmedkhamessi.com/img/cks/aks-agic-waf.png)

- Mutual TLS is an enhancement to the flat pod network that kubernetes originaly propose where any pod can communicate with any other pod in the cluster with no network restrictions nor security mesurments, I wrote a [blog post](https://ahmedkhamessi.com/2020-10-07-Service-Mesh-202/) on how to implement mTLS via Istio.

# Wrapup

In general it's a good practice to have a blick on the big picture especially when it comes to complexe system such as Kubernetes where you can get lost in details. I will be digging deeper to share pieces of my preparation and my work on the security aspect not only to get the certificate but as a general practice. Topic that i will be covering will be clusterized by the CKS [domains and competencies](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist/).

