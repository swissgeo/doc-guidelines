# Terraform Best Practices and Guidelines

General terraform guidelines and best practices

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [Code Organization](#code-organization)
- [Orchestration](#orchestration)
- [State Management](#state-management)
  - [Remote State](#remote-state)
    - [State key](#state-key)
  - [State migration](#state-migration)
- [Module](#module)
  - [Module Guidelines](#module-guidelines)
  - [Reusable/child Module Usage](#reusablechild-module-usage)
  - [Code generation via Terramate](#code-generation-via-terramate)
- [Development Guidelines](#development-guidelines)
- [Code formatting](#code-formatting)

## Introduction

Terraform is a powerful tool for provisioning and managing infrastructure as code. To ensure maintainability, scalability, and security, it's important to follow best practices and guidelines when working with Terraform.

## Code Organization

The high level code organization should look like this

```text
|- modules/                        # terraform reusable modules for the whole repository
|- CLOUD_PROVIDER                  # Github, AWS, Google, etc
|    |- ORGANIZATIONS_OR_ACCOUNT   # e.g. AWS Account name, Github organization
|           |- modules             # terraform reusable modules for this organization/account
|           |- shared              # shared resources
|           |- ENVIRONMENT         # resource pro environment (dev, int or prod)
|                |- SYSTEM         # System infrastructure
|                |- shared-env     # Infrastructure shared between all systems in that environment
|- scripts                         # Helper scripts, like for example template to create new root module
|- terramate                       # Terramate globals and global generator
     |- generators
     |- globals
```

Then it should mirror the hierarchical structure of the infrastructure.

> [!IMPORTANT]
> Each terraform module should be kept small !

## Orchestration

We use Terramate to orchestrate the Terraform root module (e.g. getting Terraform plan of all modules
changed)

See [Terramate](https://terramate.io/docs/)

## State Management

### Remote State

We use remote state management using AWS S3 backend

- Encrypt the state file at rest (see [Terraform Documentation](https://developer.hashicorp.com/terraform/language/settings/backends/s3#encrypt)).
- For AWS ressources we use one state bucket per AWS Account
- Cross account state bucket access **IS NOT ALLOWED**. This allow a simpler terraform state bucket access management and also improve security. This means that we cannot get remote state from another account.
  - Improve security as one developer might have access to one account but the other, which means developer is only allow to use terraform on the account it has access.
  - Simplify state key migration, as we have to check for remote state only in one github repository
  - In some rare case we could allow such cross account remote state sharing, but the reason of the exception to this rule MUST BE documented !
- Inside the same account (AWS, github organization, ...) we can use remote state to access other root module data to avoid hardcoding
- State backend MUST BE managed by Terramate

#### State key

We use the terraform root module path as state key in S3 prefixed with a version string `v2` and suffixed with `.tfstate`; e.g. `v2/aws/swisstopo-swissgeo/prod/systems/www/ingress.tfstate`

The version prefix is to avoid any risk of key collision in the future in case of code structure re-organization.

> [!NOTE]
> We used to use an UUID as state keys, which has the advantage that we did not have to change the state when reorganizing resources. However it has the downside that UUID key were very hard to code review and error prone due to copy paste mistake not being easily identifiable.

### State migration

To migrate the state we use the `terraform init -migrate-state` command. After migrating the state we need to manually clean up the old state by removing the S3 file

Here is an example of migrate state procedure

1. Run the `terraform init` BEFORE changing the state key on the terraform backend file :warning:
2. Update the the state key in the terraform backend file
3. Run `terraform init -migrate-state`
4. Clean the old state in S3 and DynamoDB

```bash
OLD_KEY=<old-state-key>

aws --profile AWS_ACCOUNT_PROFILE s3 rm s3://STATE_BUCKET/${OLD_KEY}
```

## Module

*Modules are containers for multiple resources that are used together.* See [Terraform Modules](https://developer.hashicorp.com/terraform/language/modules)

### Module Guidelines

- Root module must contain the terraform backend definition
- Avoid using magic number like AWS account id in your code but use variables, locals or global variables defined in reusable module instead
- Avoid unnecessary dependencies between modules whenever possible. By unnecessary dependencies, I mean using remote state for a value when the use of the value in the resource doesn't need to exists, for example for AWS policies and role you might be tempted to get the role ARN or resource ARN from a remote state to create a new role or policy, but this would add an unnecessary dependency on the terraform module where one module needs to be applied before the other, while on the AWS resource you don't have any dependencies, the ARN resource in a policy doesn't need to exists to create the policy.
- Pay attention to circlar dependencies between modules, when the resource already exists and are imported to terraform, you don't have circular dependencies but by re-creating from scratch it might be the case !
- Due to previous point, always try to create resources via terraform, instead of creating them manually and importing them afterwards.

### Reusable/child Module Usage

Use modules to encapsulate and reuse configurations. This promotes DRY (Don't Repeat Yourself) principles and makes your configurations more manageable.

- Create modules for common resources.
- Pay attention to the drawback of reusable modules, were changing it might impact lots of other root modules

### Code generation via Terramate

In order to keep code DRY, use Terramate code generation for repetitive tasks, e.g. like terraform
backend definitions or terraform provider definitions.

## Development Guidelines

- :warning: ALWAYS change resources using terraform (no manual changes via UI, e.g. AWS web console) :warning:
- :warning: NEVER EVER apply changes before opening a Pull Request :warning:
  This ease tracking of changes and because our final goal is to do GitOps which means that git is the source of truth we must have everything
  on git. We also had the case were people applied changes from a local branch and afterward forget to push and merge the changes !
- `main` branch SHOULD reflect the current infrastructure state, which means PR that are applied should be merged as soon as they have been reviewed. DON'T keep applied PR open for more that a few days.
- Keep track of applied changes in the PR using the checkbox
  - [ ] changes applied
  
  Modify/add/remove checkbox if needed (e.g. one checkbox per module/staging, etc)
  
## Code formatting

All terraform code should have the same formatting, for this we use the standard `terraform fmt` tool to format the code.
