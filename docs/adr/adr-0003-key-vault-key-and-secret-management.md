---
title: "ADR-0003: Key Vault Key and Secret Management"
status: "Proposed"
date: "2026-04-19"
authors: "Infrastructure/DevOps"
tags: ["architecture", "terraform", "azure", "keyvault", "security"]
supersedes: ""
superseded_by: ""
---

## Status

Proposed

## Context

The platform must manage cryptographic keys and application secrets in Azure Key Vault with a secure, compliant, and operationally consistent approach across environments.

Key requirements:
- No secrets may be stored in Terraform state
- No secrets may be stored in `terraform.tfvars`
- One Key Vault per environment, with separate vaults for keys and secrets considered when justified
- Use Managed Identity first and avoid legacy access policies
- Use Azure Key Vault Premium tier
- Enable purge protection with environment-differentiated retention
- Block unrestricted Public Network Access
- Require Diagnostic Settings for audit and monitoring
- Protect encryption keys with hardware security modules
- Enforce one-year expiration for keys and secrets after creation
- Enable key rotation policy with a trigger 90 days before expiration
- Store secrets with content type metadata
- Treat secrets as ephemeral resources where possible

Constraints and assumptions:
- Environments include at least `dev` and `prod`
- `prod` requires stronger retention and security controls than `dev`
- Infrastructure-as-code must not introduce credential drift or expose secrets in persisted config files
- Operational teams can manage managed identities and Azure RBAC centrally

## Decision

Adopt a centralized Azure Key Vault key and secret management strategy that enforces premium-tier security, access by managed identity only, no legacy access policies, HSM-backed keys, and strict lifecycle controls.

Key decisions:
- Provision a dedicated Key Vault per environment. Where required by separation of duties or compliance, use separate vaults for keys and secrets.
- Choose `Premium` SKU for all Key Vaults to enable HSM-protected keys, rotation policies, and advanced logging.
- Disable legacy access policies and use Azure RBAC for all vault operations.
- Assign application and platform identities via Managed Identity and least-privilege RBAC roles.
- Disable unrestricted Public Network Access and require either private endpoint access or approved network rules. If public access is enabled, enforce restrictive `network_acls` with `default_action = "Deny"`, `bypass = "AzureServices"`, and explicit `ip_rules`. Public endpoint or VNet rule access may be allowed only with additional data-plane controls and Code Runner authorization as needed.
- Set `rbac_authorization_enabled = true` to use Azure RBAC and disable legacy access policies.
- Enable `PurgeProtection` and `SoftDelete` on all vaults. Use 7-day retention for dev/test and 90-day retention for prod.
- Configure Diagnostics Settings to stream Key Vault audit logs and metrics to a Log Analytics workspace and/or storage account.
- Create HSM-protected keys using `azurerm_key_vault_key` with `key_type = "RSA-HSM"`, `key_size = 4096`, and `key_opts = ["decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"]`.
- Enforce expiration metadata on keys and secrets at creation time: `expires_on = created_at + 365 days`.
- Configure key rotation policy with `time_before_expiry` to rotate automatically 90 days before expiry, and use `expire_after = "P365D"` with `notify_before_expiry = "P89D"`.
- Store secrets with explicit `content_type` values to improve intent, usage, and parsing.
- Model secrets as ephemeral resources when possible: generate them at deployment runtime, rotate regularly, and avoid long-lived secret values in source control and config.

Rationale:
- This minimizes exposure of sensitive material in Terraform state and configuration files.
- Azure RBAC avoids legacy policy complexity and aligns with current Azure security best practices.
- Premium tier is required for hardware-protected keys and rotation policy support.
- Network restrictions and diagnostic capture reduce the attack surface and improve incident detection.
- Expiration and rotation controls enforce a strict key lifecycle, reducing the risk of stale credentials.

## Consequences

### Positive

- **POS-001**: Secrets are never persisted in Terraform state or `terraform.tfvars`, reducing risk from source control and persisted state exposure.
- **POS-002**: Managed identity-based access with Azure RBAC provides modern least-privilege controls and avoids legacy access policy drift.
- **POS-003**: Premium Key Vault tier enables HSM-protected keys, rotation policy, and enterprise auditing capabilities.
- **POS-004**: Purge protection and soft delete prevent accidental or malicious permanent deletion.
- **POS-005**: Network restrictions and diagnostics support strengthen both confidentiality and operational monitoring.
- **POS-006**: Expiration and rotation policies enforce regular credential refresh and reduce long-lived secret usage.

### Negative

- **NEG-001**: Premium Key Vault Hardware Protected Keys incurs higher cost than Software Keys.
- **NEG-002**: RBAC and managed identity configuration adds operational complexity during onboarding and troubleshooting.
- **NEG-003**: Strict network and lifecycle policies may require additional support for legacy clients or ad-hoc access scenarios.
- **NEG-004**: Ephemeral secret handling may require application changes for dynamic secret retrieval and rotation handling.
- **NEG-005**: Automated key rotation may increase the need for integration testing to verify consumers can handle rotated keys.

## Alternatives Considered

### Use Standard Key Vault Tier
- **ALT-001**: **Description**: Use Standard Key Vault tier and rely on software-protected keys.
- **ALT-002**: **Rejection Reason**: Standard tier does not support HSM-backed keys or rotation policies, violating the hardware protection and rotation requirements.

### Use Legacy Access Policies
- **ALT-003**: **Description**: Continue using legacy Key Vault access policies for identity access.
- **ALT-004**: **Rejection Reason**: Legacy policies do not align with modern Azure RBAC practices and conflict with the explicit requirement to avoid them.

### Single Key Vault for All Environments
- **ALT-005**: **Description**: Use one Key Vault instance for dev and prod with environment tagging or soft partitioning.
- **ALT-006**: **Rejection Reason**: This creates weaker isolation and violates the requirement for one vault per environment.

### Allow Public Network Access with IP Restrictions Only
- **ALT-007**: **Description**: Permit public network access and restrict by IP address ranges.
- **ALT-008**: **Rejection Reason**: Public network access increases attack surface and does not meet the intent of blocking unrestricted access; private endpoint access is preferred.

### Secrets Without Content Type Metadata
- **ALT-009**: **Description**: Store secrets without content type metadata for simplicity.
- **ALT-010**: **Rejection Reason**: Missing content type reduces operational clarity and can cause misuse or parsing errors in consuming applications.

## Implementation Notes

- **IMP-001**: Implement a Terraform module or reusable pattern that provisions a Key Vault per environment with `sku = "premium"`, `soft_delete_enabled = true`, and `purge_protection_enabled = true`.
- **IMP-002**: Enforce environment-specific `soft_delete_retention_days`: `7` for dev/test and `90` for prod.
- **IMP-003**: Use `azurerm_key_vault` with `public_network_access_enabled = false` and configure `network_acls` or private endpoints for authorized access. If public network access is permitted, require restrictive ACLs with `default_action = "Deny"`, `bypass = "AzureServices"`, and explicit `ip_rules`.
- **IMP-004**: Configure `azurerm_monitor_diagnostic_setting` to send `AuditEvent` and metric logs to a centralized Log Analytics workspace, storage account, or event hub.
- **IMP-005**: Set `rbac_authorization_enabled = true`, disable `access_policy` blocks, and use `azurerm_role_assignment` to grant `Key Vault Crypto Service Encryption User`, `Key Vault Administrator`, `Key Vault Secrets Officer`, or least privilege roles to managed identities.
- **IMP-006**: Create HSM-backed keys via `azurerm_key_vault_key` with `key_type = "RSA-HSM"`, `key_size = 4096`, and `key_opts = ["decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"]`.
- **IMP-007**: Set `expires_on = timestampadd("day", 365, utcnow())` for keys and `expiration_date` for secrets at creation time, using deployment-time logic to calculate one-year expiry.
- **IMP-008**: Configure rotation policy using `time_before_expiry` with `automatic { time_before_expiry = "P90D" }`, `expire_after = "P365D"`, and `notify_before_expiry = "P89D"`.
- **IMP-009**: Set `content_type` for all secrets and use descriptive values such as `"application/vnd.microsoft.keyvault-secret"`, `"password"`, or `"connection-string"` based on intent.
- **IMP-010**: Use ephemeral secret provisioning patterns where possible, such as temporary secrets created by deployment pipelines or application bootstrap flows, rather than long-lived static values.
- **IMP-011**: Document Key Vault operational processes, including identity onboarding, private endpoint usage, secret lifecycle, and key rotation validation.

## References
- **REF-001**: `docs/adr/adr-0001-environment-configuration-strategy.md`
- **REF-002**: `docs/adr/adr-0002-terraform-root-module-file-structure.md`
- **REF-003**: Azure Key Vault security best practices: https://learn.microsoft.com/azure/key-vault/general/security-controls
- **REF-004**: Azure Key Vault RBAC and managed identities: https://learn.microsoft.com/azure/key-vault/general/rbac-guide
- **REF-005**: Azure Key Vault rotation policy: https://learn.microsoft.com/azure/key-vault/keys/rotation-policy
- **REF-006**: Guardrail policy initiative: Enforce recommended guardrails for Azure Key Vault (`Enforce-Guardrails-KeyVault_20260203`)
- **REF-007**: Guardrail policy initiative: Enforce additional recommended guardrails for Key Vault (`Enforce-Guardrails-KeyVault-Sup`)
