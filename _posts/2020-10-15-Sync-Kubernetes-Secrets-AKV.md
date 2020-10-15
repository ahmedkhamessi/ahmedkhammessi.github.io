---
layout: post
title: Sync Kubernetes Secretes with Azure Keyvault
bigimg:
  - "/img/syncaksakv/vault.jpg": "https://unsplash.com/photos/3a7nrgYSbRE"
image: https://ahmedkhamessi.com/img/syncaksakv/vault.jpg
share-img: https://ahmedkhamessi.com/img/syncaksakv/vault.jpg
tags: [AKS, CSI, Keyvault, Azure]
comments: true
time: 5
---
Configuring applications to run on Kubernetes requires an understanding of some concepts like **ConfigMaps** and **Secret**, Those objects allow us to decouple environment-specific configuration from our container images, so that the applications are easily portable.

A keynote on the difference between ConfigMaps and Secrets is that ConfigMaps does not provide secrecy or encryption. Secrets on the other hand,let us store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys in an encoded format which is still not safe and that's where Keyvault jumps in the picture by offering a centralized and secure store for our secrets.

The main question is how to synchronize the secrets in a Keyvault with the kubernetes Secret object?

## Secrets Store CSI Driver

This driver integrates secret stores (Azure Keyvault, HashiCorp Vault) with Kubernetes via a Container Storage Interface (CSI) volume which is basically a standard for exposing block and file storage system to containerized workloads on Container Orchestration Systems like Kubernetes.
![Secret Store CSI flow](https://ahmedkhamessi.com/img/syncaksakv/csi-flow.png)

## Demo Time!!

The prerequiste for the demo:
- AKS cluster
- Azure Keyvault

```bash
#create secret in the keyvault
az keyvault secret set --name $secretName --value $secretValue --vault-name $keyVaultName

#setup csi-driver helm and namespace
helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts
kubectl create ns csi-driver

#install the csi-driver and azure keyvault provider
helm install csi-azure csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --namespace csi-driver
```

At this stage we have the secret stored in the keyvault, The csi driver and azure keyvault provider installed on the cluster. The next is to **provide the Identity** to access the keyvault.

The Azure Keyvault Provider offers four modes:
- Service Principal
- Pod Identity
- VMSS User Assigned Managed Identity
- VMSS System Assigned Managed Identitiy

For the purpose of this demo the AKS cluster has been created with [Managed Identities](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity), So for the rest of the scripts option 3 is used (VMSS User Assigned Managed Identity).

```bash
clientId=$(az aks show -g $AKSResourceGroup -n $aksName --query identityProfile.kubeletidentity.clientId -o tsv)
# set policy to access keys in your Key Vault
az keyvault set-policy -n $keyVaultName --key-permissions get --spn $clientId
# set policy to access secrets in your Key Vault
az keyvault set-policy -n $keyVaultName --secret-permissions get --spn $clientId
# set policy to access certs in your Key Vault
az keyvault set-policy -n $keyVaultName --certificate-permissions get --spn $clientId
```

## Deploy SecretProviderClass and mirror the AKV

![Architecture diagram](https://ahmedkhamessi.com/img/syncaksakv/architecture.png)

```bash
#deploy the SecretProviderClass
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: azure-kvname
spec:
  provider: azure
  secretObjects:                                 #SecretObject defines the desired state of synced K8s secret objects
  - secretName: sqlserver
    type: Opaque
    data: 
    - objectName: SECRET_1                    # name of the mounted content to sync. this could be the object name or object alias 
      key: dbpassword                   
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: $clientId 
    keyvaultName: $keyVaultName
    cloudName: ""          #[OPTIONAL] if not provided, azure environment will default to AzurePublicCloud
    cloudEnvFileName: ""   # [OPTIONAL] use to define path to file for populating azure environment
    objects:  |
      array:
        - |
          objectName: $secretName
          objectAlias: SECRET_1     # [OPTIONAL] object alias
          objectType: secret        # object types: secret, key or cert
          objectVersion: ""         # [OPTIONAL]
    resourceGroup: $KeyvaultResourceGroupName            # the resource group of the KeyVault
    subscriptionId: $subscriptionId         # the subscription ID of the KeyVault
    tenantId: $tenantId                 # the tenant ID of the KeyVault
EOF
```
At this stage the SecretProviderClass is set up and connected to the Azure Keyvault, Also the **secretObjects** section will take care of creating a Kubernetes secret object to mirror our keyvault secret and make easier for the developers reference the secret in the **Deployment** yaml files.

To note that the secret will get created once the volume is mounted, It's meant to be like this by design. Also under [secretName].[data].[objectName] section the official documentation states that it is possible to either use the secret name or the **objectAlias** (defined in the last section of the yaml file under the object array). However, I raised an issue on the official repo cause it worked only when the alias has been used. You can follow the progress of the issue [here](https://github.com/Azure/secrets-store-csi-driver-provider-azure/issues/270#issuecomment-708517765).

## Finally the Pod!

```bash
#deploy a test pod
cat <<EOF | kubectl apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: nginx-secrets-store-inline
spec:
  containers:
  - image: nginx
    name: nginx
    env:
    - name: dbpassword
      valueFrom:
        secretKeyRef:
          name: sqlserver
          key: dbpassword
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvname"
EOF
```