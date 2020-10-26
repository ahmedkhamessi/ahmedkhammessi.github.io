---
layout: post
title: Service Mesh 203 - Azure Policy for AKS
bigimg:
  - "/img/azurepolicy/background.jpg": "https://unsplash.com/photos/oZ61KFUQsus"
image: https://ahmedkhamessi.com/img/azurepolicy/background.jpg
share-img: https://ahmedkhamessi.com/img/azurepolicy/background.jpg
tags: [AKS, Azure, Policy]
comments: true
time: 4
---
The Pod Security Policy is going to be [deprecated](https://docs.microsoft.com/en-us/azure/aks/use-pod-security-policies) after February 2021, Therefor it's highly recommended to begin the preparation to migrate to Azure Policy for AKS which work with Open Policy Agent - Gatekeeper underneath. So before jumping directly to Azure Policies let’s keep our options wide open and understand how OPA/Gatekeeper works and then get into Azure Policy specification and how it can help out.

## Admission Controller

We know for a fact that the API server is the front end of the Kubernetes control plane and most importantly, it’s the only component that’s allowed to communicate with Kubernetes store ETCD. The requests comings from the cluster admins/users hitting the api-server go through a multiple step process before persisting the resource.
![Admission Controller Overview](https://ahmedkhamessi.com/img/azurepolicy/admissioncontroller.png)

Some of the baked in admission controllers to secure running containers
- PodSecuriyPolicy
- DenyEscalatingExec : Block ‘’exec’’ and ‘’attach’’ commands
- AllwaysPillImages ensure that the latest remediation are downloaded
- LimitRAnge and ResourceQuota : prevent DOS attacks

## Dynamic Admission Control

![Dynamic Admission Control Overview](https://ahmedkhamessi.com/img/azurepolicy/dynamicadmissioncontrol.png)

In a nutshell, It’s the way to write custom admission controllers using two special controllers in the sense of they don’t implement policy logic themselves instead allow us to write our custom logic and execute it whenever resources are created, updated or deleted
- ValidatingAdmissionWebhooks
- MutatingAdmissionWebhooks

> The implementation  of the webhook REST API can be in any language Go, Python or even bash in a container, but the recommended way would be to use Open Policy Agent.

## Open Policy Agent

OPA is a centralized way to handle policies, not only in a Kubernetes ecosystem, but microservices, CI-CD pipelines, API gateways.. The concept is that OPA will take a JSON input, OPA will evaluate the input against a policy written in Rego, which is a query language and send back the decision.

![Open Policy Agent Overview](https://ahmedkhamessi.com/img/azurepolicy/opa.png)

### Demo

Pre-requisits
- VS Code - OPA extension https://github.com/open-policy-agent/vscode-opa
- Source code https://github.com/ahmedkhammessi/opa

```json
{
    "apiVersion": "apps/v1",
    "kind": "Deployment",
    "metadata": {
      "name": "hello-kubernetes",
      "labels": {
        // "website": "ahmedkhamessi.com", uncomment to get the policy running 
        "app.kubernetes.io/name": "mysql",
        "app.kubernetes.io/version": "5.7.21",
        "app.kubernetes.io/component": "database",
        "app.kubernetes.io/part-of": "wordpress",
        "app.kubernetes.io/managed-by": "helm"
      }
    },
    "spec": {
      "replicas": 3,
      "selector": {
        "matchLabels": {
          "app": "hello-kubernetes"
        }
      },
      "template": {
        "metadata": {
          "labels": {
            "app": "hello-kubernetes"
          }
        },
        "spec": {
          "containers": [
            {
              "name": "hello-kubernetes",
              "image": "paulbouwer/hello-kubernetes:1.5",
              "ports": [
                {
                  "containerPort": 8080
                }
              ]
            }
          ]
        }
      }
    }
  }
```

```rego
package main

deny[msg] {
  input.kind == "Deployment" #Evaluate if kins is Deployment
  not input.metadata.labels.website #If True continue otherwise Stop & Exist

  msg := "the label website habe to be added to the metadata"
}
```
**Evaluate the policy**
![Open Policy Agent Demo](https://ahmedkhamessi.com/img/azurepolicy/opa-demo.png)

## Gatekeeper

It’s a wrapper around OPA and provides a Kubernetes native integration and functionality. The installation is quite straightforward either deploy the resources with the Kubectl command or use Helm

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper --generate-name
```
Remember the Rego policy from the OPA demo well that didn’t look native Kubernetes right!  Therefore using **ContraintTemplate** Gatekeeper extends the policy library by creating a respective CRDs (CustomResourceDefinition) and by the mean of **constraints** we inform gatekeeper that we want to enforce it or in a more technical term that we want to instantiate the policy.

The following example demonstrate how to deny sharing the host namespace, But more examples are available [here](https://github.com/open-policy-agent/gatekeeper-library)

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8spsphostnamespace
spec:
  crd:
    spec:
      names:
        kind: K8sPSPHostNamespace #New CRD
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: | # The policy definition
        package k8spsphostnamespace
        violation[{"msg": msg, "details": {}}] {
            input_share_hostnamespace(input.review.object)
            msg := sprintf("Sharing the host namespace is not allowed: %v", [input.review.object.metadata.name])
        }
        input_share_hostnamespace(o) {
            o.spec.hostPID
        }
        input_share_hostnamespace(o) {
            o.spec.hostIPC
        }
```
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPHostNamespace
metadata:
  name: psp-host-namespace
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"] # The scope
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-host-namespace
  labels:
    app: nginx-host-namespace
spec:
  hostPID: true #false
  hostIPC: true #false
  containers:
  - name: nginx
    image: nginx
```

Moreover, One of the advantages that Gatekeeper brings out of the box is the **Audit** functionality which is a huge add to the administrators in order to evaluate the compliance of the clusters against the policies. Read more about [here](https://github.com/open-policy-agent/gatekeeper#audit) 

## Azure Policy for AKS - Finally!

![Azure Policy for AKS - Overview](https://ahmedkhamessi.com/img/azurepolicy/azurepolicy.png)

By looking at the architecture we recognize that Azure Policy extends Gatekeeper to apply policies on the clusters but in a more centralized way since it allow us to manage and report the compliance state of multiple clusters from one place. By enabling the Azure Policy add-on this what we get:
-	Deploys policy definitions as **ConstaintTemplate** and **constraint** CRDs
-	Reports **Auditing ** to Azure Policy service
-	Checks with Azure Policy service for policy assignments.

![Azure Policy for AKS - Policies](https://ahmedkhamessi.com/img/azurepolicy/policies.png)

If you check any of the policies and go down in the definition you will find the links to the CRDs definitions

![Azure Policy for AKS - CRD Definition](https://ahmedkhamessi.com/img/azurepolicy/policylinktoconstraint.png)

## Wrapup

My recommendation for your production environment, even though it’s still missing custom policies, but it’s a feature in progress from the respective team where the challenge is not technical but how to provide a safe environment around custom policies.
