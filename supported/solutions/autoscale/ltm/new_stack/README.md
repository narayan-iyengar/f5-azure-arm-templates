# Deploying the BIG-IP VE in Azure - AutoScale BIG-IP LTM - VM Scale Set

[![Slack Status](https://f5cloudsolutions.herokuapp.com/badge.svg)](https://f5cloudsolutions.herokuapp.com)
[![Releases](https://img.shields.io/github/release/f5networks/f5-azure-arm-templates.svg)](https://github.com/f5networks/f5-azure-arm-templates/releases)
[![Issues](https://img.shields.io/github/issues/f5networks/f5-azure-arm-templates.svg)](https://github.com/f5networks/f5-azure-arm-templates/issues)


**Contents**
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Important Configuration Notes](#important-configuration-notes)
- [Security](#security)
- [Getting Help](#help)
- [Installation](#installation)
- [Configuration Example](#configuration-example)
- [Service Discovery](#service-discovery)


## Introduction

This solution uses an ARM template to launch the deployment of F5 BIG-IP Local Traffic Manager (LTM) Virtual Edition (VE) instances in a Microsoft Azure VM Scale Set that is configured for auto scaling. Traffic flows from the Azure load balancer to the BIG-IP VE (cluster) and then to the application servers. The BIG-IP VE(s) are configured in single-NIC mode. Auto scaling means that as certain thresholds are reached, the number of BIG-IP VE instances automatically increases or decreases accordingly. Be sure to see [Scaling Thresholds](#scaling-thresholds) for information on scaling options. 

The BIG-IP VE has the [Local Traffic Manager (LTM)](https://f5.com/products/big-ip/local-traffic-manager-ltm) module enabled to provide advanced traffic management functionality. This means you can also configure the BIG-IP VE to enable F5's L4/L7 security features, access control, and intelligent traffic management. 

You have the option of using a [BIG-IQ device](https://f5.com/products/big-iq-centralized-management) to license BIG-IP VEs using BYOL licenses in this auto scale deployment.

For information on getting started using F5's ARM templates on GitHub, see [Microsoft Azure: Solutions 101](http://clouddocs.f5.com/cloud/public/v1/azure/Azure_solutions101.html).

**Networking Stack Type:** This solution deploys into a new networking stack, which is created along with the solution.

## Prerequisites
  - **Important**: When you configure the admin password for the BIG-IP VE in the template, you cannot use the character **#**.  Additionally, there are a number of other special characters that you should avoid using for F5 product user accounts.  See https://support.f5.com/csp/article/K2873 for details.
  - If you are deploying the BYOL template, you must have a valid BIG-IP license token.
  - This template requires service principal.  See the [Service Principal Setup section](#service-principal-authentication) for details.
  - This solution uses calls to the Azure REST API to read and update Azure resources such as storage accounts, network interfaces, and route tables.  For the solution to function correctly, you must ensure that the BIG-IP(s) can connect to the Azure REST API on port 443.

## Important configuration notes
  - See the **[Configuration Example](#config)** section for a configuration diagram and description for this solution.
  - See the important note about [optionally changing the BIG-IP Management port](#changing-the-big-ip-configuration-utility-gui-port).
  - This template supports service discovery.  See the [Service Discovery section](#service-discovery) for details.
  - This template can send non-identifiable statistical information to F5 Networks to help us improve our templates.  See [Sending statistical information to F5](#sending-statistical-information-to-f5).
  - In order to pass traffic from your clients to the servers, after launching the template, you must create virtual server(s) on the BIG-IP VE.  See [Creating a virtual server](#creating-virtual-servers-on-the-big-ip-ve).
  - F5 has created a matrix that contains all of the tagged releases of the F5 ARM templates for Microsoft Azure and the corresponding BIG-IP versions, license types and throughputs available for a specific tagged release. See https://github.com/F5Networks/f5-azure-arm-templates/blob/master/azure-bigip-version-matrix.md.
  - F5 has created an iApp (currently **Experimental**) for configuring logging for BIG-IP modules to be sent to a specific set of cloud analytics solutions.  See [Experimental Logging iApp](#experimental-logging-iapp).
  - F5 Azure ARM templates now capture all deployment logs to the BIG-IP VE in **/var/log/cloud/azure**.  Depending on which template you are using, this includes deployment logs (stdout/stderr), Cloud Libs execution logs, recurring solution logs (metrics, failover, and so on), and more. 
  - Supported F5 ARM templates do not reconfigure existing Azure resources, such as network security groups.  Depending on your configuration, you may need to configure these resources to allow the BIG-IP VE(s) to receive traffic for your application.  Similarly, templates that deploy Azure load balancer(s) do not configure load balancing rules or probes on those resources to forward external traffic to the BIG-IP(s).  You must create these resources after the deployment has succeeded.
  - This template includes a master election feature, which ensures that if the existing master BIG-IP VE is unavailable, a new master is selected from the BIG-IP VEs in the cluster.
  - This template has some optional post-deployment configuration.  See the [Post-Deployment Configuration section](#post-deployment-configuration) for details.
  - After deploying the template, if you make manual changes to the BIG-IP configuration, you must see [this section](#backup-big-ip-configuration-for-cluster-recovery).
  - For important information on choosing a metric on which to base autoscaling events and the thresholds used by the template, see [Scaling Thresholds](#scaling-thresholds).
  - You have the option of using a [BIG-IQ device](https://f5.com/products/big-iq-centralized-management) with a pool of BIG-IP licenses in order to license BIG-IP VEs using BYOL licenses. This solution only supports only BIG-IQ versions 5.0 - 5.3, and your BIG-IQ system must have at least 2 NICs.


## Security
This ARM template downloads helper code to configure the BIG-IP system. If you want to verify the integrity of the template, you can open the template and ensure the following lines are present. See [Security Detail](#securitydetail) for the exact code.
In the *variables* section:
  - In the *verifyHash* variable: **script-signature** and then a hashed signature.
  - In the *installCloudLibs* variable: **tmsh load sys config merge file /config/verifyHash**.
  - In the *installCloudLibs* variable: ensure this includes **tmsh run cli script verifyHash /config/cloud/f5-cloud-libs.tar.gz**.


Additionally, F5 provides checksums for all of our supported templates. For instructions and the checksums to compare against, see https://devcentral.f5.com/codeshare/checksums-for-f5-supported-cft-and-arm-templates-on-github-1014.

## Supported BIG-IP versions
The following is a map that shows the available options for the template parameter **bigIpVersion** as it corresponds to the BIG-IP version itself. Only the latest version of BIG-IP VE is posted in the Azure Marketplace. For older versions, see downloads.f5.com.

**NOTE**:  Due to changes within the Azure environment, only BIG-IP version 13.1.0200 is available (even if you select **Latest** in the template). We will update the template to include a new BIG-IP v12.1 image as soon as it is available. 

| Azure BIG-IP Image Version | BIG-IP Version |
| --- | --- |
| 13.1.0200 | 13.1.0 Build 0.0.6 |
| latest | This will select the latest BIG-IP version available |


## Supported instance types and hypervisors
  - For a list of supported Azure instance types for this solution, see the [Azure instances for BIG-IP VE](http://clouddocs.f5.com/cloud/public/v1/azure/Azure_singleNIC.html#azure-instances-for-big-ip-ve).

  - For a list versions of the BIG-IP Virtual Edition (VE) and F5 licenses that are supported on specific hypervisors and Microsoft Azure, see https://support.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/ve-supported-hypervisor-matrix.html.

### Help
Because this template has been created and fully tested by F5 Networks, it is fully supported by F5. This means you can get assistance if necessary from [F5 Technical Support](https://support.f5.com/csp/article/K40701984).

**Community Support**  
We encourage you to use our [Slack channel](https://f5cloudsolutions.herokuapp.com) for discussion and assistance on F5 ARM templates. There are F5 employees who are members of this community who typically monitor the channel Monday-Friday 9-5 PST and will offer best-effort assistance. This slack channel community support should **not** be considered a substitute for F5 Technical Support for supported templates. See the [Slack Channel Statement](https://github.com/F5Networks/f5-azure-arm-templates/blob/master/slack-channel-statement.md) for guidelines on using this channel.


## Installation

You have three options for deploying this solution:
  - Using the Azure deploy buttons
  - Using [PowerShell](#powershell)
  - Using [CLI Tools](#cli)

### <a name="azure"></a>Azure deploy buttons

Use the appropriate button, depending on what type of BIG-IP licensing required:
   - **PAYG**: This allows you to use pay-as-you-go hourly billing. <br><a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FF5Networks%2Ff5-azure-arm-templates%2Fv4.4.0.1%2Fsupported%2Fsolutions%2Fautoscale%2Fltm%2Fnew_stack%2FPAYG%2Fazuredeploy.json">
    <img src="http://azuredeploy.net/deploybutton.png"/></a><br><br>

   - **BIG-IQ**: This allows you to launch the template using an existing BIG-IQ device with a pool of licenses to license the BIG-IP VE(s). <br><a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FF5Networks%2Ff5-azure-arm-templates%2Fv4.4.0.1%2Fsupported%2Fsolutions%2Fautoscale%2Fltm%2Fnew_stack%2FBIGIQ%2Fazuredeploy.json">
    <img src="http://azuredeploy.net/deploybutton.png"/></a><br><br>



### Template parameters

| Parameter | Required | Description |
| --- | --- | --- |
| vmScaleSetMinCount | Yes | The minimum (and default) number of BIG-IP VEs that will be deployed into the VM Scale Set. |
| vmScaleSetMaxCount | Yes | The maximum number of BIG-IP VEs that can be deployed into the VM Scale Set. |
| autoScaleMetric | Yes | Select the metric on which auto scale events should be triggered. The following parameters determine individual settings for the scaling rules based on the metric you choose. Note: Custom BIG-IP metrics (for use by additional autoscale rules or for device visibility) are sent to Application Insights regardless of the metric you select. |
| appInsights | Yes | Enter the name of your existing Application Insights environment that will be used to receive custom BIG-IP metrics you can use for Scale Set rules and device visibility. If the Application Insights environment is in a different Resource Group than this deployment, specify it as **app_insights_name;app_insights_rg**).  If you do not have an Application Insights environment, leave the default (CREATE_NEW) and the template will create one. Note: By default, the new Application Insights environment will be created in **East US**, if necessary you can specify a different region as **CREATE_NEW:app_insights_region**). |
| calculatedBandwidth | Yes | Specify the amount of bandwidth (in Mbps) that should be used to base the throughput percentage calculation on for scale events. For PAYG, we recommend this matches the parameter **licensedBandwidth**, or at minimum is a lower value. See [Scaling Thresholds](#scaling-thresholds) for more information. |
| scaleOutThreshold | Yes | The percentage the metric should be above to trigger a Scale Out event.  Note: For network utilization metrics this is factored as a percentage of the parameter 'calculatedBandwidth'. See [Scaling Thresholds](#scaling-thresholds) for more information. |
| scaleInThreshold | Yes | The percentage the metric should be below to trigger a Scale In event.  Note: For network utilization metrics this is factored as a percentage of the parameter 'calculatedBandwidth'.  See [Scaling Thresholds](#scaling-thresholds) for more information. |
| scaleTimeWindow | Yes | The time window required to trigger a scale event (in and out). This is used to determine the amount of time needed for a threshold to be breached, as well as to prevent excessive scaling events (flapping). |
| adminUsername | Yes | User name for the Virtual Machine. |
| adminPassword | Yes | Password to login to the Virtual Machine. |
| dnsLabel | Yes | Unique DNS Name for the Public IP address used to access the Virtual Machine. |
| instanceType | Yes | Azure instance size of the Virtual Machine. |
| imageName | Yes | F5 SKU (IMAGE) to you want to deploy. Note: The disk size of the VM will be determined based on the option you select. |
| bigIpVersion | Yes | F5 BIG-IP version you want to use. |
| licensedBandwidth | PAYG only: | The amount of licensed bandwidth (Mbps) you want the PAYG image to use. |
| bigIqLicenseHost | BIG-IQ licensing only: | The IP address (or hostname) for the BIG-IQ to be used when licensing the BIG-IP.  Note: The BIG-IP will make a REST call to the BIG-IQ (already existing) to let it know a BIG-IP needs to be licensed. It will then license the BIG-IP using the provided BIG-IQ credentials and license pool. |
| bigIqLicenseUsername | BIG-IQ licensing only: | The BIG-IQ username to use during BIG-IP licensing via BIG-IQ. |
| bigIqLicensePassword | BIG-IQ licensing only: | The BIG-IQ password to use during BIG-IP licensing via BIG-IQ. |
| bigIqLicensePool | BIG-IQ licensing only: | The BIG-IQ license pool to use during BIG-IP licensing via BIG-IQ. |
| vnetAddressPrefix | Yes | The start of the CIDR block the BIG-IP VEs use when creating the Vnet and subnets.  You MUST type just the first two octets of the /16 virtual network that will be created, for example '10.0', '10.100', 192.168'. |
| tenantId | Yes | Your Azure service principal application tenant ID. |
| clientId | Yes | Your Azure service principal application client ID. |
| servicePrincipalSecret | Yes | Your Azure service principal application secret. |
| notificationEmail | Yes | If you want email notifications on scale events, specify an email address, otherwise leave the parameter as **OPTIONAL**. Note: You can specify multiple emails by separating them with a semi-colon, such as *email@domain.com;email2@domain.com*. |
| ntpServer | Yes | Leave the default NTP server the BIG-IP uses, or replace the default NTP server with the one you want to use. |
| timeZone | Yes | If you would like to change the time zone the BIG-IP uses, enter the time zone you want to use. This is based on the tz database found in /usr/share/zoneinfo. Example values: UTC, US/Pacific, US/Eastern, Europe/London or Asia/Singapore. |
| restrictedSrcAddress | Yes | This field restricts management access to a specific network or address. Enter an IP address or address range in CIDR notation, or asterisk for all sources |
| tagValues | Yes | Default key/value resource tags will be added to the resources in this deployment, if you would like the values to be unique adjust them as needed for each key. |
| allowUsageAnalytics | Yes | This deployment can send anonymous statistics to F5 to help us determine how to improve our solutions. If you select **No** statistics are not sent. |

### Programmatic deployments
As an alternative to deploying through the Azure Portal (GUI) each solution provides example scripts to deploy the ARM template.  The example commands can be found below along with the name of the script file, which exists in the current directory.

#### <a name="powershell"></a>PowerShell Script Example

```powershell
## Example Command: .\Deploy_via_PS.ps1 -licenseType PAYG -licensedBandwidth 200m -vmScaleSetMinCount 2 -vmScaleSetMaxCount 4 -autoScaleMetric Host_Throughput -appInsights CREATE_NEW -calculatedBandwidth 200m -scaleOutThreshold 90 -scaleInThreshold 10 -scaleTimeWindow 10 -adminUsername azureuser -adminPassword <value> -dnsLabel <value> -instanceType Standard_DS2_v2 -imageName Good -bigIpVersion 13.1.0200 -vnetAddressPrefix 10.0 -tenantId <value> -clientId <value> -servicePrincipalSecret <value> -notificationEmail OPTIONAL -ntpServer 0.pool.ntp.org -timeZone UTC -allowUsageAnalytics Yes -resourceGroupName <value>
```

=======

#### <a name="cli"></a>Azure CLI(1.0) Script Example

```bash
## Example Command: ./deploy_via_bash.sh --licenseType PAYG --licensedBandwidth 200m --vmScaleSetMinCount 2 --vmScaleSetMaxCount 4 --autoScaleMetric Host_Throughput --appInsights CREATE_NEW --calculatedBandwidth 200m --scaleOutThreshold 90 --scaleInThreshold 10 --scaleTimeWindow 10 --adminUsername azureuser --adminPassword <value> --dnsLabel <value> --instanceType Standard_DS2_v2 --imageName Good --bigIpVersion 13.1.0200 --vnetAddressPrefix 10.0 --tenantId <value> --clientId <value> --servicePrincipalSecret <value> --notificationEmail OPTIONAL --ntpServer 0.pool.ntp.org --timeZone UTC --allowUsageAnalytics Yes --resourceGroupName <value> --azureLoginUser <value> --azureLoginPassword <value>
```

## Scaling Thresholds

 You have three choices on which metric to use for auto scale events, each based on a percentage of the metric which you set in the ARM template:  

  -	**F5_TMM_CPU** - Choosing this option means scaling events are triggered based on the utilization of the BIG-IP VE CPU, specifically the F5 TMM (Traffic Management Microkernel) CPU.  
  -	**F5_TMM_Traffic** - Choosing this option means that scaling events are triggered based on traffic going through the BIG-IP VE TMM.  These thresholds are based on an aggregate of traffic both in and out, and are based on a percentage of the value you chose in the Calculated Bandwidth option (see below).
  -	**Host_Throughput** - Choosing this option means scaling events are based on a metric being gathered by Azure on the host itself, specifically Network_Out. This is based on a percentage of the value you chose in the Calculated Bandwidth option (see below).

Both F5_TMM_Traffic and Host_Throughput are based on a percentage of the value you choose from the **Calculated Bandwidth** list.  For PAYG deployments, this value should match (or could be lower than) the **Licensed Bandwidth** value you are using.  For example, if you plan to use 200Mbps BIG-IP VEs, you should select 200m from the Calculated Bandwidth list.  The system then uses this value together with the percentages you enter in **Scale Out Threshold** and **Scale In Threshold** to determine when scaling events occur for these two metrics.  If you are using the BIG-IQ with a pool of BIG-IP BYOL licenses, there is no Licensed Bandwidth field, so you must specify the bandwidth level in the Calculated Bandwidth field in order for scaling to function properly. 


## Configuration Example <a name="config">

The following is an example configuration diagram for this solution deployment. In this scenario, all access to the BIG-IP VE appliance is through an Azure Load Balancer. The Azure Load Balancer processes both management and data plane traffic into the BIG-IP VEs, which then distribute the traffic to web/application servers according to normal F5 patterns.

![Configuration Example](images/azure-example-diagram.png)

## Post-Deployment Configuration
This solution deploys an ARM template that fully configures BIG-IP VE(s) and handles clustering (DSC) and Azure creation of objects needed for management of those BIG-IP VEs.  However, once deployed the assumption is configuration will be performed on the BIG-IP VE(s) to create virtual servers, pools, and other objects used for processing application traffic.  An example of the steps required to add an application are listed [here](#post-deployment-application-configuration).

### Backup BIG-IP configuration for cluster recovery
After you initially launch the template, if you make manual changes to the BIG-IP configuration, you must make a backup of your BIG-IP configuration and store the resulting UCS file in the **backup** container of the storage account ending in **data000** created by the template to ensure the master election process will include the custom configuration in the cluster in the event of failure. Note: If necessary to recover from this UCS it will pick the one with the latest timestamp.
  1. Backup your BIG-IP configuration (ideally the cluster primary) by creating a [UCS](https://support.f5.com/csp/article/K13132) archive.  Use the following syntax to save the backup UCS file:
     - From the CLI command: ```# tmsh save /sys ucs /var/tmp/original.ucs```
     - From the Configuration utility: **System > Archives > Create**
  2. Upload the UCS into the **backup** container of the storage account ending in **data000** (it is a Blob container).

### Post-deployment application configuration
Note: Steps are for an example application on port 443
  1. Add a "Health Probe" to the ALB (Azure Load Balancer) for port 443, you can choose TCP or HTTP depending on your needs.  This queries each BIG-IP at that port to determine if it is available for traffic.
  2. Add a "Load Balancing Rule" to the ALB where the port is 443 and the backend port is also 443 (assuming you are using same port on the BIG-IP), make sure the backend pool is selected (there should only be one backend pool which was created and is managed by the VM Scale set)
  3. Add an "Inbound Security Rule" to the Network Security Group (NSG) for port 443 as the NSG is added to the subnet where the BIG-IP VE(s) are deployed - You could optionally just remove the NSG from the subnet as the VM Scale Set is fronted by the ALB.

### Additional Optional Configuration Items
Here are some post-deployment options that are entirely optional but could be useful based on your needs.

#### BIG-IP Lifecycle Management
As new BIG-IP versions are released, existing VM scale sets can be upgraded to use those new images. In an existing implementation, we assume you have created different types of BIG-IP configuration objects (such as virtual servers, pools, and monitors), and you want to retain this BIG-IP configuration after an upgrade. This section describes the process of upgrading and retaining the configuration.

When this ARM template was initially deployed, a storage account was created in the same Resource Group as the VM scale set. This account name ends with **data000*** (the name of storage accounts have to be globally unique, so the prefix is a unique string). In this storage account, the template created a container named **backup**.  We use this backup container to hold backup [UCS](https://support.f5.com/csp/article/K13132) configuration files. Once the UCS is present in the container, you update the scale set "model" to use the newer BIG-IP version. Once the scale set is updated, you upgrade the BIG-IP VE(s). As a part of this upgrade, the provisioning checks the backup container for a UCS file and if one exists, it uploads the configuration (if more than one exists, it uses the latest).

**To upgrade the BIG-IP VE Image**
  1. Backup configuration as outlined [here](#backup-big-ip-configuration-for-cluster-recovery)
  3. Update the VM Scale Set Model to the new BIG-IP version
     - From PowerShell: Use the PowerShell script in the **scripts** folder in this directory
     - Using the Azure redeploy functionality: From the Resource Group where the ARM template was initially deployed, click the successful deployment and then select to redeploy the template. If necessary, re-select all the same variables, and **only change** the BIG-IP version to the latest.
  4. Upgrade the Instances
     1. In Azure, navigate to the VM Scale Set instances pane and verify the *Latest model* does not say **Yes** (it should have a caution sign instead of the word Yes)
     2. Select either all instances at once or each instance one at a time (starting with instance ID 0 and working up).
     3. Click the **Upgrade** action button.

#### Configure Scale Event Notifications
**Note:** Email addresses for notification can now be specified within the solution and will be applied automatically, they can also be configured manually via the VM Scale Set configuration options available within the Azure Portal.

You can add notifications when scale up/down events happen, either in the form of email or webhooks. The following shows an example of adding an email address via the Azure Resources Explorer that receives an email from Azure whenever a scale up/down event occurs.

Log in to the [Azure Resource Explorer](https://resources.azure.com) and then navigate to the Auto Scale settings (**Subscriptions > Resource Groups >** *resource group where deployed* **> Providers > Microsoft.Insights > Autoscalesettings > autoscaleconfig**).  At the top of the screen click Read/Write, and then from the Auto Scale settings, click **Edit**.  Replace the current **notifications** json key with the example below, making sure to update the email address(es). Select PUT and notifications will be sent to the email addresses listed.

```json
    "notifications": [
      {
        "operation": "Scale",
        "email": {
          "sendToSubscriptionAdministrator": false,
          "sendToSubscriptionCoAdministrators": false,
          "customEmails": [
            "email@f5.com"
          ]
        },
        "webhooks": null
      }
    ]
```

## Documentation

For more information on F5 solutions for Azure, including manual configuration procedures for some deployment scenarios, see the Azure section of http://clouddocs.f5.com/cloud/public/v1/.

### Service Discovery
Once you launch your BIG-IP instance using the ARM template, you can use the Service Discovery iApp template on the BIG-IP VE to automatically update pool members based on auto-scaled cloud application hosts.  In the iApp template, you enter information about your cloud environment, including the tag key and tag value for the pool members you want to include, and then the BIG-IP VE programmatically discovers (or removes) members using those tags.  See our [Service Discovery video](https://www.youtube.com/watch?v=ig_pQ_tqvsI) to see this feature in action.

#### Tagging
In Microsoft Azure, you have three options for tagging objects that the Service Discovery iApp uses. Note that you select public or private IP addresses within the iApp.
  - *Tag a VM resource*<br>
The BIG-IP VE will discover the primary public or private IP addresses for the primary NIC configured for the tagged VM.
  - *Tag a NIC resource*<br>
The BIG-IP VE will discover the primary public or private IP addresses for the tagged NIC.  Use this option if you want to use the secondary NIC of a VM in the pool.
  - *Tag a Virtual Machine Scale Set resource*<br>
The BIG-IP VE will discover the primary private IP address for the primary NIC configured for each Scale Set instance.  Note you must select Private IP addresses in the iApp template if you are tagging a Scale Set.

The iApp first looks for NIC resources with the tags you specify.  If it finds NICs with the proper tags, it does not look for VM resources. If it does not find NIC resources, it looks for VM resources with the proper tags. In either case, it then looks for Scale Set resources with the proper tags.

**Important**: Make sure the tags and IP addresses you use are unique. You should not tag multiple Azure nodes with the same key/tag combination if those nodes use the same IP address.

To launch the template:
  1.	From the BIG-IP VE web-based Configuration utility, on the Main tab, click **iApps > Application Services > Create**.
  2.	In the **Name** field, give the template a unique name.
  3.	From the **Template** list, select **f5.service_discovery**.  The template opens.
  4.	Complete the template with information from your environment.  For assistance, from the Do you want to see inline help? question, select Yes, show inline help.
  5.	When you are done, click the **Finished** button.

### Service Principal Authentication
This solution requires access to the Azure API to determine how the BIG-IP VEs should be configured.  The most efficient and security-conscious way to handle this is to utilize Azure service principal authentication, for all the typical security reasons.  The following provides information/links on the options for configuring a service principal within Azure if this is the first time it is needed in a subscription.

_Ensure that however the creation of the service principal occurs to verify it only has minimum required access based on the solutions need(read vs read/write) prior to this template being deployed and used by the solution within the resource group selected(new or existing)._

**Minimum Required Access:** **Read** access is required, it can be limited to the resource group used by this solution.

The end result should be possession of a client (application) ID, tenant ID and service principal secret that can login to the same subscription this template will be deployed into.  Ensuring this is fully functioning prior to deploying this ARM template will save on some troubleshooting post-deployment if the service principal is in fact not fully configured.

**NOTE:** Service principal information is stored locally in the **/config/cloud/.azCredentials file.  If for any reason you need to update the service principal information, you must manually edit the .azCredentials file on both BIG-IP systems.

#### 1. Azure Portal

Follow the steps outlined in the [Azure Portal documentation](https://azure.microsoft.com/en-us/documentation/articles/resource-group-create-service-principal-portal/) to generate the service principal.

#### 2. Azure CLI

This method can be used with either the [Azure CLI v2.0 (Python)](https://github.com/Azure/azure-cli) or the [Azure Cross-Platform CLI (npm module)](https://github.com/Azure/azure-xplat-cli).

_Using the Python Azure CLI v2.0 - requires just one step_
```shell
$ az ad sp create-for-rbac
```

_Using the Node.js cross-platform CLI - requires additional steps for setting up_
https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authenticate-service-principal-cli

#### 3. Azure PowerShell
Follow the steps outlined in the [Azure Powershell documentation](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authenticate-service-principal) to generate the service principal.

## Creating virtual servers on the BIG-IP VE

In order to pass traffic from your clients to the servers through the BIG-IP system, you must create a virtual server on the BIG-IP VE. To create a BIG-IP virtual server you need to know the private IP address of the IP configuration(s) for each BIG-IP VE network interface created by the template. If you need additional virtual servers for your applications/servers, you can add more secondary IP configurations on the Azure network interface, and corresponding virtual servers on the BIG-IP system. See https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-multiple-ip-addresses-portal for information on multiple IP addresses.
**Important**: For 1 NIC BIG-IP VEs, the virtual server configuration uses a wildcard destination address (example: 0.0.0.0/0) and should use different ports (if behind an ALB) or hostnames to differentiate services.

**To create virtual servers on the BIG-IP system**

1. Once your BIG-IP VE has launched, open the BIG-IP VE Configuration utility.
2. On the Main tab, click **Local Traffic > Virtual Servers** and then click the **Create** button.
3. In the **Name** field, give the Virtual Server a unique name.
4. In the **Destination/Mask** field, type the Azure secondary private IP address.
5. In the **Service Port** field, type the appropriate port. 
6. Configure the rest of the virtual server as appropriate.
7. If you used the Service Discovery iApp template: <br>In the Resources section, from the **Default Pool** list, select the name of the pool created by the iApp.
8. Click the **Finished** button.
9. Repeat as necessary.

### Deploying Custom Configuration to the BIG-IP (Azure Virtual Machine)

Once the solution has been deployed there may be a need to perform some additional configuration of the BIG-IP.  This can be accomplished via traditional methods such as via the GUI, logging into the CLI or using the REST API.  However, depending on the requirements it might be preferred to perform this custom configuration as a part of the initial deployment of the solution.  This can be accomplished in the below manner.

Within the Azure Resource Manager (ARM) template there is a variable called **customConfig**, this contains text similar to "### START(INPUT) CUSTOM CONFIGURATION", that can be replaced with custom shell scripting to perform additional configuration of the BIG-IP.  An example of what it would look like to configure the f5.ip_forwarding iApp is included below.

Warning: F5 does not support the template if you change anything other than the **customConfig** ARM template variable.

```json
"variables": {
    "customConfig": "### START (INPUT) CUSTOM CONFIGURATION HERE\ntmsh create sys application service my_deployment { device-group none template f5.ip_forwarding traffic-group none variables replace-all-with { basic__addr { value 0.0.0.0 } basic__forward_all { value No } basic__mask { value 0.0.0.0 } basic__port { value 0 } basic__vlan_listening { value default } options__advanced { value no }options__display_help { value hide } } }"
}
```
### Changing the BIG-IP Configuration utility (GUI) port
Depending on the deployment requirements, the default management port for the BIG-IP may need to be changed. To change the Management port, see [Changing the Configuration utility port](https://support.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/bigip-ve-setup-msft-azure-12-0-0/2.html#GUID-3E6920CD-A8CD-456C-AC40-33469DA6922E) for instructions.<br>
***Important***: The default port provisioned is dependent on 1) which BIG-IP version you choose to deploy as well as 2) how many interfaces (NICs) are configured on that BIG-IP. BIG-IP v13.x and later in a single-NIC configuration uses port 8443. All prior BIG-IP versions default to 443 on the MGMT interface.<br>
***Important***: If you perform the procedure to change the port, you must check the Azure Network Security Group associated with the interface on the BIG-IP that was deployed and adjust the ports accordingly.

### Experimental Logging iApp
F5 has created an iApp for configuring logging for BIG-IP modules to be sent to a specific set of cloud analytics solutions. The iApp creates logging profiles which can be attached to the appropriate objects (virtual servers, APM policy, and so on) which results in logs being sent to the selected cloud analytics solution, Azure in this case. 

Note that even if the F5 ARM template is F5 Supported, this iApp template is still Experimental.

We recommend you watch the [Viewing ASM Data in Azure Analytics video](https://www.youtube.com/watch?v=X3B_TOG5ZpA&feature=youtu.be) that shows this iApp in action, everything from downloading and importing the iApp, to configuring it, to a demo of an attack on an application and the resulting ASM violation log that is sent to ASM Analytics.

**Important**: Be aware that this may (depending on the level of logging required) affect performance of the BIG-IP as a result of the processing to construct and send the log messages over HTTP to the cloud analytics solution.  
It is also important to note this cloud logging iApp template is a *different solution and iApp template* than the F5 Analytics iApp template (https://f5.com/solutions/deployment-guides/analytics-big-ip-v114-v1212-ltm-apm-aam-asm-afm).  

Use the following guidance for downloading and importing the iApp template.
  1. From a web browser, go to https://github.com/F5Networks/f5-cloud-iapps/tree/master/f5-cloud-logger.
  2. Click **f5.cloud_logger.v1.0.0.tmpl** (or later version if applicable).
  3. On the right side of the code box, click **Raw**.
  4. Save the code to a location accessible by your BIG-IP system (for example, right-click the raw output and then click Save as).
  5. Log on to the BIG-IP system web-based Configuration utility.
  6. On the Main tab, expand **iApp**, and then click **Templates**.
  7. Click the **Import** button on the right side of the screen.
  8. Click the **Browse** button, and then browse to the location you saved the Cloud Logger iApp file.
  9. Click the **Upload** button. The iApp is now available for use.
  10. From the iApp menu, click **Application Services > Applications > Create**.
  11. From the **Template** list, select f5.cloud_logger.v1.0.0.tmpl (or later version if applicable). 

For assistance running the iApp template, once you open the iApp, from the *Do you want to see inline help?* question, select **Yes, show inline help**.


### Sending statistical information to F5
All of the F5 templates now have an option to send anonymous statistical data to F5 Networks to help us improve future templates.  
None of the information we collect is personally identifiable, and only includes:  

- Customer ID: this is a hash of the customer ID, not the actual ID
- Deployment ID: hash of stack ID
- F5 template name
- F5 template version
- Cloud Name
- Azure region 
- BIG-IP version 
- F5 license type
- F5 Cloud libs version
- F5 script name

This information is critical to the future improvements of templates, but should you decide to select **No**, information will not be sent to F5.

## Security Details <a name="securitydetail"></a>
This section has the code snippet for each the lines you should ensure are present in your template file if you want to verify the integrity of the helper code in the template.

Note the hashed script-signature may be different in your template.<br>

```json
"variables": {
    "apiVersion": "2015-06-15",
    "location": "[resourceGroup().location]",
    "singleQuote": "'",
    "f5CloudLibsTag": "release-2.0.0",
    "expectedHash": "8bb8ca730dce21dff6ec129a84bdb1689d703dc2b0227adcbd16757d5eeddd767fbe7d8d54cc147521ff2232bd42eebe78259069594d159eceb86a88ea137b73",
    "verifyHash": "[concat(variables('singleQuote'), 'cli script /Common/verifyHash {\nproc script::run {} {\n        if {[catch {\n            set file_path [lindex $tmsh::argv 1]\n            set expected_hash ', variables('expectedHash'), '\n            set computed_hash [lindex [exec /usr/bin/openssl dgst -r -sha512 $file_path] 0]\n            if { $expected_hash eq $computed_hash } {\n                exit 0\n            }\n            tmsh::log err {Hash does not match}\n            exit 1\n        }]} {\n            tmsh::log err {Unexpected error in verifyHash}\n            exit 1\n        }\n    }\n    script-signature fc3P5jEvm5pd4qgKzkpOFr9bNGzZFjo9pK0diwqe/LgXwpLlNbpuqoFG6kMSRnzlpL54nrnVKREf6EsBwFoz6WbfDMD3QYZ4k3zkY7aiLzOdOcJh2wECZM5z1Yve/9Vjhmpp4zXo4varPVUkHBYzzr8FPQiR6E7Nv5xOJM2ocUv7E6/2nRfJs42J70bWmGL2ZEmk0xd6gt4tRdksU3LOXhsipuEZbPxJGOPMUZL7o5xNqzU3PvnqZrLFk37bOYMTrZxte51jP/gr3+TIsWNfQEX47nxUcSGN2HYY2Fu+aHDZtdnkYgn5WogQdUAjVVBXYlB38JpX1PFHt1AMrtSIFg==\n}', variables('singleQuote'))]",
    "installCloudLibs": "[concat(variables('singleQuote'), '#!/bin/bash\necho about to execute\nchecks=0\nwhile [ $checks -lt 120 ]; do echo checking mcpd\n/usr/bin/tmsh -a show sys mcp-state field-fmt | grep -q running\nif [ $? == 0 ]; then\necho mcpd ready\nbreak\nfi\necho mcpd not ready yet\nlet checks=checks+1\nsleep 1\ndone\necho loading verifyHash script\n/usr/bin/tmsh load sys config merge file /config/verifyHash\nif [ $? != 0 ]; then\necho cannot validate signature of /config/verifyHash\nexit\nfi\necho loaded verifyHash\necho verifying f5-cloud-libs.targ.gz\n/usr/bin/tmsh run cli script verifyHash /config/cloud/f5-cloud-libs.tar.gz\nif [ $? != 0 ]; then\necho f5-cloud-libs.tar.gz is not valid\nexit\nfi\necho verified f5-cloud-libs.tar.gz\necho expanding f5-cloud-libs.tar.gz\ntar xvfz /config/cloud/f5-cloud-libs.tar.gz -C /config/cloud\ntouch /config/cloud/cloudLibsReady', variables('singleQuote'))]",
```


## Filing Issues
If you find an issue, we would love to hear about it.
You have a choice when it comes to filing issues:
  - Use the **Issues** link on the GitHub menu bar in this repository for items such as enhancement or feature requests and non-urgent bug fixes. Tell us as much as you can about what you found and how you found it.
  - Contact us at [solutionsfeedback@f5.com](mailto:solutionsfeedback@f5.com?subject=GitHub%20Feedback) for general feedback or enhancement requests.
  - Use our [Slack channel](https://f5cloudsolutions.herokuapp.com) for discussion and assistance on F5 cloud templates.  There are F5 employees who are members of this community who typically monitor the channel Monday-Friday 9-5 PST and will offer best-effort assistance.
  - For templates in the **supported** directory, contact F5 Technical support via your typical method for more time sensitive changes and other issues requiring immediate support.


## Copyright

Copyright 2014-2018 F5 Networks Inc.


## License


### Apache V2.0

Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and limitations
under the License.

### Contributor License Agreement

Individuals or business entities who contribute to this project must have
completed and submitted the [F5 Contributor License Agreement](http://f5-openstack-docs.readthedocs.io/en/latest/cla_landing.html).