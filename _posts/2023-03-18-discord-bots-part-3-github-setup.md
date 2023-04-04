---
title: "Discord Bot Pipeline - GitHub Setup"
excerpt: "Create and set up a new GitHub project for Terraform deployments"
date: 2023-03-18 12:07:35 -0400
categories:
  - discord
tags:
  - infrastructure
  - cicd
  - terraform
  - github
permalink: discordbots/part3-github-setup
series: discordbots
---

{% include series.html %}

Previously in this series, we set up our GCP organization and created an admin
project that will carry out Terraform actions. Now let's set up a GitHub project
to integrate with this GCP project. The goal here is to use GitHub workflows in
order to manage your projects with Terraform. The details of Terraform are
beyond the scope of this series, but the next section will provide a brief
overview.

## GitHub Setup

First, create a new GitHub project by navigating to https://github.com/new.
Create a project like `terraform-controller` and clone it locally to your
machine. From the repository's settings tab, go to Secrets and variables ->
Actions.

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
this file when creating the `GCP_SA_KEY` secret. These steps are shown in the
gallery below. Note: I created these images with a different repository name but
the gist is the same.

{% include image-gallery.html folder="/assets/images/discordbots/github_setup/repo" size=200 %}

We will be leveraging branch naming conventions in order to handle Terraform
operations. Our default branch is `main`, which will contain workflows and
templates for a basic Terraform configuration. Each of our projects will be
handled by a branch with a prefix like `_deploy`. So for a project like
`discord-bots`, we will have a branch `discord-bots_deploy`. This branch will be
used to apply Terraform configs specifically for our `discord-bots` project. In
the next section, we will add the basic Terraform configurations necessary for
spinning up new projects using our infrastructure.
