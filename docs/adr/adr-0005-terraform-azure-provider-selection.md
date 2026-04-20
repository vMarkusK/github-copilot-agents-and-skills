---
title: "ADR-0005: Terraform Azure Provider Selection"
status: "Proposed"
date: "2026-04-20"
authors: ["Platform Engineering Team"]
tags: ["architecture", "terraform", "azure", "provider"]
supersedes: ""
superseded_by: ""
---

## Status

Proposed

## Context

When provisioning Azure infrastructure using Terraform, engineers have two primary providers available:

- **hashicorp/azurerm**: The official, community-maintained provider with broad resource coverage and long-term stability
- **Azure/azapi**: The first-party provider by Microsoft, offering early access to preview features and newer Azure resources that may not yet be available in hashicorp/azurerm

**Forces at play:**

- **Stability vs. Innovation**: hashicorp/azurerm prioritizes stability and long-term maintenance, while Azure/azapi provides access to cutting-edge Azure features
- **Community Maturity**: hashicorp/azurerm has a larger community, more Stack Overflow answers, and extensive documentation. Azure/azapi is newer and less widely adopted
- **Resource Coverage**: hashicorp/azurerm covers ~95% of Azure resources with stable, documented support. Azure/azapi provides access to preview resources and experimental features
- **Maintenance Burden**: Managing multiple providers in the same codebase increases complexity and requires clear guidelines for when to use each
- **Preview Feature Access**: Some Azure features launch in preview and may only be available via Azure/azapi before being added to hashicorp/azurerm

**Organizational Requirements:**

- Need a clear, consistent approach to provider selection across all Terraform projects
- Must balance access to new features with operational stability
- Should minimize context switching and maintain predictable infrastructure-as-code patterns
- Prefer a primary provider to reduce cognitive load for teams

## Decision

**Use `hashicorp/azurerm` as the primary Terraform provider for all Azure infrastructure provisioning.**

Secondary use of `Azure/azapi` is permitted only in two specific scenarios:

1. **Preview Features**: When a new Azure feature is only available as a preview and not yet supported by hashicorp/azurerm
2. **Coverage Gaps**: When hashicorp/azurerm does not provide a resource or datasource needed for your infrastructure

**Rationale:**

- hashicorp/azurerm provides stable, well-documented, and community-vetted resource definitions suitable for production infrastructure
- The provider has extensive community support, making troubleshooting and knowledge-sharing easier
- Using one primary provider reduces complexity and improves maintainability
- Azure/azapi can be selectively used as a temporary bridge for preview features, with a clear migration path to hashicorp/azurerm once features reach GA
- This approach follows the principle of "stable by default, innovative when necessary"

## Consequences

### Positive

- **POS-001**: **Stability and Predictability**: hashicorp/azurerm is battle-tested in production environments across thousands of organizations, reducing the risk of unexpected behavior changes
- **POS-002**: **Community Support**: Larger community base means better documentation, more Stack Overflow answers, GitHub issue discussions, and faster resolution of common problems
- **POS-003**: **Reduced Maintenance Burden**: Standardizing on one primary provider simplifies onboarding, reduces training overhead, and makes infrastructure code more consistent across teams
- **POS-004**: **Clear Guidelines**: Defining when to use Azure/azapi (preview features, coverage gaps) eliminates ambiguity and reduces decision-making friction during development
- **POS-005**: **Long-term Cost Efficiency**: Avoiding premature adoption of unstable preview APIs reduces technical debt and refactoring costs when features change or reach GA
- **POS-006**: **Improved Code Readability**: hashicorp/azurerm provides declarative, type-specific resource blocks (e.g., `azurerm_resource_group`, `azurerm_virtual_network`) that are intuitive and self-documenting. Azure/azapi uses generic `azapi_resource` blocks with JSON body payloads, making code less readable and requiring more effort to understand resource configuration

### Negative

- **NEG-001**: **Preview Feature Delays**: Teams will experience a delay in using new Azure features compared to early adopters who use Azure/azapi directly. Some features may take months to reach hashicorp/azurerm
- **NEG-002**: **Multi-Provider Complexity**: When Azure/azapi is necessary for preview features, maintaining two providers in the same codebase adds complexity and requires context switching
- **NEG-003**: **Preview Feature Migration**: Infrastructure using Azure/azapi for preview features must eventually be migrated to hashicorp/azurerm resources once available, creating refactoring work
- **NEG-004**: **Feature Gaps**: Some Azure resources available in Azure/azapi may never be ported to hashicorp/azurerm if they have low adoption, leaving those resources only available via Azure/azapi

## Alternatives Considered

### Alternative 1: Azure/azapi as Primary Provider

**ALT-001**: **Description**: Use Azure/azapi as the primary provider for all infrastructure, with hashicorp/azurerm as a fallback for only well-established, stable resources

**ALT-001**: **Rejection Reason**: While Azure/azapi offers early access to new features, it lacks the maturity, community support, and documentation stability needed for production infrastructure. The provider is newer and less battle-tested. This approach would increase operational risk and reduce team productivity due to fewer community resources and documentation gaps.

### Alternative 2: Dual-Provider Strategy (No Clear Hierarchy)

**ALT-002**: **Description**: Allow teams to choose between hashicorp/azurerm and Azure/azapi on a per-resource basis without a primary provider guideline

**ALT-002**: **Rejection Reason**: This creates decision-making paralysis and inconsistency across projects. Different teams would standardize on different providers, making knowledge-sharing difficult and increasing onboarding complexity. Code reviews would require evaluating provider choices rather than infrastructure logic.

### Alternative 3: Do Nothing (Status Quo)

**ALT-003**: **Description**: Let individual teams or projects make provider selection decisions independently without organizational guidance

**ALT-003**: **Rejection Reason**: Without clear organizational standards, teams would fragment their approaches, reducing code consistency, increasing support overhead, and making knowledge transfer difficult. New team members would face unclear expectations about which provider to use for new infrastructure.

## Implementation Notes

### Default Provider Configuration

When configuring `hashicorp/azurerm`, use this default configuration as a baseline:

```hcl
provider "azurerm" {
  features {}
  
  subscription_id                 = var.subscription_id
  resource_provider_registrations = "core"
  storage_use_azuread             = true
}
```

**Configuration Rationale:**

- **`features {}`**: Enables default feature flags for the provider; add specific overrides only when necessary
- **`subscription_id = var.subscription_id`**: Allows flexible subscription targeting across environments via variables
- **`resource_provider_registrations = "core"`**: Registers only core resource providers on apply, reducing unnecessary provider registrations and improving deployment speed
- **`storage_use_azuread = true`**: Uses Microsoft Entra ID (formerly Azure AD) for storage authentication instead of shared keys, improving security posture

### Using Azure/azapi for Preview Features

When a preview feature requires Azure/azapi, follow these steps:

1. **Document the requirement**: Add a comment in the Terraform code explaining why Azure/azapi is needed and which preview feature(s) it enables
2. **Track GA timeline**: Create a task to migrate the resource to hashicorp/azurerm once the feature reaches general availability
3. **Minimize scope**: Use Azure/azapi only for the specific preview resources needed; keep everything else on hashicorp/azurerm
4. **Version constraints**: Pin Azure/azapi version in required_providers to prevent unexpected changes during preview phases

### Example: Azure/azapi for Preview Feature

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    azapi = {
      source  = "Azure/azapi"
      version = "~> 1.12"
    }
  }
}

# Primary provider: hashicorp/azurerm
provider "azurerm" {
  features {}
  subscription_id                 = var.subscription_id
  resource_provider_registrations = "core"
  storage_use_azuread             = true
}

provider "azapi" {
  subscription_id = var.subscription_id
}

# Standard resource with hashicorp/azurerm
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

# Preview resource with Azure/azapi
# TODO: Migrate to azurerm_<resource_type> once feature reaches GA (target: Q3 2026)
resource "azapi_resource" "preview_feature" {
  type      = "Microsoft.SomeService/resources@2024-01-15-preview"
  name      = var.resource_name
  parent_id = azurerm_resource_group.main.id

  body = jsonencode({
    properties = {
      previewProperty = "value"
    }
  })
}
```

### Migration Path for Preview Features

When a preview feature reaches general availability in hashicorp/azurerm:

1. Update hashicorp/azurerm provider version to include the new resource
2. Convert azapi_resource blocks to azurerm_<resource_type> blocks
3. Test thoroughly in dev/test environments before promoting to production
4. Remove Azure/azapi configuration once all resources have been migrated
5. Close the migration task created in step 2 above

## References

- **REF-001**: [ADR-0002: Terraform Root Module File Structure](./adr-0002-terraform-root-module-file-structure.md) — Related decision on organizing Terraform projects
- **REF-002**: [ADR-0004: Azure Encryption Strategy](./adr-0004-azure-encryption-strategy.md) — Security baseline that complements provider selection
- **REF-003**: [Terraform Azure Provider Documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) — Official hashicorp/azurerm provider docs
- **REF-004**: [Azure API Terraform Provider Documentation](https://registry.terraform.io/providers/Azure/azapi/latest/docs) — Official Azure/azapi provider docs
- **REF-005**: [Azure Resource Manager API Versions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/api-versions-list) — Track preview vs GA API versions for Azure services
- **REF-006**: [Cloud Adoption Framework: Naming Conventions](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming) — Ensures Terraform resources follow organizational naming standards
