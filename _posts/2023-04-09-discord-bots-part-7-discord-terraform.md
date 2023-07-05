---
title: "Discord Bot Pipeline - Discord Specific Terraform Setup"
excerpt: "This post covers setting up Terraform configs to handle Discord bot
creation."
date: 2023-04-09 20:23:00 -0400
categories:
  - discord
tags:
  - discord
  - infrastructure
  - cicd
  - gcp
  - terraform
permalink: discordbots/part7-discord-terraform
series: discordbots
---

{% include series.html %}

Now that we have a pipeline that can generate new projects and apply Terraform
for us, let's customize our new `discord-bots` project and create some specific
Terraform configs for bot development.

The first thing is to update `terraform/project.tf` again:

```tf
// Allow the Cloud Run Service Agent to access the Artifact Registry.
resource "google_project_iam_member" "serverless_robot_prod_roles" {
  for_each = toset([
    "roles/artifactregistry.reader",
    "roles/run.serviceAgent"
  ])
  project = google_project.project.project_id
  role    = each.key
  member  = "serviceAccount:service-${google_project.project.number}@serverless-robot-prod.iam.gserviceaccount.com"
}

// Grant Cloud Run Admin and Builder roles to the Cloud Build service account.
// This is needed to build new releases and deploy them to Cloud Run.
resource "google_project_iam_member" "cloudbuild_roles" {
  for_each = toset([
    "roles/cloudbuild.builds.builder",
    "roles/run.admin"
  ])
  project = google_project.project.project_id
  role    = each.key
  member  = "serviceAccount:${google_project.project.number}@cloudbuild.gserviceaccount.com"
}

// Give the default compute service account the Cloud Run Admin role.
// This is needed for deploying Cloud Run services.
resource "google_project_iam_member" "compute_roles" {
  for_each = toset([
    "roles/run.admin"
  ])
  project = google_project.project.project_id
  role    = each.key
  member  = "serviceAccount:${google_project.project.number}-compute@developer.gserviceaccount.com"
}

// Allow Cloud Build to manage Cloud Run services
resource "google_project_iam_member" "gcp_sa_roles" {
  for_each = toset([
    "roles/cloudbuild.serviceAgent"
  ])
  project = google_project.project.project_id
  role    = each.key
  member  = "serviceAccount:service-${google_project.project.number}@gcp-sa-cloudbuild.iam.gserviceaccount.com"
}

// Allow CloudBuild to act on behalf of another service account.
// This is needed to deploy services since each bot will use its own service account.
resource "google_service_account_iam_binding" "admin-account-iam" {
  service_account_id = "${google_project.project.id}/serviceAccounts/${google_project.project.number}-compute@developer.gserviceaccount.com"
  role               = "roles/iam.serviceAccountUser"

  members = [
    "serviceAccount:${google_project.project.number}@cloudbuild.gserviceaccount.com",
  ]
}

// Needed to automate generating new Cloud Run revisions
resource "google_service_account_iam_binding" "run-service-agent" {
  service_account_id = "${google_project.project.id}/serviceAccounts/${google_project.project.number}-compute@developer.gserviceaccount.com"
  role               = "roles/run.serviceAgent"

  members = [
    "serviceAccount:service-${google_project.project.number}@serverless-robot-prod.iam.gserviceaccount.com",
  ]
}

// Create a bucket for Cloud Build
resource "google_storage_bucket" "cloud-build-bucket" {
  project       = google_project.project.project_id
  name          = "${google_project.project.project_id}_cloudbuild"
  location      = "US"
  force_destroy = true
}

// Create a new Secret to hold a GitHub key for automation.
// This is only needed if your repository is private.
resource "google_secret_manager_secret" "github-key" {
  project   = google_project.project.project_id
  secret_id = "github-private-repo-key"

  replication {
    user_managed {
      replicas {
        location = var.zone
      }
    }
  }
}

// Allow Cloud Build to access the GitHub key secret.
resource "google_secret_manager_secret_iam_binding" "github-access" {
  project   = google_project.project.project_id
  secret_id = google_secret_manager_secret.github-key.secret_id
  role      = "roles/secretmanager.secretAccessor"
  members = [
    "serviceAccount:${google_project.project.number}@cloudbuild.gserviceaccount.com",
  ]
}

// Create a notification channel to be used by all bots.
// Replace the email address with your own.
resource "google_monitoring_notification_channel" "email-owner" {
  project      = google_project.project.project_id
  display_name = "Email Me"
  type         = "email"
  labels = {
    email_address = "joshua@thompsonja.com"
  }
}
```

## Terraform modules

[Terraform modules](https://developer.hashicorp.com/terraform/language/modules)
organize groups of Terraform resources together in a cohesive and reusable way.
They are particularly useful if we want to create multiple Discord bots so that
we can define the configuration for a single bot, and then instantiate them
easily for each bot, avoiding a lot of duplicate Terraform config. Create a new
folder in your `terraform` folder called `modules/discord_bot`. The first file
we'll want to create in this folder is one to define the variables that
configure an individual bot. Create `variables.tf` and add the following to it:

```tf
variable "bot_name" {
  description = "Name of the discord bot"
  type        = string
}

variable "github_trigger" {
  type = object({
    branch          = string
    included_files  = list(string)
    repo_owner      = string
    repo_name       = string
    cloudbuild_file = string
  })
  description = <<EOT
    github_trigger = {
      branch: "Branch to use to trigger builds"
      included_files: "Optional regex glob of files to use to trigger builds"
      repo_owner: "GitHub repository owner to use to trigger builds"
      repo_name: "GitHub repository name to use to trigger builds"
      cloudbuild_file: "GCP Cloud Build config file"
    }
  EOT
}

variable "gcp" {
  type = object({
    additional_roles       = list(string)
    additional_secrets     = list(string)
    artifact_repository_id = string
    notification_channels  = list(string)
    owner                  = string
    project_id             = string
    project_number         = string
    zone                   = string
  })
  description = <<EOT
    gcp = {
      additional_roles:       = "Additional GCP roles to generate for this bot"
      additional_secrets:     = "Additional GCP Cloud Secrets to generate for this bot"
      artifact_repository_id: = "GCP Artifact Registry Repository ID"
      notification_channels: = "List of GCP monitoring notification channel IDs"
      owner: "Owner to grant build/deploy access to"
      project_id: "GCP Project ID"
      project_number: "GCP Project Number"
      zone: "GCP Zone to deploy bot resource to"
    }
  EOT
}
```

Note that we can define nested variables, useful for organizing variables into
cohesive groups. In this case, we will define a Discord bot with a name, some
GCP configuration, and some GitHub configuration. Most of the variables are
relatively straightforward, but I want to highlight a few notable ones:

- `github_trigger.included_files` - The intent is to include all of our Discord
  bots in a single repository, meaning we'll need to differentiate which files
  should trigger a new build based on filepath globs.
- `github_trigger.cloudbuild_file` - We will define a top level
  `cloudbuild.yaml` file that configures GCP's Cloud Build to build our bots
  whenever we push a new commit to GitHub.
- `gcp.additional_roles` - Any additional roles necessary for a Discord bot.
  This may be necessary depending on what kind of interactions the bot needs.
  For example, if a bot will publish a GCP Pub Sub message, then it will need
  appropriate roles to access Pub Sub.
- `gcp.additional_secrets` - Similarly, there may be secrets, like API keys to
  other services, that you may need to specify here.

Now let's define a file in the same folder called `main.tf` which defines the
resources needed for a Discord bot, using the variables we defined in
`variables.tf`:

```tf
// Create a new Service Account. This is needed so that each Discord bot has a
// separate identity to act as, allowing us to separate access to different
// parts of our cloud infrastructure.
resource "google_service_account" "bot-service-account" {
  project      = var.gcp.project_id
  account_id   = "${var.bot_name}-sa"
  display_name = "Service Account for ${var.bot_name} discord bot"
}

// Grant yourself and the Cloud Build service account the ability to act as the
// bot Service Account. This is needed for deploying bots.
resource "google_service_account_iam_binding" "bot-iam-binding" {
  service_account_id = google_service_account.bot-service-account.name
  role               = "roles/iam.serviceAccountUser"

  members = [
    "serviceAccount:${var.gcp.project_number}@cloudbuild.gserviceaccount.com",
    "user:${var.gcp.owner}"
  ]
}

// Grant the bot Service Account the ability to write logs, in addition to any
// specified as a var.
resource "google_project_iam_member" "bot-role" {
  for_each = toset(concat(var.gcp.additional_roles, ["roles/logging.logWriter"]))
  project  = var.gcp.project_id
  role     = each.key
  member   = "serviceAccount:${google_service_account.bot-service-account.email}"
}

// Every bot gets at least a single secret created to store the Discord bot key
// generated when you create a new bot.
resource "google_secret_manager_secret" "bot-key" {
  project   = var.gcp.project_id
  secret_id = "${var.bot_name}-key"

  replication {
    user_managed {
      replicas {
        location = var.gcp.zone
      }
    }
  }
}

// Create any additional secrets specified as a var.
resource "google_secret_manager_secret" "bot-secrets" {
  for_each  = toset(var.gcp.additional_secrets)
  project   = var.gcp.project_id
  secret_id = each.key

  replication {
    user_managed {
      replicas {
        location = var.gcp.zone
      }
    }
  }
}

// Make sure you and the bot can access these secrets.
resource "google_secret_manager_secret_iam_binding" "bot-secrets-access" {
  for_each  = merge({ "bot-key" = google_secret_manager_secret.bot-key }, google_secret_manager_secret.bot-secrets)
  project   = var.gcp.project_id
  secret_id = each.value.secret_id
  role      = "roles/secretmanager.secretAccessor"
  members = [
    "serviceAccount:${google_service_account.bot-service-account.email}",
    "user:${var.gcp.owner}"
  ]
}

resource "google_secret_manager_secret_iam_binding" "bot-secrets-viewer" {
  for_each  = merge({ "bot-key" = google_secret_manager_secret.bot-key }, google_secret_manager_secret.bot-secrets)
  project   = var.gcp.project_id
  secret_id = each.value.secret_id
  role      = "roles/secretmanager.viewer"
  members = [
    "serviceAccount:${google_service_account.bot-service-account.email}",
    "user:${var.gcp.owner}"
  ]
}

// This lets you be able to update the secret when you obtain it from Discord.
resource "google_secret_manager_secret_iam_binding" "bot-secrets-writer" {
  for_each  = merge({ "bot-key" = google_secret_manager_secret.bot-key }, google_secret_manager_secret.bot-secrets)
  project   = var.gcp.project_id
  secret_id = each.value.secret_id
  role      = "roles/secretmanager.secretVersionManager"
  members = [
    "user:${var.gcp.owner}"
  ]
}

// Define a Cloud Run service
resource "google_cloud_run_service" "service" {
  name     = var.bot_name
  location = var.gcp.zone
  project  = var.gcp.project_id

  template {
    spec {
      containers {
        image   = "us-docker.pkg.dev/cloudrun/container/hello:latest"
        command = ["/server"]
        args    = ["--project_id=${var.gcp.project_id}"]
      }
      service_account_name = google_service_account.bot-service-account.email
    }
    metadata {
      annotations = {
        // For simplicity, I'm scaling from 0 to 1, if you anticipate more
        // traffic you will want to update this and make it a variable.
        "autoscaling.knative.dev/minScale" = "0",
        "autoscaling.knative.dev/maxScale" = "1"
      }
    }
  }
  autogenerate_revision_name = true

  traffic {
    percent         = 100
    latest_revision = true
  }
}

// This allows us to make the service public facing, necessary for Discord bots.
data "google_iam_policy" "noauth" {
  binding {
    role = "roles/run.invoker"
    members = [
      "allUsers",
    ]
  }
}

resource "google_cloud_run_service_iam_policy" "noauth" {
  location = google_cloud_run_service.service.location
  project  = google_cloud_run_service.service.project
  service  = google_cloud_run_service.service.name

  policy_data = data.google_iam_policy.noauth.policy_data
}

// Create a Cloud Build trigger that fires when the GitHub repo is updated.
resource "google_cloudbuild_trigger" "service-trigger" {
  // Make location global in order to avoid any quota issues with a zone
  location = "global"
  project  = google_cloud_run_service.service.project
  name     = "${google_cloud_run_service.service.name}-trigger"

  included_files = var.github_trigger.included_files != null ? var.github_trigger.included_files : []

  github {
    owner = var.github_trigger.repo_owner
    name  = var.github_trigger.repo_name
    push {
      branch = var.github_trigger.branch
    }
  }

  substitutions = {
    _APP         = var.bot_name
    _ZONE        = var.gcp.zone
  }

  filename = var.github_trigger.cloudbuild_file

  include_build_logs = "INCLUDE_BUILD_LOGS_WITH_STATUS"
}

// Create a GCP logging metric to count errors
resource "google_logging_metric" "error_logging_metric" {
  name        = "${var.bot_name}_errors"
  project     = var.gcp.project_id
  filter      = "severity >= ERROR AND logName=\"projects/${var.gcp.project_id}/logs/${var.bot_name}-logs\""
  description = "Logged errors for ${var.bot_name} bot"
  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "INT64"
  }
}

// Monitor the logging metric and send emails whenever this alert fires.
resource "google_monitoring_alert_policy" "error_logging_alert_policy" {
  display_name = "${var.bot_name} Error Alert"
  project      = var.gcp.project_id
  combiner     = "OR"
  conditions {
    display_name = "Error logs"
    condition_threshold {
      filter          = "metric.type=\"logging.googleapis.com/user/${var.bot_name}_errors\" AND resource.type=\"cloud_run_revision\""
      duration        = "300s"
      comparison      = "COMPARISON_GT"
      threshold_value = 0.5
      aggregations {
        alignment_period   = "600s"
        per_series_aligner = "ALIGN_SUM"
      }
    }
  }

  notification_channels = var.gcp.notification_channels != null ? var.gcp.notification_channels : []
  alert_strategy {
    auto_close = "3600s"
  }
}

```

Nice! That was a lot of configuration, but you should be able to make another
pull request and merge it much the same way you did before:

```bash
git add .
git commit -m "Add discord specific Terraform config"
git checkout -b bot_config
git push
```

You should be able to view the output of the `Terraform Plan` workflow, and it
should look reasonable based on all the resources we added in this section.
The only terraform piece we're missing now is the definition of the actual bot
itself. But for now, we'll switch gears into finally writing the bot!
