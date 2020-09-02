---
layout: post
title: SQL Server on Containers and AKS
bigimg:
  - "/img/sqlserver.jpg": "https://unsplash.com/photos/M5tzZtFCOfs"
image: https://ahmedkhamessi.com/img/sqlserver.jpg
share-img: https://ahmedkhamessi.com/img/sqlserver.jpg
tags: [AKS, K8s, SQL Server, Container]
comments: true
time: 3
---
Let’s go through some facts before we drill down to the concrete stuff, Containers are ephemeral and stateless in nature which raises already a couple of question marks around the topic of having SQL Server running on containers within your Kubernetes cluster, Cause the fact is when we think about SQL Servers then data durability comes in mind, constant network availability/access and the state is critical.. 
that’s how Kubernetes and in our case AKS comes in the picture where we will separate the compute plane (stateless) from the data planes by providing persistent storage, consistent DNS and High Availability.

## Why should you consider running SQL Server on containers?

- **Ease of use** in the development and test environment without the need for spinning up VMs, managing them go into endless version conflicts etc etc etc..
- **Standardization** that’s the word that your management board would like to hear but it also make sense in this case, containers are becoming more and more the way to do things in software development so if your apps and services are running in containers then DBs should follow along.
- A huge gain on the **Portability** aspect among different cloud vendors and even on premises systems.
- Last but not least is the great **DevOps** integration.

## How to deploy SQL Server in Kubernetes

Before jumping over the AKS and setting up SQL Server pods and service let's check out what is actually needed to run it as a container just locally. By checking out [Docker Hub](https://hub.docker.com/_/microsoft-mssql-server) you can see the different images and the required options to run Sql Server on containers, it as simple as:

```shell
docker pull mcr.microsoft.com/mssql/server

docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=rWpzZx~YL+7mEw<z" `
   -p 1433:1433 --name mySqlContainer `
   -v sqldata1:/var/opt/mssql `
   -d mcr.microsoft.com/mssql/server
```

Notice that we mounted a volume where the data will be persisted and that’s exactly the same behaviour that we will map with AKS and Azure Disks instead of Docker desktop and the Docker host directory.

The steps and the concept is described in details in the [official documentation](https://docs.microsoft.com/en-us/sql/linux/tutorial-sql-server-containers-kubernetes?view=sql-server-ver15) but again for the sake of simplicity let's speed through the essentials.

# Create the secret

```shell
kubectl create secret generic mssql --from-literal=SA_PASSWORD="rWpzZx~YL+7mEw<z"
```

# Create the storage

The idea behind Storage Classes is to provide a way for administrators to describe the type of storage that the platform will offer.

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
     name: azure-disk
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Standard_LRS
  kind: Managed
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mssql-data
  annotations:
    volume.beta.kubernetes.io/storage-class: azure-disk
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

# Finally create the deplyoment and expose it!

The deployment will specify the number of replicas and via a ReplicationSet will ensure High Availability and self healing of the workload. Furthermore, We will define the different environment variables, that we’ve seen in the simple local docker version command, and mount the respective volume in the pod template.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mssql-deployment
spec:
  replicas: 1
  selector:
     matchLabels:
       app: mssql
  template:
    metadata:
      labels:
        app: mssql
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: mssql
        image: mcr.microsoft.com/mssql/server:2017-latest
        ports:
        - containerPort: 1433
        env:
        - name: MSSQL_PID
          value: "Developer"
        - name: ACCEPT_EULA
          value: "Y"
        - name: SA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mssql
              key: SA_PASSWORD 
        volumeMounts:
        - name: mssqldb
          mountPath: /var/opt/mssql
      volumes:
      - name: mssqldb
        persistentVolumeClaim:
          claimName: mssql-data
---
apiVersion: v1
kind: Service
metadata:
  name: mssql-deployment
spec:
  selector:
    app: mssql
  ports:
    - protocol: TCP
      port: 1433
      targetPort: 1433
  type: LoadBalancer
```

The secret that has been created will expose our workload/Sql Server in this case to the outside word via a Loadbalancer that will be provisioned automatically which is not the recommended way for production scale.

# Coming up

The failover mechanism for now is relying on the Kubernetes controllers such as Deployments and ReplicaSets which are  not application aware components therefor Microsoft is working on SQL Always On Availability Groups in AKS (in Preview).