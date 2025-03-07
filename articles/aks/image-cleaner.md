---
title: Use Image Cleaner on Azure Kubernetes Service (AKS)
description: Learn how to use Image Cleaner to clean up stale images on Azure Kubernetes Service (AKS)
ms.author: nickoman
author: nickomang
ms.topic: article
ms.custom: devx-track-azurecli
ms.date: 03/02/2023
---

# Use Image Cleaner to clean up stale images on your Azure Kubernetes Service cluster (preview)

It's common to use pipelines to build and deploy images on Azure Kubernetes Service (AKS) clusters. While great for image creation, this process often doesn't account for the stale images left behind and can lead to image bloat on cluster nodes. These images can present security issues as they may contain vulnerabilities. By cleaning these unreferenced images, you can remove an area of risk in your clusters. When done manually, this process can be time intensive, which Image Cleaner can mitigate via automatic image identification and removal.

> [!NOTE]
> Image Cleaner is a feature based on [Eraser](https://github.com/Azure/eraser).
> On an AKS cluster, the feature name and property name is `Image Cleaner` while the relevant Image Cleaner pods' names contain `Eraser`.

[!INCLUDE [preview features callout](./includes/preview/preview-callout.md)]

## Prerequisites

* An Azure subscription. If you don't have an Azure subscription, you can create a [free account](https://azure.microsoft.com/free).
* [Azure CLI][azure-cli-install] or [Azure PowerShell][azure-powershell-install] and the `aks-preview` 0.5.96 or later CLI extension installed.
* The `EnableImageCleanerPreview` feature flag registered on your subscription:

### [Azure CLI](#tab/azure-cli)

First, install the aks-preview extension by running the following command:

```azurecli
az extension add --name aks-preview
```

Run the following command to update to the latest version of the extension released:

```azurecli
az extension update --name aks-preview
```

Then register the `EnableImageCleanerPreview` feature flag by using the [az feature register][az-feature-register] command, as shown in the following example:

```azurecli-interactive
az feature register --namespace "Microsoft.ContainerService" --name "EnableImageCleanerPreview"
```

It takes a few minutes for the status to show *Registered*. Verify the registration status by using the [az feature show][az-feature-show] command:

```azurecli-interactive
az feature show --namespace "Microsoft.ContainerService" --name "EnableImageCleanerPreview"
```

When the status reflects *Registered*, refresh the registration of the *Microsoft.ContainerService* resource provider by using the [az provider register][az-provider-register] command:

```azurecli-interactive
az provider register --namespace Microsoft.ContainerService
```

### [Azure PowerShell](#tab/azure-powershell)

Register the `EnableImageCleanerPreview` feature flag by using the [Register-AzProviderPreviewFeature][register-azproviderpreviewfeature] cmdlet, as shown in the following example:

```azurepowershell-interactive
Register-AzProviderPreviewFeature -ProviderNamespace Microsoft.ContainerService -Name EnableImageCleanerPreview
```

It takes a few minutes for the status to show *Registered*. Verify the registration status by using the [Get-AzProviderPreviewFeature][get-azproviderpreviewfeature] cmdlet:

```azurepowershell-interactive
Get-AzProviderPreviewFeature -ProviderNamespace Microsoft.ContainerService -Name EnableImageCleanerPreview |
 Format-Table -Property Name, @{name='State'; expression={$_.Properties.State}}
```

When ready, refresh the registration of the *Microsoft.ContainerService* resource provider by using the [Register-AzResourceProvider][register-azresourceprovider] command:

```azurepowershell-interactive
Register-AzResourceProvider -ProviderNamespace Microsoft.ContainerService
```

---

## Limitations

Image Cleaner does not support the following:

* ARM64 node pools. For more information, see [Azure Virtual Machines with ARM-based processors][arm-vms].
* Windows node pools.

## How Image Cleaner works

When enabled, an `eraser-controller-manager` pod is deployed, which generates an `ImageList` CRD. The eraser pods running on each nodes will clean up the unreferenced and vulnerable images according to the ImageList. Vulnerability is determined based on a [trivy][trivy] scan, after which images with a `LOW`, `MEDIUM`, `HIGH`, or `CRITICAL` classification are flagged. An updated `ImageList` will be automatically generated by Image Cleaner based on a set time interval, and can also be supplied manually.



Once an `ImageList` is generated, Image Cleaner will remove all the images in the list from node VMs.

:::image type="content" source="./media/image-cleaner/image-cleaner.jpg" alt-text="Screenshot of a diagram showing ImageCleaner's workflow. The ImageCleaner pods running on the cluster can generate an ImageList, or manual input can be provided.":::

## Configuration options

In addition to choosing between manual and automatic mode, there are several options for Image Cleaner:

|Name|Description|Required|
|----|-----------|--------|
|--enable-image-cleaner|Enable the Image Cleaner feature for an AKS cluster|Yes, unless disable is specified|
|--disable-image-cleaner|Disable the Image Cleaner feature for an AKS cluster|Yes, unless enable is specified|
|--image-cleaner-interval-hours|This parameter determines the interval time (in hours) Image Cleaner will use to run. The default value for Azure CLI is one week, the minimum value is 24 hours and the maximum is three months.|Not required for Azure CLI, required for ARM template or other clients|

> [!NOTE]
> After disabling Image Cleaner, the old configuration still exists. This means that if you enable the feature again without explicitly passing configuration, the existing value will be used rather than the default.

## Enable Image Cleaner on your AKS cluster

To create a new AKS cluster using the default interval, use [az aks create][az-aks-create]:

```azurecli-interactive
az aks create -g MyResourceGroup -n MyManagedCluster \ 
  --enable-image-cleaner 
```

To enable on an existing AKS cluster, use [az aks update][az-aks-update]:

```azurecli-interactive
az aks update -g MyResourceGroup -n MyManagedCluster \ 
  --enable-image-cleaner 
```

The `--image-cleaner-interval-hours` parameter can be specified at creation time or for an existing cluster. For example, the following command updates the interval for a cluster with Image Cleaner already enabled:

```azurecli-interactive
az aks update -g MyResourceGroup -n MyManagedCluster \
  --image-cleaner-interval-hours 48
```

After the feature is enabled, the `eraser-controller-manager-xxx` pod and `collector-aks-xxx` pod will be deployed.
Based on your configuration, Image Cleaner will generate an `ImageList` containing non-running and vulnerable images at the desired interval. Image Cleaner will automatically remove these images from cluster nodes.

## Manually remove images

To manually remove images from your cluster using Image Cleaner, first create an `ImageList`. For example, save the following as `image-list.yml`:

```yml
apiVersion: eraser.sh/v1alpha1
kind: ImageList
metadata:
  name: imagelist
spec:
  images:
    - docker.io/library/alpine:3.7.3   # You can also use "*" to specify all non-running images
```

And apply it to the cluster:

```bash
kubectl apply -f image-list.yml
```

A job named `eraser-aks-xxx`will be triggered which causes Image Cleaner to remove the desired images from all nodes.

## Disable Image Cleaner

To stop using Image Cleaner, you can disable it via the `--disable-image-cleaner` flag:

```azurecli-interactive
az aks update -g MyResourceGroup -n MyManagedCluster
  --disable-image-cleaner
```

## Logging

Deletion image logs are stored in `eraser-aks-nodepool-xxx` pods for manually deleted images, and in `collector-aks-nodes-xxx` pods for automatically deleted images.

You can view these logs by running `kubectl logs <pod name> -n kubesystem`. However, this command may return only the most recent logs, since older logs are routinely deleted. To view all logs, follow these steps to enable the [Azure Monitor add-on](./monitor-aks.md) and use the Container Insights pod log table.

1. Ensure that Azure monitoring is enabled on the cluster. For detailed steps, see [Enable Container Insights for AKS cluster](../azure-monitor/containers/container-insights-enable-aks.md#existing-aks-cluster).

1. Get the Log Analytics resource ID:

   ```azurecli
   az aks show -g <resourceGroupofAKSCluster> -n <nameofAksCluster>
   ```

   After a few minutes, the command returns JSON-formatted information about the solution, including the workspace resource ID:

   ```json
   "addonProfiles": {
    "omsagent": {
      "config": {
        "logAnalyticsWorkspaceResourceID": "/subscriptions/<WorkspaceSubscription>/resourceGroups/<DefaultWorkspaceRG>/providers/Microsoft.OperationalInsights/workspaces/<defaultWorkspaceName>"
      },
      "enabled": true
    }
   }
   ```

1. In the Azure portal, search for the workspace resource ID, then select **Logs**.

1. Copy this query into the table, replacing `name` with either `eraser-aks-nodepool-xxx` (for manual mode) or `collector-aks-nodes-xxx` (for automatic mode).

   ```kusto
      let startTimestamp = ago(1h);
   KubePodInventory
   | where TimeGenerated > startTimestamp
   | project ContainerID, PodName=Name, Namespace
   | where PodName contains "name" and Namespace startswith "kube-system"
   | distinct ContainerID, PodName
   | join
   (
       ContainerLog
       | where TimeGenerated > startTimestamp
   )
   on ContainerID
   // at this point before the next pipe, columns from both tables are available to be "projected". Due to both
   // tables having a "Name" column, we assign an alias as PodName to one column which we actually want
   | project TimeGenerated, PodName, LogEntry, LogEntrySource
   | summarize by TimeGenerated, LogEntry
   | order by TimeGenerated desc
   ```

1. Select **Run**. Any deleted image logs will appear in the **Results** area.

   :::image type="content" source="media/image-cleaner/eraser-log-analytics.png" alt-text="Screenshot showing deleted image logs in the Azure portal." lightbox="media/image-cleaner/eraser-log-analytics.png":::

<!-- LINKS -->

[azure-cli-install]: /cli/azure/install-azure-cli
[azure-powershell-install]: /powershell/azure/install-az-ps

[az-aks-create]: /cli/azure/aks#az_aks_create
[az-aks-update]: /cli/azure/aks#az_aks_update
[az-feature-register]: /cli/azure/feature#az-feature-register
[register-azproviderpreviewfeature]: /powershell/module/az.resources/register-azproviderpreviewfeature
[az-feature-show]: /cli/azure/feature#az-feature-show
[get-azproviderpreviewfeature]: /powershell/module/az.resources/get-azproviderpreviewfeature
[az-provider-register]: /cli/azure/provider#az-provider-register
[register-azresourceprovider]: /powershell/module/az.resources/register-azresourceprovider

[arm-vms]: https://azure.microsoft.com/blog/azure-virtual-machines-with-ampere-altra-arm-based-processors-generally-available/
[trivy]: https://github.com/aquasecurity/trivy
