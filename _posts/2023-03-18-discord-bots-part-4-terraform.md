---
title: "Discord Bot Pipeline - Terraform Setup"
excerpt: "This post covers setting up Terraform configs to handle Discord bot
creation."
date: 2023-03-18 12:06:35 +0000
categories:
  - discord
tags:
  - infrastructure
  - cicd
  - gcp
  - terraform
permalink: discordbots/part4-terraform
series: discordbots
published: false
---

{% include series.html %}

Previously we set up a GCP organization and created an admin project to be used
to carry out terraform operations.

Create a folder named terraform in your git project folder and add the following
files:

1. backend.tf
1. outputs.tf
1. project.tf
1. provider.tf
1. terraform.tfvars
1. variables.tf

This will form the foundation of our Terraform configuration and will be applied
to every project we create. Therefore, this should contain the bare minimum
configuration we would expect to use across any project, even those beyond
Discord bots.

Below we'll discuss the purpose of each file.

## variables.tf and terraform.tfvars

The `variables.tf` file unsurprisingly defines variables to be used by other
Terraform configs. Its contents are below:

```
variable "project_name" {}
variable "billing_account" {}
variable "org_id" {}
variable "region" {}
variable "zone" {}
```

These are basically the variables that define the environment the project will
use to operate. Some of these, like `billing_account` and `org_id`, will be
essentially global across all of your projects, since they are tied to your GCP
account. Others, like `project_name`, are project-specific and will be different
for each project you create.

The `terraform.tfvars` file contains the values for these variables:

```
project_name = "{{PROJECT_NAME}}"
billing_account = "{{BILLING_ACCOUNT}}"
org_id = "{{ORGANIZATION_ID}}"
region = "{{REGION}}"
zone = "{{ZONE}}"
```

Note that these are still parameterized and will be populated by our workflow
that creates a new project.

On the next post, we'll talk about how we can leverage GitHub Actions to perform
Terraform actions for us.

## backend.tf

The `backend.tf` file will be configured to tell Terraform where to find the
persistent Terraform state. We created this as a GCS bucket earlier in this
series, and each project will use its own state file, as shown in the file
below:

```
terraform {
  backend "gcs" {
    bucket = "${DOMAIN}-tf-controller"
    prefix = "{{PROJECT_NAME}}/terraform/state"
  }
}
```

Where `${DOMAIN}` is the domain you used earlier in this series.

## provider.tf

We are using GCP, so add GCP providers here:

```
provider "google" {
  region = var.region
}

provider "google-beta" {
  region = var.region
}
```

## outputs.tf

This file contains exposed values that are essentially the results of applying
the Terraform config. This is mainly for other Terraform configs to use, and is
essentially the GCP project ID of the project that will be created:

```
output "project_id" {
  value = google_project.project.project_id
}
```

## project.tf

This file consists of the core set of Terraform objects that will be part of
every project you create. This is a bit subjective, as not all projects may use
every resource you define here, but I typically like to keep this as the set of
resources that are pretty commonly used. In my case, I set this up as:

```
resource "random_id" "id" {
  byte_length = 4
  prefix      = var.project_name
}

resource "google_project" "project" {
  name            = var.project_name
  project_id      = random_id.id.hex
  billing_account = var.billing_account
  org_id          = var.org_id
}

resource "google_project_service" "service" {
  provider = google-beta

  for_each = toset([
    "artifactregistry.googleapis.com",
    "cloudbilling.googleapis.com",
    "cloudresourcemanager.googleapis.com",
    "compute.googleapis.com",
    "firebase.googleapis.com",
    "firestore.googleapis.com",
    "iam.googleapis.com",
    "secretmanager.googleapis.com",
    "serviceusage.googleapis.com"
  ])

  service = each.key

  project            = google_project.project.project_id
  disable_on_destroy = false
}

resource "time_sleep" "artifact_registry_api_enabling" {
  depends_on = [
    google_project_service.service["artifactregistry.googleapis.com"]
  ]

  create_duration = "2m"
}

resource "google_artifact_registry_repository" "artifact-repo" {
  provider      = google-beta
  location      = var.zone
  project       = google_project.project.project_id
  repository_id = "services"
  description   = "Docker images for services"
  format        = "DOCKER"

  depends_on = [
    time_sleep.artifact_registry_api_enabling
  ]
}
```

This adds a GCP service resource for many common services, like Firebase,
Secret Manager, and Artifact Registry. it also creates a default Artifact
Registry Repository for Docker images, as it's fairly common for projects to
need to push Docker images to some private repository (and this one is no
exception).
