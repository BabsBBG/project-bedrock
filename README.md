# Project Bedrock — Azure Secure Baseline

## Scenario

ClearVault Financial is a UK-based fintech preparing to migrate core operations to Azure. As the engaged cloud security engineer, the objective was to design and deploy a governed, identity-aware infrastructure baseline before any application workloads land. The principle: security and governance are not retrofitted — they are the foundation everything else is built on.

This project establishes the secure baseline environment that all subsequent ClearVault Azure workloads inherit. Every control is enforced at the platform level, validated through deliberate testing, and documented with evidence.

---

## Architecture Overview

- **Management Group** → subscription → platform and workload resource groups
- **Hub VNet** (UK South) peered with **Spoke VNet** — hub and spoke topology
- **West Europe VNet** hosting compute resources — multi-region deployment due to subscription quota constraints
- Three-tier subnet design: app, management, data — each with dedicated NSGs
- Private DNS Zone linked to hub VNet with auto-registration enabled
- Storage account locked to app-subnet via service endpoint — default network action: Deny
- Ubuntu management VM with system-assigned managed identity scoped to audit-logs container only

*Full architecture diagram — see `/diagrams/bedrock-architecture.drawio`*

---

## Modules

### Module 1 — Governance Foundation

Governance is defined before infrastructure is deployed. Azure Policy enforces required tags — Environment, Owner, CostCentre — and restricts deployments to UK South and West Europe only. No resource can land outside approved regions or without classification metadata. Both policies are assigned at the management group level and inherit down to every resource in the subscription.

RBAC is assigned to Entra ID security groups rather than individuals, establishing a clean JML-ready access model from day one. Three groups — platform admins, workload contributors, and readonly auditors — are scoped precisely to their operational boundaries. A custom role `ClearVault-NetworkReviewer` provides read-only network visibility for auditors without any write permissions. A CanNotDelete lock on the platform resource group ensures no accidental or unauthorised destruction of core infrastructure regardless of RBAC role.

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
*ClearVault-NetworkReviewer custom role — Microsoft.Network read permissions*

![Auditors NetworkReviewer Assignment](Governance/bedrock-m1-t8b-auditors-networkreviewer-assignment.png)
*Auditors group assigned ClearVault-NetworkReviewer role*

---

### Module 2 — Network Architecture

A hub and spoke VNet topology separates platform shared services from workload resources. Three subnets — app, management, and data — each carry a dedicated NSG with rules reflecting their trust level. App subnet accepts HTTPS inbound only. Management subnet accepts SSH from a single approved IP only. Data subnet denies all inbound and restricts outbound to the app subnet only. Each NSG is independently managed — no shared NSGs across subnets.

VNet peering connects hub and spoke with bidirectional links. A Private DNS Zone `internal.clearvault.com` provides internal name resolution with auto-registration enabled — VMs deployed into the linked VNet register automatically. A service endpoint on app-subnet extends the private network boundary directly to the storage account, enabling subnet-scoped access controls without private endpoint overhead.

![Hub VNet Overview](Networking/bedrock-m2-t1a-hub-vnet-overview.png)
*Hub VNet overview — address space 10.10.0.0/16*

![Hub VNet Subnets](Networking/bedrock-m2-t1b-hub-vnet-subnets.png)
*Three subnets — app, mgmt, data — with address ranges*

![NSGs Created](Networking/bedrock-m2-t2-nsgs-created.png)
*Three dedicated NSGs created in UK South*

![App Subnet NSG Rules](Networking/bedrock-m2-t3a-app-subnet-nsg-rules.png)
*App subnet NSG — Allow HTTPS inbound, Deny all*

![Mgmt Subnet NSG Rules](Networking/bedrock-m2-t3b-mgmt-subnet-nsg-rules.png)
*Management subnet NSG — Allow SSH from approved IP only*

![Data Subnet NSG Rules](Networking/bedrock-m2-t3c-data-subnet-nsg-rules.png)
*Data subnet NSG — Deny all inbound, restrict outbound to app-subnet only*

![NSG Subnet Associations](Networking/bedrock-m2-t4-nsg-subnet-associations.png)
*All three subnets with respective NSGs associated*

![Spoke VNet Created](Networking/bedrock-m2-t5-spoke-vnet-created.png)
*Spoke VNet — address space 10.20.0.0/16*

![VNet Peering Connected](Networking/bedrock-m2-t6-vnet-peering-connected.png)
*Hub-to-spoke peering status Connected*

![Private DNS Zone Created](Networking/bedrock-m2-t7-private-dns-zone-created.png)
*internal.clearvault.com Private DNS Zone*

![Hub VNet Link CLI](Networking/bedrock-m2-t8a-hub-vnet-link-cli-connected.png)
*VNet link created via CLI — provisioningState Succeeded with tags confirmed*

![Hub VNet Link Portal](Networking/bedrock-m2-t8b-hub-vnet-link-portal-connected.png)
*hub-vnet-link visible in portal with auto-registration enabled*

![A Record Created](Networking/bedrock-m2-t9-a-record-created.png)
*test.internal.clearvault.com A record pointing to 10.10.1.10*

![Service Endpoint Enabled](Networking/bedrock-m2-t10-service-endpoint-enabled.png)
*Microsoft.Storage service endpoint enabled on app-subnet*

---

### Module 3 — Storage

A storage account with public blob access disabled and TLS 1.2 enforced as minimum. Network rules restrict access exclusively to app-subnet via service endpoint — the default network action is Deny, meaning any traffic not originating from the approved subnet is rejected regardless of authentication credentials.

Two containers — `app-data` and `audit-logs` — both set to private access. A SAS token scoped to Read and List permissions on Blob resources only, with a 24-hour expiry and HTTPS-only protocol, demonstrates time-limited delegated access without exposing account keys. A lifecycle policy automates data tiering: Cool at 30 days, Archive at 90 days, deletion at 365 days — aligned to standard fintech data retention requirements. Blob versioning on audit-logs ensures tamper evidence and log recoverability. Microsoft Defender for Storage provides continuous threat detection.

![Storage Account Created](Storage/bedrock-m3-t1-storage-account-created.png)
*clearvaultstg01 in UK South under clearvault-workload-rg*

![Public Access Disabled TLS12](Storage/bedrock-m3-t2-annonymous-access-disabled-tls12.png)
*Blob annonymous access disabled, minimum TLS 1.2 enforced*

![Network Rules App Subnet](Storage/bedrock-m3-t3-network-rules-app-subnet.png)
*Storage network rules — app-subnet only, default action Deny*

![Containers Created](Storage/bedrock-m3-t4-containers-created.png)
*app-data and audit-logs containers with private access*

![SAS Token Configuration](Storage/bedrock-m3-t5-sas-token-configuration.png)
*SAS token — Read and List only, HTTPS only, 24 hour expiry*

![Lifecycle Policy Configured](Storage/bedrock-m3-t6-lifecycle-policy-configured.png)
*Lifecycle policy — Cool at 30 days, Archive at 90 days, Delete at 365 days*

![Defender Storage Enabled](Storage/bedrock-m3-t7-defender-storage-enabled.png)
*Microsoft Defender for Storage enabled*

![Blob Versioning Enabled](Storage/bedrock-m3-t8-blob-versioning-enabled.png)
*Blob versioning enabled on storage account*

---

### Module 4 — Compute

An Ubuntu 22.04 management VM deployed into the management subnet with no public IP. The intended production access method is Azure Bastion — removing the public IP eliminates direct internet exposure entirely. A NIC-level NSG applies a second layer of network control on top of the subnet NSG — SSH is restricted to traffic originating from within the management subnet only. This defence in depth means traffic must pass two independent NSG gates to reach the VM.

System-assigned managed identity is enabled and granted `Storage Blob Data Reader` scoped precisely to the `audit-logs` container — not the storage account, not the resource group, the exact container. This eliminates stored credentials entirely and enforces least privilege at the identity layer. Boot diagnostics is enabled for operational observability during incident response.

![VM Basics Configuration](Compute/bedrock-m4-t1-vm-basics-configuration.png)
*clearvault-mgmt-vm01 — Ubuntu 22.04, West Europe*

![Networking No Public IP](Compute/bedrock-m4-t2-networking-no-public-ip.png)
*VM networking — mgmt-subnet selected, no public IP*

![NIC NSG Rules](Compute/bedrock-m4-t3-nic-nsg-rules.png)
*NIC-level NSG — SSH restricted to mgmt-subnet only*

![Managed Identity Enabled](Compute/bedrock-m4-t4-managed-identity-enabled.png)
*System-assigned managed identity enabled on VM*

![Storage Role Assignment](Compute/bedrock-m4-t5-storage-role-assignment-audit-logs.png)
*Storage Blob Data Reader assigned to VM managed identity — audit-logs container scope only*

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

## Notes & Deviations

**VM region — West Europe**
The subscription does not support VM creation in UK South due to quota limitations. The compute module was deployed in West Europe, which is within the approved regions defined by the Allowed Locations policy. A dedicated VNet `clearvault-hub-vnet-we` was created in West Europe to host the management VM, preserving the correct subnet architecture. This is documented as a subscription constraint, not a design compromise.

**Tag policy scope — resource groups exempt**
During Module 5 validation it was discovered that the Require a tag on resources policy does not apply to resource groups themselves — only to resources deployed within them. This is expected Azure Policy behaviour. The tag enforcement test was adjusted to attempt a storage account creation without tags, which correctly produced a policy denial. This discovery was captured as part of the validation findings.

**Private DNS Zone VNet link — portal bug**
The VNet link creation failed in the Azure portal with an undefined error. The link was successfully created via Azure Cloud Shell CLI. The CLI command included the required tags, which also incidentally triggered a policy denial on the first attempt — an unplanned but valid demonstration of the tag policy enforcement. The correctly tagged command succeeded. This is a known intermittent portal issue and does not affect the functionality of the DNS zone.

**Module 5 RBAC test — contributor visibility**
The workload contributor test account could only see `clearvault-workload-rg` in the portal — `clearvault-platform-rg` was not visible at all. This is correct RBAC behaviour: Reader is required to see a resource group in the portal, and the contributor account has no role at subscription or platform scope. The test was reframed to demonstrate that the account could not create a new resource group — an action requiring subscription-level permissions — which returned AuthorizationFailed as expected.

---

## Skills Demonstrated

- Azure governance — management groups, Policy deny effects, resource locks
- Identity & access management — RBAC scoping, custom roles, Entra ID groups, managed identity least privilege
- Network security — hub and spoke VNet design, per-subnet NSGs, VNet peering, private DNS, service endpoints, defence in depth
- Storage security — network rules, SAS token scoping, lifecycle management, Defender for Storage, blob versioning
- Governance validation — deliberate control testing with documented evidence and deviation analysis

---

## AZ-104 Domain Coverage

- Manage Identities & Governance
- Implement & Manage Storage
- Deploy & Manage Virtual Networking
- Deploy & Manage Compute Resources

---

## Related Projects

- [Project Citadel](https://github.com/blvckbillgates/project-citadel) — Azure secure environment with Sentinel SIEM detection and incident response
