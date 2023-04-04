---
title: "Discord Bot Pipeline - GitHub Actions Setup"
excerpt: "GitHub Actions are a powerful way to handle CI/CD for Terraform and
our Discord bots. Read more below the fold."
date: 2023-04-03 21:57:35 -0400
categories:
  - discord
tags:
  - infrastructure
  - cicd
  - terraform
  - github
permalink: discordbots/part5-github-actions
series: discordbots
---

{% include series.html %}

Previously in this series, we created a GitHub project and populated it with
Terraform configurations. In this post, we'll go over how to leverage GitHub
Workflows as our CI/CD infrastructure for deploying Terraform updates.

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

permissions:
  contents: write

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
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
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

From your Terraform controller repository, click on `Actions` and select
`New Project` from the left-hand panel. Click the `Run Workflow` button, which
will bring up a menu with various arguments. Add your domain name (without the
top level domain like .com) and set the project name. This project name will be
used as a prefix for the GCP project. The GCP region defaults to us-east1-a,
leave it as is or override it if you prefer your bot to be hosted somewhere
else.

{% include image-gallery.html folder="/assets/images/discordbots/github_actions/workflow" size=200 %}

Clicking on `Run workflow` will cause the workflow to begin. Once it starts, an
entry will appear in the workflow runs page. Clicking on it will reveal a status
page. You can expand the dropdowns to see logs for each step of the workflow.
For instance, expanding `terraform plan` will show you Terraform logs for the
`plan` stage, which generally includes which resources will be created,
modified, or destroyed, and similarly for `terraform apply`, which shows which
resources actually were created, modified, or destroyed.

{% include image-gallery.html folder="/assets/images/discordbots/github_actions/logs" size=200 %}
