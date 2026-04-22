---
title: "ADR-0004: Azure Encryption Strategy"
status: "Proposed"
date: "2026-04-19"
authors: "Infrastructure/DevOps"
tags: ["architecture", "terraform", "azure", "security", "encryption"]
supersedes: ""
superseded_by: ""
---

## Status

Proposed

## Context

The platform must enforce an Azure encryption strategy that protects data in transit, at rest, and at host level while aligning with existing key management decisions from `ADR-0003`.

Forces at play:
- Regulatory and customer requirements increasingly expect explicit controls for encryption in transit and at rest.
- Azure service capabilities are inconsistent: some resource types support CMK, versionless key references, or host encryption, while others do not.
- A uniform "CMK wherever supported" policy increases cost, rollout friction, and operational burden, but the platform accepts these trade-offs to maintain a strong encryption baseline.
- Platform teams need a default position that is secure by default, but still pragmatic for workloads with different risk profiles.
- Terraform modules and policy controls must be able to express both mandatory guardrails and justified exceptions.

Key requirements:
- Use Customer Managed Keys (CMK) as a mandatory requirement for storage, compute, and database encryption wherever the Azure service supports CMK.
- Use versionless key references where possible to simplify rotation and reduce deployment complexity.
- Encrypt all virtual machines at host level.
- Use Disk Encryption Sets with `encryption_type = "EncryptionAtRestWithPlatformAndCustomerKeys"` for double encryption when enabling disk encryption.
- **Azure Log Analytics Workspaces and Event Hubs**: Only encrypt with CMK on explicit request, as CMK encryption incurs significant base costs and should not be enabled by default.
- Treat infrastructure encryption as optional; do not require it for every workload.
- Encryption in transport is mandatory.
- Require TLS 1.2 or newer for all supported network communication.
- Activate Virtual Network encryption only on explicit request and only where the workload has a documented requirement that justifies its limitations and operational complexity.

Constraints and assumptions:
- The solution must remain compatible with Azure-native managed services and Terraform-based infrastructure.
- Existing key and secret management decisions from `ADR-0003` are authoritative for Key Vault usage.
- Customers may have workloads that cannot use CMK immediately; fallback to platform-managed encryption must remain available.
- Operational teams can manage Key Vault identities, RBAC, and key rotation policies.
- Private network encryption via VNet encryption is an exception-based control and must not be enabled by default.
- Workload teams are expected to justify exceptions where a service does not support the platform default or where CMK would create disproportionate cost or operational impact.

## Decision

Adopt an Azure encryption strategy that prioritizes transport security, customer-managed keys, host-level encryption for VMs, and double encryption for disks when enabled, while allowing VNet encryption only on explicit workload request.

Key decisions:
- Treat encryption in transit as non-negotiable. All workload traffic, including private and east-west traffic where the platform can control it, must use TLS 1.2 or newer.
- Use Customer Managed Keys (CMK) as a mandatory control for all supported PaaS and IaaS resources where Azure support exists and where the service can be integrated with the Key Vault model defined in `ADR-0003`.
- Allow platform-managed encryption only where Azure does not support CMK for the specific resource or feature path. Cost, convenience, delivery speed, or operational preference are not valid reasons to bypass CMK where support exists.
- **Azure Log Analytics Workspaces and Event Hubs** must use CMK when the service and selected SKU support it and when the workload requires those services. If a chosen service configuration cannot support CMK, that limitation must be documented explicitly as a platform constraint rather than treated as a discretionary exception.
- Prefer versionless key references in Key Vault for resources that support them, allowing key rollover without forced resource redeployment.
- Enable Encryption at Host for all Azure virtual machines and VM scale sets that support the feature, to protect VM state and temporary data at the physical host layer.
- When customer-managed disk encryption is required, use Azure Disk Encryption Sets with `encryption_type = "EncryptionAtRestWithPlatformAndCustomerKeys"` to provide platform and customer-key-backed encryption at rest.
- Treat infrastructure-level encryption controls beyond platform defaults as risk-based. They are mandatory only when required by regulation, contractual commitment, data classification, or explicit security design review.
- Activate Azure VNet encryption only on explicit customer or workload request with documented justification. It is not part of the default platform baseline because of its current limitations, uneven service compatibility, and operational complexity.

Rationale:
- Mandatory transport encryption and TLS 1.2+ address the most important threat vector for data in motion.
- CMK improves customer control, supports separation of duties for key ownership, and aligns encryption dependencies with the Key Vault strategy in `ADR-0003`.
- Treating CMK as mandatory wherever supported establishes a clearer and stronger security baseline and removes ambiguity from workload-level design decisions.
- Versionless key references reduce deployment churn and simplify operational rotation lifecycles.
- Encryption at host provides an additional protection layer for VM workloads beyond storage-level encryption.
- Double encryption in Disk Encryption Sets delivers a stronger defense-in-depth posture when customer-managed keys are used for disks.
- **Log Analytics and Event Hubs CMK encryption**: where supported, these services must follow the same CMK baseline as other platform components; additional cost does not override the encryption standard.
- A risk-based model for optional infrastructure encryption applies only to controls beyond the CMK and transport baseline; it does not weaken mandatory encryption requirements where Azure capabilities exist.
- Keeping VNet encryption as an explicit opt-in control prevents accidental reliance on a feature with known limitations while preserving it for workloads that specifically require it.

## Consequences

### Positive

- **POS-001**: Transport encryption becomes mandatory, eliminating clear-text transit and improving compliance with modern security standards.
- **POS-002**: Customer Managed Keys increase key ownership visibility and support centralized key lifecycle controls via Key Vault.
- **POS-003**: The decision removes ambiguity by making CMK a hard baseline wherever Azure support exists, preventing weaker per-workload interpretations.
- **POS-004**: Versionless key references reduce redeployment scope when keys are rotated or rolled.
- **POS-005**: Host-level encryption for VMs protects against physical host compromise and secures ephemeral guest state.
- **POS-006**: Disk Encryption Sets with double encryption strengthen at-rest protections when customer-managed keys are required.
- **POS-007**: Restricting VNet encryption to explicit requests reduces dependency on a feature with known limitations while preserving it for justified workload scenarios.

### Negative

- **NEG-001**: CMK adoption increases operational overhead for key provisioning, identity assignment, and key lifecycle management.
- **NEG-002**: A mandatory-CMK posture can constrain service selection, SKU selection, and rollout sequencing where teams might otherwise choose lower-friction platform defaults.
- **NEG-003**: Versionless keys can hide explicit key version changes from some deployment workflows, requiring disciplined key rotation tracking.
- **NEG-004**: Encryption at Host may introduce compatibility constraints for legacy VM images or unsupported guest configurations.
- **NEG-005**: Disk Encryption Sets may increase cost and complexity for workloads that do not strictly require customer key-backed disk encryption.
- **NEG-006**: Risk-based infrastructure encryption may still produce different control levels between workloads, which must be visible and explicitly accepted.
- **NEG-007**: CMK for observability and messaging services can introduce substantial recurring cost that the platform must absorb when those services support CMK.

## Alternatives Considered

### Platform-Managed Encryption Only
- **ALT-001**: **Description**: Use only Azure-managed keys and platform encryption for all resources.
- **ALT-002**: **Rejection Reason**: This fails the requirement to use Customer Managed Keys where possible and reduces customer control over key management.

### Treat CMK as Optional by Workload
- **ALT-003**: **Description**: Let individual workloads decide whether to use CMK based on cost, convenience, or local delivery priorities.
- **ALT-004**: **Rejection Reason**: This would allow teams to trade away key ownership and encryption consistency for local preference, undermining the platform security baseline.

### Require CMK Wherever Supported
- **ALT-005**: **Description**: Mandate CMK for every supported resource and treat lack of Azure support as the only valid reason to fall back to platform-managed encryption.
- **ALT-006**: **Rejection Reason**: This is the selected approach.

### Require Infrastructure Encryption for All Workloads
- **ALT-007**: **Description**: Make infrastructure encryption mandatory for every workload regardless of risk.
- **ALT-008**: **Rejection Reason**: This adds unnecessary complexity and cost for controls beyond the CMK and transport baseline and contradicts the risk-based requirement for optional infrastructure encryption.

### Use VNet Encryption as Default
- **ALT-009**: **Description**: Use VNet encryption broadly to protect traffic between VMs across subnets.
- **ALT-010**: **Rejection Reason**: VNet encryption may be used for specific workloads, but enabling it broadly as a default would add unnecessary operational complexity and compatibility risk.

### Disable Encryption at Host on VMs
- **ALT-011**: **Description**: Do not enable host-level encryption for VMs and rely only on disk encryption.
- **ALT-012**: **Rejection Reason**: Host-level encryption is required to protect VM state and temporary data at the physical host layer.

### Use Versioned Key References Everywhere
- **ALT-013**: **Description**: Use explicit key versions for all resources instead of versionless references.
- **ALT-014**: **Rejection Reason**: This increases deployment complexity and makes key rotation more invasive than necessary.

## Implementation Notes

- **IMP-001**: Extend Terraform modules to accept CMK-enabled key references and to deploy or reference Key Vault keys only for resource types that support the required encryption pattern.
- **IMP-002**: Implement support for versionless Key Vault references such as `https://<vault-name>.vault.azure.net/keys/<key-name>` when the Azure resource type supports them.
- **IMP-003**: Make CMK-capable configuration the mandatory Terraform module path where supported, and require an explicit documented platform limitation when falling back to platform-managed encryption.
- **IMP-004**: Prevent module consumers from disabling CMK for supported resources through convenience flags alone; unsupported-service cases must be represented as explicit design constraints.
- **IMP-005**: Enable `enable_host_encryption = true` or the equivalent setting on all Azure VM and VMSS modules that support the feature.
- **IMP-006**: Use Azure Disk Encryption Sets with `encryption_type = "EncryptionAtRestWithPlatformAndCustomerKeys"` when customer-managed disk encryption is required.
- **IMP-007**: Document workload categories that require additional infrastructure encryption and the approval process for enabling it.
- **IMP-008**: Enforce TLS 1.2 or newer in application and service configuration, and validate certificate chains for HTTPS endpoints.
- **IMP-009**: Provide Terraform guardrails or Azure Policy controls that deny deployment of supported resources without CMK and prevent VNet encryption from being enabled unless an explicit approved input is provided.
- **IMP-010**: Ensure all CMK usages reference the Key Vault key via managed identity roles and Azure RBAC following `ADR-0003`.
- **IMP-011**: Validate transport encryption using automated checks or scan tools to ensure no endpoints support TLS 1.1 or earlier.
- **IMP-012**: Make encryption decisions and any unsupported-service constraints visible in architecture documentation so reviewers can understand where the baseline is fully met and where Azure capability limits apply.

## References
- **REF-001**: `docs/adr/adr-0003-key-vault-key-and-secret-management.md`
- **REF-002**: Azure Disk Encryption Set documentation: https://learn.microsoft.com/azure/virtual-machines/disk-encryption-set
- **REF-003**: Azure encryption at host: https://learn.microsoft.com/azure/virtual-machines/encryption-at-host
- **REF-004**: TLS guidance: https://learn.microsoft.com/azure/security/fundamentals/tls
- **REF-005**: Azure Key Vault customer-managed key usage: https://learn.microsoft.com/azure/key-vault/keys/about-keys
- **REF-006**: Azure Storage encryption with customer-managed keys: https://learn.microsoft.com/azure/storage/common/customer-managed-keys-overview
- **REF-007**: Azure Event Hubs encryption at rest: https://learn.microsoft.com/azure/event-hubs/configure-customer-managed-key
