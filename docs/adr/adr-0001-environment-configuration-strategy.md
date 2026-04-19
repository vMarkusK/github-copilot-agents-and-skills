---
title: "ADR-0001: Environment Configuration Strategy for Terraform"
status: "Proposed"
date: "2026-04-19"
authors: "Infrastructure/DevOps"
tags: ["architecture", "terraform", "environment-config"]
supersedes: ""
superseded_by: ""
---

## Status

Proposed

## Context

The repository requires a standardized Terraform environment configuration pattern for multiple deployment stages. The implementation must separate stage-specific values from module and resource definitions, keep backend configuration external to the main code, and avoid encoding stage-driven provisioning decisions directly in the resource graph.

Constraints and requirements:
- Stage-specific configuration must be stored in `environments/`.
- Two deployment stages are required: `dev` and `prod`.
- Each stage has a dedicated `.tfvars` file for variable values.
- Each stage has a dedicated `.tfbackend` file for backend configuration.
- The Terraform codebase uses a `backend "azurerm" {}` block with no inline `config` settings.
- Specific service or SKU values must not be hardcoded in code; they must come from variables.
- Stage-based resource enablement must be controlled by explicit variables such as `enable_natgateway` rather than stage conditionals in code.
- The current environment must also be represented by an `env` variable populated per stage.

## Decision

Adopt a stage-focused Terraform environment configuration strategy using external stage files in `environments/` and a minimal backend declaration in code.

Key decisions:
- Create `environments/dev.tfvars` and `environments/prod.tfvars` for stage-specific variable values.
- Create `environments/dev.tfbackend` and `environments/prod.tfbackend` for backend configuration.
- Keep `backend "azurerm" {}` in Terraform code without inline `config` values.
- Declare all stage-specific parameters as variables in the Terraform module, including `env` and resource toggles such as `enable_natgateway`.
- Avoid using stage values to decide whether a resource is created; use explicit boolean variables instead.

Rationale:
- This keeps Terraform code reusable and stage-agnostic.
- It supports clear separation between configuration and infrastructure logic.
- It enables safe stage switching by selecting the appropriate `.tfvars` and `.tfbackend` files during `terraform init` and `terraform apply`.
- It avoids hidden stage-driven behavior in code and reduces accidental drift between environments.

## Consequences

### Positive

- **POS-001**: Environment-specific values are isolated from Terraform modules, improving maintainability and reuse.
- **POS-002**: Backend configuration can vary per stage without modifying Terraform code, enabling distinct state management for `dev` and `prod`.
- **POS-003**: Explicit variables such as `env` and `enable_natgateway` make stage controls predictable and auditable.
- **POS-004**: The codebase remains compatible with standard Terraform workflows and best practices for environment configuration.

### Negative

- **NEG-001**: Developers must remember to pass the correct `.tfvars` and `.tfbackend` files for each stage during commands.
- **NEG-002**: Additional files are required for each stage, increasing repository surface area and file management overhead.
- **NEG-003**: Any stage-specific differences must be expressed through variables, which can require more upfront design of variable APIs.

## Alternatives Considered

### Inline Stage Logic in Terraform Code
- **ALT-001**: **Description**: Use conditional logic in the module code to switch behavior based on a stage variable like `env`.
- **ALT-002**: **Rejection Reason**: This would mix environment-specific behavior into the infrastructure definition, reducing reuse and increasing the risk of unintended stage-specific side effects.

### Single `.tfvars` plus per-stage backend options in CLI only
- **ALT-003**: **Description**: Keep one shared `.tfvars` file and use CLI-supplied options or environment variables for stage-specific values.
- **ALT-004**: **Rejection Reason**: This makes stage configuration less discoverable in source control and harder for collaborators to use consistently.

### Backend config inline in Terraform code
- **ALT-005**: **Description**: Add backend configuration directly in the `terraform { backend "azurerm" { ... } }` block with stage-aware values.
- **ALT-006**: **Rejection Reason**: Inline backend config prevents clean stage isolation and violates the requirement to keep backend config external to the code.

## Implementation Notes

- **IMP-001**: Create `environments/dev.tfvars` and `environments/prod.tfvars` with values for `env`, named resource settings, SKUs, and boolean feature toggles.
- **IMP-002**: Create `environments/dev.tfbackend` and `environments/prod.tfbackend` containing `resource_group_name`, `storage_account_name`, `container_name`, and `key` for Azure backend state.
- **IMP-003**: Keep the Terraform module code generic, with variables such as `env`, `enable_natgateway`, `location`, and `sku_name` instead of stage-specific constants.
- **IMP-004**: Document the stage initialization workflow in repository README or contributing docs: `terraform init -backend-config=environments/dev.tfbackend`, `terraform apply -var-file=environments/dev.tfvars`.
- **IMP-005**: Review stage variables periodically to ensure `dev` and `prod` remain aligned and only differ where intentional.

## References

- **REF-001**: `docs/adr/` directory for ADR standards.
- **REF-002**: Terraform recommended pattern: backend config externalized from code.
- **REF-003**: Azure Terraform best practice: stage-specific state backends per environment.
