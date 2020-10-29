---
layout: post
title: AKS - Encrypt OS and Data Disks BYOK and Mutate Default Storage
bigimg:
  - "img/aksencryption/background.jpg": "https://unsplash.com/photos/FXFz-sW0uwo"
image: https://ahmedkhamessi.com/img/aksencryption/background.jpg
share-img: https://ahmedkhamessi.com/img/aksencryption/background.jpg
tags: [AKS, AzureKeyVault, BYOK]
comments: true
time: 2
---
Working on Azure Kubernetes Service on an Enterprise scale imposes security first approaches, Therefor encrypting OS and Data disks should be on the top of your security checklist for AKS and it's ecosystem. The challenge is actually how to make sure that the cluster users when requesting **PersistentVolumes** through **PersistentVolumeClaims**, AKS is going to provision the disk with a custom encryption key the resides in an Azure Keyvault.

Moreover, Azure Kubernetes Service mutate default storage class feature went GA on the 22 of September, So now AKS customers can now use a different storage class in place of the default storage class to better fit their workload needs. This is actually a good fit in our scenario so we will apply it to the cluster.

# The Concept

So the idea is actually to create a **Disk Encryption Set** and configure, assign the required permissions, it to encrypt the disks with a key within our Keyvault and finally set up the AKS cluster to use the encryption set for OS and Data disks.

The following script will get you throught the following steps
- Create the resourcegroup and the Keyvault
- Create an instance of a DiskEncryptionSet
- Grant the DiskEncryptionSet access to the Keyvault
- Create the AKS and assign the DiskEncryptionSet
- Create a new StorageClass
- Mutate the Default StorageClass

```bash
myKeyVaultName={PlaceHolder}
myKeyName={PlaceHolder}
myResourceGroup={PlaceHolder}
myAzureRegionName={PlaceHolder}
myDiskEncryptionSetName={PlaceHolder}
myAKSCluster={PlaceHolder}
myAzureSubscriptionId=$(az account list --query [id] -o tsv)
storageClass={PlaceHolder}
spAppId={PlaceHolder}
# greater than 1.17 
KUBERNETES_VERSION=1.18.6

#Create the resourcegroup and the Keyvault
az group create -l $myAzureRegionName -n $myResourceGroup
az keyvault create -n $myKeyVaultName -g $myResourceGroup -l $myAzureRegionName  --enable-purge-protection true --enable-soft-delete true

#Create an instance of a DiskEncryptionSet
keyVaultId=$(az keyvault show --name $myKeyVaultName --query [id] -o tsv)
keyVaultKeyUrl=$(az keyvault key show --vault-name $myKeyVaultName  --name $myKeyName  --query [key.kid] -o tsv)

az disk-encryption-set create -n $myDiskEncryptionSetName  -l $myAzureRegionName  -g $myResourceGroup --source-vault $keyVaultId --key-url $keyVaultKeyUrl

#Grant the DiskEncryptionSet access to key vault
desIdentity=$(az disk-encryption-set show -n $myDiskEncryptionSetName  -g $myResourceGroup --query [identity.principalId] -o tsv)

# Update security policy settings
az keyvault set-policy -n $myKeyVaultName -g $myResourceGroup --object-id $desIdentity --key-permissions wrapkey unwrapkey get

# Create the AKS cluster
diskEncryptionSetId=$(az disk-encryption-set show -n $myDiskEncryptionSetName -g $myResourceGroup --query [id] -o tsv)

az aks create -n $myAKSCluster -g $myResourceGroup --node-osdisk-diskencryptionset-id $diskEncryptionSetId --kubernetes-version $KUBERNETES_VERSION --generate-ssh-keys

#Encrypt AKS data disk
az role assignment create --assignee $spAppId --scope /subscriptions/$myAzureSubscriptionId/resourceGroups/$myResourceGroup/providers/Microsoft.Compute/diskEncryptionSets/$myDiskEncryptionSetNam --role Contributor

#create the storage class
cat << EOF | kubectl apply -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1  
metadata:
  name: $storageClass
provisioner: kubernetes.io/azure-disk
parameters:
  skuname: Standard_LRS
  kind: managed
  diskEncryptionSetID: "/subscriptions/$myAzureSubscriptionId/resourceGroups/$myResourceGroup/providers/Microsoft.Compute/diskEncryptionSets/$myDiskEncryptionSetName"
allowVolumeExpansion: true
EOF

#Mutate the default StorageClass
kubectl patch storageclass default -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass $storageClass -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

```

# Demo!

### The DiskEncryptionSet Configuration

Under the Access Control blade of the DiskEncryptionSet check how the contributor role is assigned to the AKS cluster.

![The DiskEncryptionSet configuration](https://ahmedkhamessi.com/img/aksencryption/DiskEncryptionSet_Config.png)

### Storage Class and Data Disk Encryption

Checkout the Mutated StorageClass, as the newly created one is set to default, and the PVC myclaim is bound to it.

```shell
ahmed@Azure:~$ kubectl get sc,pvc
NAME                                                PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   
storageclass.storage.k8s.io/azurefile               kubernetes.io/azure-file   Delete          Immediate                   
storageclass.storage.k8s.io/azurefile-premium       kubernetes.io/azure-file   Delete          Immediate 
storageclass.storage.k8s.io/default                 kubernetes.io/azure-disk   Delete          Immediate            
storageclass.storage.k8s.io/encryptedsc (default)   kubernetes.io/azure-disk   Delete          Immediate                   
storageclass.storage.k8s.io/managed-premium         kubernetes.io/azure-disk   Delete          Immediate                      

NAME                              STATUS   VOLUME                                     CAPACITY   STORAGECLASS   
persistentvolumeclaim/myclaim     Bound    pvc-cc6eb71f-f2ac-4925-9a29-3e71678e1d27   8Gi        encryptedsc    
persistentvolumeclaim/testclaim   Bound    pvc-4fc72059-df07-4a1d-b12a-3facce852ee9   8Gi        default       
```
Jump into the portal and check out the attached disk and it's encryption setting

![Data Disk Encryption](https://ahmedkhamessi.com/img/aksencryption/DataDiskEncryption.png)

### OS Disk Encryption

Check any instance (Worker Node) of the VMSS to check out if the Encryption is set to use the custom key

![Data Disk Encryption](https://ahmedkhamessi.com/img/aksencryption/OSDiskEncryption.png)