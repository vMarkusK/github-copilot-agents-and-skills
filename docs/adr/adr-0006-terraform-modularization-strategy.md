---
title: "ADR-0006: Terraform Modularization Strategy"
status: "Proposed"
date: "2026-04-20"
authors: "Architecture Team"
tags: ["terraform", "architecture", "modules", "infrastructure", "best-practices"]
supersedes: ""
superseded_by: ""
---

## Status

Proposed

## Context

Terraform projects can become unnecessarily complex through excessive modularization. Teams need clear guidelines to decide:

- When local modules are beneficial and when they are not
- How to handle external modules and Azure Verified Modules (AVM)
- How to optimally structure simple resources

Common problems with insufficient guidelines:

- **MOD-001**: Modules as wrappers for individual resources to "enforce" standards
- **MOD-002**: Excessive nesting through unnecessary abstraction layers
- **MOD-003**: Uncontrolled usage of external modules and AVM
- **MOD-004**: Missing documentation on modularization decisions
- **MOD-005**: Reduced maintainability due to too many module variations

## Decision

The following Terraform modularization strategy is introduced:

### Local Modules

**Usage**: Local modules are used EXCLUSIVELY when a **specific combination of multiple resources** is needed multiple times in the project.

**Criteria**: A resource combination qualifies for a module if:
- It combines at least two different resource types
- It is used at least two times in the same context
- It represents a consistent business logic pattern

**Example - Justified Modularization**:
- Complete VM package: Virtual Machine + NIC + NSG + Cloud-Init configuration
- Subnet with associated NSG rules and Route Table (when used multiple times)

**Example - Anti-Pattern**:
- Module for a single VM (Use: `for_each` loops instead)
- Module for a single Subnet (Use: `for_each` loops instead)
- Module as a wrapper for individual storage resources

### External Modules and AVM

**Policy**: External modules and Azure Verified Modules (AVM) are only integrated on **explicit request**.

**Rationale**:
- **DEC-001**: Local control and understanding take priority
- **DEC-002**: External dependencies increase maintenance effort and complexity
- **DEC-003**: AVM can be introduced later as needed without refactoring existing code

### Simple Resources

**Policy**: Individual or similar resources are managed with `for_each` loops instead of modules.

**Examples**:
- Multiple subnets in the same VNet → use `for_each` loop
- Multiple Storage Accounts with similar configuration → use `for_each` loop
- Multiple Network Security Groups → use `for_each` loop

## Consequences

### Positive

- **POS-001**: **Reduced unnecessary complexity** - Codebase remains understandable and maintainable
- **POS-002**: **Faster development** - Less time spent on modularization decisions
- **POS-003**: **Better debugging** - Direct resource access without module abstraction
- **POS-004**: **Easier code navigation** - Fewer files and directory levels to search through
- **POS-005**: **More flexible adjustments** - Simpler changes to resource configurations without module parameters
- **POS-006**: **Lower learning curve** - New team members understand the structure faster

### Negative

- **NEG-001**: **Potential code duplication** - Frequently used patterns may repeat
- **NEG-002**: **Manual standardization** - Standards must be enforced through code reviews instead of modules
- **NEG-003**: **Later modularization** - If a `for_each` loop later proves to be a module candidate, refactoring is required
- **NEG-004**: **Increased code review effort** - More duplicated code requires more attention in reviews
- **NEG-005**: **Missing version control** - External modules offer versioning; local patterns do not

## Alternatives Considered

### Alternative 1: Maximum Modularization

- **ALT-001**: **Description**: All resource combinations are extracted into modules, even if used only once.
- **ALT-001**: **Rejection Reason**: Leads to unnecessary complexity, complicates debugging, and requires extensive maintenance. Team feedback indicates frustration with too many abstraction layers.

### Alternative 2: Use Only External AVM Modules

- **ALT-002**: **Description**: Standardize on Azure Verified Modules for all resource combinations.
- **ALT-002**: **Rejection Reason**: External dependencies increase complexity uncontrollably. AVM updates can cause breaking changes. No local control over implementation details. Inflexible for project-specific requirements.

### Alternative 3: No Modules Except Large Components

- **ALT-003**: **Description**: Use modules only for large-scale, complex infrastructure components (e.g., complete hub-spoke topology).
- **ALT-003**: **Rejection Reason**: Too rigid. Common patterns (VM + NIC + NSG) should be reusable to support the DRY principle.

### Alternative 4: Policy-as-Code Instead of Modules

- **ALT-004**: **Description**: Enforce standards through Terraform policies and linting instead of using modules.
- **ALT-004**: **Rejection Reason**: Policies and linting are complementary, not alternatives. Modules for proven patterns remain valuable for code reuse.

## Implementation Notes

### Modularization Checklist for New Patterns

- **IMP-001**: **Multiple-use criterion**: Is the resource combination needed at least 2 times in the project? → Consider a module
- **IMP-002**: **Cohesion test**: Do the resources form a logical, coherent business pattern?
- **IMP-003**: **Maintainability test**: Is the module easier to understand than the `for_each` loop?
- **IMP-004**: **Dependency test**: Are external modules or AVM needed? Only on explicit request

### Code Review Focus

- **IMP-005**: **Pattern recognition** - Point out repeating resource combinations in code reviews
- **IMP-006**: **Module justification** - Question modularization decisions against this ADR
- **IMP-007**: **for_each application** - Ensure simple resources are structured with `for_each`

### Documentation

- **IMP-008**: **Module documentation** - Every module must document usage examples and justification
- **IMP-009**: **ADR reference** - Reference this ADR in module documentation
- **IMP-010**: **Pattern catalog** - Collect and share proven patterns and examples

### Transition and Implementation

- **IMP-011**: **Existing modules**: Review existing modules against this policy and refactor if needed
- **IMP-012**: **Rollout strategy**: New projects follow this strategy immediately; adapt existing projects progressively
- **IMP-013**: **Training**: Team training on policies and decision criteria

## References

- **REF-001**: [ADR-0002 Terraform Root Module File Structure](./adr-0002-terraform-root-module-file-structure.md) - Complementary structure guidelines
- **REF-002**: [ADR-0005 Terraform Azure Provider Selection](./adr-0005-terraform-azure-provider-selection.md) - Provider strategy
- **REF-003**: [Terraform Modules Documentation](https://www.terraform.io/language/modules) - Official Terraform module documentation
- **REF-004**: [Azure Verified Modules (AVM)](https://github.com/Azure/terraform-azurerm-avm-template) - Microsoft's AVM initiative
- **REF-005**: [Terraform Best Practices - DRY Principle](https://www.terraform.io/language/modules#when-to-write-a-module) - Best-practice guidance for modules
- **REF-006**: [Terraform for_each Documentation](https://www.terraform.io/language/meta-arguments/for_each) - Documentation on `for_each` alternative
