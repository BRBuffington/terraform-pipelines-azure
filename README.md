[[_TOC_]]

# Introduction
This repository contains azure devops pipeline template for terraform; it will be used to standardize, provide guardrails, and simplify deployments of terraform compositions. The templates are all YAML-based and will be used by the composition pipelines.

**Templates:**
* Terraform Validate
* Terraform CICD

A composition repository template can be found [here](https://dev.azure.com/GuggenheimPartners/SS-Terraform-Compositions/_git/zzTemplate.Terraform).

To jumpstart terraform compositions, a [runbook](https://dev.azure.com/GuggenheimPartners/SS-Terraform-Compositions/_build?definitionId=20) has been created that will create all the required pipelines, branch policies, and bare-bones terraform composition repo.

# Pipeline Templates

## Terraform Validate
This pipeline is used to do a quick verification of terraform code syntax. This pipeline can be ran manually and on every commit to the feature branch. The triggers can be changed to accomodate the composition's needs.

### *Steps*
* `terraform fmt -check`
* `terraform validate` (no backend)

### *How to use*
The template can be used by extending the composition pipeline like below:
```
trigger:                                                    # Recommended triggers
  branches:
    include:
      - feature/*
      - hotfix/*
      - bug/*

  paths:                                                    # Recommended paths to include and exclude
    include:
      - "*.tf"
      - "*.hcl"
      - "*.tfvars"
      - ".azuredevops/"
      - "infra/"
    exclude:
      - "infra/documentation/"
      - "infra/docs/"
      - "infra/*.md"

variables:
  - group: terraform-secrets                                # [Required] Do not change
  - group: terraform-common                                 # [Required] Do not change
  - group: solution-specific                                # [Optional] This will be rare!

resources:
  repositories:
    - repository: templates                                 # Templates repository
      type: git
      name: SS-IaC-Management/terraform-pipelines-azure
      ref: "refs/heads/main"

extends:
  template: terraform-validate.yaml@templates
    parameters:
        pre_run_steps: <StepList>                           # Accepts any Azure DevOps steps/tasks
        post_run_steps: <StepList>                          # Accepts any Azure DevOps steps/tasks
        pool: <object>                                      # Accepts Azure DevOps pool (VmImage or Name)
        working_dir: <string>                               # Override the default working directory
```

## Terraform CICD
This is the main terraform pipeline template. It will be used during PR reviews as build validation, speculative terraform plan runs, and actual terraform apply (deployment). The release stages are dynamic and it will create deployment stages for each environment depending on which branch triggered the pipeline. The pipeline allows the following hooks for extensibility:
* Pre/Post-Plan
* Pre/Post-Apply

### *Steps*
**CI**
* Download Terraform
* `Terraform fmt -check`
* Policy Checks - OPA/Conftest (Terraform HCL2 and Plan)
* Dynamic Tagging - Bridgecrew Yor
* Terraform Plan + Artifact (publish, if release)
* Static Code Analysis - Bridgecrew Checkov
* *FUTURE: Dynamic Documentation - Terraform-Docs*

**CD**
* Download plan artifact
* Pause (Manual Validation)
* `Terraform apply`

### *How to use*
The template can be used by extending the composition pipeline like below:
```
parameters:
  - name: configs                                           # TFvars file name, adjust to the required environments
    displayName: Environment Configs (tfvars)?
    type: object
    default:
      - eus2-sbp
      - eus2-si
      - eus2-qa
      - eus2-uat

  - name: plan_only                                         # Used during manual triggers to allow plan only runs
    displayName: Plan Only?
    type: string
    default: Yes
    values:
      - Yes
      - No

  - name: destroy                                           # Changes the plan and apply to destroy
    displayName: Destroy?
    type: boolean
    default: false

trigger:                                                    # Recommended triggers, do not change unless directed by Cloud Engineering
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - "*.tf"
      - "*.hcl"
      - "*.tfvars"
      - ".azuredevops/"
      - "infra/"
    exclude:
      - "infra/documentation/"
      - "infra/docs/"
      - "infra/*.md"

variables:
  - group: terraform-secrets                                # [Required] Do not change
  - group: terraform-common                                 # [Required] Do not change
  - group: solution-specific                                # [Optional] This will be rare!

resources:
  repositories:
    - repository: templates                                 # Main Terraform CICD pipeline template
      type: git
      name: SS-IaC-Management/terraform-pipelines-azure
      ref: "refs/heads/main"                                # Use main branch, unless directed by Cloud Engineering

extends:
  template: terraform-cicd.yaml@templates
  parameters:
    configs: ${{ parameters.configs }}                      # [Required] Do not change, should be from parameters
    plan_only: ${{ parameters.plan_only }}                  # [Required] Do not change, should be from parameters
    destroy: ${{ parameters.destroy }}                      # [Required] Do not change, should be from parameters
    policy_branch_name: main                                # [Required] OPA/Conftest Policies branch name
    pool: <object>                                          # [Optional] Accepts Azure DevOps pool (VmImage or Name)
    working_dir: <string>                                   # [Optional] Override the default working directory
    backend_file: <string>                                  # [Optional] Override the default backend file tfvars
    pat: <string>                                           # [Optional] Override the default AZDO_PAT from the terraform-secrets variable group
    terraform_env: <object>                                 # [Optional] Override the default `Environment Variables` during terraform runs
    pre_run_steps: <StepsList>                              # [Optional] Azure DevOps Steps/Tasks. This will run BEFORE Plan and Apply steps
    plan_pre_run_steps: <StepsList>                         # [Optional] Azure DevOps Steps/Tasks
    plan_post_run_steps: <StepsList>                        # [Optional] Azure DevOps Steps/Tasks
    apply_pre_run_steps: <StepsList>                        # [Optional] Azure DevOps Steps/Tasks
    apply_post_run_steps:                                   # [Optional] Azure DevOps Steps/Tasks
      - script: |                                           # Sample step/task
          printenv
        displayName: "DEBUG"
        condition: always()
```