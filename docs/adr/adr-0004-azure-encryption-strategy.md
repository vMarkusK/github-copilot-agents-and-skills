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

Key requirements:
- Use Customer Managed Keys (CMK) where possible for storage, compute, and database encryption.
- Use versionless key references where possible to simplify rotation and reduce deployment complexity.
- Encrypt all virtual machines at host level.
- Use Disk Encryption Sets with `encryption_type = "EncryptionAtRestWithPlatformAndCustomerKeys"` for double encryption when enabling disk encryption.
- Treat infrastructure encryption as optional; do not require it for every workload.
- Encryption in transport is mandatory.
- Require TLS 1.2 or newer for all supported network communication.
- Avoid Virtual Network encryption due to its limitations and operational complexity.

Constraints and assumptions:
- The solution must remain compatible with Azure-native managed services and Terraform-based infrastructure.
- Existing key and secret management decisions from `ADR-0003` are authoritative for Key Vault usage.
- Customers may have workloads that cannot use CMK immediately; fallback to platform-managed encryption must remain available.
- Operational teams can manage Key Vault identities, RBAC, and key rotation policies.
- Private network encryption via VNet encryption is not a preferred design pattern for this platform.

## Decision

Adopt an Azure encryption strategy that prioritizes transport security, customer-managed keys, host-level encryption for VMs, and double encryption for disks when enabled, while explicitly avoiding VNet encryption as a general pattern.

Key decisions:
- Use Customer Managed Keys (CMK) for all supported PaaS and IaaS resources where CMK support exists and does not impose undue operational burden.
- Prefer versionless key references in Key Vault for platform resources that support them, allowing automatic key rollover without resource redeployment.
- Enable Encryption at Host for all Azure virtual machines to protect VM state and temporary data at the physical host layer.
- For disk encryption, use Azure Disk Encryption Sets with `encryption_type = "EncryptionAtRestWithPlatformAndCustomerKeys"` to achieve both platform and customer key encryption when customer keys are applied.
- Make infrastructure encryption optional: only require additional encryption controls for workloads with a clear regulatory or risk-based need.
- Enforce TLS 1.2 or later for all network communication, including service endpoints, APIs, and application traffic.
- Treat encryption in transport as mandatory for all workloads, with no exception for internal or private traffic.
- Avoid using Azure VNet encryption as a platform-wide requirement due to current limitations and compatibility issues.

Rationale:
- Mandatory transport encryption and TLS 1.2+ address the most important threat vector for data in motion.
- CMK improves customer control and supports separation of duties for key ownership, aligning with the Key Vault strategy in `ADR-0003`.
- Versionless key references reduce deployment churn and simplify rotation lifecycles.
- Encryption at host provides an additional layer of protection for VM workloads beyond disk and platform encryption.
- Double encryption in Disk Encryption Sets delivers a stronger defense-in-depth posture when customer-managed keys are used for disk storage.
- Keeping infrastructure encryption optional avoids unnecessary complexity for workloads that do not require it and maintains flexibility.
- Avoiding VNet encryption prevents reliance on a feature with known limitations and provides a more predictable operational model.

## Consequences

### Positive

- **POS-001**: Transport encryption becomes mandatory, eliminating clear-text transit and improving compliance with modern security standards.
- **POS-002**: Customer Managed Keys increase key ownership visibility and support centralized key lifecycle controls via Key Vault.
- **POS-003**: Versionless key references reduce redeployment scope when keys are rotated or rolled.
- **POS-004**: Host-level encryption for VMs protects against physical host compromise and secures ephemeral guest state.
- **POS-005**: Disk Encryption Sets with double encryption strengthen at-rest protections when customer-managed keys are required.
- **POS-006**: Avoiding VNet encryption reduces dependency on a problematic feature and simplifies network architecture.

### Negative

- **NEG-001**: CMK adoption increases operational overhead for key provisioning, identity assignment, and key lifecycle management.
- **NEG-002**: Versionless keys can hide explicit key version changes from some deployment workflows, requiring disciplined key rotation tracking.
- **NEG-003**: Encryption at Host may introduce compatibility constraints for legacy VM images or unsupported guest configurations.
- **NEG-004**: Disk Encryption Sets may increase cost and complexity for workloads that do not strictly require customer key-backed disk encryption.
- **NEG-005**: Optional infrastructure encryption may lead to inconsistent encryption posture across workloads if not governed by clear policy.

## Alternatives Considered

### Platform-Managed Encryption Only
- **ALT-001**: **Description**: Use only Azure-managed keys and platform encryption for all resources.
- **ALT-002**: **Rejection Reason**: This fails the requirement to use Customer Managed Keys where possible and reduces customer control over key management.

### Require Infrastructure Encryption for All Workloads
- **ALT-003**: **Description**: Make infrastructure encryption mandatory for every workload regardless of risk.
- **ALT-004**: **Rejection Reason**: This adds unnecessary complexity and cost for workloads that do not require it and contradicts the optional infrastructure encryption requirement.

### Use VNet Encryption as Default
- **ALT-005**: **Description**: Use VNet encryption broadly to protect traffic between VMs across subnets.
- **ALT-006**: **Rejection Reason**: VNet encryption is avoided due to its limitations, compatibility issues, and lack of support as a general platform pattern.

### Disable Encryption at Host on VMs
- **ALT-007**: **Description**: Do not enable host-level encryption for VMs and rely only on disk encryption.
- **ALT-008**: **Rejection Reason**: Host-level encryption is required to protect VM state and temporary data at the physical host layer.

### Use Versioned Key References Everywhere
- **ALT-009**: **Description**: Use explicit key versions for all resources instead of versionless references.
- **ALT-010**: **Rejection Reason**: This increases deployment complexity and makes key rotation more invasive than necessary.

## Implementation Notes

- **IMP-001**: Extend Terraform modules to accept CMK-enabled key references and to conditionally deploy Key Vault keys for supported resources.
- **IMP-002**: Implement support for versionless Key Vault references such as `https://<vault-name>.vault.azure.net/keys/<key-name>` when resource types allow it.
- **IMP-003**: Enable `enable_host_encryption = true` or the equivalent setting on all Azure VM and VMSS modules.
- **IMP-004**: Use Azure Disk Encryption Sets with `encryption_type = "EncryptionAtRestWithPlatformAndCustomerKeys"` when customer-managed disk encryption is required.
- **IMP-005**: Document workload categories that require optional infrastructure encryption and the approval process for enabling it.
- **IMP-006**: Enforce TLS 1.2 or newer in application and service configuration, and validate certificate chains for HTTPS endpoints.
- **IMP-007**: Provide Terraform guardrails or policies that prevent unsupported VNet encryption patterns from being deployed as the default option.
- **IMP-008**: Ensure all CMK usages reference the Key Vault key via managed identity roles and Azure RBAC following `ADR-0003`.
- **IMP-009**: Validate transport encryption using automated checks or scan tools to ensure no endpoints support TLS 1.1 or earlier.
- **IMP-010**: Make infrastructure encryption decisions visible in architecture documentation so reviewers understand when and why it is optional.

## References
- **REF-001**: `docs/adr/adr-0003-key-vault-key-and-secret-management.md`
- **REF-002**: Azure Disk Encryption Set documentation: https://learn.microsoft.com/azure/virtual-machines/disk-encryption-set
- **REF-003**: Azure encryption at host: https://learn.microsoft.com/azure/virtual-machines/encryption-at-host
- **REF-004**: TLS guidance: https://learn.microsoft.com/azure/security/fundamentals/tls
- **REF-005**: Azure Key Vault customer-managed key usage: https://learn.microsoft.com/azure/key-vault/keys/about-keys
