---
title: "Discord Bot Pipeline - GitHub Actions Setup"
excerpt: "GitHub Actions are a powerful way to handle CI/CD for Terraform and
our Discord bots. Read more below the fold."
date: 2023-03-18 12:07:35 +0000
categories:
  - discord
tags:
  - infrastructure
  - cicd
  - terraform
  - github
permalink: discordbots/part3-github-actions
series: discordbots
---

{% include series.html %}

Previously in this series, we set up our GCP organization and created an admin
project that will carry out Terraform actions. Now let's set up a GitHub project
to integrate with this GCP project. The goal here is to use GitHub workflows in
order to manage your Terraform projects. The details of Terraform are beyond the
scope of this series, but the next section will provide a brief overview of
Terraform.

## Terraform Basics

If you've ever worked with a cloud computing provider, you've likely encountered
the stage of your journey where you've forgotten what settings you've applied,
what permissions and service accounts you've created, and have made breaking
changes that you don't know how to revert. Terraform is an Infrastructure-as-
Code tool that allows you to define your cloud infrastructure as configuration
files.

There are two main operations you should be familiar with:

- `terraform plan` - This displays the proposed changes based on a current
  configuration.
- `terraform apply` - This applies the proposed changes, resulting in new or
  updated infrastructure.

Terraform knows about the current state of the system by maintaining a state
file. This state file will be located in the GCS bucket we created in the
previous step. When Terraform applies configuration, it also updates the state
file in addition to updating your cloud infrastructure.

## GitHub Setup

First, create a new GitHub project by navigating to https://github.com/new.
Create a project like `terraform-admin-repo` and clone it locally to your
machine.

From the repository's settings tab, go to Secrets and variables -> Actions. We
are going to create three repository secrets that will be used by the workflows
to interact with Terraform and our GCP project. Click `New Repository secret`
and add the following:

1. GCP_BILLING_ACCOUNT_ID
1. GCP_ORGANIZATION_ID
1. GCP_SA_KEY

As in the previous step, your GCP billing account ID can be found by running:

```bash
gcloud beta billing accounts list --format=json \
  | jq -r '.[0].name' \
  | cut -d'/' -f2
```

And your organization ID can be found by running:

```bash
gcloud organizations list --format=json \
  | jq -r '.[0].name' \
  | cut -d'/' -f2
```

The service account was created in the previous step and stored in
`~/.config/gcloud/<your domain>-tf-controller.json`. Copy the entire contents of
this file when creating the `GCP_SA_KEY` secret.

The goal here is to leverage branch naming conventions in order to handle
Terraform operations. So the first thing we need is a workflow to create a new
project.

## GitHub Workflows

We can use Terraform in GitHub by creating workflows to support the following:

1. A dispatch (manually-triggered) workflow that can be used to create new
   GCP projects.
1. A workflow that runs when Pull Requests are opened that triggers
   `terraform plan`. This workflow should output the result of `terraform plan`
   so that you can visually inspect the proposed changes. This workflow will be
   read-only; no changes will be made to your infrastructure when it runs.
1. A workflow that runs when Pull Requests are merged that triggers
   `terraform apply`. This workflow will perform actual changes to your
   infrastructure when it runs.

### Creating new projects

Create a file `.github/workflows/new_project.yaml` with the following contents:

{% raw %}

```yaml
name: New Project
on:
  workflow_dispatch:
    inputs:
      project_name:
        description: "GCP project name"
        required: true
      region:
        description: "GCP region for terraform configs"
        required: true
        default: "us-east1-a"

env:
  GOOGLE_CREDENTIALS: ${{ secrets.GCP_SA_KEY }}
  TF_WORKING_DIR: terraform

jobs:
  new_project:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ env.TF_WORKING_DIR }}
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v1
        with:
          terraform_wrapper: false
      - name: Create GitHub branch
        run: |
          git config --global user.name "Terraform Admin"
          git config --global user.email "you@example.com"
          git checkout -b ${{ inputs.project_name }}_deploy
          zone="$(echo ${{ inputs.region }} | cut -d'-' -f-2)"
          sed -i -e "s/{{PROJECT_NAME}}/${{ inputs.project_name }}/g" *
          sed -i -e "s/{{BILLING_ACCOUNT}}/${{ secrets.GCP_BILLING_ACCOUNT_ID }}/g" *
          sed -i -e "s/{{ORGANIZATION_ID}}/${{ secrets.GCP_ORGANIZATION_ID }}/g" *
          sed -i -e "s/{{REGION}}/${{ inputs.region }}/g" *
          sed -i -e "s/{{ZONE}}/${zone}/g" *
          rm ../.github/workflows/new_project.yaml ../admin_setup.sh
          git commit -am "Initial setup for ${{ inputs.project_name }}"
          git push origin ${{ inputs.project_name }}_deploy
      - name: terraform init
        run: terraform init
      - name: terraform plan
        run: terraform plan -no-color -lock-timeout=5m
      - name: terraform apply
        run: terraform apply -no-color -lock-timeout=5m -auto-approve
```

{% endraw %}

Note that there is a global git `user.email` configuration that can be set to
your custom domain.

### Terraform plan

Add the following file to `.github/workflows/terraform_plan.yaml`:

{% raw %}

```yaml
name: Terraform Plan
on:
  pull_request:
    paths:
      - "terraform/**"

env:
  GOOGLE_CREDENTIALS: ${{ secrets.GOOGLE_SA_KEY }}
  TF_WORKING_DIR: terraform

jobs:
  terraform_plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ env.TF_WORKING_DIR }}
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2
      - name: terraform fmt
        id: fmt
        run: terraform fmt -check
        continue-on-error: true
      - name: terraform init
        id: init
        run: terraform init
      - name: terraform validate
        id: validate
        run: terraform validate -no-color
        continue-on-error: true
      - name: terraform plan
        id: plan
        run: terraform plan -no-color -lock-timeout=5m
      - uses: actions/github-script@0.9.0
        if: github.event_name == 'pull_request'
        env:
          PLAN: "terraform\n${{ steps.plan.outputs.stdout }}"
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const output = `
            #### Terraform Format and Style 🖌\`${{ steps.fmt.outcome }}\`
            #### Terraform Initialization ⚙️\`${{ steps.init.outcome }}\`
            #### Terraform Validation 🤖\`${{ steps.validate.outcome }}\`
            #### Terraform Plan 📖\`${{ steps.plan.outcome }}\`

            <details><summary>Show Plan</summary>

            \`\`\`\n
            ${process.env.PLAN}
            \`\`\`

            </details>

            github.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
```

{% endraw %}

### Terraform apply

Add the following file to `.github/workflows/terraform_apply.yaml`

{% raw %}

```yaml
name: Terraform Apply
on:
  push:
    branches:
      - "**_deploy"
  workflow_dispatch:

env:
  GOOGLE_CREDENTIALS: ${{ secrets.GOOGLE_SA_KEY }}
  TF_WORKING_DIR: terraform

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ env.TF_WORKING_DIR }}
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v1
        with:
          terraform_wrapper: false
      - name: terraform init
        run: terraform init
      - name: terraform plan
        run: terraform plan -no-color -lock-timeout=5m
      - name: terraform apply
        run: terraform apply -no-color -lock-timeout=5m -auto-approve
```

{% endraw %}

You should end up with something very similar to
https://github.com/thompsonja/terraform-controller.

## Creating your first project
