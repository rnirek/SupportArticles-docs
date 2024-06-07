---
title: Troubleshoot Azure Fence Agent Issues in SLES
description: Provides troubleshooting guidance for Azure Fence Agent failing to start
author: rnirek
ms.author: rnirek
ms.topic: troubleshooting
ms.date: 05/30/20204
ms.service: virtual-machines
ms.collection: linux
---

<!---Recommended: Remove all the comments in this template before you sign-off or merge to the main branch.--->

# Overview <!-- Write a title descriptive enough to cover the specific issue to troubleshoot. This H1 is required and is the only one to exist. -->

The Azure fence agent makes use of the python program located at /usr/sbin/fence_azure_arm. The cluster RA used to implement STONITH calls this program with the appropriate parameters and uses it to communicate with the Azure platform using API calls.

As documented in [SUSE - Create Azure Fence agent STONITH device](https://learn.microsoft.com/en-us/azure/sap/workloads/high-availability-guide-suse-pacemaker?tabs=msi#create-azure-fence-agent-stonith-device), the roles provide permissions to the fence agent to perform the following actions.

1. powerOff
2. start
   
When the VM is detected as being unhealthy, the fence agent uses the above actions to power off the VM and then start it up again thus providing a STONITH device.

## Prerequisites

Pacemaker clusters running on SLES and using Azure fence agent  for STONITH purposes.

## Description 

Azure Fencing Agent resource fails to start and reports "unknown error" and shows in "stopped" state.

```VM1:/home/azureadmin1 # crm status
Stack: corosync
Current DC: VM2 (version 2.0.1+20190417.13d370ca9-3.6.1-2.0.1+20190417.13d370ca9) - partition with quorum
Last updated: Mon Apr  6 13:58:59 2020
Last change: Mon Apr  6 13:58:53 2020 by root via crm_attribute on VM1
 
2 nodes configured
7 resources configured
 
Online: [ VM1 VM2 ]
 
Full list of resources:
 
Clone Set: cln_SAPHanaTopology_SS2_HDB00 [rsc_SAPHanaTopology_SS2_HDB00]
	 Started: [ VM1 VM2 ]
Clone Set: msl_SAPHana_SS2_HDB00 [rsc_SAPHana_SS2_HDB00] (promotable)
	 Masters: [ VM1 ]
	 Slaves: [ VM2 ]
Resource Group: g_ip_SS2_HDB00
	 rsc_ip_SS2_HDB00   (ocf::heartbeat:IPaddr2):       Started VM1
	 rsc_nc_SS2_HDB00   (ocf::heartbeat:azure-lb):      Started VM1
rsc_st_azure   (stonith:fence_azure_arm):      Stopped
 
Failed Resource Actions:
* rsc_st_azure_start_0 on VM2 'unknown error' (1): call=102, status=complete, exitreason='',
	last-rc-change='Mon Apr  6 13:50:57 2020', queued=0ms, exec=1790ms
* rsc_st_azure_start_0 on VM1 'unknown error' (1): call=121, status=complete, exitreason='',
	last-rc-change='Mon Apr  6 13:50:59 2020', queued=0ms, exec=1760ms
```

## Scenario 1: <!--Required: Here goes the Scenario short title-->
```/var/log/messages
2021-03-15T20:23:15.441083+00:00 NodeName pacemaker-fenced[2550]:  warning: fence_azure_arm[21839] stderr: [ 2021-03-15 20:23:15,398 ERROR: Failed: Azure Error: AuthenticationFailed ]
2021-03-15T20:23:15.441260+00:00 NodeName pacemaker-fenced[2550]:  warning: fence_azure_arm[21839] stderr: [ Message: Authentication failed. ]
2021-03-15T20:23:15.441668+00:00 NodeName pacemaker-fenced[2550]:  warning: fence_azure_arm[21839] stderr: [  ]
```
### Resolution:

1. Connectivity to the Azure management API public endpoints

As documented in Public endpoint connectivity for Virtual Machines using [Azure Standard Load Balancer in SAP high-availability scenarios](https://learn.microsoft.com/en-us/azure/sap/workloads/high-availability-guide-standard-load-balancer-outbound-connections#option-3-using-proxy-for-pacemaker-calls-to-azure-management-api), it is essential to check outbound connectivity to the Azure management API available at the URLs below:
```
  https://management.azure.com
  https://login.microsoftonline.com
```
This can be done by using either telnet, curl or nc (telnet or curl are not normally available on customer VMs) to test connectivity. The tests to both URLs must be done as shown below.

Connectivity to https://management.azure.com 
```
telnet management.azure.com 443
OR
curl -v telnet://management.azure.com:443
OR
nc -z -v management.azure.com 443
```
Connectivity to https://login.microsoftonline.com 
```
telnet login.microsoftonline.com 443
OR
curl -v telnet://login.microsoftonline.com:443
OR
nc -z -v login.microsoftonline.com 443
```

2. Valid information set up in username/password for the STONITH resource
One of the major causes of the STONITH resource failing is the use of invalid values for the username or password if using Managed Identity. This can be tested using the fence_azure_arm command as shown below. The values for username and password are those created per the document [SUSE - Create Azure Fence agent STONITH device](https://learn.microsoft.com/en-us/azure/sap/workloads/high-availability-guide-suse-pacemaker?tabs=msi#create-azure-fence-agent-stonith-device), as appropriate for the customer distribution.

```/usr/sbin/fence_azure_arm --action=list --username='<user name>' --password='<password>' --tenantId=<tenant ID> --resourceGroup=<resource group> ```

This command should return the nodenames of the VMs in the cluster.

If the command does not return successfully, it should be re-run with the -v flag to enable verbose output and -D flag to enable debug output as shown below.

```/usr/sbin/fence_azure_arm --action=list --username='<user name>' --password='<password>' --tenantId=<tenant ID> --resourceGroup=<resource group> -v -D /var/tmp/debug-fence.out ```

If using Managed Indentity: 

```/usr/sbin/fence_azure_arm --action=list --msi --resourceGroup=<resource group> -v -D /var/tmp/debug-fence.out```

### Next Steps:
If you require further help, see [Support and troubleshooting for Azure VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/vm-support-help). That article can also help you file an Azure support incident, if necessary. As you follow the instructions, keep a copy of the *debug-fence.out* because it might be requested for inspection by the support engineer.

## Scenario 2: <!--Required: Here goes the Scenario short title-->

```/var/log/messages
2020-04-06T10:06:47.779470+00:00 VM1 pacemaker-controld[29309]: notice: Result of probe operation for rsc_st_azure on VM1: 7 (not running)
2020-04-06T10:06:51.045519+00:00 VM1 pacemaker-execd[29306]: notice: executing - rsc:rsc_st_azure action:start call_id:52
2020-04-06T10:06:52.826702+00:00 VM1 /fence_azure_arm: Failed: AdalError: Get Token request returned http error: 400 and server response: {"error":"unauthorized_client","error_description":"AADSTS700016: Application with identifier '5005c556-e620-4297-8398-abcc239fa706'
was not found in the directory 'b37c8d34-016c-4e9e-9283-56234092be50'. This can happen if the application has not been installed by the administrator of the tenant or consented to by any user in the tenant.
You may have sent your authentication request to the wrong tenant.\r\nTrace ID: 9d86f824-52c1-45a8-b24f-c81473122d00\r\nCorrelation ID: 7dd6de5d-1d6a-4950-be8b-9a2cb2df8553\r\nTimestamp:2020-04-06 10:06:52Z","error_codes":[700016],"timestamp":"2020-04-06 10:06:52Z","trace_id":"9d86f824-52c1-45a8-b24f-c81473122d00",
"correlation_id":"7dd6de5d-1d6a-4950-be8b-9a2cb2df8553","error_uri":"https://login.microsoftonline.com/error?code=700016 "}
```

### Resolution:

Please check with customer and re-verify the AAD app tenant ID, application ID, login and password details from the Azure portal.

Once the IDs are verified and replaced, reconfigure the fencing agent in the cluster.

```# crm configure property maintenance-mode=true

# crm configure edit <fencing agent resource>

Change the parameters accordingly

Save the changes

# crm configure property maintenance-mode=false

Now verify the cluster status to confirm if fencing agent issue is fixed

# crm status
```

## Scenario 3: <!--Optional: Here goes the Scenario short title-->

```/var/log/messages
Apr 2 00:49:56 VM1 fence_azure_arm: Please use '-h' for usage
Apr 2 00:49:57 VM1 stonith-ng[105424]: warning: fence_azure_arm[109393] stderr: [ 2020-04-02 00:49:56,978 ERROR: Failed: Azure Error: AuthorizationFailed ]
Apr 2 00:49:57 VM1 stonith-ng[105424]: warning: fence_azure_arm[109393] stderr: [ Message: The client 'd36bc109-bdfd-4b6d-bf28-d3990d3c22ea' with object id 'd36bc109-bdfd-4b6d-bf28-d3990d3c22ea' does not have authorization to perform action 'Microsoft.Compute/virtualMachines/read' over scope '/subscriptions/e2d1c3ed-77d2-47f5-a2af-27aa0f9d79a8/resourceGroups/DPG-RG-MAIN01-PROD/providers/Microsoft.Compute' or the scope is invalid.If access was recently granted, please refresh your credentials. ]
```
### Resolution:
Involve AAD suppport to verify the role permissions for the AAD app Once the permissions are fixed, issue should be resolved
Now verify the cluster status to confirm fencing agent issue is fixed crm status

## Scenario 4: <!--Optional: Here goes the Scenario short title-->
After reviewing the logs and outputs, found the agent is failing with authorization issues.
```/var/log/messages
Apr 2 00:49:57 heeudpgscs01 stonith-ng[105424]: warning: fence_azure_arm[109393] stderr: [ 2020-04-02 00:49:56,978 ERROR: Failed: Azure Error: AuthorizationFailed ]
Apr 2 00:49:57 heeudpgscs01 stonith-ng[105424]: warning: fence_azure_arm[109393] stderr: [ Message: The client 'd36bc109-bdfd-4b6d-bf28-d3990d3c22ea' with object id 'd36bc109-bdfd-4b6d-bf28-d3990d3c22ea' does not have authorization to perform action 'Microsoft.Compute/virtualMachines/read' over scope '/subscriptions/e2d1c3ed-77d2-47f5-a2af-27aa0f9d79a8/resourceGroups/DPG-RG-MAIN01-PROD/providers/Microsoft.Compute' or the scope is invalid. If access was recently granted, please refresh your credentials. ]
```
### Resolution:
1. Based on https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/sap/high-availability-guide-suse-pacemaker#1-create-a-custom-role-for-the-fence-agent, check the custom role definition for "Linux Fence Agent Role". The role name may be different for different customers.
3. Check if the app fencing-agent" having this custom role is assigned to the impacted VM or not.
4. If it doesn't, assign the app to the VM via Access Control.
5. Start the pacemaker cluster and it should along with the fencing agent (Azure Fencing Agent).

## Scenario 5: <!--Optional: Here goes the Scenario short title-->
/var/log/messages
```
warning: fence_azure_arm[28114] stderr: [ 2021-06-24 07:59:29,832 ERROR: Failed: Error occurred in request., SSLError: HTTPSConnectionPool(host='management.azure.com ', port=443): Max retries exceeded with url: /subscriptions/a8964342-5414-40a0-aade-5c3d23482016/resourceGroups/rgglbp01/providers/Microsoft.Compute/virtualMachines?api-version=2019-03-01 (Caused by SSLError(SSLError('bad handshake: SysCallError(-1, 'Unexpected EOF')',),)) ]
```
### Resolution:
1.Tested connectivity from affected nodes using openssl:
```openssl s_client -connect management.azure.com:443```

Noticed that the output was not showing the full certificate handshake:
``` CONNECTED(00000003)
write:errno=0
---
no peer certificate available
---
No client certificate CA names sent
---
SSL handshake has read 0 bytes and written 176 bytes
Verification: OK
---
New, (NONE), Cipher is (NONE)
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : 0000
    Session-ID:
    Session-ID-ctx:
    Master-Key:
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1625235527
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
```
2. These errors are likely to a network appliance / firewall doing packet inspection or modifying the TLS connections in a way that is causing certificate verification to fail (mtu sizes can also cause these problems)
3. If you have Azure Firewall in front of the nodes, make sure these tags are added to the application/network rules:
```Application Rules:
ApiManagement , AppServiceManagement, AzureCloud
Network Rules:
AppServiceEnvironment
```
[!INCLUDE [Azure Help Support](../../includes/azure-help-support.md)]
