---
title: Use Cluster Shared Volumes in a Failover Cluster in Windows Threshold
ms.custom: na
ms.reviewer: na
ms.suite: na
ms.tgt_pltfrm: na
ms.topic: article
ms.assetid: ce57fdf2-2760-4412-bb43-dadbcff6dc73
---
# Use Cluster Shared Volumes in a Failover Cluster in Windows Threshold
Cluster Shared Volumes \(CSV\) enable multiple nodes in a failover cluster to simultaneously have read\-write access to the same LUN \(disk\) that is provisioned as an NTFS volume. \(In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], the disk can be provisioned as NTFS or Resilient File System \(ReFS\).\) With CSV, clustered roles can fail over quickly from one node to another node without requiring a change in drive ownership, or dismounting and remounting a volume. CSV also help simplify the management of a potentially large number of LUNs in a failover cluster.

CSV provide a general\-purpose, clustered file system, which is layered above NTFS \(or ReFS in [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)]\). CSV applications include:

-   Clustered virtual hard disk \(VHD\) files for clustered Hyper\-V virtual machines

-   Scale\-out file shares to store application data for the Scale\-Out File Server clustered role. Examples of the application data for this role include Hyper\-V virtual machine files and Microsoft SQL Server data. \(Be aware that ReFS is not supported for a Scale\-Out File Server.\) For more information about Scale\-Out File Server, see [Scale-Out File Server for Application Data Overview](assetId:///0a6029b2-9390-414f-b486-98d31d033ff0).

> [!NOTE]
> CSV does not support the Microsoft SQL Server clustered workload in SQL Server 2012 and earlier versions of SQL Server.

In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], CSV functionality was significantly enhanced. For example, dependencies on Active Directory Domain Services were removed. Support was added for the functional improvements in **chkdsk**, for interoperability with antivirus and backup applications, and for integration with general storage features such as BitLocker\-encrypted volumes and Storage Spaces. For an overview of CSV functionality that was introduced in [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], see [What's New in Failover Clustering in Windows Server 2012 [redirected]](assetId:///187d6191-4f92-4f98-9cae-c5e6d5b74e76).

[!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)] introduces additional functionality, such as distributed CSV ownership, increased resiliency through availability of the Server service, greater flexibility in the amount of physical memory that you can allocate to CSV cache, better diagnosibility, and enhanced interoperability that includes support for ReFS and deduplication. For more information, see [What's New in Failover Clustering in Windows Server 2012 R2](assetId:///444c4ac3-9866-41ba-8888-8724777b8f0a).

> [!NOTE]
> For information about using data deduplication on CSV for Virtual Desktop Infrastructure \(VDI\) scenarios, see the blog posts [Deploying Data Deduplication for VDI storage in Windows Server 2012 R2](http://blogs.technet.com/b/filecab/archive/2013/07/31/deploying-data-deduplication-for-vdi-storage-in-windows-server-2012-r2.aspx) and [Extending Data Deduplication to new workloads in Windows Server 2012 R2](http://blogs.technet.com/b/filecab/archive/2013/07/31/extending-data-deduplication-to-new-workloads-in-windows-server-2012-r2.aspx).

**In this topic**

-   [Review requirements and considerations for using CSV in a failover cluster](#BKMK_Reqs)

    -   [Network configuration considerations](#BKMK_network)

    -   [Storage and disk configuration requirements](#BKMK_storage)

    -   [Node requirements](#BKMK_node)

-   [Plan to use CSV in a failover cluster](#BKMK_planning)

    -   [Arrangement of LUNs, volumes, and VHD files](#BKMK_LUNs)

    -   [Number and size of LUNs and volumes for CSVs](#BKMK_loads)

-   [Add a disk as a CSV to a failover cluster](#BKMK_deploy)

-   [Enable CSV block cache for read-intensive workloads (optional)](#BKMK_cache)

-   [Back up CSV](#BKMK_backup)

## <a name="BKMK_Reqs"></a>Review requirements and considerations for using CSV in a failover cluster
Before using CSV in a failover cluster, review the network, storage, and other requirements and considerations in this section.

**In this section**

-   [Network configuration considerations](#BKMK_network)

-   [Storage and disk configuration requirements](#BKMK_storage)

-   [Node requirements](#BKMK_node)

### <a name="BKMK_network"></a>Network configuration considerations
Consider the following when you configure the networks that support CSV.

-   **Multiple networks and multiple network adapters**. To enable fault tolerance in the event of a network failure, we recommend that multiple cluster networks carry CSV traffic or that you configure teamed network adapters.

    If the cluster nodes are connected to networks that should not be used by the cluster, you should disable them. For example, we recommend that you disable iSCSI networks for cluster use to prevent CSV traffic on those networks. To disable a network, in Failover Cluster Manager, click **Networks**, click the network, click the **Properties** action, and then select **Do not allow cluster network communication on this network**. Alternatively, you can configure the **Role** property of the network by using the [Get-ClusterNetwork](http://technet.microsoft.com/library/hh847303)[!INCLUDE[wps_2](includes/wps_2_md.md)] cmdlet.

-   **Network adapter properties**. In the properties for all adapters that carry cluster communication, make sure that the following settings are enabled:

    -   **Client for Microsoft Networks** and **File and Printer Sharing for Microsoft Networks**. These settings support Server Message Block \(SMB\) 3.0, which is used by default to carry CSV traffic between nodes. To enable SMB, also ensure that the Server service and the Workstation service are running and that they are configured to start automatically on each cluster node.

        > [!NOTE]
        > In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], there are multiple Server service instances per failover cluster node. There is the default instance that handles incoming traffic from SMB clients that access regular file shares, and a second CSV instance that handles only inter\-node CSV traffic. Also, if the Server service on a node becomes unhealthy, CSV ownership automatically transitions to another node.

        SMB 3.0 includes the SMB Multichannel and SMB Direct features, which enable CSV traffic to stream across multiple networks in the cluster and to leverage network adapters that support Remote Direct Memory Access \(RDMA\). By default, SMB Multichannel is used for CSV traffic. For more information, see [Server Message Block overview](assetId:///d5399314-77e2-4d73-889d-d62f062cb0cc).

    -   **Microsoft Failover Cluster Virtual Adapter Performance Filter**. This setting improves the ability of nodes to perform I\/O redirection when it is required to reach CSV, for example, when a connectivity failure prevents a node from connecting directly to the CSV disk. For more information, see [About I/O synchronization and redirected I/O mode in CSV communication](#BKMK_redirected) in this topic.

-   **Cluster network prioritization**. We generally recommend that you do not change the cluster\-configured preferences for the networks.

-   **IP subnet configuration**. No specific subnet configuration is required for nodes in a network that use CSV. CSV can support multisubnet clusters.

-   **Policy\-based Quality of Service \(QoS\)**. We recommend that you configure a QoS priority policy and a minimum bandwidth policy for network traffic to each node when you use CSV. For more information, see [Policy-based QoS](assetId:///aec1ca7c-9f81-4789-ad02-20d9084ead10).

-   **Storage network**. For storage network recommendations, review the guidelines that are provided by your storage vendor. For additional considerations about storage for CSV, see [Storage and disk configuration requirements for using CSVs](#BKMK_storage), later in this topic.

For an overview of the hardware, network, and storage requirements for failover clusters, see [Failover Clustering Hardware Requirements and Storage Options](assetId:///c72342a0-7bf8-4e42-b8d2-b4a48659ba7c).

#### <a name="BKMK_redirected"></a>About I\/O synchronization and I\/O redirection in CSV communication
**I\/O synchronization** CSV enable multiple nodes to have simultaneous read\-write access to the same shared storage. When a node performs disk input\/output \(I\/O\) on a CSV volume, the node communicates directly with the storage, for example, through a storage area network \(SAN\). However, at any time, a single node \(called the coordinator node\) “owns” the physical disk resource that is associated with the LUN. The coordinator node for a CSV volume is displayed in Failover Cluster Manager as **Owner Node** under **Disks**. It also appears in the output of the [Get-ClusterSharedVolume](http://technet.microsoft.com/library/hh847282)[!INCLUDE[wps_2](includes/wps_2_md.md)] cmdlet.

> [!NOTE]
> In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], CSV ownership is evenly distributed across the failover cluster nodes based on the number of CSV volumes that each node owns. Additionally, ownership is automatically rebalanced when there are conditions such as CSV failover, a node rejoins the cluster, you add a new node to the cluster, you restart a cluster node, or you start the failover cluster after it has been shut down.

When certain small changes occur in the file system on a CSV volume, this metadata must be synchronized on each of the physical nodes that access the LUN, not only on the single coordinator node. For example, when a virtual machine on a CSV volume is started, created, or deleted, or when a virtual machine is migrated, this information needs to be synchronized on each of the physical nodes that access the virtual machine. These metadata update operations occur in parallel across the cluster networks by using SMB 3.0. These operations do not require all the physical nodes to communicate with the shared storage.

**I\/O redirection** Storage connectivity failures and certain storage operations can prevent a given node from communicating directly with the storage. To maintain function while the node is not communicating with the storage, the node redirects the disk I\/O through a cluster network to the coordinator node where the disk is currently mounted. If the current coordinator node experiences a storage connectivity failure, all disk I\/O operations are queued temporarily while a new node is established as a coordinator node.

The server uses one of the following I\/O redirection modes, depending on the situation:

-   **File system redirection** Redirection is per volume—for example, when CSV snapshots are taken by a backup application when a CSV volume is manually placed in redirected I\/O mode.

-   **Block redirection** Redirection is at the file\-block level—for example, when storage connectivity is lost to a volume. Block redirection is significantly faster than file system redirection.

In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], you can view the state of a CSV volume on a per node basis. For example, you can see whether I\/O is direct or redirected, or whether the CSV volume is unavailable. If a CSV volume is in I\/O redirected mode, you can also view the reason. Use the Windows PowerShell cmdlet **Get\-ClusterSharedVolumeState** to view this information.

> [!NOTE]
> -   In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], because of improvements in CSV design, CSV perform more operations in direct I\/O mode than occurred in [!INCLUDE[nextref_server_7](includes/nextref_server_7_md.md)].
> -   Because of the integration of CSV with SMB 3.0 features such as SMB Multichannel and SMB Direct, redirected I\/O traffic can stream across multiple cluster networks.
> -   You should plan your cluster networks to allow for the potential increase in network traffic to the coordinator node during I\/O redirection.

### <a name="BKMK_storage"></a>Storage and disk configuration requirements
To use CSV, your storage and disks must meet the following requirements:

-   **File system format**. In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], a disk or storage space for a CSV volume must be a basic disk that is partitioned with NTFS or ReFS. In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], a disk or storage space for a CSV volume must be a basic disk that is partitioned with NTFS.

    A CSV has the following additional requirements:

    -   In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], you cannot use a disk for a CSV that is formatted with FAT or FAT32.

    -   In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], you cannot use a disk for a CSV that is formatted with FAT, FAT32, or ReFS.

    -   If you want to use a storage space for a CSV, you can configure a simple space or a mirror space. In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], you can also configure a parity space. \(In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], CSV does not support parity spaces.\)

    -   A CSV cannot be used as a quorum witness disk. For more information about the cluster quorum, see [Configure and manage the quorum in a failover cluster](assetId:///e5cef0e4-cf8e-48be-a5bc-2182d416fab1).

    -   After you add a disk as a CSV, it is designated in the CSVFS format \(for CSV File System\). This allows the cluster and other software to differentiate the CSV storage from other NTFS or ReFS storage. Generally, CSVFS supports the same functionality as NTFS or ReFS. However, certain features are not supported. For example, in [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], you cannot enable compression on CSV. In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], you cannot enable data deduplication or compression on CSV.

-   **Resource type in the cluster**. For a CSV volume, you must use the Physical Disk resource type. By default, a disk or storage space that is added to cluster storage is automatically configured in this way.

-   **Choice of CSV disks or other disks in cluster storage**. When choosing one or more disks for a clustered virtual machine, consider how each disk will be used. If a disk will be used to store files that are created by Hyper\-V, such as VHD files or configuration files, you can choose from the CSV disks or the other available disks in cluster storage. If a disk will be a physical disk that is directly attached to the virtual machine \(also called a pass\-through disk\), you cannot choose a CSV disk, and you must choose from the other available disks in cluster storage.

-   **Path name for identifying disks**. Disks in CSV are identified with a path name. Each path appears to be on the system drive of the node as a numbered volume under the **\\ClusterStorage** folder. This path is the same when viewed from any node in the cluster. You can rename the volumes if needed.

For storage requirements for CSV, review the guidelines that are provided by your storage vendor. For additional storage planning considerations for CSV, see [Plan to use CSV in a failover cluster](#BKMK_planning) later in this topic.

### <a name="BKMK_node"></a>Node requirements
To use CSV, your nodes must meet the following requirements:

-   **Drive letter of system disk**. On all nodes, the drive letter for the system disk must be the same.

-   **Authentication protocol**. The NTLM protocol must be enabled on all nodes. This is enabled by default.

## <a name="BKMK_planning"></a>Plan to use CSV in a failover cluster
This section lists planning considerations and recommendations for using CSV in a failover cluster running [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)] or [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)].

> [!IMPORTANT]
> Ask your storage vendor for recommendations about how to configure your specific storage unit for CSV. If the recommendations from the storage vendor differ from information in this topic, use the recommendations from the storage vendor.

**In this section**

-   [Arrangement of LUNs, volumes, and VHD files](#BKMK_LUNs)

-   [Number and size of LUNs and volumes](#BKMK_loads)

### <a name="BKMK_LUNs"></a>Arrangement of LUNs, volumes, and VHD files
To make the best use of CSV to provide storage for clustered virtual machines, it is helpful to review how you would arrange the LUNs \(disks\) when you configure physical servers. When you configure the corresponding virtual machines, try to arrange the VHD files in a similar way.

Consider a physical server for which you would organize the disks and files as follows:

-   System files, including a page file, on one physical disk

-   Data files on another physical disk

For an equivalent clustered virtual machine, you should organize the volumes and files in a similar way:

-   System files, including a page file, in a VHD file on one CSV

-   Data files in a VHD file on another CSV

If you add another virtual machine, where possible, you should keep the same arrangement for the VHDs on that virtual machine.

### <a name="BKMK_loads"></a>Number and size of LUNs and volumes
When you plan the storage configuration for a failover cluster that uses CSV, consider the following recommendations:

-   To decide how many LUNs to configure, consult your storage vendor. For example, your storage vendor may recommend that you configure each LUN with one partition and place one CSV volume on it.

-   There are no limitations for the number of virtual machines that can be supported on a single CSV volume. However, you should consider the number of virtual machines that you plan to have in the cluster and the workload \(I\/O operations per second\) for each virtual machine. Consider the following examples:

    -   One organization is deploying virtual machines that will support a virtual desktop infrastructure \(VDI\), which is a relatively light workload. The cluster uses high\-performance storage. The cluster administrator, after consulting with the storage vendor, decides to place a relatively large number of virtual machines per CSV volume.

    -   Another organization is deploying a large number of virtual machines that will support a heavily used database application, which is a heavier workload. The cluster uses lower\-performing storage. The cluster administrator, after consulting with the storage vendor, decides to place a relatively small number of virtual machines per CSV volume.

-   When you plan the storage configuration for a particular virtual machine, consider the disk requirements of the service, application, or role that the virtual machine will support. Understanding these requirements helps you avoid disk contention that can result in poor performance. The storage configuration for the virtual machine should closely resemble the storage configuration that you would use for a physical server that is running the same service, application, or role. For more information, see [Plan the arrangement of LUNs, volumes, and VHD files](#BKMK_LUNs) earlier in this topic.

    You can also mitigate disk contention by having storage with a large number of independent physical hard disks. Choose your storage hardware accordingly, and consult with your vendor to optimize the performance of your storage.

-   Depending on your cluster workloads and their need for I\/O operations, you can consider configuring only a percentage of the virtual machines to access each LUN, while other virtual machines do not have connectivity and are instead dedicated to compute operations.

## <a name="BKMK_deploy"></a>Add a disk to CSV on a failover cluster
The CSV feature is enabled by default in Failover Clustering. To add a disk to CSV, you must add a disk to the **Available Storage** group of the cluster \(if it is not already added\), and then add the disk to CSV on the cluster. You can use Failover Cluster Manager or the Failover Clusters [!INCLUDE[wps_2](includes/wps_2_md.md)] cmdlets to perform these procedures.

#### To add a disk to Available Storage

1.  In Failover Cluster Manager, in the console tree, expand the name of the cluster, and then expand **Storage**.

2.  Right\-click **Disks**, and then click **Add Disk**. A list appears showing the disks that can be added for use in a failover cluster.

3.  Select the disk or disks you want to add, and then click **OK**.

    The disks are now assigned to the **Available Storage** group.

![](media/PowerShellLogoSmall.gif)**[!INCLUDE[wps_proc_title](includes/wps_proc_title_md.md)]**

[!INCLUDE[wps_proc_intro](includes/wps_proc_intro_md.md)]

The following example identifies the disks that are ready to be added to the cluster, and then adds them to the **Available Storage** group.

```powershell
Get-ClusterAvailableDisk | Add-ClusterDisk
```

#### To add a disk in Available Storage to CSV

1.  In Failover Cluster Manager, in the console tree, expand the name of the cluster, expand **Storage**, and then click **Disks**.

2.  Select one or more disks that are assigned to **Available Storage**, right\-click the selection, and then click **Add to Cluster Shared Volumes**.

    The disks are now assigned to the **Cluster Shared Volume** group in the cluster. The disks are exposed to each cluster node as numbered volumes \(mount points\) under the %SystemDisk%ClusterStorage folder. The volumes appear in the CSVFS file system.

> [!NOTE]
> You can rename CSV volumes in the %SystemDisk%ClusterStorage folder.

![](media/PowerShellLogoSmall.gif)**[!INCLUDE[wps_proc_title](includes/wps_proc_title_md.md)]**

[!INCLUDE[wps_proc_intro](includes/wps_proc_intro_md.md)]

The following example adds *Cluster Disk 1* in **Available Storage** to CSV on the local cluster.

```powershell
Add-ClusterSharedVolume –Name "Cluster Disk 1"
```

## <a name="BKMK_cache"></a>Enable the CSV cache for read\-intensive workloads \(optional\)
The CSV cache provides caching at the block level of read\-only unbuffered I\/O operations by allocating system memory \(RAM\) as a write\-through cache. \(Unbuffered I\/O operations are not cached by the cache manager.\) This can improve performance for applications such as Hyper\-V, which conducts unbuffered I\/O operations when accessing a VHD. The CSV cache can boost the performance of read requests without caching write requests. Enabling the CSV cache is also useful for Scale\-Out File Server scenarios.

> [!NOTE]
> We recommend that you enable the CSV cache for all clustered Hyper\-V and Scale\-Out File Server deployments.

By default in [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], the CSV cache is disabled. In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], the CSV cache is enabled by default. However, you must still allocate the size of the block cache to reserve.

The following table describes the two configuration settings that control the CSV cache.

|Property Name in [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)]|Property Name in [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)]|Description|
|----------------------------------------------------------------------------------|----------------------------------------------------------------------------|---------------|
|**BlockCacheSize**|**SharedVolumeBlockCacheSizeInMB**|This is a cluster common property that allows you to define how much memory \(in megabytes\) to reserve for the CSV cache on each node in the cluster. For example, if a value of 512 is defined, then 512 MB of system memory is reserved on each node. \(In many clusters, 512 MB is a recommended value.\) The default setting is 0 \(for disabled\).|
|**EnableBlockCache**|**CsvEnableBlockCache**|This is a private property of the cluster Physical Disk resource. It allows you to enable CSV cache on an individual disk that is added to CSV. In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], the default setting is 0 \(for disabled\). To enable CSV cache on a disk, configure a value of 1. By default, in [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], this setting is enabled.|

You can monitor the CSV cache in Performance Monitor by adding the counters under **Cluster CSV Volume Cache**.

#### To configure the CSV cache

1.  Start [!INCLUDE[wps_2](includes/wps_2_md.md)] as an administrator.

2.  To define a cache of *512* MB to be reserved on each node, type the following:

    -   For [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)]:

        ```powershell
        (Get-Cluster).BlockCacheSize = 512
        ```

    -   For [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)]:

        ```powershell
        (Get-Cluster).SharedVolumeBlockCacheSizeInMB = 512
        ```

3.  In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], to enable the CSV cache on a CSV named *Cluster Disk 1*, type the following:

    ```powershell
    Get-ClusterSharedVolume "Cluster Disk 1" | Set-ClusterParameter CsvEnableBlockCache 1
    ```

> [!NOTE]
> -   In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], you can allocate only 20% of the total physical RAM to the CSV cache. In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], you can allocate up to 80%. Because Scale\-Out File Servers are not typically memory constrained, you can accomplish large performance gains by using the extra memory for the CSV cache.
> -   In [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)], to avoid resource contention, you should restart each node in the cluster after you modify the memory that is allocated to the CSV cache. In [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], a restart is no longer required.
> -   After you enable or disable CSV cache on an individual disk, for the setting to take effect, you must take the Physical Disk resource offline and bring it back online. \(By default, in [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)], the CSV cache is enabled.\)
> -   For more information about CSV cache that includes information about performance counters, see the blog post [How to Enable CSV Cache](http://blogs.msdn.com/b/clustering/archive/2012/03/22/10286676.aspx).

## <a name="BKMK_backup"></a>Back up CSV
There are multiple methods to back up information that is stored on CSV in a failover cluster. You can use a Microsoft backup application or a non\-Microsoft application. In general, CSV do not impose special backup requirements beyond those for clustered storage formatted with NTFS or ReFS. CSV backups also do not disrupt other CSV storage operations.

You should consider the following factors when you select a backup application and backup schedule for CSV:

-   Volume\-level backup of a CSV volume can be run from any node that connects to the CSV volume.

-   Your backup application can use software snapshots or hardware snapshots. Depending on the ability of your backup application to support them, backups can use application\-consistent and crash\-consistent Volume Shadow Copy Service \(VSS\) snapshots.

-   If you are backing up CSV that have multiple running virtual machines, you should generally choose a management operating system\-based backup method. If your backup application supports it, multiple virtual machines can be backed up simultaneously.

-   CSV support backup requestors that are running [!INCLUDE[winblue_server_2](includes/winblue_server_2_md.md)] Backup, [!INCLUDE[win8_server_2](includes/win8_server_2_md.md)] Backup or [!INCLUDE[nextref_server_7](includes/nextref_server_7_md.md)] Backup. However, Windows Server Backup generally provides only a basic backup solution that may not be suited for organizations with larger clusters. Windows Server Backup does not support application\-consistent virtual machine backup on CSV. It supports crash\-consistent volume\-level backup only. \(If you restore a crash\-consistent backup, the virtual machine will be in the same state it was if the virtual machine had crashed at the exact moment that the backup was taken.\) A backup of a virtual machine on a CSV volume will succeed, but an error event will be logged indicating that this is not supported.

-   You may require administrative credentials when backing up a failover cluster.

> [!IMPORTANT]
> Be sure to carefully review what data your backup application backs up and restores, which CSV features it supports, and the resource requirements for the application on each cluster node.

> [!CAUTION]
> If you need to restore the backup data onto a CSV volume, be aware of the capabilities and limitations of the backup application to maintain and restore application\-consistent data across the cluster nodes. For example, with some applications, if the CSV is restored on a node that is different from the node where the CSV volume was backed up, you might inadvertently overwrite important data about the application state on the node where the restore is taking place.

## See also

-   [Failover Clustering Overview](assetId:///6eeffe4f-3558-495b-bcea-c640fe4d6c49)

-   [Deploy Clustered Storage Spaces](assetId:///7b33931e-7971-49b0-b385-b1e5a90d94fe)

