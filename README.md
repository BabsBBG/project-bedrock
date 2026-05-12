# Project Bedrock — Azure Secure Baseline

## Scenario

ClearVault Financial, a fintech company based in the UK, is set to transition its core operations to Azure. As the dedicated cloud security engineer, the goal was to create and implement a governed, identity-aware infrastructure baseline prior to the deployment of any application workloads. The guiding principle is that security and governance should not be added later; they must serve as the foundation upon which everything else is constructed.

This project establishes a secure baseline environment that all future ClearVault Azure workloads will inherit. Every control is enforced at the platform level, validated through intentional testing, and documented with supporting evidence.

---

<div align="center">
  <a href="https://medium.com/@tobibabalola21/before-you-deploy-anything-building-a-governed-azure-baseline-from-scratch-95cf385aa32b" target="_blank">
    <img src="https://img.shields.io/badge/📄_Read_the_Full_Deep_Dive_on-Medium-black?style=for-the-badge" alt="Medium Article" />
  </a>
</div>

## Architecture Overview

- **Management Group** → subscription → platform and workload resource groups
- **Hub VNet** (UK South) peered with **Spoke VNet** - hub and spoke topology
- **West Europe VNet** hosting compute resources - multi-region deployment due to subscription quota constraints
- Three-tier subnet design: app, management, data - each with dedicated NSGs
- Private DNS Zone linked to hub VNet with auto-registration enabled
- Storage account locked to app-subnet via service endpoint - default network action: Deny
- Ubuntu management VM with system-assigned managed identity scoped to audit-logs container only

---

## Modules

### Module 1 — Governance Foundation

Governance is established first before the deployment of infrastructure. Azure Policy mandates essential tags - Environment, Owner, CostCentre, and limits deployments to the UK South and West Europe regions exclusively. No resource is permitted to exist outside of approved areas or without the necessary classification metadata. Both policies are implemented at the management group level and roll down to every resource within the subscription.

RBAC is allocated to Entra ID security groups instead of individual users, creating a streamlined JML-ready access model from the outset. Three groups - platform admins, workload contributors, and readonly auditors, are defined specifically within their operational limits. A custom role `ClearVault-NetworkReviewer` grants read-only access to network information for auditors, prohibiting any write capabilities. A CanNotDelete lock on the platform resource group guarantees that the core infrastructure cannot be accidentally or unauthorizedly destroyed, irrespective of the RBAC role.

![Management Group Created](Governance/bedrock-m1-t1-management-group-created.png)
*Management group with subscription nested underneath*

![Tag Policies Assigned](Governance/bedrock-m1-t2-tag-policies-assigned.png)
*Three tag policies assigned at management group scope*

![Allowed Regions Policy](Governance/bedrock-m1-t3-allowed-regions-policy.png)
*Allowed Locations policy scoped to UK South and West Europe*

![Resource Groups Created](Governance/bedrock-m1-t4-resource-groups-created.png)
*Platform and workload resource groups in UK South*

![Platform RG Lock](Governance/bedrock-m1-t5-platform-rg-lock.png)
*CanNotDelete lock on clearvault-platform-rg*

![Entra Groups Created](Governance/bedrock-m1-t6-entra-groups-created.png)
*Three Entra ID security groups created*

![Platform Admins Assignment](Governance/bedrock-m1-t7a-platform-admins-owner-assignment.png)
*clearvault-platform-admins assigned Owner on platform RG*

![Workload Contributors Assignment](Governance/bedrock-m1-t7b-workload-contributors-assignment.png)
*clearvault-workload-contributors assigned Contributor on workload RG*

![Auditors Subscription Assignment](Governance/bedrock-m1-t7c-readonly-auditors-subscription-assignment.png)
*clearvault-readonly-auditors assigned Reader at subscription scope*

![NetworkReviewer Permissions](Governance/bedrock-m1-t8a-auditor-networkreviwer-permissions.png)
*ClearVault-NetworkReviewer custom role - Microsoft.Network read permissions*

![Auditors NetworkReviewer Assignment](Governance/bedrock-m1-t8b-auditors-networkreviewer-assignment.png)
*Auditors group assigned ClearVault-NetworkReviewer role*

---

### Module 2 — Network Architecture

A hub and spoke VNet topology effectively seperates platform shared services from workload resources.

Three subnets - app, management, and data - each have a dedicated NSG that reflects their respective trust levels. The app subnet only allows HTTPS inbound traffic. The management subnet permits SSH access solely from a single approved IP address. The data subnet blocks all inbound traffic and limits outbound traffic exclusively to the app subnet.

Each NSG is managed independently, ensuring there are no shared NSGs across the subnets.

VNet peering establishes bidirectional links between the hub and spoke. A Private DNS Zone `internal.clearvault.com` facilitates internal name resolution with auto-registration enabled, allowing VMs deployed in the linked VNet to register automatically. Additionally, a service endpoint on the app subnet extends the private network boundary directly to the storage account, allowing for subnet-scoped access controls without the complications of private endpoint overhead.

![Hub VNet Overview](Networking/bedrock-m2-t1a-hub-vnet-overview.png)
*Hub VNet overview - address space 10.10.0.0/16*

![Hub VNet Subnets](Networking/bedrock-m2-t1b-hub-vnet-subnets.png)
*Three subnets - app, mgmt, data — with address ranges*

![NSGs Created](Networking/bedrock-m2-t2-nsgs-created.png)
*Three dedicated NSGs created in UK South*

![App Subnet NSG Rules](Networking/bedrock-m2-t3a-app-subnet-nsg-rules.png)
*App subnet NSG - Allow HTTPS inbound, Deny all*

![Mgmt Subnet NSG Rules](Networking/bedrock-m2-t3b-mgmt-subnet-nsg-rules.png)
*Management subnet NSG - Allow SSH from approved IP only*

![Data Subnet NSG Rules](Networking/bedrock-m2-t3c-data-subnet-nsg-rules.png)
*Data subnet NSG - Deny all inbound, restrict outbound to app-subnet only*

![NSG Subnet Associations](Networking/bedrock-m2-t4-nsg-subnet-associations.png)
*All three subnets with respective NSGs associated*

![Spoke VNet Created](Networking/bedrock-m2-t5-spoke-vnet-created.png)
*Spoke VNet - address space 10.20.0.0/16*

![VNet Peering Connected](Networking/bedrock-m2-t6-vnet-peering-connected.png)
*Hub-to-spoke peering status Connected*

![Private DNS Zone Created](Networking/bedrock-m2-t7-private-dns-zone-created.png)
*internal.clearvault.com Private DNS Zone*

![Hub VNet Link CLI](Networking/bedrock-m2-t8a-hub-vnet-link-cli-connected.png)
*VNet link created via CLI - provisioningState Succeeded with tags confirmed*

![Hub VNet Link Portal](Networking/bedrock-m2-t8b-hub-vnet-link-portal-connected.png)
*hub-vnet-link visible in portal with auto-registration enabled*

![A Record Created](Networking/bedrock-m2-t9-a-record-created.png)
*test.internal.clearvault.com A record pointing to 10.10.1.10*

![Service Endpoint Enabled](Networking/bedrock-m2-t10-service-endpoint-enabled.png)
*Microsoft.Storage service endpoint enabled on app-subnet*

---

### Module 3 — Storage

A storage account has public blob access disabled and enforces TLS 1.2 as the minimum standard. Network rules limit access solely to the app-subnet through a service endpoint, with the default network action set to Deny. This means that any traffic not coming from the approved subnet is rejected, regardless of the authentication credentials.

There are two containers: `app-data` and `audit-logs`, both configured for private access. A SAS token is issued with Read and List permissions on Blob resources only, featuring a 24-hour expiration and an HTTPS-only protocol. This setup illustrates time-limited delegated access without revealing account keys. A lifecycle policy is in place to automate data tiering: Cool after 30 days, Archive after 90 days, and deletion after 365 days, in accordance with standard fintech data retention requirements. Blob versioning on audit-logs guarantees tamper evidence and the ability to recover logs. Microsoft Defender for Storage offers ongoing threat detection.

![Storage Account Created](Storage/bedrock-m3-t1-storage-account-created.png)
*clearvaultstg01 in UK South under clearvault-workload-rg*

![Public Access Disabled TLS12](Storage/bedrock-m3-t2-annonymous-access-disabled-tls12.png)
*Blob annonymous access disabled, minimum TLS 1.2 enforced*

![Network Rules App Subnet](Storage/bedrock-m3-t3-network-rules-app-subnet.png)
*Storage network rules - app-subnet only, default action Deny*

![Containers Created](Storage/bedrock-m3-t4-containers-created.png)
*app-data and audit-logs containers with private access*

![SAS Token Configuration](Storage/bedrock-m3-t5-sas-token-configuration.png)
*SAS token - Read and List only, HTTPS only, 24 hour expiry*

![Lifecycle Policy Configured](Storage/bedrock-m3-t6-lifecycle-policy-configured.png)
*Lifecycle policy - Cool at 30 days, Archive at 90 days, Delete at 365 days*

![Defender Storage Enabled](Storage/bedrock-m3-t7-defender-storage-enabled.png)
*Microsoft Defender for Storage enabled*

![Blob Versioning Enabled](Storage/bedrock-m3-t8-blob-versioning-enabled.png)
*Blob versioning enabled on storage account*

---

### Module 4 — Compute

An Ubuntu 22.04 management virtual machine has been deployed in the management subnet without a public IP address. The planned method for production access is through Azure Bastion, which completely removes direct exposure to the internet. A Network Interface Card (NIC)-level Network Security Group (NSG) adds an additional layer of network control on top of the subnet NSG, ensuring that SSH access is limited to traffic that originates solely from within the management subnet. This layered defense strategy requires that traffic must navigate through two separate NSG checkpoints before it can access the VM.

A system-assigned managed identity has been activated and assigned the `Storage Blob Data Reader` role, specifically scoped to the `audit-logs` container, excluding the storage account and the resource group, focusing solely on the exact container. This approach completely removes the need for stored credentials and enforces the principle of least privilege at the identity level. Additionally, boot diagnostics have been enabled to provide operational visibility during incident response.

![VM Basics Configuration](Compute/bedrock-m4-t1-vm-basics-configuration.png)
*clearvault-mgmt-vm01 - Ubuntu 22.04, West Europe*

![Networking No Public IP](Compute/bedrock-m4-t2-networking-no-public-ip.png)
*VM networking - mgmt-subnet selected, no public IP*

![NIC NSG Rules](Compute/bedrock-m4-t3-nic-nsg-rules.png)
*NIC-level NSG - SSH restricted to mgmt-subnet only*

![Managed Identity Enabled](Compute/bedrock-m4-t4-managed-identity-enabled.png)
*System-assigned managed identity enabled on VM*

![Storage Role Assignment](Compute/bedrock-m4-t5-storage-role-assignment-audit-logs.png)
*Storage Blob Data Reader assigned to VM managed identity - audit-logs container scope only*

![Boot Diagnostics Enabled](Compute/bedrock-m4-t6-boot-diagnostics-enabled.png)
*Boot diagnostics enabled with managed storage account*

![VM Tags Verified](Compute/bedrock-m4-t7-vm-tags-verified.png)
*Environment, Owner, CostCentre tags confirmed on VM*

---

### Module 5 — Governance Validation

Controls are only meaningful when tested. Each governance layer was validated through deliberate failure attempts and documented with evidence.

| Test | Control Validated | Result |
|---|---|---|
| Deploy resource to East US | Allowed Locations Policy | Denied |
| Create resource without tags | Require Tag Policy | Denied |
| Auditor attempts resource creation | Reader RBAC scope | AuthorizationFailed |
| Contributor accesses platform RG | Contributor scope boundary | AuthorizationFailed |
| Owner attempts to delete platform RG | CanNotDelete lock | Blocked |
| Managed identity role scope | Least privilege on audit-logs | Storage Blob Data Reader — audit-logs container only |
| Portal access to storage from public internet | Storage network rules | 403 Unauthorised |

![Region Policy Denial](Governance%20Validation/bedrock-m5-t1-region-policy-denial-eastus.png)
*Allowed Locations policy blocking deployment to East US*

![Tag Policy Denial](Governance%20Validation/bedrock-m5-t2-tag-policy-denial-no-tags.png)
*Tag policy denying resource creation without required tags*

![Auditor Authorization Failed](Governance%20Validation/bedrock-m5-t3-auditor-authorization-failed.png)
*Reader RBAC scope blocking auditor account from creating resources*

![Contributor Platform RG Blocked](Governance%20Validation/bedrock-m5-t4-contributor-platform-rg-blocked.png)
*Contributor account blocked from accessing clearvault-platform-rg*

![Lock Deletion Blocked](Governance%20Validation/bedrock-m5-t5-lock-deletion-blocked.png)
*CanNotDelete lock preventing platform RG deletion by subscription Owner*

![Managed Identity Role Scope](Governance%20Validation/bedrock-m5-t6-managed-identity-role-scope-cli.png)
*CLI output confirming Storage Blob Data Reader scoped to audit-logs container only*

![Storage 403 Public Access](Governance%20Validation/bedrock-m5-t7-storage-403-public-access-blocked.png)
*Storage network rules returning 403 on public internet access attempt*

---

## Observations & Deviations

**VM region — West Europe**
The subscription is unable to support VM creation in UK South because of quota restrictions. The compute module has been deployed in West Europe, which falls within the approved regions specified by the Allowed Locations policy. A dedicated VNet `clearvault-hub-vnet-we` was established in West Europe to accommodate the management VM, ensuring the proper subnet architecture is maintained. This is recorded as a subscription limitation, rather than a design compromise.

**Tag policy scope — resource groups exempt**
During Module 5 validation i discovered that the Require a tag on resources policy does not apply to resource groups themselves  only to resources deployed within them. This is expected Azure Policy behaviour. The tag enforcement test was adjusted to attempt a storage account creation without tags, which correctly produced a policy denial. This discovery was captured as part of the validation findings.

**Private DNS Zone VNet link — portal bug**
The creation of the VNet link in the Azure portal encountered an undefined error. However, the link was successfully established using the Azure Cloud Shell CLI. The CLI command contained the necessary tags, which inadvertently caused a policy denial during the initial attempt, serving as an unintentional yet valid example of the enforcement of the tag policy. The command with the correct tags was successful. This is a recognized intermittent issue with the portal and does not impact the functionality of the DNS zone.

**Module 5 RBAC test — contributor visibility**
The contributor test account for the workload was only able to view `clearvault-workload-rg`, while `clearvault-platform-rg` was completely hidden. This aligns with the correct RBAC behavior: a Reader role is necessary to view a resource group in the portal, and the contributor account lacks any role at the subscription or platform level. The test was adjusted to show that the account was unable to create a new resource group, an action that necessitates subscription-level permissions, resulting in an AuthorizationFailed message as anticipated.

---

## Skills Demonstrated

- Azure governance - management groups, Policy deny effects, resource locks
- Identity & access management - RBAC scoping, custom roles, Entra ID groups, managed identity least privilege
- Network security - hub and spoke VNet design, per-subnet NSGs, VNet peering, private DNS, service endpoints, defence in depth
- Storage security - network rules, SAS token scoping, lifecycle management, Defender for Storage, blob versioning
- Governance validation - deliberate control testing with documented evidence and deviation analysis

---

## AZ-104 Domain Coverage

- Manage Identities & Governance
- Implement & Manage Storage
- Deploy & Manage Virtual Networking
- Deploy & Manage Compute Resources

---

## Related Projects

- [Project Citadel](https://github.com/blvckbillgates/project-citadel) - Azure secure environment with Sentinel SIEM detection and incident response
