# Terraform Style Guide

The flexibility of Terraform's configuration language gives you many options to choose from as you write your code, structure your directories, and test your configuration. While some design decisions depend on your organization's needs or preferences, there are common patterns that we recommend you adopt. Adopting and adhering to a style guide keeps your Terraform code legible, scalable, and maintainable.

This style guide covers code style recommendations, formatting and resource organization, plus operational and workflow recommendations such as meta-argument usage, versioning, and sensitive data management.

## Code style

- Run `terraform fmt` and `terraform validate` before committing code to version control.
- Use a linter such as [TFLint](https://github.com/terraform-linters/tflint) to enforce your organization's coding best practices.
- Use `#` for single-line and multi-line comments.
- Use nouns for resource names and do not include the resource type in the name.
- Use underscores to separate multiple words in names.
- Wrap the resource type and name in double quotes in resource definitions.
- Let your code build on itself: define dependent resources after the resources they reference.
- Include a type and description for every variable.
- Include a description for every output.
- Avoid overuse of variables and local values.
- Always include a default provider configuration.
- Use `count` and `for_each` sparingly.

## Code formatting

The Terraform parser allows flexibility in layout, but these idiomatic conventions improve consistency across files and modules.

- Indent two spaces for each nesting level.
- When multiple arguments with single-line values appear on consecutive lines at the same nesting level, align their equals signs.

```hcl
ami           = "abc123"
instance_type = "t2.micro"
```

- When both arguments and blocks appear together inside a block body, place all arguments together at the top and nested blocks below them.
- Use one blank line to separate arguments from nested blocks.
- Use empty lines to separate logical groups of arguments within a block.
- For blocks that contain both arguments and meta-arguments, list meta-arguments first and separate them from other arguments with one blank line.
- Place meta-argument blocks last and separate them from other blocks with one blank line.

```hcl
resource "aws_instance" "example" {
  count = 2

  ami           = "abc123"
  instance_type = "t2.micro"

  network_interface {
    # ...
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

- Top-level blocks should always be separated from one another by one blank line.
- Nested blocks should also be separated by blank lines, except when grouping together related blocks of the same type (for example, multiple `provisioner` blocks in a resource).
- Avoid grouping multiple blocks of the same type with other blocks of a different type unless the block types are defined by semantics to form a family.

The `terraform fmt` command formats Terraform code to a subset of these recommendations. By default, `terraform fmt` only modifies code in the current directory, but you can use `-recursive` to format all subdirectories.

We recommend running `terraform fmt` before each commit. Use mechanisms such as [Git pre-commit hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) to automatically run the command on commit.

## Code validation

The `terraform validate` command checks that your configuration is syntactically valid and internally consistent. It verifies the correct type of arguments, but it does not validate provider-specific values or evaluate existing state.

- `terraform validate` is safe to run automatically and frequently.
- Use it to catch syntax and semantic issues early.

For more information, refer to the [Terraform validate documentation](https://developer.hashicorp.com/terraform/cli/commands/validate).

## Linting and static code analysis

Terraform does not have a built-in linter, so many organizations rely on third-party tools such as [TFLint](https://github.com/terraform-linters/tflint). A linter enforces code standards using static analysis, and most linting tools support custom rules.

## Comments

Write code so it is easy to understand. Use comments only when necessary to clarify complexity for other maintainers.

- Use `#` for both single-line and multi-line comments.
- Avoid `//` and `/* */` comment syntax, which are supported for backward compatibility but not idiomatic.

```hcl
# Each tunnel is responsible for encrypting and decrypting traffic exiting
# and leaving its associated gateway.
resource "google_compute_vpn_tunnel" "tunnel1" {
  # ...
}
```

## Resource naming

Every resource within a configuration must have a unique name. For consistency and readability:

- Use a descriptive noun.
- Separate words with underscores.
- Do not include the resource type in the identifier.
- Wrap the resource type and name in double quotes.

❌ Bad:

```hcl
resource aws_instance webAPI-aws-instance {
  # ...
}
```

✅ Good:

```hcl
resource "aws_instance" "web_api" {
  # ...
}
```

## Resource order

The order of resources and data sources in code does not affect how Terraform builds them; Terraform determines creation order from cross-resource dependencies.

Organize resources for readability and make code build on itself. Define data sources before the resources that reference them.

Example:

```hcl
data "aws_ami" "web" {
  # ...
}

data "aws_availability_zones" "available" {
  # ...
}

resource "aws_instance" "web" {
  ami               = data.aws_ami.web.id
  availability_zone = data.aws_availability_zones.available.names[0]
  # ...
}
```

Recommended order for resource parameters:

1. `count` or `for_each` meta-argument, if present.
2. Resource-specific non-block parameters.
3. Resource-specific block parameters.
4. `lifecycle` block, if required.
5. `depends_on` parameter, if required.

## Variables

While variables make modules more flexible, overusing them can make code harder to understand. When deciding whether to expose a variable, consider whether the value will actually change between deployments.

- Define a `type` and `description` for every variable.
- If a variable is optional, provide a reasonable `default`.
- For sensitive variables, set `sensitive = true`. Terraform still stores the value in plaintext in state, but it will hide the value in CLI output.
- Use [input variable validation](https://developer.hashicorp.com/terraform/language/block/variable#validation) when variable values require uniquely restrictive rules.

Example:

```hcl
variable "web_instance_count" {
  type        = number
  description = "Number of web instances to deploy. This application requires at least two instances."

  validation {
    condition     = var.web_instance_count > 1
    error_message = "This application requires at least two web instances."
  }
}
```

Recommended order for variable parameters:

1. Type
2. Description
3. Default (optional)
4. Sensitive (optional)
5. Validation blocks

## Outputs

Output values expose data about infrastructure and make it easier to reference from other Terraform configurations.

- Provide a `description` for every output.
- Use the following order for output parameters:
  1. Description
  2. Value
  3. Sensitive (optional)
- Use descriptive nouns and underscores for output names.

Example:

```hcl
output "web_public_ip" {
  description = "Public IP of the web instance"
  value       = aws_instance.web.public_ip
}
```

## Local values

Local values let you reference expressions or values multiple times. Use them sparingly, because overuse can make code harder to understand.

- If a local value is referenced in multiple files, define it in `locals.tf`.
- If it is specific to a single file, define it at the top of that file.
- Use descriptive nouns and underscores for names.

Example:

```hcl
locals {
  name_suffix = "${var.region}-${var.environment}"
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${local.name_suffix}"
  }
}
```

For more information, refer to the [locals block documentation](https://developer.hashicorp.com/terraform/language/block/locals) and the [Simplify Terraform configuration with locals](https://developer.hashicorp.com/terraform/tutorials/configuration-language/locals) tutorial.

## Provider aliasing

Provider aliasing lets you define multiple `provider` blocks for the same provider, such as provisioning resources in different regions.

Example:

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "example" {
  provider = aws.west
  # ...
}

module "aws_vpc" {
  source = "./aws_vpc"
  providers = {
    aws = aws.west
  }
}
```

- Any provider block without `alias` is the default provider configuration.
- Always include a default provider configuration.
- Define all provider configurations in the same file.
- If you define multiple providers, define the default first.
- For non-default providers, define `alias` as the first parameter of the provider block.

## Dynamic resource count

The `for_each` and `count` meta-arguments let you create multiple resources from a single `resource` block based on runtime conditions.

- Use `count` when resources are almost identical.
- Use `for_each` when resource instances require distinct values that cannot be derived from an integer.
- `for_each` accepts a `map` or `set`; Terraform creates one instance per element.

Example:

```hcl
variable "web_instances" {
  type        = list(string)
  description = "A list of instances for the web application"
  default = [
    "ui",
    "api",
    "db",
    "metrics"
  ]
}

resource "aws_instance" "web" {
  for_each = toset(var.web_instances)
  ami           = data.aws_ami.webapp.id
  instance_type = "t3.micro"

  tags = {
    Name = "web_${each.key}"
  }
}

output "web_private_ips" {
  description = "Private IPs of the web instances"
  value = {
    for k, v in aws_instance.web : k => v.private_ip
  }
}

output "web_ui_public_ip" {
  description = "Public IP of the web UI instance"
  value       = aws_instance.web["ui"].public_ip
}
```

Meta-arguments simplify code but add complexity, so use them in moderation.

Conditional creation with `count` is common:

```hcl
variable "enable_metrics" {
  description = "True if the metrics server should be deployed"
  type        = bool
  default     = true
}

resource "aws_instance" "web" {
  count = var.enable_metrics ? 1 : 0

  ami           = data.aws_ami.webapp.id
  instance_type = "t3.micro"
}
```

If the effect of a meta-argument is not immediately obvious, add a comment for clarification.

## .gitignore

Define a `.gitignore` file that excludes files you should not publish to version control, such as Terraform state files.

Do not commit:

- `terraform.tfstate` and `terraform.tfstate.*` backup files.
- `.terraform.tfstate.lock.info`.
- The dependency lock file `.terraform.lock.hcl`.
- The `.terraform` directory.
- Saved plan files created with `terraform plan -out`.
- Any `.tfvars` files containing sensitive information.

Always commit:

- All Terraform code files.
- A `README.md` describing inputs, outputs, and module usage.

Refer to [GitHub's Terraform .gitignore file](https://github.com/github/gitignore/blob/main/Terraform.gitignore).

## Workflow style

This section reviews standards that enable predictable and secure Terraform workflows.

- Pin your Terraform, provider, and module versions.
- Name module repositories using `terraform-<PROVIDER>-<NAME>` when publishing to a registry.
- Store local modules at `./modules/<module_name>`.
- Use the `tfe_outputs` data source or provider-specific data sources to share state between workspaces.
- Protect credentials with dynamic provider credentials or a secrets manager such as HashiCorp Vault.
- Write tests for your modules.
- Use policy enforcement on HCP Terraform to set guardrails.

## Version pinning

To prevent upstream upgrades from introducing unintentional changes, declare version constraints for providers and Terraform itself.

- Every provider dependency should include a `version` constraint in the `required_providers` block.
- At minimum, declare the oldest provider version the module is known to work with using `>=`.
- Root modules should also constrain the maximum provider version they are intended to work with to avoid accidental upgrades to incompatible releases.
- Use the `~>` operator to allow patch releases within a specific minor release when that behavior is appropriate.
- Do not use maximum-version constraints like `~>` in reusable modules intended to be consumed by many configurations. That can force consumers to update multiple modules at once during routine upgrades.
- Set a minimum required Terraform version with `required_version` in the `terraform` block.

Example:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.34.0"
    }
  }
  required_version = "~> 1.14.8"
}
```

- For registry modules, use the `version` parameter in the `module` block.
- Local modules ignore the `version` parameter.

## Module repository names

Registry module repositories should follow the naming convention `terraform-<PROVIDER>-<NAME>`.

Examples: `terraform-google-vault`, `terraform-aws-ec2-instance`.

## Module structure

Terraform modules define self-contained, reusable pieces of infrastructure-as-code.

Use modules to group logically related resources that are provisioned together, such as:

- A networking module for VPC, subnets, gateways, and security groups.
- An application module for compute, storage, and supporting networking.

Review the [module creation recommended pattern documentation](https://developer.hashicorp.com/terraform/tutorials/modules/pattern-module-creation) and [standard module structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure).

## Local modules

Local modules are sourced from disk rather than a remote registry.

- Prefer publishing reusable modules to a registry such as the [HCP Terraform private registry](https://developer.hashicorp.com/terraform/cloud-docs/registry).
- If you cannot use a registry, define child modules under `./modules/<module_name>`.

## Repository structure

How you structure modules and Terraform configuration in version control impacts versioning and operations.

- Store actual infrastructure configuration separately from module code.
- Use one repository per module when possible for independent versioning.
- Organize infrastructure configuration repositories by logical resource groups.

A monorepo is also valid, but it requires workflows that target modified directories and may reduce access control granularity.

Example monorepo layout:

```text
.
├── modules
│   ├── function
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variables.tf
│   ├── queue
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variables.tf
│   └── vpc
│       ├── main.tf
│       ├── outputs.tf
│       └── variables.tf
├── main.tf
├── outputs.tf
└── variables.tf
```

## State sharing

Avoid sharing full Terraform state files when possible because state can contain sensitive information.

- In HCP Terraform or Terraform Enterprise, use the `tfe_outputs` data source to reference resources across workspaces.
- Otherwise, use provider data sources to query existing resources by ID or tags.

## Secrets management

If you do not configure remote state storage, Terraform stores state in plaintext locally. State can include passwords, private keys, and other sensitive data.

- In HCP Terraform or Terraform Enterprise, use state encryption and dynamic provider credentials when available.
- In Terraform Community Edition, configure provider credentials using environment variables and retrieve secrets from a secure secrets manager such as HashiCorp Vault.
- In CI/CD pipelines, use native secret storage for environment variables and avoid embedding secrets in code.

## Integration and unit testing

Write tests for your Terraform modules and run them as part of pull request validation or CI/CD.

- Tests validate module behavior and logic.
- They are complementary to validation features such as variable validation, preconditions, postconditions, and check blocks.

Refer to the [Terraform test documentation](https://developer.hashicorp.com/terraform/language/tests) and the [Write Terraform tests tutorial](https://developer.hashicorp.com/terraform/tutorials/configuration-language/test).
