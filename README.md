[![Banner][banner-image]](https://masterpoint.io/)

# spacelift-automation

[![Release][release-badge]][latest-release]

💡 Learn more about Masterpoint [below](#who-we-are-𐦂𖨆𐀪𖠋).

## Purpose and Functionality

This Terraform child module provides infrastructure automation for projects in [Spacelift](https://docs.spacelift.io/).

### Overview

This `spacelift-automation` child module is designed to streamline the deployment and management of all Spacelift infrastructure, including creating a Spacelift Stack to manage itself.

Check out our quick introduction to this child module here: [![[External] terraform-spacelift-automation quick intro - Watch Video](https://cdn.loom.com/sessions/thumbnails/8de21afb732048a58fdee90042b4840f-11908d1d42de3247-full-play.gif)](https://www.loom.com/share/8de21afb732048a58fdee90042b4840f)

It automates the creation of "child" stacks and all the required accompanying Spacelift resources. For each enabled root module it creates:

1. [Spacelift Stack](https://docs.spacelift.io/concepts/stack/)
   You can think about a stack as a combination of source code, state file and configuration in the form of environment variables and mounted files.
2. [Spacelift Stack Destructor](https://docs.spacelift.io/concepts/stack/stack-dependencies.html#ordered-stack-creation-and-deletion)
   Required to destroy the resources of a Stack before deleting it. Destroying this resource will delete the resources in the stack. If this resource needs to be deleted and the resources in the stacks are to be preserved, ensure that the deactivated attribute is set to true.
3. [Spacelift AWS Integration Attachment](https://docs.spacelift.io/integrations/cloud-providers/aws#lets-explain)
   Associates a specific AWS IAM role with a stack to allow it to assume that role. The IAM role typically has permissions to manage specific AWS resources, and Spacelift assumes this role to run the operations required by the stack.
4. [Spacelift Initialization Hook](https://docs.spacelift.io/concepts/run#initializing)
   Prepares your environment before executing infrastructure code. This custom script copies corresponding Terraform tfvars files into a working directory before any Spacelift run or task as a `spacelift.auto.tfvars` file. This ensures your tfvars are [automatically loaded](https://opentofu.org/docs/v1.7/language/values/variables/#variable-definitions-tfvars-files) into the OpenTofu/Terraform execution environment.

## Usage

Spacelift Automation logic is opinionated and heavily relies on certain repository structures.
This module is configured to track all the files in the given root module directory and create Spacelift Stacks based on the provided configuration.

We support the following root module directory structures, which are controlled by the `var.root_modules_structure` variable:

### `MultiInstance` (the default)

This is the default structure that we expect and recommend. This is intended for root modules that manage multiple state files (instances) through [workspaces](https://opentofu.org/docs/cli/workspaces/) or [Dynamic Backend configurations](https://opentofu.org/docs/intro/whats-new/#early-variablelocals-evaluation).

Structure requirements:

- Stack configs are placed in `<root_modules_path>/<root_module>/stacks` directory for each workspace / instance of that stack. e.g. `root-modules/k8s-cluster/stacks/dev.yaml` and `root-modules/k8s-cluster/stacks/stage.yaml`
- Terraform variables are placed in `<root_modules_path>/<root_module>/tfvars` directory for each workspace / instance of that stack. e.g. `root-modules/k8s-cluster/tfvars/dev.tfvars` and `root-modules/k8s-cluster/tfvars/stage.tfvars`
- Stack config files and tfvars files must be equal to OpenTofu/Terraform workspace, e.g. `stacks/dev.yaml` and `tfvars/dev.tfvars` for a workspace named `dev`.
- Common configs are placed in `<root_modules_path>/<root_module>/stacks/common.yaml` file (or `var.common_config_file` value). This is useful when you know that some values should be shared across all the stacks created for a root module. For example, all stacks that manage Spacelift Policies must use the `administrative: true` setting or all stacks must share the same labels.

We have an example of this structure in the [examples/complete](./examples/complete/root-modules/), which looks like the following:

```sh
├── root-modules
│   ├── spacelift-aws-role
│   │   ├── stacks
│   │   │   └── dev.yaml
│   │   │   └── stage.yaml
│   │   │   └── common.yaml
│   │   ├── tfvars
│   │   │   └── dev.tfvars
│   │   │   └── stage.tfvars
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   └── versions.tf
│   ├── k8s-cluster
│   │   ├── stacks
│   │   │   └── dev.yaml
│   │   │   └── prod.yaml
│   │   │   └── common.yaml
│   │   ├── tfvars
│   │   │   └── dev.tfvars
│   │   │   └── prod.tfvars
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   └── versions.tf
...
```

The `spacelift-automation/main.tf` file looks something like this:

```hcl
github_enterprise = {
  namespace = "masterpointio"
}
repository = "terraform-spacelift-automation"

# Stacks configurations
root_modules_path        = "root-modules"
all_root_modules_enabled = true

aws_integration_id = "ZDPP8SKNVG0G27T4"
```

The configuration above creates the following stacks:

- `spacelift-aws-role-dev`
- `spacelift-aws-role-stage`
- `k8s-cluster-dev`
- `k8s-cluster-prod`

These stacks have the following configuration:

- Stacks track changes in GitHub repo `github.com/masterpointio/terraform-spacelift-automation`, branch `main` (the default), and directrory `root-modules`.
- Common configuration is defined in `root-modules/spacelift-aws-role/stacks/common.yaml` and applied to both Stacks. However, if there is an override in a Stack config (e.g. `root-modules/spacelift-aws-role/stacks/dev.yaml`), it takes precedence over common configs.
- Corresponding Terraform variables are generated by an [Initialization Hook](https://docs.spacelift.io/concepts/run#initializing) and placed in the root of each Stack's working directory during each run or task. For example, the content of the file `root-modules/spacelift-aws-role/tfvars/dev.tfvars` will be copied to working directory of the Stack `spacelift-aws-role-dev` as file `spacelift.auto.tfvars` allowing the OpenTofu/Terraform inputs to be automatically loaded.
  - If you would like to disable this functionality, you can set `tfvars.enabled` in the Stack's YAML file to `false`.

### `SingleInstance`

This is a special case where each root module directory only manages one state file (instance). Each time you want to create a new instance of a root module, you need to create a new directory with the same code and change your inputs. **We do not recommend this structure** as it is less flexible and easily leads to anti-patterns, but it is supported.

Structure requirements:

- Stack configs are placed in `<root_modules_path>/<root_module>/stack.yaml` directory. e.g. `root-modules/rds-cluster/stack.yaml`
- Tfvars values are not supported in this structure. In this structure, we suggest you just add your tfvars as `***.auto.tfvars` or hardcode your values directly in root module code.

Here is an example of this structure that we have in the [examples/single-instance](./examples/single-instance/) directory:

```sh
├── root-modules
│   ├── spacelift-automation
│   │   ├── stack.yaml
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   └── versions.tf
│   ├── rds-cluster-dev
│   │   ├── stack.yaml
│   │   ├── main.tf
│   │   └── versions.tf
│   ├── rds-cluster-prod
│   │   ├── stack.yaml
│   │   ├── main.tf
│   │   └── versions.tf
│   ├── random-pet
│   │   ├── stack.yaml
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   └── versions.tf
...
```

The configuration above creates the following Spacelift Stacks:

- `spacelift-automation`
- `rds-cluster-dev`
- `rds-cluster-prod`
- `random-pet`

These stacks will be configured using the settings in the `stack.yaml` file.

## FAQs

### Can I create a Spacelift Stack for Spacelift Automation? (Recommended)

Spacelift Automation can manage itself as a Stack as well, and we recommend this so you can fully automate your Stack management upon merging to your given branch. Follow these steps to achieve that:

1. Create a new vanilla OpenTofu/Terraform root module in `<root_modules_path>/spacelift-automation` that consumes this child module and supplies the necessary configuration for your unique setup. e.g.

   ```hcl
   # root-modules/spacelift-automation/main.tf

   module "spacelift-automation" {
     source  = "masterpointio/automation/spacelift"
     version = "x.x.x" # Always pin a version, use the latest version from the release page.

     # GitHub configuration
     github_enterprise = {
       namespace = "masterpointio"
     }
     repository = "your-infrastructure-repo"

     # Stacks configurations
     root_modules_path        = "../../root-modules"
     all_root_modules_enabled = true

     aws_integration_id = "ZDPP8SKNVG0G27T4"
   }
   ```

2. Optionally, create a Terraform workspace that will be used for your Automation configuration, e.g.:

   ```sh
   tofu workspace new main
   ```

   Remember that Stack config and tfvars file name must be equal to the workspace e.g. `main.yaml` and `main.tfvars`. If you choose not to create a new workspace, this can be `default.yaml` and `default.tfvars`.

3. Apply the `spacelift-automation` root module.
4. Move the Automation configs to the `<root-modules>/spacelift-automation/stacks` directory and push the changes to the tracked repo and branch.
5. After pushed to your repo's tracked branch, Spacelift Automation will track the addition of new root modules and create Stacks for them.

Check out an example configuration in the [examples/complete](./examples/complete/root-modules/spacelift-automation/tfvars/example.tfvars).

<!-- NOTE to Masterpoint team: We might want to create a small wrapper to automatize this using Taskit. On hold for now. -->

### What goes in a Stack config file? e.g. `stacks/dev.yaml`, `stacks/common.yaml`, `stack.yaml`, and similar

Most settings that you would set on [the Spacelift Stack resource](https://search.opentofu.org/provider/spacelift-io/spacelift/latest/docs/resources/stack) are supported. Additionally, you can include certain automation settings that will override this module's defaults like `automation_settings.default_tf_workspace_enabled`, `automation_settings.tfvars_enabled`, `space_name`, and similar.

Below is a brief example. You can also see the full schema in our [JSON Schema file](./stack-config.schema.json).

```yaml
kind: StackConfigV1
stack_settings:
  administrative: true
  autodeploy: true
  autoretry: true
  description: "Production EKS cluster configuration"
  labels:
    - "prod"

  terraform_version: "1.9.0"
  terraform_workflow_tool: "OPEN_TOFU"

  # Security and protection
  protect_from_deletion: true
  enable_local_preview: false

  # Hooks and scripts
  before_init:
    - "echo hello-world"
  after_apply:
    - "./scripts/notify-slack.sh"

automation_settings:
  default_tf_workspace_enabled: true
  tfvars_enabled: false
```

### Why are variable values provided separately in `tfvars/` and not in the `yaml` file?

This is to support easy local and outside-spacelift operations. Keeping variable values in a `tfvars` file per workspace allows you to simply pass that file to the relevant CLI command locally via the `-var-file` option so that you don't need to provide values individually. e.g. `tofu plan -var-file=tfvars/dev.tfvars`

<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.9 |
| <a name="requirement_jsonschema"></a> [jsonschema](#requirement\_jsonschema) | >= 0.2.1 |
| <a name="requirement_spacelift"></a> [spacelift](#requirement\_spacelift) | >= 1.14 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_jsonschema"></a> [jsonschema](#provider\_jsonschema) | >= 0.2.1 |
| <a name="provider_spacelift"></a> [spacelift](#provider\_spacelift) | >= 1.14 |

## Modules

| Name | Source | Version |
|------|--------|---------|
| <a name="module_deep"></a> [deep](#module\_deep) | cloudposse/config/yaml//modules/deepmerge | 1.0.2 |

## Resources

| Name | Type |
|------|------|
| [spacelift_aws_integration_attachment.default](https://registry.terraform.io/providers/spacelift-io/spacelift/latest/docs/resources/aws_integration_attachment) | resource |
| [spacelift_drift_detection.default](https://registry.terraform.io/providers/spacelift-io/spacelift/latest/docs/resources/drift_detection) | resource |
| [spacelift_space.default](https://registry.terraform.io/providers/spacelift-io/spacelift/latest/docs/resources/space) | resource |
| [spacelift_stack.default](https://registry.terraform.io/providers/spacelift-io/spacelift/latest/docs/resources/stack) | resource |
| [spacelift_stack_destructor.default](https://registry.terraform.io/providers/spacelift-io/spacelift/latest/docs/resources/stack_destructor) | resource |
| [jsonschema_validator.runtime_overrides](https://registry.terraform.io/providers/bpedman/jsonschema/latest/docs/data-sources/validator) | data source |
| [spacelift_spaces.all](https://registry.terraform.io/providers/spacelift-io/spacelift/latest/docs/data-sources/spaces) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_additional_project_globs"></a> [additional\_project\_globs](#input\_additional\_project\_globs) | Project globs is an optional list of paths to track stack changes of outside of the project root. Push policies are another alternative to track changes in additional paths. | `set(string)` | `[]` | no |
| <a name="input_administrative"></a> [administrative](#input\_administrative) | Flag to mark the stack as administrative | `bool` | `false` | no |
| <a name="input_after_apply"></a> [after\_apply](#input\_after\_apply) | List of after-apply scripts | `list(string)` | `[]` | no |
| <a name="input_after_destroy"></a> [after\_destroy](#input\_after\_destroy) | List of after-destroy scripts | `list(string)` | `[]` | no |
| <a name="input_after_init"></a> [after\_init](#input\_after\_init) | List of after-init scripts | `list(string)` | `[]` | no |
| <a name="input_after_perform"></a> [after\_perform](#input\_after\_perform) | List of after-perform scripts | `list(string)` | `[]` | no |
| <a name="input_after_plan"></a> [after\_plan](#input\_after\_plan) | List of after-plan scripts | `list(string)` | `[]` | no |
| <a name="input_after_run"></a> [after\_run](#input\_after\_run) | List of after-run (aka `finally` hook) scripts | `list(string)` | `[]` | no |
| <a name="input_all_root_modules_enabled"></a> [all\_root\_modules\_enabled](#input\_all\_root\_modules\_enabled) | When set to true, all subdirectories in root\_modules\_path will be treated as root modules. | `bool` | `false` | no |
| <a name="input_autodeploy"></a> [autodeploy](#input\_autodeploy) | Flag to enable/disable automatic deployment of the stack | `bool` | `true` | no |
| <a name="input_autoretry"></a> [autoretry](#input\_autoretry) | Flag to enable/disable automatic retry of the stack | `bool` | `false` | no |
| <a name="input_aws_integration_attachment_read"></a> [aws\_integration\_attachment\_read](#input\_aws\_integration\_attachment\_read) | Indicates whether this attachment is used for read operations. | `bool` | `true` | no |
| <a name="input_aws_integration_attachment_write"></a> [aws\_integration\_attachment\_write](#input\_aws\_integration\_attachment\_write) | Indicates whether this attachment is used for write operations. | `bool` | `true` | no |
| <a name="input_aws_integration_enabled"></a> [aws\_integration\_enabled](#input\_aws\_integration\_enabled) | Indicates whether the AWS integration is enabled. | `bool` | `false` | no |
| <a name="input_aws_integration_id"></a> [aws\_integration\_id](#input\_aws\_integration\_id) | ID of the AWS integration to attach. | `string` | `null` | no |
| <a name="input_before_apply"></a> [before\_apply](#input\_before\_apply) | List of before-apply scripts | `list(string)` | `[]` | no |
| <a name="input_before_destroy"></a> [before\_destroy](#input\_before\_destroy) | List of before-destroy scripts | `list(string)` | `[]` | no |
| <a name="input_before_init"></a> [before\_init](#input\_before\_init) | List of before-init scripts | `list(string)` | `[]` | no |
| <a name="input_before_perform"></a> [before\_perform](#input\_before\_perform) | List of before-perform scripts | `list(string)` | `[]` | no |
| <a name="input_before_plan"></a> [before\_plan](#input\_before\_plan) | List of before-plan scripts | `list(string)` | `[]` | no |
| <a name="input_branch"></a> [branch](#input\_branch) | Specify which branch to use within the infrastructure repository. | `string` | `"main"` | no |
| <a name="input_common_config_file"></a> [common\_config\_file](#input\_common\_config\_file) | Name of the common configuration file for the stack across a root module. | `string` | `"common.yaml"` | no |
| <a name="input_default_tf_workspace_enabled"></a> [default\_tf\_workspace\_enabled](#input\_default\_tf\_workspace\_enabled) | Enables the use of `default` Terraform workspace instead of managing multiple workspaces within a root module.<br/><br/>NOTE: We encourage the use of Terraform workspaces to manage multiple environments.<br/>However, you will want to disable this behavior if you're utilizing different backends for each instance<br/>of your root modules (we call this "Dynamic Backends"). | `bool` | `false` | no |
| <a name="input_description"></a> [description](#input\_description) | A description for the created Stacks. This is a template string that will be rendered with the final config object for the stack.<br/>    See the main.tf for full internals of that object and the documentation on templatestring for usage.<br/>    https://opentofu.org/docs/language/functions/templatestring/ | `string` | `"Root Module: ${root_module}\nProject Root: ${project_root}\nWorkspace: ${terraform_workspace}\nManaged by spacelift-automation Terraform root module."` | no |
| <a name="input_destructor_deactivated"></a> [destructor\_deactivated](#input\_destructor\_deactivated) | Whether to deactivate the stack destructor by default | `bool` | `true` | no |
| <a name="input_destructor_enabled"></a> [destructor\_enabled](#input\_destructor\_enabled) | Whether to enable the stack destructor by default | `bool` | `true` | no |
| <a name="input_drift_detection_enabled"></a> [drift\_detection\_enabled](#input\_drift\_detection\_enabled) | Flag to enable/disable Drift Detection configuration for a Stack. | `bool` | `false` | no |
| <a name="input_drift_detection_ignore_state"></a> [drift\_detection\_ignore\_state](#input\_drift\_detection\_ignore\_state) | Controls whether drift detection should be performed on a stack<br/>in any final state instead of just 'Finished'. | `bool` | `false` | no |
| <a name="input_drift_detection_reconcile"></a> [drift\_detection\_reconcile](#input\_drift\_detection\_reconcile) | Flag to enable/disable automatic reconciliation of drifts. | `bool` | `false` | no |
| <a name="input_drift_detection_schedule"></a> [drift\_detection\_schedule](#input\_drift\_detection\_schedule) | The schedule for drift detection. | `list(string)` | <pre>[<br/>  "0 4 * * *"<br/>]</pre> | no |
| <a name="input_drift_detection_timezone"></a> [drift\_detection\_timezone](#input\_drift\_detection\_timezone) | The timezone for drift detection. | `string` | `"UTC"` | no |
| <a name="input_enable_local_preview"></a> [enable\_local\_preview](#input\_enable\_local\_preview) | Indicates whether local preview runs can be triggered on this Stack. | `bool` | `false` | no |
| <a name="input_enable_well_known_secret_masking"></a> [enable\_well\_known\_secret\_masking](#input\_enable\_well\_known\_secret\_masking) | Indicates whether well-known secret masking is enabled. | `bool` | `true` | no |
| <a name="input_enabled_root_modules"></a> [enabled\_root\_modules](#input\_enabled\_root\_modules) | List of root modules where to look for stack config files.<br/>Ignored when all\_root\_modules\_enabled is true.<br/>Example: ["spacelift-automation", "k8s-cluster"] | `list(string)` | `[]` | no |
| <a name="input_github_action_deploy"></a> [github\_action\_deploy](#input\_github\_action\_deploy) | Indicates whether GitHub users can deploy from the Checks API. | `bool` | `true` | no |
| <a name="input_github_enterprise"></a> [github\_enterprise](#input\_github\_enterprise) | The GitHub VCS settings | <pre>object({<br/>    namespace = string<br/>    id        = optional(string)<br/>  })</pre> | `null` | no |
| <a name="input_labels"></a> [labels](#input\_labels) | List of labels to apply to the stacks. | `list(string)` | `[]` | no |
| <a name="input_manage_state"></a> [manage\_state](#input\_manage\_state) | Determines if Spacelift should manage state for this stack. | `bool` | `false` | no |
| <a name="input_protect_from_deletion"></a> [protect\_from\_deletion](#input\_protect\_from\_deletion) | Protect this stack from accidental deletion. If set, attempts to delete this stack will fail. | `bool` | `false` | no |
| <a name="input_repository"></a> [repository](#input\_repository) | The name of your infrastructure repo | `string` | n/a | yes |
| <a name="input_root_module_structure"></a> [root\_module\_structure](#input\_root\_module\_structure) | The root module structure of the Stacks that you're reading in. See README for full details.<br/><br/>MultiInstance - You're using Workspaces or Dynamic Backend configuration to create multiple instances of the same root module code.<br/>SingleInstance - You're using copies of a root module and your directory structure to create multiple instances of the same Terraform code. | `string` | `"MultiInstance"` | no |
| <a name="input_root_modules_path"></a> [root\_modules\_path](#input\_root\_modules\_path) | The path, relative to the root of the repository, where the root module can be found. | `string` | `"root-modules"` | no |
| <a name="input_runner_image"></a> [runner\_image](#input\_runner\_image) | URL of the Docker image used to process Runs. Defaults to `null` which is Spacelift's standard (Alpine) runner image. | `string` | `null` | no |
| <a name="input_runtime_overrides"></a> [runtime\_overrides](#input\_runtime\_overrides) | Runtime overrides that are merged into the stack config.<br/>  This allows for per-root-module overrides of the stack resources at runtime<br/>  so you have more flexibility beyond the variable defaults and the static stack config files.<br/>  Keys are the root module names and values match the StackConfig schema.<br/>  See `stack-config.schema.json` for full details on the schema and<br/>  `tests/fixtures/multi-instance/root-module-a/stacks/default-example.yaml` for a complete example. | `any` | `{}` | no |
| <a name="input_space_id"></a> [space\_id](#input\_space\_id) | Place the created stacks in the specified space\_id. Mutually exclusive with space\_name. | `string` | `null` | no |
| <a name="input_space_name"></a> [space\_name](#input\_space\_name) | Place the created stacks in the specified space\_name. Mutually exclusive with space\_id. | `string` | `null` | no |
| <a name="input_spaces"></a> [spaces](#input\_spaces) | A map of Spacelift Spaces to create | <pre>map(object({<br/>    description      = optional(string, null)<br/>    inherit_entities = optional(bool, false)<br/>    labels           = optional(list(string), null)<br/>    parent_space_id  = optional(string, "root")<br/>  }))</pre> | `{}` | no |
| <a name="input_terraform_smart_sanitization"></a> [terraform\_smart\_sanitization](#input\_terraform\_smart\_sanitization) | Indicates whether runs on this will use terraform's sensitive value system to sanitize<br/>the outputs of Terraform state and plans in spacelift instead of sanitizing all fields. | `bool` | `false` | no |
| <a name="input_terraform_version"></a> [terraform\_version](#input\_terraform\_version) | OpenTofu/Terraform version to use. Defaults to the latest available version of the `terraform_workflow_tool`. | `string` | `null` | no |
| <a name="input_terraform_workflow_tool"></a> [terraform\_workflow\_tool](#input\_terraform\_workflow\_tool) | Defines the tool that will be used to execute the workflow.<br/>This can be one of OPEN\_TOFU, TERRAFORM\_FOSS or CUSTOM. | `string` | `"OPEN_TOFU"` | no |
| <a name="input_worker_pool_id"></a> [worker\_pool\_id](#input\_worker\_pool\_id) | ID of the worker pool to use.<br/>NOTE: worker\_pool\_id is required when using a self-hosted instance of Spacelift. | `string` | `null` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_spacelift_stacks"></a> [spacelift\_stacks](#output\_spacelift\_stacks) | A map of Spacelift stacks with selected attributes.<br/>To reduce the risk of accidentally exporting sensitive data, only a subset of attributes is exported. |
<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
<!-- markdownlint-enable -->
<!-- prettier-ignore-end -->

## Built By

Powered by the [Masterpoint team](https://masterpoint.io/who-we-are/) and driven forward by contributions from the community ❤️

[![Contributors][contributors-image]][contributors-url]

## Contribution Guidelines

Contributions are welcome and appreciated!

Found an issue or want to request a feature? [Open an issue][issues-url]

Want to fix a bug you found or add some functionality? Fork, clone, commit, push, and PR — we'll check it out.

## Who We Are 𐦂𖨆𐀪𖠋

Established in 2016, Masterpoint is a team of experienced software and platform engineers specializing in Infrastructure as Code (IaC). We provide expert guidance to organizations of all sizes, helping them leverage the latest IaC practices to accelerate their engineering teams.

### Our Mission

Our mission is to simplify cloud infrastructure so developers can innovate faster, safer, and with greater confidence. By open-sourcing tools and modules that we use internally, we aim to contribute back to the community, promoting consistency, quality, and security.

### Our Commitments

- 🌟 **Open Source**: We live and breathe open source, contributing to and maintaining hundreds of projects across multiple organizations.
- 🌎 **1% for the Planet**: Demonstrating our commitment to environmental sustainability, we are proud members of [1% for the Planet](https://www.onepercentfortheplanet.org), pledging to donate 1% of our annual sales to environmental nonprofits.
- 🇺🇦 **1% Towards Ukraine**: With team members and friends affected by the ongoing [Russo-Ukrainian war](https://en.wikipedia.org/wiki/Russo-Ukrainian_War), we donate 1% of our annual revenue to invasion relief efforts, supporting organizations providing aid to those in need. [Here's how you can help Ukraine with just a few clicks](https://masterpoint.io/updates/supporting-ukraine/).

## Connect With Us

We're active members of the community and are always publishing content, giving talks, and sharing our hard earned expertise. Here are a few ways you can see what we're up to:

[![LinkedIn][linkedin-badge]][linkedin-url] [![Newsletter][newsletter-badge]][newsletter-url] [![Blog][blog-badge]][blog-url] [![YouTube][youtube-badge]][youtube-url]

... and be sure to connect with our founder, [Matt Gowie](https://www.linkedin.com/in/gowiem/).

## License

[Apache License, Version 2.0][license-url].

[![Open Source Initiative][osi-image]][license-url]

Copyright © 2016-2025 [Masterpoint Consulting LLC](https://masterpoint.io/)

<!-- MARKDOWN LINKS & IMAGES -->

[banner-image]: https://masterpoint-public.s3.us-west-2.amazonaws.com/v2/standard-long-fullcolor.png
[license-url]: https://opensource.org/license/apache-2-0
[osi-image]: https://i0.wp.com/opensource.org/wp-content/uploads/2023/03/cropped-OSI-horizontal-large.png?fit=250%2C229&ssl=1
[linkedin-badge]: https://img.shields.io/badge/LinkedIn-Follow-0A66C2?style=for-the-badge&logoColor=white
[linkedin-url]: https://www.linkedin.com/company/masterpoint-consulting
[blog-badge]: https://img.shields.io/badge/Blog-IaC_Insights-55C1B4?style=for-the-badge&logoColor=white
[blog-url]: https://masterpoint.io/updates/
[newsletter-badge]: https://img.shields.io/badge/Newsletter-Subscribe-ECE295?style=for-the-badge&logoColor=222222
[newsletter-url]: https://newsletter.masterpoint.io/
[youtube-badge]: https://img.shields.io/badge/YouTube-Subscribe-D191BF?style=for-the-badge&logo=youtube&logoColor=white
[youtube-url]: https://www.youtube.com/channel/UCeeDaO2NREVlPy9Plqx-9JQ
[release-badge]: https://img.shields.io/github/v/release/masterpointio/terraform-spacelift-automation?color=0E383A&label=Release&style=for-the-badge&logo=github&logoColor=white
[latest-release]: https://github.com/masterpointio/terraform-spacelift-automation/releases/latest
[contributors-image]: https://contrib.rocks/image?repo=masterpointio/terraform-spacelift-automation
[contributors-url]: https://github.com/masterpointio/terraform-spacelift-automation/graphs/contributors
[issues-url]: https://github.com/masterpointio/terraform-spacelift-automation/issues
