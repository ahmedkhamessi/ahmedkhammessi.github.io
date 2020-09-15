---
layout: post
title: Service Mesh 201 - Traffic Management
bigimg:
  - "/img/servicemesh/trafficmanagement/tm.jpg": "https://unsplash.com/photos/7nrsVjvALnA"
image: https://ahmedkhamessi.com/img/servicemesh/trafficmanagement/tm.jpg
share-img: https://ahmedkhamessi.com/img/servicemesh/trafficmanagement/tm.jpg
tags: [Service Mesh, Istio]
comments: true
time: 7
---
Istio comes with a handsfull of features aiming to solve the modern technical problem arising from the shift of monoloth application to distributed microservice architecture, In this blog entry we are going to explore **Traffic Management**, one of the core features of a service mesh. For a better understanding a basic knowledge about Istio is required for that you can check this [Introduction](https://ahmedkhamessi.com/2020-09-11-Service-Mesh-101/).

## Catchup

Once Istio is configured, It becomes the control plane of the application network allowing to manage service traffic without remodeling the services themselfs. This unleashes new capabilities espacially in regards wtih deployment strategy such as Canary and Blue/Green deployments but also Circuit Breaker pattern and more.. Those capabilities relys on a couple of Istio custom resources, **Virtual Service** and **Destination Rule**, So let's checkout how they look like in action
![Catchup Diagram](https://ahmedkhamessi.com/img/servicemesh/trafficmanagement/catchup.jpg)

As we can see in the diagram the service, kubernetes resource, loadbalance the traffic between all the pods with the label configured in the selector property ignoring the fact that they operate different verions wich Istio approch with DestinationRules and Subsets quite easly offering a fine grained traffic control.

```yaml
#-----------/*Virtual Service*/----------
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-virtualservice
spec:
  hosts:
  - my-host
  http:
  - route:
    - destination: #Routing all traffic coming to my-host to the v2 subset
        host: my-host
        subset: v2 #Defined in the Destination Rule

#-----------/*Destination Rule*/----------
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destinationrule
spec:
  host: my-host
  subsets: # A subset is a selection of the pods based on labels.
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

Istio custom resources as such open the door to hanle different secnarios and comes handy in daily DevOps and test task

## Dark lunch

The concept is to lunch a version of the application that's only accessible to the test team or specific users, It's a great way to gain confidence in new features. Admiting that we have the same Destination Rule created we need to fine tune the Virtual Service to achieve this kind of behaviour
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers: 
        end-user: # header key 
          exact: tester
    route: # Route if pattern matches
    - destination:
        host: reviews
        subset: v2
  - route: # Else
    - destination:
        host: reviews
        subset: v1
```
The previous example uses an http header with a user name which is not a production scenario, normally we would have cookies or JWT tokens which we will get into in a later stage of the series. 
But for now, something that i found really handy in my last projects is **fault injection** to test the behaviour of my services when i get a specific http status code. To condifgure this with istion it's a simple as adding the fault condition to the virtual service

```yaml
# same as before
  http:
  - match:
    - headers:
        end-user:
          exact: tester    
    fault:
      abort:
        percent: 50 # Only 50% of the request will fault
        httpStatus: 503
# same as before
```

## Blue/Green Deployment

After gaining confidence in the new features by making sure they work as expected and that the system is resiliant enough through dark lunches, It's time to release it to big public and this is a typical scenario where you want to have a sort of a backup plan in case of things go wrong.
That's where Blue/Green deployment comes into play by offering a sort of smooth transition between versions and allowong easy rollbacks.

A pre-requirement to understand how perform such a release on the network level only, we need to get familiar with the concept of **Gateway** in Istio. It's a custom resource that 
> describes a load balancer operating at the edge of the mesh receiving incoming or outgoing HTTP/TCP connections.

That's the definition from the official Istio documentation but let me try to break that down for you with the following diagram
![Gateway Diagram](https://ahmedkhamessi.com/img/servicemesh/trafficmanagement/gateway.jpg)

The takeway is that when users are trying to reach test.myapp.com the gateway is going to redirect the traffic to the virtual service with the same host name and it's actually the subsets bound to virtual services which will deliver the Blue/Green deployment capability as just changing it the traffic will be routed to a different version/set of pod. Let's do it
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway
spec:
  servers:
    hosts:
    - "*"
  - Port:
      number: 80
      name: http
      protocol: Http
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  gateways:
    - my-gateway
  http:
  - route:
    - destination:
        host: myapp
        subset: v2
        port:
          number: 9080
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-test
spec:
  hosts:
    - test.myapp
  gateways:
    - my-gateway
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
        port:
          number: 9080
```

## Canary Deployment

Very similar to the Blue/Green deployment but instead of being radical and switching a 100% the traffic from version 1 to version 2, This time we are going to make use of the **Weight** property to dose the traffic going to the new release.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  gateways:
    - my-gateway
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
        port:
          number: 9080
      weight: 25
    - destination:
        host: myapp
        subset: v2
        port:
          number: 9080
      weight: 75
```

## Circuit Breaker

This pattern is widely used in a distributed architecture aiming to handle faults scenario that might auto-heal in a certain amount of time and this ofcourse increase the stability and resilency of the system. One of the great power of Istio, It allows us to implement this pattern only by handling the network level by adding a **Traffic Policy** to the **Destination Rule**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
      trafficPolicy:
        outlierDetection:
          consecutiveErrors: 2 # If 2 errors in a 1min interval then eject for 5min
          interval: 1m
          baseEjectionTime: 5m
          maxEjectionPercent: 100 # The percentage of the endpoints that the policy can eject
```

## Wrapup

We've seen what a powerfull set of capabilites offered by the custom resources that Istio brings up allowing a fine grain service traffic management. In the next blog entry Security is going to be in the spot light, we will check out mTLS and Istio policies, Also some practical examples of how to secure access to services using the AuthorizationPolicy resource.
