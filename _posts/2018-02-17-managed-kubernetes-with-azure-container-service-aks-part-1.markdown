---
layout: default
title:  "Managed Kubernetes with Azure Container Service (AKS) - Part 1"
date:   2018-02-17
categories: kubernetes docker azure
post_author: Daniel Musiał
---

It has been clear for quite some time that Kubernetes is winning the Container Orchestration Wars. Being one of the most actively developed open-source projects on GitHub with great community and many successfull production cases at Google, Kubernetes is gaining a lot of interest from developers regardless of their technology or platform of choice. It's not a surprise that big cloud players such as Amazon or Microsoft are heavily investing in Container Orchestration and Container as a Service solutions. In this and the next couple of articles, I'll focus on one of the latest additions to Microsoft's Azure product family - Azure Container Service AKS.

## AKS vs ACS / Managed vs Unmanaged

AKS allows to build managed Kubernetes clusters. With AKS, users do not have to worry about maintaining the control plane of the cluster. Apart from automating the provisioning, AKS also releases us from activites such as patching, upgrading, scaling and backing up of the Kubernetes master nodes. In fact, users are not even given access to those servers ie via SSH. The only thing we will need to worry about are the worker nodes (minions) running our application containers. This is a big step forward from the previous Azure Container Service ACS, that offered unmanaged Kubernetes, DC/OS and Docker Swarm clusters and a great simplification from developer's perspective.

## What do I need to start using AKS?

To play around with AKS you'll need the following:
 * Azure subscription
 * [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) installed on your machine

## Provisioning Kubernetes cluster

Thanks to nice support of AKS in Azure CLI, provisioning a cluster is very simple. First, make sure that you're properly logged in to your Azure account by executing:

```
$ az login
```

Next, create a Resource Group that will host your cluster. Since AKS is still in preview, you'll have to verify regional availability before proceeding. You can do so by visiting the [AKS github page](https://github.com/Azure/AKS/blob/master/preview_regions.md).

```
$ az group create -l westeurope -n AksDemo
```

At this point you're ready to provision the cluster. To minimize cost of the demo environment I suggest going with 3 worker nodes.

```
$ az aks create -n AksDemo -g AksDemo --node-count 3 --kubernetes-version 1.8.2 --generate-ssh-keys
```

Notice that I'm using Kubernetes version 1.8.2 and generating new SSH keys for the worker nodes. If you want to further customize the cluster or provide additional settings the [az aks online docs](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest) will help you get going. After the command executes successfully you'll notice the following:

 * Your Resource Group (AksDemo in my example) contains a single resource that represents the AKS cluster
 * An additional Resource Group has been created automatically that contains the worker node VMs and all related resources (NICs, Disks, Security group etc)
 * Service Principal / Application Registration has been created automatically

So far the only problems I have encountered during cluster provisioning were related to either insufficient access rights or regional availability so before executing `az aks create` make sure you have proper rights to create Resource Groups and Service Principals. Also check for any announcements on the [AKS github page](https://github.com/Azure/AKS/blob/master/preview_regions.md) already mentioned before.

## Accessing Kubernetes web portal

After the cluster has been provisioned you can open up Kubernetes web UI to check if everything worked fine. By default it is not exposed to the internet so cannot be accessed directly. Fortunatelly we can use `kubectl` to set up port forwarding. Before doing so, you'll need `kubectl` on your system. If you dont have it already downloaded, run the following Azure CLI command:

```
$ az aks install-cli
```

If you're runnning Windows it is worth adding the download location (Usually C:\Program Files (x86)) to system PATH before proceeding. Next, we'll need to create a config file for `kubectl` that will contain the Kubernetes API address as well as authentication details. Azure CLI can help us with that as well:

```
$ az aks get-credentials -g AksDemo -n AksDemo
```

On a Windows machine, the `config` file will be created in you user's home directory, ie `C:\Users\user_name\.kube\config`. Now we're ready to set up port forwarding and access the Kubernetes web UI:

```
$ az aks browse -n AksDemo -g AksDemo
```

After executing the above command, the Kubernetes web UI will open up in your default browser. You can also use `kubectl` to interact with your cluster like you normally would with any Kubernetes cluster. For example, you can run the below to get the list of worker nodes:

```
$ kubectl get nodes
```

## Beyond Azure - other managed Kubernetes solutions

Azure AKS was one of the first managed Kubernetes services but definitely not the only one. There are a few similiar alternatives already on the market that are worth exploring:

 * [Amazon EKS](https://aws.amazon.com/eks/)
 * [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/)
 * [Platform9](https://platform9.com/managed-kubernetes/)

If you're looking for more, it is worth to visit Kubernetes setup documentation that discusess [Picking the right solution] (https://kubernetes.io/docs/setup/pick-right-solution/).