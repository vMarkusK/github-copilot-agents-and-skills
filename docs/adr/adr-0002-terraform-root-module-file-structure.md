---
title: "ADR-0002: Terraform Root-Module File Structure"
status: "Proposed"
date: "2026-04-19"
authors: "Infrastructure/DevOps"
tags: ["architecture", "terraform", "file-structure"]
supersedes: ""
superseded_by: ""
---

## Status

Proposed

## Context

The repository requires a standardized file structure for Terraform root modules to ensure consistency, maintainability, and simplicity across infrastructure deployments. This builds upon the environment configuration strategy established in ADR-0001, which separates stage-specific values into external files.

Key requirements:
- Focus on simplicity to reduce cognitive load for developers
- Organize files logically by purpose
- Support the environment-based configuration from ADR-0001
- Enable clear separation of concerns between configuration, resources, and outputs
- Maintain compatibility with standard Terraform workflows

## Decision

Adopt a simple, purpose-driven file structure for Terraform root modules with dedicated files for core components and resource organization by type.

Key decisions:
- Use `main.tf` for Terraform version, provider versions, backend block, and provider configuration
- Use `locals.tf` for all local values
- Use `variables.tf` for all input variables
- Use `outputs.tf` for all output values
- Organize application resources in separate `.tf` files structured by resource type (e.g., `storage.tf`, `database.tf`, `app-service.tf`)
- Maintain environment-specific files in `environments/` directory as per ADR-0001
- Include standard repository files (`.gitignore`, `README.md`)

Rationale:
- This structure promotes simplicity by grouping related concerns together
- It follows Terraform best practices for file organization
- Resource files by type improve readability and maintainability for larger modules
- It aligns with the external environment configuration approach from ADR-0001
- The structure scales well while remaining intuitive

## Consequences

### Positive

- **POS-001**: Simple, logical organization reduces time spent navigating and understanding the codebase
- **POS-002**: Purpose-driven files make it easy to locate specific components (providers, variables, resources)
- **POS-003**: Resource files grouped by type improve maintainability and reduce merge conflicts
- **POS-004**: Maintains compatibility with ADR-0001's environment configuration strategy
- **POS-005**: Follows Terraform community conventions for better collaboration

### Negative

- **NEG-001**: May require additional files for very large modules, potentially increasing repository complexity
- **NEG-002**: Developers must adhere to the naming conventions for resource files
- **NEG-003**: Less flexibility for alternative organizational patterns if specific use cases arise

## Alternatives Considered

### Single-file approach (all in main.tf)
- **ALT-001**: **Description**: Place all Terraform code in a single `main.tf` file
- **ALT-002**: **Rejection Reason**: Violates simplicity goal and makes the file unwieldy for larger modules

### Resource files by environment
- **ALT-003**: **Description**: Organize resource files by environment (e.g., `dev.tf`, `prod.tf`) instead of by type
- **ALT-004**: **Rejection Reason**: Would duplicate resource definitions and conflict with ADR-0001's external configuration approach

### Complex nested directory structure
- **ALT-005**: **Description**: Use subdirectories for different resource types and components
- **ALT-006**: **Rejection Reason**: Increases complexity and navigation overhead, contradicting the simplicity focus

## Implementation Notes

- **IMP-001**: Create `main.tf` containing terraform block, required_providers, backend block, and provider configurations
- **IMP-002**: Create `locals.tf` for all local value definitions used across the module
- **IMP-003**: Create `variables.tf` with all input variable declarations and descriptions
- **IMP-004**: Create `outputs.tf` with all output value definitions and descriptions
- **IMP-005**: Create resource files named by type (e.g., `storage.tf`, `database.tf`) containing related resources and data sources
- **IMP-006**: Ensure `environments/` directory contains `.tfvars` and `.tfbackend` files as per ADR-0001
- **IMP-007**: Update `.gitignore` to exclude Terraform state and lock files with the following content:

```
# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log
crash.*.log

# Exclude all .tfvars files, which are likely to contain sensitive data, such as
# password, private keys, and other secrets. These should not be part of version 
# control as they are data points which are potentially sensitive and subject 
# to change depending on the environment.
*.tfvars
*.tfvars.json

# Ignore override files as they are usually used to override resources locally and so
# are not checked in
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Include override files you do wish to add to version control using negated pattern
# !example_override.tf

# Include tfplan files to ignore the plan output of command: terraform plan -out=tfplan
# example: *tfplan*

# Ignore CLI configuration files
.terraformrc
terraform.rc

# Ignore hcl lock files
.terraform.lock.hcl
```

- **IMP-008**: Document the file structure and purpose in `README.md`

## References

- **REF-001**: ADR-0001: Environment Configuration Strategy for Terraform
- **REF-002**: Terraform documentation on module structure best practices
- **REF-003**: HashiCorp Terraform style guide recommendations