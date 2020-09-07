# Compute

## Virtual Machines

### Design Considerations

#### Service Level Agreements

* Microsoft provides a 1) 95% SLA for single instance virtual machines using Standard HDD storage for all OS and Data disks, 2) 99.5% SLA for single instance virtual machines using Standard SSD storage for all OS and Data disks, 3) 99.9% SLA for single instance virtual machines using Premium storage for all OS and Data disks, 4) 99.95% SLA for all virtual machines that have two or more instances in the same Availability Set or Dedicated Host Group, and a 5) 99.99% SLA for all virtual machines that have two or more instances deployed across two or more Availability Zones in the same region.

  [Virtual Machine Service Level Agreements](https://azure.microsoft.com/support/legal/sla/virtual-machines/v1_9/)

### Configuration Recommendations

* For all virtual machines requiring resiliency, it is highly recommended that:

    - Managed Disks should be used for all virtual machine OS and Data disks to ensure resilience across underlying storage stamps within a datacenter.
        [Managed Disk Benefits](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/managed-disks-overview#benefits-of-managed-disks)

    - Singleton workloads should use Premium Managed Disks to enhance resiliency and obtain a 99.9% SLA as well as dedicated performance characteristics.

    - Non-Singleton workloads should consider two or more replica instances with Managed disks (Standard or Premium) that are deployed within an Availability Set to obtain a 99.95% SLA or across Availability Zones to obtain a 99.95% SLA.

    -	Where appropriate virtual machines should be deployed across Availability Zones to maximize resilience within a region.
        [Datacenter Fault Tolerance](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/manage-availability#use-availability-zones-to-protect-from-datacenter-level-failures)

* Azure Metadata Service Scheduled Events should be used to proactively respond to maintenance events (i.e. reboots) and limit disruption to virtual machines.

  [Azure Metadata Service Scheduled Events](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/scheduled-events)

* Azure Backup should be used to back-up virtual machines within a Recovery Services Vault, to protect against accidental data loss.

  [Azure Backup](https://docs.microsoft.com/en-us/azure/backup/backup-azure-vms-introduction)
  
    -	Enable Soft Delete for the Recovery Services vault to protect against accidental or malicious deletion of backup data, ensuring the ability to recover.
    
  [Azre Backup Soft Delete](https://docs.microsoft.com/en-us/azure/backup/backup-azure-security-feature-cloud)

* Enable diagnostic logging for all virtual machines to ensure health metrics, boot diagnostics and infrastructure logs are routed to Log Analytics or an alternative log aggregation technology.

  [Diagnostic Logs](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/platform-logs-overview)


* Establish virtual machine Resource Health alerts to notify key stakeholders when resource health events occur.
    >*An appropriate threshold for resource unavailability must be set to minimize signal to noise ratios so that transient faults do not generate an alert. For example, configuring a virtual machine alert with an unavailability threshold of 1 minute before an alert is triggered.*

  [Resource Health Alerts](https://docs.microsoft.com/en-gb/azure/service-health/resource-health-alert-arm-template-guide)

 * To ensure application scalability while navigating within disk sizing thresholds, it is highly recommended that applications be installed on data disks rather than thee OS disk.

### Supporting Artifacts

* To identify resiliency risks to existing compute resources and support continuous compliance for new resources within a customer tenant, it is recommended that Azure Policy and Azure Resource Graph be used to Audit the use of non-resilient deployment configurations.

#### Azure Resource Graph

* Query to **identify standalone single instance VMs that are not protected by a minimum SLA of at least 99.5%**. It will return all VM instances that are not deployed within an Availability Set or across Availability Zones and are not using either Standard SSD or Premium SSD for both OS and Data disks.
    - This query can easily be altered to identify all single instance VMs including those using Premium Storage which are protected by a minimum SLA of at least 99.5%; simply remove the trailing where condition.

```kql
Resources
| where
    type =~ 'Microsoft.Compute/virtualMachines'
        and isnull(properties.availabilitySet.id)
    or type =~ 'Microsoft.Compute/virtualMachineScaleSets'
        and sku.capacity <= 1
        or properties.platformFaultDomainCount <= 1
| where 
    tags != '{"Skip":""}'
| where 
    isnull(zones)
| where
	properties.storageProfile.osDisk.managedDisk.storageAccountType !in ('Premium_LRS'
	or properties.storageProfile.dataDisks.managedDisk.storageAccountType != 'Premium_LRS'
	    and array_length(properties.storageProfile.dataDisks) != 0
```


* The following query expands on the identification of standalone instances by **identifying any Availability Sets containing single instance VMs**, which are exposed to the same risks as standalone single instances outside of an Availability Set.

```kql
Resources
| where 
    type =~ 'Microsoft.Compute/availabilitySets'
| where 
    tags != '{"Skip":""}'
| where 
	array_length(properties.virtualMachines) <= 1
| where
	properties.platformFaultDomainCount <= 1
```

#### Azure Policy

* Azure policy definition to **audit standalone single instance VMs that are not protected by a SLA**. It will flag an audit event for all Virtual Machine instances that are not deployed witin an Availability Set or across Availability Zones and are not using Premium Storage for both OS and Data disks. It also encompasses both Virtual Machine and Virtual Machine Scale Set resources.

[Audit VM/VMSS Standalone Instances](../src/compute/policydefinition_Audit-VMStandaloneInstances.json)

---

* Azure policy definition to **audit Availability Sets containing single instance VMs that are not protected by a SLA**. It will flag an audit event for all Availability Sets that does not contain multiple instances.

[Audit Availability Sets With Single Instances](../src/compute/policydefinition_Audit-AvailabilitySetSingleInstances.json)

## Azure Kubernetes Service (AKS)

### Design Considerations

#### Service Level Agreements

* For customers subscribing to the Azure Kubernetes Service (AKS) Uptime SLA, Microsoft guarantees 1) 99.95% availability of the Kubernetes API server endpoint for AKS Clusters that use Azure Availability Zones, and 2) 99.9% availability for AKS Clusters that not use Azure Availability Zones. For customers that do not wish to subscribe to the AKS uptime SLA, Microsoft provides a service level objective (SLO) of 99.5%.

* The SLA for agent (worker) nodes within an AKS cluster is covered by the standard [Virtual Machine SLA](#virtual-machines) which is dependent on the chosen deployment configuration and whether an Availability Set or Availability Zones are used.

  [AKS Service Level Agreements](https://azure.microsoft.com/support/legal/sla/kubernetes-service/v1_1/)
  [AKS Uptime SLA Offering](https://docs.microsoft.com/en-us/azure/aks/uptime-sla)

### Configuration Recommendations

* For all AKS clusters requiring resiliency, it is highly recommended that:

    - Use [Availability Zones](https://docs.microsoft.com/azure/aks/availability-zones) to maximize resilience within a region by distributing AKS agent nodes across physically separate data centers.

    - Where co-locality requirements exist, an Availability Set deployment can be used to minimize inter-node latency.

* Virtual Machine Scale Set deployment configurations should be used to unlock cluster autoscaling and the use of multiple node pools.

    -	Enable cluster [autoscaling](https://docs.microsoft.com/en-us/azure/aks/cluster-autoscaler) to adjust the number of agent nodes in response to resource constraints.

* Use [Managed Identities](https://docs.microsoft.com/azure/aks/use-managed-identity) to avoid having to manage and rotate service principles.

* Adopt a [multi-region strategy](https://docs.microsoft.com/en-gb/azure/aks/operator-best-practices-multi-region#plan-for-multiregion-deployment) by deploying AKS clusters deployed across different Azure regions to maximize availability and provide business continuity.

    -	Internet facing workloads should leverage Azure Front Door, [Azure Traffic Manager](https://docs.microsoft.com/en-gb/azure/aks/operator-best-practices-multi-region#use-azure-traffic-manager-to-route-traffic), or a third-party CDN to route traffic globally across AKS clusters.

* Store container images within Azure Container Registry and enable [geo-replication](https://docs.microsoft.com/azure/aks/operator-best-practices-multi-region#enable-geo-replication-for-container-images) to replicate container images across leveraged AKS regions. 

### Supporting Source Artifacts

#### Azure Resource Graph

* Query to identify AKS clusters that are not deployed across **Availability Zones**:

```kql
Resources
| where
    type =~ 'Microsoft.ContainerService/managedClusters'
	and isnull(zones)
```

* Query to identify AKS clusters that are deployed within a AvailabilitySet:

```kql
Resources
| where
    type =~ 'Microsoft.ContainerService/managedClusters'
	and properties.agentPoolProfiles[0].type != 'VirtualMachineScaleSets'
| project name, location, resourceGroup, subscriptionId, properties.agentPoolProfiles[0].type
```

* Query to identify AKS clusters that are not deployed using a **Managed Identity**:

```kql
Resources
| where
    type =~ 'Microsoft.ContainerService/managedClusters'
	and isnull(identity)
```
## Azure App Service

### Design Considerations

#### Service Level Agreements

* Microsoft guarantees that Apps will be available 99.95% of the time. However, no SLA is provided for Apps using either the Free or Shared tiers.

[SLA for App Service](https://azure.microsoft.com/en-us/support/legal/sla/app-service/v1_4/)

### Configuration Recommendations

#### App Service Plans

* Azure App Service provides a number of configuration options that are not enabled by default. For all App Services requiring resiliency, it is highly recommended that:

    - Use Basic or higher plans with 2 or more worker instances for high availability.

    - Evaluate the use of [TCP and SNAT ports](https://docs.microsoft.com/en-us/azure/app-service/troubleshoot-intermittent-outbound-connection-errors#cause) to avoid outbound connection errors
        >*TCP connections are used for all outbound connections whereas SNAT ports are used when making outbound connections to public IP addresses.*
        >
        >*SNAT port exhaustion is a common failure scenario that can be predicted by load testing while monitoring ports using Azure Diagnostics. If a load test results in SNAT errors, it is necessary to either scale across more/larger workers, or implement coding practices to help preserve and re-use SNAT ports, such as connection pooling and the lazy loading of resources.*
        >*It is recommended not to exceed 100 simultaneous outbound connections to a public IP Address per worker, and to avoid communicating with downstream services via public IP addresses when a private address (Private Endpoint) or Service Endpoint through vNet Integration could be used.*
        >
        >*TCP port exhaustion happens when the sum of connection from a given worker exceeds the capacity. The number of available TCP ports depend on the size of the worker. The following table lists the current limits:*
        >|  |Small (B1, S1, P1, I1)|Medium (B2, S2, P2, I2)|Large (B3, S3, P3, I3)|
        >|---------|---------|---------|---------|
        >|TCP ports|1920|3968|8064|
        >
        >*Applications with lots of longstanding connections require ports to be left open for long periods of time, which can lead to TCP Connection exhaustion. TCP Connection limits are fixed based on instance size, so it is necessary to scale up to a larger worker size to increase the allotment of TCP connections, or implement code level mitigations to govern connection usage. Similar to SNAT port exhaustion, Azure Diagnostics can be used to identify if a problem exists with TCP port limits.*

    - Enable [AutoHeal](https://azure.github.io/AppService/2018/09/10/Announcing-the-New-Auto-Healing-Experience-in-App-Service-Diagnostics.html) to automatically recycle unhealthy workers.
        >*This feature is currently only available to Windows Plans.*

    - Enable [Health Check](https://aka.ms/appservicehealthcheck) to identify non-responsive workers.
        >*Any health check is better than none at all, however, the logic behind endpoint tests should assess all critical downstream dependencies to ensure overall health. It is also recommended practice to track application health and cache status in real time as this removes unnecessary delays before  action can be taken.*

    - Enable [AutoScale](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/autoscale-get-started?toc=/azure/app-service/toc.json) to ensure adequate resources are available to service requests.
        >*The default limit of App Service workers is 30.  If the App Service routinely uses 15 or more instances, consider opening a support ticket to increase the maximum number of workers to 2x the instance count required to serve normal peak load.*

    - Enable [Local Cache](https://docs.microsoft.com/en-us/azure/app-service/overview-local-cache) to reduce dependencies on cluster file servers.
        >*Enabling local cache is not always appropriate because it can lead to slower worker startup times. However, when coupled with Deployment Slots, it can improve resiliency by removing dependencies on file servers and also reduces storage-related recycle events. However, Local cache should not be used with a single worker instance or when shared storage is required.*

    - Enable [Diagnostic Logging](https://docs.microsoft.com/en-us/Azure/app-service/troubleshoot-diagnostic-logs) to provide insight into application behavior.
        >*Diagnostic logging provides the ability to ingest rich application and platform level logs into either Log Analytics, Azure Storage, or a third party tool via Event Hub.*

    - Enable [Application Insights Alerts](https://docs.microsoft.com/en-us/Azure/azure-monitor/app/azure-web-apps) to be made aware of fault conditions.
        >*Application performance monitoring with Application Insights provides deep insights into application performance. For Windows Plans a 'codeless deployment' approach is possible to quickly get insights without changing any code.*

    - Review [Azure App Service diagnostics](https://docs.microsoft.com/en-us/azure/app-service/overview-diagnostics) to ensure common problems are addressed.
        >*It is a good practice to regularly review service-related diagnostics and recommendations and take action as appropriate.*

#### App Service Environments

* For App Service Environments, ensure ASE is deployed within in [highly available configuration](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/enterprise-integration/ase-high-availability-deployment) across Availability Zones.
    >*Configuring ASE to use Availability Zones by deploying ASE across specific zones ensures applications can continue to operate even in the event of a data center level failure. This provides excellent redundancy without requiring multiple deployments in different Azure regions.*

* For App Service Environments, ensure the [ASE Network](https://docs.microsoft.com/en-us/azure/app-service/environment/network-info) is configured correctly.
    >*One common ASE pitfall occurs when ASE is deployed into a subnet with an IP Address space that is too small to support future expansion. In such cases, ASE can be left unable to scale without redeploying the entire environment into a larger subnet. It is highly recomended that adequate IP addresses be used to support either the maximum number of workers or the largest number considered workloads will need. A single ASE cluster can scale to 201 instance, which would require a /24 subnet.*

* For App Service Environments, consider configuring [Upgrade Preference](https://docs.microsoft.com/en-us/azure/app-service/environment/using-an-ase#upgrade-preference) if multiple environments are used.
    >*If lower environments are used for staging or testing, consideration should be given to configuring these environments to receive updates sooner than the production environment. This will help to identify any conflicts or problems with an update and provides a window to mitigate issues before they reach the production environment.
    >If multiple load balanced (zonal) production deployments are used, upgrade preference can also be used to protect the broader environment against issues from platform upgrades.*

* For App Service Environments, plan for scaling out the ASE cluster
    >*Scaling ASE instances vertically or horizontally currently takes 30-60 minutes as new private instances need to be provisioned. It is highly recomended that effort be invested up-front to plan for scaling during spikes in load or transient failure scenarios.*

#### Application Deployment

* When deploying application code or configuration, it is highly recommended that:

- Use [Deployment Slots](https://docs.microsoft.com/en-us/azure/app-service/deploy-staging-slots) for resilient code deployments.
    >*Deployment Slots allow for code to be deployed to instances that are warmed-up before serving production traffic.*
    [Azure Friday](https://www.youtube.com/watch?v=MP8fXgxq6xo)
    [blog post](https://ruslany.net/2019/06/azure-app-service-deployment-slots-tips-and-tricks/)

- Avoid Unnecessary Worker restarts
    >There are a number of events that can lead App Service workers to restart, such as content deployment, App Settings changes, and VNet intergration configuration changes. It is best practice to make changes in a deployment slot other than the slot currently configured to accept production traffic. After workers are recycled and warmed up, a "swap" can be performed without unnecessary down time.

- Use ["Run From Package"](https://docs.microsoft.com/en-us/azure/app-service/deploy-run-package) to avoid deployment conflicts
    >*Run from Package provides several advantages:*
    >* *Eliminates file lock conflicts between deployment and runtime.*
    >* *Ensures only full-deployed apps are running at any time.*
    >* *May reduce cold-start times, particularly for JavaScript functions with large npm package trees.*

### Supporting Source Artefacts

#### Azure Resource Graph

* Query to identify App Service Plans with **only 1 instance**:

```kql
Resources
| where type == "microsoft.web/serverfarms" and properties.computeMode == 'Dedicated'
| where sku.capacity == 1
```

#### Additional Resources

* [The Ultimate Guide to Running Healthy Apps in the Cloud](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html)


## Service Fabric


### Design Considerations

#### Service Level Agreements

* Azure Service Fabric does not provide its own SLA. The availability of Service Fabric clusters is based on the underlying Virtual Machine and Storage resources used.

* Virtual Machine Scale Sets also do not have an SLA, since the SLA for Virtual Machines applies here. If the Virtual Machine Scale Set includes Virtual Machines in at least 2 Fault Domains, the availability of the underlying Virtual Machines SLA for two or more instances applies. If the scale set contains a single Virtual Machine, the availability for a Single Instance Virtual Machine applies.

[Service Fabric](https://azure.microsoft.com/en-us/support/legal/sla/service-fabric/v1_0/)

[Virtual Machine Scale Set](https://azure.microsoft.com/en-us/support/legal/sla/virtual-machine-scale-sets/v1_1/)

### Design Recommendations

#### Cluster Management & Deployment

* Review the [Service Fabric production readiness checklist](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-production-readiness-checklist)

* Use durability level Silver (5 VMs) or higher for production scenarios. 
> This will ensure the Azure infrastructure communicates with the Service Fabric controller on scheduling reboots, etc.

* Use reliability level Silver or higher for production scenarios.

* For critical workloads, consider using Availability Zones for your Service Fabric clusters. 
> This means deploying a primary NodeType (and by extension a VM ScaleSet) to each AZ. This will ensure that the Service Fabric system services are spread across zones.

### Networking

* To expose services on the Service Fabric cluster, use a reverse proxy such as the Service Fabric reverse proxy or Traefik. When exposing APIs hosted on the cluster, consider using Azure API Management.
> API Management can [integrate](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-api-management-overview) with Service Fabric directly.

* For production scenarios, use the Standard tier load balancer. The Basic SKU is free, but does not have an SLA.

* Keep the different node types and gateway services on different subnets.

* Apply Network Security Groups (NSG) to restrict traffic flow between subnets/node types. Ensure that the [correct ports](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-best-practices-networking#cluster-networking) are opened for managing the cluster. 
> For example, you may have an API Management instance (one subnet), a frontend subnet (exposing a website directly) and a backend subnet (accessible only to frontend), each implemented on a different VM Scale Set.

#### Security

* When using secrets (connection strings, passwords) in SF services, either retrieve them directly from Keyvault at runtime or use the [Service Fabric Secrets Store](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-application-secret-store)

* When using the Service Fabric Secret Store to distribute secrets, use a separate data encipherment certificate to encrypt the values. 
> This certificate is deployed to the VM scaleset nodes to decrypt the secret values. When using this approach, ensure that secrets are inserted and encrypted at release time. Using this approach means that changing the secrets requires a deployment. Make sure your key-rotation process is fully automated to do this without downtime.

* Do not use self-signed certificates for production scenarios. Either provision a certificate through your PKI or use a public certificate authority.

* Deploy certificates by adding them to Azure Keyvault and referencing the URI in your deployment.

* Have a process in place for monitoring the expiration date of certificates. 
> For example, Key Vault offers a feature that sends an email when x% of the certificate's lifespan has elapsed.

* Enable Azure Active Directory integration for your cluster to ensure users can access Service Fabric Explorer using their AAD credentials. Do not distribute the cluster certificate among users to access Explorer. 

* Exclude the Service Fabric processes from Windows Defender to improve performance
> By default, Windows Defender antivirus is installed on Windows Server 2016 and 2019. To reduce any performance impact and resource consumption overhead incurred by Windows Defender, and if your security policies allow you to exclude processes and paths for open-source software, you can [exclude](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-best-practices-security#windows-defender) the Service Fabric executables from Defender scans.


# Data 

## Azure SQL Database 	

###  Design Considerations	


* Azure SQL Database is a fully managed platform as a service (PaaS) database engine that handles most of the database management functions . Azure SQL Database is always running on the latest stable version of the SQL Server database engine and patched OS with 99.99% availability. PaaS capabilities that are built into Azure SQL Database enable you to focus on the domain-specific database administration and optimization activities that are critical for your business. 


* Azure SQL Database is having built-in regional high availability and turnkey geo-replication to any Azure region. It includes intelligence to support self-driving features such as performance tuning, threat monitoring, and vulnerability assessments and provides fully automated patching and updating of the code base.

* Azure SQL Database Business Critical or Premium tiers configured as Zone Redundant Deployments have an availability guarantee of at least 99.995%.
* Azure SQL Database Business Critical or Premium tiers not configured for Zone Redundant Deployments, General Purpose, Standard, or Basic tiers, or Hyperscale tier with two or more replicas have an availability guarantee of at least 99.99%.
* Azure SQL Database Hyperscale tier with one replica has an availability guarantee of at least 99.95% and 99.9% for zero replicas.
* Azure SQL Database Business Critical tier configured with geo-replication has a guarantee of Recovery point objective (RPO) of 5 sec for 100% of deployed hours.
* Azure SQL Database Business Critical tier configured with geo-replication has a guarantee of Recovery time objective (RTO) of 30 sec for 100% of deployed hours.

Find more details about SLA for Azure SQL Database [here](https://azure.microsoft.com/en-us/support/legal/sla/sql-database/v1_4/) . 

Read following section for Design and Configuration Recommendation to know how you can implement and acheive your Traget SLA 
	


* **Use point-in-time restore**  to recover from human error. Point-in-time restore returns your database to an earlier point in time to recover data from changes done inadvertently. For more information, read the PITR documentation for https://docs.microsoft.com/en-us/azure/azure-sql/database/recovery-using-backups#point-in-time-restore 
	
* **Use geo-restore to recover from a service outage**. You can restore a database on any SQL Database server or an instance database on any managed instance in any Azure region from the most recent geo-replicated backups. Geo-restore uses a geo-replicated backup as its source. You can request geo-restore even if the database or datacenter is inaccessible due to an outage. Geo-restore restores a database from a geo-redundant backup. For more information, see [Recover an Azure SQL database using automated database backups](https://docs.microsoft.com/azure/sql-database/sql-database-recovery-using-backups)
	
* **Use sharding**: Sharding is a technique of distributing data and processing across many identically structured databases , provides an alternative to traditional scale-up approaches both in terms of cost and elasticity. Consider using sharding to partition the database horizontally. Sharding can provide fault isolation. For more information, see [Scaling out with Azure SQL Database.](https://docs.microsoft.com/azure/sql-database/sql-database-elastic-scale-introduction)

* **Define an application performance SLA and monitor it with alerts**. Detecting quickly when your application performance inadvertently degrades below an acceptable level is important to maintain high resiliency. Use the monitoring solution defined above to set alerts on key query performance metrics to you can take action when the performance breaks the SLA. [Monitor Your Database](https://docs.microsoft.com/en-us/azure/azure-sql/database/monitor-tune-overview)

### Configuration Recommendations

* **Configure HA/DR that best feed your needs from following options**   : Azure Sql DB  offers the following capabilities for recovering from an outage, It is advised to implement one or combination of one or more depending on Business RTO/RPO requirements : 

	- **Use Active Geo-Replication** :  Use Active Geo-Replication to create a readable secondary in a different region. If your primary database fails, perform a manual failover to the secondary database. Until you fail over, the secondary database remains read-only. [Active geo-replication](https://docs.microsoft.com/en-us/azure/azure-sql/database/active-geo-replication-overview) enables you to create readable replicas and manually failover to any replica in case of a datacenter outage or application upgrade. Up to 4 secondaries are supported in the same or different regions, and the secondaries can also be used for read-only access queries. The failover must be initiated manually by the application or the user. After failover, the new primary has a different connection end point. 
	
	- **Use Auto Failover Groups** : A failover group can include one or multiple databases, typically used by the same application. Additionally, you can use the readable secondary databases to offload read-only query workloads. Because auto-failover groups involve multiple databases, these databases must be configured on the primary server. Auto-failover groups support replication of all databases in the group to only one secondary server or instance in a different region. Learn more about [AutoFailover Groups](https://docs.microsoft.com/en-us/azure/azure-sql/database/auto-failover-group-overview?tabs=azure-powershell)  and  [DR design](https://docs.microsoft.com/en-us/azure/azure-sql/database/designing-cloud-solutions-for-disaster-recovery).	
	
	- **Use Zone-Redundant database** : By default, the cluster of nodes for the premium availability model is created in the same datacenter. With the introduction of Azure Availability Zones, SQL Database can place different replicas of the Business Critical database to different availability zones in the same region. To eliminate a single point of failure, the control ring is also duplicated across multiple zones as three gateway rings (GW). The routing to a specific gateway ring is controlled by [Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview) (ATM). Because the zone redundant configuration in the Premium or Business Critical service tiers does not create additional database redundancy, you can enable it at no extra cost. Learn More on Zone-redundant databases [here](https://docs.microsoft.com/en-us/azure/azure-sql/database/high-availability-sla) 
		
* **Monitor your Azure Sql DB in near-real time to detect reliability incidents**. Use one of the available solutions to monitor SQL DB to detect potential reliability incidents early and make your databases more reliable. Choosing a near real-time monitoring solution is key to quickly react to incidents. Learn more details about Azure SQL Analytics [Here](https://docs.microsoft.com/en-us/azure/azure-monitor/insights/azure-sql#analyze-data-and-create-alerts)  
	
* **Backup your keys**: If you are not [using encryption keys in Azure key vault to protect your data](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-always-encrypted-azure-key-vault) , backup your keys!

* **Implement Retry Logic** :  Although Azure SQL Database is resilient on the transitive infrastructure failures, these failures might affect your connectivity. When a transient error occurs while working with SQL Database, make sure your code must be able to retry the call. Follow the link for detailed instruction on how to [Implement retry logic](https://docs.microsoft.com/en-us/azure/azure-sql/database/troubleshoot-common-connectivity-issues). 
	
## Azure SQL Managed Instance


### Design Considerations 

* **Use point-in-time restore to recover from human error**. Point-in-time restore returns your database to an earlier point in time to recover data from changes done inadvertently. For more information, read the [PITR documentation for managed instance](https://docs.microsoft.com/en-us/azure/azure-sql/database/recovery-using-backups#point-in-time-restore)

* **Use geo-restore to recover from a service outage**. Geo-restore restores a database from a geo-redundant backup into a managed instance in a different region. For more information, see [Recover a database using Geo-restore documentation](https://docs.microsoft.com/en-us/azure/azure-sql/database/auto-failover-group-overview)

* **Define an application performance SLA and monitor it with alerts**. Detecting quickly when your application performance inadvertently degrades below an acceptable level is important to maintain high resiliency. Use the monitoring solution defined above to set alerts on key query performance metrics to you can take action when the performance breaks the SLA.

* **Consider the time required for certain operations**. Make sure you separate time to thoroughly test the amount of time required to scale up and down (change the size) your existing managed instance, and to create a new managed instance. This will ensure that you understand completely how these time consuming operations will affect your RTO and RPO.
 
### Configuration Recommendations

* **Use Business Critical tier**. This tier provides higher resiliency to failures and faster failover times due to the underlying HA architecture, among other benefits. For more information, see [SQL Managed Instance High availability](https://docs.microsoft.com/en-us/azure/azure-sql/database/high-availability-sla)

* **Configure a secondary instance and an Auto-failover group to enable failover to another region**. If an outage impacts one or more of the databases in the managed instance, you can manually or automatically failover all the databases inside the instance to a secondary region. For more information, read the [Auto-failover groups documentation for managed instance](https://docs.microsoft.com/en-us/azure/azure-sql/database/auto-failover-group-overview)

* **Monitor your SQL MI instance in near-real time to detect reliability incidents**. Use one of the available solutions to monitor your SQL MI to detect potential reliability incidents early and make your databases more reliable. Choosing a near real-time monitoring solution is key to quickly react to incidents. For more information, read the [Azure SQL Managed Instance monitoring options](http://aka.ms/mi-monitoring)

* **Implement Retry Logic** :  Although Azure SQL MI is resilient on the transitive infrastructure failures, these failures might affect your connectivity. When a transient error occurs while working with SQL MI, make sure your code must be able to retry the call. Follow the link for detailed instruction on how to [Implement retry logic](https://docs.microsoft.com/en-us/azure/azure-sql/database/troubleshoot-common-connectivity-issues). 

## Cosmos DB

### Design Considerations
 
#### Service Level Agreements

* 99.99% SLAs for throughput, consistency, availability and latency for Database Accounts scoped to a single Azure region configured with any of the five Consistency Levels or Database Accounts spanning to multiple Azure regions, configured with any of the four relaxed Consistency Levels. 

* 99.999% SLA for read availability for Database Accounts spanning two or more Azure region.
 
* 99.999% SLA for both read and write availability with the configuration of multiple Azure regions as writable endpoints.
 
 [Cosmos DB Service Level Agreements](https://azure.microsoft.com/en-us/support/legal/sla/cosmos-db/v1_3/)

### Configuration Recommendations

* It is strongly recommended that you configure the Azure Cosmos accounts used for production workloads to enable [automatic failover](https://docs.microsoft.com/en-us/azure/cosmos-db/high-availability#multi-region-accounts-with-a-single-write-region-write-region-outage). 

* Session is default consistency level, and it is the most widely used [consistency level](https://docs.microsoft.com/en-us/azure/cosmos-db/consistency-levels). It is the recommended consistency level to start with as it receives data later but in the same order as the writes.

* Use [Azure Monitor](https://docs.microsoft.com/en-us/azure/cosmos-db/monitor-cosmos-db) to see the provisioned autoscale max RU/s (Autoscale Max Throughput) and the RU/s the system is currently scaled to (Provisioned Throughput).

* Understand your traffic pattern in order to pick the right option for [provisioned throughput types](https://docs.microsoft.com/en-us/azure/cosmos-db/how-to-choose-offer).

* For new applications, if you don’t know your traffic pattern yet, start at the entry point RU/s to avoid over-provisioning in the beginning.

* For existing applications:

    - Use Azure Monitor metrics to determine if your traffic pattern is suitable for autoscale. 

    - Find the normalized request unit consumption metric of your database or container. Normalized utilization is a measure of how much you are currently using your standard (manual) provisioned throughput. 

    - The closer the number is to 100%, the more you are fully using your provisioned RU/s. 

* Of all hours in a month, if you set provisioned RU/s T and use the full amount for 66% of the hours or more, it's estimated you'll save with standard (manual) provisioned RU/s. If you set autoscale max RU/s Tmax and use the full amount Tmax for 66% of the hours or less, it's estimated you'll save with autoscale.

* If multi-master option is enabled on Cosmos DB, it is important to understand [Conflict Types and Resolution Policies](https://docs.microsoft.com/en-us/azure/cosmos-db/conflict-resolution-policies).

* [Selecting partition key](https://docs.microsoft.com/en-us/azure/cosmos-db/partitioning-overview#choose-partitionkey) is a simple, but very important design choice:

    - You cannot change partition key after it's been created with the collection.

    - Your partition key should be a property that has a value which doesn not change. If a property is your partition key, you can't update that property's value.

    - Make sure picking a partition key which has a high cardinality. The property should have a wide range of possible values.

    - Your partition key should spread RU consumption and data storage evenly across all logical partitions. This ensures even RU consumption and storage distribution across your physical partitions.

    - Make sure you are running read queries with the partitioned column as it will reduce RU consumption and latency.

  * For query-intensive workloads, use Windows 64-bit instead of Linux or Windows 32-bit host processing. 

  * If client is consuming more than 50,000 RU/s, there could be bottleneck due to machine capping out on CPU or network utilization. If you reach this point, it is recommended to scale out client applications across multiple servers.

  * Call [OpenAsync](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.documents.client.documentclient.openasync?view=azure-dotnet) to avoid startup latency on first request.

  * In order to avoid network latency, collocate client in same region as cosmos db.

  * Increase the number of threads /tasks.

  * To reduce latency and CPU jitter, it is recommended to enable accelerated networking on client virtual machines both [Windows](https://docs.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-powershell) and [Linux](https://docs.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-cli).    

  * Implement [retry logic](https://docs.microsoft.com/en-us/azure/architecture/best-practices/retry-service-specific#cosmos-db) in your client.
  
### Supporting Source Artefacts

#### Azure Resource Graph

* In order to check if multi location is not selected you can use the following query: 

 ```kql
Resources
|where  type =~ 'Microsoft.DocumentDb/databaseAccounts'
|where array_length( properties.locations) <=1
``` 

* To check for cosmosdb instances where automatic failover is not enabled:

 ```kql
Resources
|where  type =~ 'Microsoft.DocumentDb/databaseAccounts'
|where properties.enableAutomaticFailover!=True
``` 

* Query to see the list of multi-region writes:

```kql
resources
| where type == "microsoft.documentdb/databaseaccounts"
 and properties.enableMultipleWriteLocations == "true"
``` 

* To see the consistency levels for your cosmos db accounts you can use the query below: 

 ```kql
Resources
| project name, type, location, consistencyLevel = properties.consistencyPolicy.defaultConsistencyLevel 
| where type == "microsoft.documentdb/databaseaccounts" 
| order by name asc
``` 

#### Additional Resources

* [High Availability in Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/high-availability)

* [Auto-Scale FAQ](https://docs.microsoft.com/en-us/azure/cosmos-db/autoscale-faq)

* [Performance Tips for Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/performance-tips)


# Messaging

## Event Grid

### Design Considerations

#### Service Level Agreements

* Azure EventGrid provides a 99.99% uptime SLA

### Configuration Recommendations

* In case of a multi-region Azure solution, deploy an Event Grid instance per region.

* In high-throughput scenarios, use batched events. This means that the service will deliver a json array with multiple events to the subscribers, instead of an array with one event. The consuming application must be able to process these arrays.

* Event batches cannot exceed 1MB in size. This means that if the message payload is large, only one or a few messages will fit in the batch. This means that the consuming service will need to process more event batches. If your event has a large payload, consider storing it elsewhere (e.g. blob storage) and passing a reference in the event. When integrating with third-party services through the CloudEvents schema, it is recommended [not to exceed 64kb events](https://github.com/cloudevents/spec/blob/v1.0/spec.md#size-limits).

* Batch size selection depends on the payload size and the message volume. This should be a configurable parameter and optimizing this should be done during load-testing. 

* If EventGrid delivers to an endpoint that holds custom code, ensure that the message is accepted with an HTTP 200-204 response only when it can be successfully processed. 

* Monitor EventGrid for failed event publishing (Publish Failed metric). Additionally, the 'Unmatched' metric will show messages that are published, but not matched to any subscription. Depending on your application architecutre, the latter may be intentional.

* Monitor EventGrid for failed event delivery. The 'Delivery Failed' metric will increase every time a message cannot be delivered to an event handler (timeout or a non 200-204 HTTP status code). Additionally, if an event must not be lost, set up a Dead-Letter-Queue (DLQ) storage account. This is where events that cannot be delivered after the maximum retry count will be placed. Optionally, implement a notification system on the DLQ storage account, e.g. by handling a 'new file' event through Event Grid.

## Event Hub


### Design Considerations

#### Service Level Agreements

* Azure Event Hubs has a [published SLA](https://azure.microsoft.com/en-us/support/legal/sla/event-hubs) of 99.95% for the Basic and Standard Tiers, and 99.99% for the Dedicated Tier.

### Configuration Recommendations

* The number of partitions reflect the degree of downstream paralellism you can achieve. For maximum throughput, use the maximum number of partitions (32) when creating the Event Hub. This will allow you to scale up to 32 concurrent processing entities and will offer the highest send/receive availability.

* In high-throughput scenarios, use batched events. This means that the service will deliver a json array with multiple events to the subscribers, instead of an array with one event. The consuming application must be able to process these arrays.

* As part of your solution-wide availability and disaster recovery strategy, consider enabling the EventHub geo disaster-recovery option. This will allow the creation of a seconary namespace in a different region. Note that only the active namespace receives messages at any time and that messages and events themselves are not replicated to the secondary region. 
> Note: The RTO for the regional failover is 'up to 30 minutes'. Confirm this aligns with the requirements of the customer and fits in the broader availability strategy. If a higher RTO is required, consider implementing a client-side failover pattern too.

* When developing new applications, use EventProcessorClient (.Net and Java) or EventHubConsumerClient (Python and Javascript) as the client SDK. EventProcessorHost has been deprecated. 

* Every consumer can read events from 1 to 32 partitions. To achieve maximum scale on the side of the consuming application, every consumer should read from a single partition. 

* Do not publish events to a specific partition. If ordering of events is essential, implement this downstream or use a different messaging service instead.

* Create SendOnly and ListenOnly policies for the event publisher and consumer, respecively. 

* When publishing events frequently, use the AMQP protocol when possible. AMQP has higher network costs when initializing the session, however HTTPS requires additional TLS overhead for every request. AMQP has higher performance for frequent publishers.

* When a solution has a large number of independent event publishers, consider using Event Publishers for fine-grained access control. Note that is automatically sets the partition key to the publisher name, so this should only be used if the events originate from all publishers evenly. 

* When using the Capture feature, carefully consider the configuration of the time window and file size, especially with low event volumes. Data Lake will charge small for a minimal file size for storage (gen1) or minimal transaction size (gen2). This means that if you set the time window so low that the file has not reached minimum size, you will incur a lot of extra cost.

* Ensure each consuming application uses a seprate consumer group and only one active receiver per consumer group is in place. 

* When using the SDK to send events to Event Hubs, ensure the exceptions thrown by the [retry policy](https://docs.microsoft.com/en-us/azure/architecture/best-practices/retry-service-specific#event-hubs) (EventHubsException or OperationCancelledException) are properly caught. When using HTTPS, ensure a proper retry pattern is implemented.

### Supporting Source Artefacts

#### Azure Resource Graph
* Find Event Hub namespaces with 'Basic' SKU:

```KQL
Resources 
| where type == 'microsoft.eventhub/namespaces'
| where sku.name == 'Basic'
| project resourceGroup, name, sku.name
```

## Service Bus

### Design Considerations

#### Service Level Agreements

* For Service Bus Queues and Topics, Microsoft guarantees that at least 99.9% of the time, properly configured applications will be able to send or receive messages or perform other operations on a deployed Queue or Topic. [SLA Documentation](https://azure.microsoft.com/en-us/support/legal/sla/service-bus)

* However, when deploying Service Bus with Geo-disaster recovery and in availability zones, the SLO increases dramatically, but does not change the financially backed SLA of 99.9% availability.

#### Service Bus Standard Features (not supported in Premium)

* [Partitioned Queues and Topics](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-partitioning)
* [Express Entities](https://docs.microsoft.com/en-us/dotnet/api/microsoft.servicebus.messaging.queuedescription.enableexpress)

#### Service Bus Premium Features

* In addition to the documentation on [Service Bus Premium and Standard messaging tiers](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-premium-messaging), these features are only available on the Premium SKU.

    - Dedicated resources
    - Virtual network integration: Limits the networks that can connect to the Service Bus instance. Requires Service Endpoints to be enabled on the subnet. There are Trusted Microsoft services that are not supported when Virtual Networks are implemented (i.e., integration with Event Grid) https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-service-endpoints
    - Private endpoints
    - IP Filtering/Firewall: Restrict connections to only defined IPv4 addresses or IPv4 address ranges
    - [Availability zones](https://docs.microsoft.com/en-us/azure/availability-zones/az-overview): Provides enhanced availability by spreading replicas across availability zones within one region at no additional cost
    - Event Grid Integration: [Available event types](https://docs.microsoft.com/en-us/azure/event-grid/event-schema-service-bus)
    - Scale Messaging Units
    - [Geo-Disaster Recovery](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-geo-dr) (paired namespace)
    - BYOK (Bring Your Own Key): Azure Service Bus encrypts data at rest and automatically decrypts it when accessed, but customers can also bring their own customer-managed key.

### Configuration Recommendations

* If you need mission critical messaging with queues/topics, Service Bus Premium is recommended with Geo-Disaster Recovery. Choosing the pattern is dependent on the busincess requirements and the recovery time objective (RTO).

* Geo-Disaster
    - [Active/Active](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-outages-disasters#active-replication)
    - [Active/Passive](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-outages-disasters#passive-replication)
    - [Paired Namespace (Active/Passive)](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-geo-dr)
    - ** NOTE: the secondary region should be an Azure paired region. https://docs.microsoft.com/en-us/azure/best-practices-availability-paired-regions 

* Make the namespace zone redundant (only available with Premium)

* Review the Service Bus performance improvements documentation https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-performance-improvements

* Connect to Service Bus with the AMQP protocol and utilize Service Endpoints or Private Endpoints when possible. This keeps traffic on the Azure Backbone. Note: the default connection protocol for Microsoft.Azure.ServiceBus and Windows.Azure.ServiceBus namespaces is AMQP.

#### Messaging Exceptions
* Ensure that Service Bus messaging exceptions are handled properly.
[Service Bus Messaging Exceptions](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-exceptions)

### Supporting Source Artefacts

#### Azure Resource Graph

* Query to identify Service Bus Instances that are not on the premium tier:

```kql
Resources
| where
	type == 'microsoft.servicebus/namespaces'
| where
	sku.tier != 'Premium'
```

* Query to identify premium Service Bus Instances that are not zone redundant:

```kql
Resources
| where
	type == 'microsoft.servicebus/namespaces'
| where
	sku.tier == 'Premium'
	and properties.zoneRedundant == 'false'
```

* Query to identify premium Service Bus Instances that are not using private endpoints:

```kql
Resources
| where
	type == 'microsoft.servicebus/namespaces'
| where
	sku.tier == 'Premium'
	and isempty(properties.privateEndpointConnections)
```


## Storage Queues

### Design Considerations

#### Service Level Agreements
* Azure Storage Queues follow the SLA statements of the general [Storage Account service](https://azure.microsoft.com/en-us/support/legal/sla/storage/v1_5/). Currently (v1.5) this specifies a 99.9% guarantee for LRS, ZRS and GRS accounts and a 99.99% guarantee for RA-GRS (provided that requests to RA-GRS switch to secondary endpoints if there is no success on the primary endpoint)

### Configuration Recommendations

* Since Storage Queues are part of the Azure Storage service, please refer to the general storage guidance, in addition to the configuration recommendations mentioned below.

* Using geo-zone-redundant storage (GZRS) or read-access geo-zone-redundant storage (RA-GZRS) will provide 16 nines or durability and will protect against failover if an entire datacenter becomes unavailable. https://docs.microsoft.com/en-us/azure/storage/common/storage-redundancy 

* For an SLA increase from three to four nines, use geo-redundant storage with read access and configure the client application to fail over to secondary read endpoints if the primary endpoints fail to respond. This consideration should be part of the overall reliability strategy of your solution.

* Refer to the 'Storage' guidance for specifics on data recovery for storage accounts

* Ensure that for all clients accessing the storage account, a proper [retry policy](https://docs.microsoft.com/en-us/azure/architecture/best-practices/retry-service-specific#azure-storage) is implemented.

### Supporting Source Artefacts

#### Azure Resource Graph

* Query to identify storage accounts using V1 storage accounts:

```kql
Resources
| where
	type == 'microsoft.storage/storageaccounts'
	and kind == 'Storage'
```

* Query to identify storage accounts using locally redundant storage (LRS):

```kql
Resources
| where
	type == 'microsoft.storage/storageaccounts'
	and sku.name =~ 'Standard_LRS'
```


# Hybrid

## Azure Stack Hub

### Design Considerations

#### Service Level Agreements

* **Microsoft does not provide an SLA for Azure Stack Hub** because Microsoft does not have control over customer datacenter reliability, people, and processes.

#### Current Limitations

* Azure Stack Hub currently only supports a single Scale Unit (SU) within in a single Region, which can consist of between 4 and 16 servers that use Hyper-V failover clustering; each region serves as an independent Azure Stack Hub "stamp" with separate portal and API endpoints.
  > Azure Stack Hub does therefore **not support Availability Zones** as it currently consists only of a single "region" (aka a single physical location). High availability to cope with outages of a single location should be implemented by using two Azure Stack Hub instances deployed into different pyhsical locations.

* Azure Stack Hub supports **Premium Storage** to ensure compatibility, however, provisioning premium storage accounts or disks does not guarantee that storage objects will be allocated onto SSD or NVMe drives.

* Azure Stack Hub supports only a subset of [VPN Gateway SKUs](https://docs.microsoft.com/en-us/azure-stack/user/azure-stack-vpn-gateway-about-vpn-gateways#estimated-aggregate-throughput-by-sku) available in Azure with a limited bandwidth of 100 or 200 Mbps. 
  > Only one site-to-site (S2S) VPN connection can be created between two Azure Stack Hub deployments. This is due to a limitation in the platform that only allows a single VPN connection to the same IP address. Multiple S2S VPN connections with higher throughput can be established using 3rd-party NVAs.

* Azure Stack Hub does currently not support [Virtual network peering](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview). 
  > Two networks (on the same Azure Stack Hub "stamp") can also not be connected via Azure (Stack) VPN GWs as they're sharing the same IP address. Virtual networks on Azure Stack Hub can be connected using 3rd-party NVAs (e.g. [Fortinet Fortigate](https://docs.microsoft.com/en-us/azure-stack/user/azure-stack-network-howto-vnet-to-vnet?view=azs-2002)).

### Configuration Recommendations

* Treat Azure Stack Hub as a scale unit and deploy multiple instances to remove Azure Stack Hub as a single point of failure for encompassed workloads. 

    -	Deploy workloads in either an active-active or active-passive configuration across Azure Stack Hub stamps and/or Azure.

* Apply general Azure configuration recommendations for all Azure Stack Hub services.