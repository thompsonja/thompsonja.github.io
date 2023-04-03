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
order to manage your projects with Terraform. The details of Terraform are
beyond the scope of this series, but the next section will provide a brief
overview.

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
Create a project like `terraform-controller` and clone it locally to your
machine. Note: I created these images with a different repository name.

![Create a Repository](/assets/images/discordbots/github_setup/1%20-%20Repo%20Creation.png)

From the repository's settings tab, go to Secrets and variables -> Actions,
shown highlighted below:

![Navigating to the Secrets Page](/assets/images/discordbots/github_setup/2%20-%20Actions%20Secrets.png)

We are going to create three repository secrets that will be used by the
workflows to interact with Terraform and our GCP project. Click
`New Repository secret` and add the following:

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
this file when creating the `GCP_SA_KEY` secret. Your GitHub secrets page should
look like:

![Project Secrets](/assets/images/discordbots/github_setup/3%20-%20Adding%20Secrets.png)

The goal now is to leverage branch naming conventions in order to handle
Terraform operations. Our default branch is `main`, which will contain workflows
and templates for a basic Terraform configuration. Each of our projects will be
handled by a branch with a prefix like `_deploy`. So for a project like
`discord-bots`, we will have a branch `discord-bots_deploy`. This branch will be
used to apply Terraform configs specifically for our `discord-bots` project. So
the first thing we need is a workflow to create a new project.

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
`New Project` from the left-hand panel, as shown below:

![New Project Workflow](/assets/images/discordbots/github_setup/4%20-%20Running%20the%20New%20Project%20Workflow.png)

Click the `Run Workflow` button, which will bring up a menu with various
arguments. Add your domain name (without the top level domain like .com) and set
the project name. This project name will be used as a prefix for the GCP
project. The GCP region defaults to us-east1-a, leave it as is or override it if
you prefer your bot to be hosted somewhere else.

![Setting Workflow Arguments](/assets/images/discordbots/github_setup/5%20-%20Setting%20Workflow%20Arguments.png)

Clicking on `Run workflow` will cause the workflow to begin. Clicking on it will
reveal a status page, as shown below in a successful run:

![Checking Workflow Status](/assets/images/discordbots/github_setup/6%20-%20Checking%20Status.png)

You can expand the dropdowns to see logs for each step of the workflow. For
instance, expanding `terraform plan` will show you Terraform logs for the `plan`
stage, which generally includes which resources will be created, modified, or
destroyed:

![Viewing Terraform Plan Logs](/assets/images/discordbots/github_setup/7%20-%20Expanding%20Plan%20Logs.png)

Similarly, you can look at the logs for `terraform apply`, which will show what
resources actually were created, modified, or destroyed:

![Viewing Terraform Apply Logs](/assets/images/discordbots/github_setup/8%20-%20Expanding%20Apply%20Logs.png)
