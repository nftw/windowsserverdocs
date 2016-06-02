---
title: Dfsrmig
ms.custom: na
ms.prod: windows-server-2012
ms.reviewer: na
ms.suite: na
ms.technology: 
  - techgroup-storage
ms.tgt_pltfrm: na
ms.topic: article
ms.assetid: e1b6a464-6a93-4e66-9969-04f175226d8d
author: JasonGerend
---
# Dfsrmig
The `dfsrmig` command migrates SYSVOL replication from File Replication Service \(FRS\) to Distributed File System \(DFS\) Replication, provides information about the progress of the migration, and modifies Active Directory Domain Services \(AD DS\) objects to support the migration.  
  
For examples of how to use this command, see the [Examples](#BKMK_examples) section later in this document.  
  
## Syntax  
  
```  
dfsrmig [/SetGlobalState <state> | /GetGlobalState | /GetMigrationState | /CreateGlobalObjects |   
/DeleteRoNtfrsMember [<read_only_domain_controller_name>] | /DeleteRoDfsrMember [<read_only_domain_controller_name>] | /?]  
```  
  
## Parameters  
  
|Parameter|Description|  
|-------------|---------------|  
|\/SetGlobalState <state>|Sets the desired global migration state for the domain to the state that corresponds to the value specified by *state*.<br /><br />To proceed through the migration or the rollback processes, use this command to cycle through the valid states. This option enables you to initiate and control the migration process by setting the global migration state in AD DS on the PDC emulator. If the PDC emulator is not available, this command fails.<br /><br />For information about the states for migration and rollback, see [SYSVOL Migration States](assetId:///ce98f69f-4b1c-4249-b526-1631a614e5eb). You can only set the global migration state to a stable state. The valid values for *state*, therefore, are **0** for the Start state, **1** for the Prepared state, **2** for the Redirected state, and **3** for the Eliminated state.<br /><br />Migration to the Eliminated state is irreversible and rollback from that state is not possible, so use a value of **3** for *state* only when you are fully committed to using DFS Replication for SYSVOL replication.|  
|\/GetGlobalState|Retrieves the current global migration state for the domain from the local copy of the AD DS database, when run on the PDC emulator.<br /><br />Use this option to confirm that you set the correct global migration state. Only stable migration states can be global migration states, so the results that the **dfsrmig** command reports with the **\/GetGlobalState** option correspond to the states you can set with the **\/SetGlobalState** option.<br /><br />You should run the **dfsrmig** command with the **\/GetGlobalState** option only on the PDC emulator. Active Directory replication replicates the global state to the other domain controllers in the domain, but replication latencies can cause inconsistencies if you run the **dfsrmig** command with the **\/GetGlobalState** option on a domain controller other than the PDC emulator. To check the local migration status of a domain controller other than the PDC emulator, use the **\/GetMigrationState** option instead.|  
|\/GetMigrationState|Retrieves the current local migration state for all domain controllers in the domain, and determines whether those local states match the current global migration state.<br /><br />Use this option to determine if all domain controllers have reached the global migration state. The output of the **dsfrmig** command when you use the **\/GetMigrationState** option indicates whether or not migration to the current global state is complete, and it lists the local migration state for any domain controllers that have not reached the current global migration state. Local migration state for domain controllers can include transition states for domain controllers that have not reached the current global migration state.|  
|\/CreateGlobalObjects|Creates the global objects and settings in AD DS that DFS Replication uses.<br /><br />You should not need to use this option during a normal migration process, because the DFS Replication service automatically creates these AD DS objects and settings during the migration from the Start state to the Prepared state. Use this option to manually create these objects and settings in the following situations:<br /><br />-   **A new read\-only domain controller is promoted during migration**. The DFS Replication service automatically creates the AD DS objects and settings for DFS Replication during the migration from the Start state to the Prepared state. If a new read\-only domain controller is promoted in the domain after this transition, but before migration to the Eliminated state, then the objects that correspond to the newly activated read\-only domain controller are not created in AD DS causing replication and migration to fail.<br />-   In this case, you can run the **dfsrmig** command with the **\/CreateGlobalObjects** option to manually create the objects on any read\-only domain controllers that do not already have them. Running this command does not affect the domain controllers that already have the objects and settings for the DFS Replication service.<br />-   **The global settings for the DFS Replication service are missing or were deleted**. If these settings are missing for a particular domain controller, migration from the Start state to the Prepared state stalls at the Preparing transition state for the domain controller. In this case, you can use the **dfsrmig** command with the **\/CreateGlobalObjects** option to manually create the settings. **Note:** Because the global AD DS settings for the DFS Replication service for a read\-only domain controller are created on the PDC emulator, these settings need to replicate to the read\-only domain controller from the PDC emulator before the DFS Replication service on the read\-only domain controller can use these settings. Because of Active Directory replication latencies, this replication can take some time to occur.|  
|\/DeleteRoNtfrsMember \[<read\_only\_domain\_controller\_name>\]|Deletes the global AD DS settings for FRS replication that correspond to the specified read\-only domain controller, or deletes the global AD DS settings for FRS replication for all read\-only domain controllers if no value is specified for *read\_only\_domain\_controller\_name*.<br /><br />You should not need to use this option during a normal migration process, because the DFS Replication service automatically deletes these AD DS settings during the migration from the Redirected state to the Eliminated state. Because read\-only domain controllers cannot delete these settings from AD DS, the PDC emulator performs this operation, and the changes eventually replicate to the read\-only domain controllers after the applicable latencies for Active Directory replication.<br /><br />You use this option to manually delete the AD DS settings only when the automatic deletion fails on a read\-only domain controller and stalls the read\-only domain controller for a long time during the migration from the Redirected state to the Eliminated state.|  
|\/DeleteRoDfsrMember \[<read\_only\_domain\_controller\_name>\]|Deletes the global AD DS settings for DFS Replication that correspond to the specified read\-only domain controller, or deletes the global AD DS settings for DFS Replication for all read\-only domain controllers if no value is specified for *read\_only\_domain\_controller\_name*.<br /><br />Use this option to manually delete the AD DS settings only when the automatic deletion fails on a read\-only domain controller and stalls the read\-only domain controller for a long time when rolling back the migration from the Prepared state to the Start state.|  
|\/?|Displays Help at the command prompt. Equivalent to running **dfsrmig** without any options.|  
  
## Remarks  
  
-   Dfsrmig.exe, the migration tool for the DFS Replication service, is installed with the DFS Replication service.  
  
    For a new [!INCLUDE[nextref_longhorn](includes/nextref_longhorn_md.md)] server, Dcpromo.exe installs and starts the DFS Replication service when you promote the computer to a domain controller. When you upgrade a server from Windows Server 2003 to [!INCLUDE[nextref_longhorn](includes/nextref_longhorn_md.md)], the upgrade process installs and starts the DFS Replication service. You do not need to install the DFS Replication role service to have the DFS Replication service installed and started.  
  
-   The **dfsrmig** tool is supported only on domain controllers that run at the [!INCLUDE[nextref_longhorn](includes/nextref_longhorn_md.md)] domain functional level, because SYSVOL migration from FRS to DFS Replication is only possible on domain controllers that operate at the [!INCLUDE[nextref_longhorn](includes/nextref_longhorn_md.md)] domain functional level.  
  
-   You can run the **dfsrmig** command on any domain controller, but operations that create or manipulate AD DS objects are only allowed on read\-write capable domain controllers \(not on read\-only domain controllers\).  
  
-   Running **dfsrmig** without any options displays Help at the command prompt.  
  
## <a name="BKMK_examples"></a>Examples  
To set the global migration state to prepared \(**1**\) and initiate migration to or rollback from the Prepared state, type:  
  
```  
dfsrmig /SetGlobalState 1  
```  
  
To set the global migration state to start \(**0**\) and initiate rollback to the Start state, type:  
  
```  
dfsrmig /SetGlobalState 0  
```  
  
To display the global migration state, type:  
  
```  
dfsrmig /GetGlobalState  
```  
  
This example shows typical output from the **dfsrmig \/GetGlobalState** command.  
  
```  
Current DFSR global state: ‘Prepared’  
Succeeded.  
```  
  
To display the information about whether the local migration states on all of the domain controllers match the global migration state and the local migration states for any domain controllers where the local state does not match the global state, type:  
  
```  
dfsrmig /GetMigrationState  
```  
  
This example shows typical output from the **dfsrmig \/GetMigrationState** command when the local migration states on all of the domain controllers match the global migration state.  
  
```  
All Domain Controllers have migrated successfully to Global state (‘Prepared’).  
Migration has reached a consistent state on all Domain Controllers.  
Succeeded.  
```  
  
This example shows typical output from the **dfsrmig \/GetMigrationState** command when the local migration states on some domain controllers do not match the global migration state.  
  
```  
The following Domain Controllers are not in sync with Global state (‘Prepared’):  
  
Domain Controller (Local Migration State) – DC Type  
=========  
CONTOSO-DC2 (‘Start’) – ReadOnly DC  
CONTOSO-DC3 (‘Preparing’) – Writable DC  
  
Migration has not yet reached a consistent state on all domain controllers  
State information might be stale due to AD latency.  
```  
  
To create the global objects and settings that DFS Replication uses in AD DS on domain controllers where those settings were not created automatically during migration or where those settings are missing, type:  
  
```  
dfsrmig /CreateGlobalObjects  
```  
  
To delete the global AD DS settings for FRS replication for a read\-only domain controller named contoso\-dc2 if those settings were not deleted automatically deleted by the migration process, type:  
  
```  
dfsrmig /DeleteRoNtfrsMember contoso-dc2  
```  
  
To delete the global AD DS settings for FRS replication for all read\-only domain controllers if those settings were not deleted automatically by the migration process, type:  
  
```  
dfsrmig /DeleteRoNtfrsMember  
```  
  
To delete the global AD DS settings for DFS Replication for a read\-only domain controller named contoso\-dc2 if those settings were not deleted automatically by the migration process, type:  
  
```  
dfsrmig /DeleteRoDfsrMember contoso-dc2  
```  
  
To delete the global AD DS settings for DFS Replication for all read\-only domain controllers if those settings were not deleted automatically by the migration process, type:  
  
```  
dfsrmig /DeleteRoDfsrMember  
```  
  
## Additional references  
[Command\-Line Syntax Key](http://go.microsoft.com/fwlink/?LinkId=122056)  
  
[SYSVOL Replication Migration Guide: FRS to DFS Replication](assetId:///458d7d84-a8bd-4111-9156-953cadb52728)  
  
[Migrating to the Prepared State](assetId:///e33aff1a-8062-4c8a-9354-dc8476155e09)  
  
[Migrating to the Redirected State](assetId:///ee3510ce-12c7-4f33-93da-3b8dc1afb49e)  
  
[Migrating to the Eliminated State](assetId:///65642101-6ba6-4769-8eee-7daa77245b47)  
  
[Rolling Back SYSVOL Migration to a Previous Stable State](assetId:///8c9a84c7-59d5-4daf-b66c-4073aa51ebb8)  
  
[SYSVOL Migration Series: Part 2–Dfsrmig.exe: The SYSVOL Migration Tool](http://go.microsoft.com/fwlink/?LinkID=121757)  
  
