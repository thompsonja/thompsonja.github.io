---
title: "Discord Bot Pipeline - GCP Setup"
excerpt: "We go over the steps involved for setting up a GCP organization that
will host GCP projects for Terraform management and the Discord bots themselves."
date: 2023-03-18 12:05:35 +0000
categories:
  - discord
tags:
  - infrastructure
  - cicd
  - gcp
  - terraform
collections:
  - discordbot
permalink: discordbots/part2-gcp
series: discordbots
---

{% include series.html %}

Creating an organization in GCP is a great way to organize your projects. For
just a single discord bot application, this is likely overkill, but this will
greatly simplify your ability to iterate on other projects. I am using Cloud
Identity, primarily because it's the cheapest option and I am not currently
running a Google Workspace.

Cloud Identity needs to be tied to a domain, which I have created for this blog,
thompsonja.com. I happen to also use [Google Domains](https://domains.google.com/)
for domain management, although any registrar will do.

I'm using the following resources from GCP:

- https://cloud.google.com/resource-manager/docs/creating-managing-organization
- https://cloud.google.com/identity/docs/set-up-cloud-identity-admin

Once you finish creating the Identity, navigating to the cloud console should
result in an organization being created for you automatically when you navigate
to the [GCP Console](https://console.cloud.google.com/).

You'll want to set up billing by navigating to
https://console.cloud.google.com/billing. Usually GCP will run a new user trial
credit option.

Now we can start setting up our new organization by adding a new GCP project.
This project will be in charge of running Terraform commands.

First, make sure you have `gcloud` installed on your machine. Follow the
instructions here if you haven't already:
https://cloud.google.com/sdk/docs/install

Once you have it installed, run the following command in a terminal:

```bash
gcloud auth login
```

And login with your newly created email you created during the Cloud Identity
creation process. Once completed, you can verify success with:

```bash
gcloud auth list
```

Run the following:

```bash
# Install jq for parsing json data from commands if you don't have it already
sudo apt-get -y install jq

GCP_BILLING_ACCOUNT_ID="$(gcloud beta billing accounts list --format=json \
      | jq -r '.[0].name' \
      | cut -d'/' -f2)"

GCP_ORGANIZATION_ID="$(gcloud organizations list --format=json \
      | jq -r '.[0].name' \
      | cut -d'/' -f2)"

# Replace DOMAIN with something unique to you, like your domain
DOMAIN="thompsonja"
TF_ADMIN="${DOMAIN}-tf-controller"
TF_CREDS="${HOME}/.config/gcloud/${TF_ADMIN}.json"
```

This performs some variable setup. `TF_ADMIN` will be the name of the GCP
project we create and must be globally unique, hence why I prefixed it with my
domain name. Also know that there is a character limit on GCP project names, so
be cautious of that if `DOMAIN` is too long. `TF_CREDS` will be where a private
service account key will be created. BE CAREFUL with this key, as it will have
admin rights over any generated project.

The first thing we need to do is create the Terraform GCP project and link it to
our billing account:

```bash
gcloud projects create ${TF_ADMIN} \
  --organization ${GCP_ORGANIZATION_ID} \
  --set-as-default

gcloud beta billing projects link ${TF_ADMIN} \
  --billing-account ${GCP_BILLING_ACCOUNT_ID}
```

Then we can create the service account and download the key:

```bash
gcloud iam service-accounts create terraform \
  --display-name "Terraform admin account"

gcloud iam service-accounts keys create ${TF_CREDS} \
  --iam-account terraform@${TF_ADMIN}.iam.gserviceaccount.com
```

We grant this service account a few bindings onto our GCP project, including
`storage.admin`, `resourcemanager.projectCreator`, and `billing.user`. These are
important as we will be using GCP Cloud Storage to store the Terraform state in
a storage bucket and will need permissions to create projects and assign billing
on newly created projects. Run the following:

```bash
gcloud projects add-iam-policy-binding ${TF_ADMIN} \
  --member serviceAccount:terraform@${TF_ADMIN}.iam.gserviceaccount.com \
  --role roles/viewer

gcloud projects add-iam-policy-binding ${TF_ADMIN} \
  --member serviceAccount:terraform@${TF_ADMIN}.iam.gserviceaccount.com \
  --role roles/storage.admin

gcloud organizations add-iam-policy-binding ${GCP_ORGANIZATION_ID} \
  --member serviceAccount:terraform@${TF_ADMIN}.iam.gserviceaccount.com \
  --role roles/resourcemanager.projectCreator

gcloud organizations add-iam-policy-binding ${GCP_ORGANIZATION_ID} \
  --member serviceAccount:terraform@${TF_ADMIN}.iam.gserviceaccount.com \
  --role roles/billing.user
```

Now we can create the Cloud Storage bucket that will contain the Terraform
state. Buckets also need to be globally unique, so it's a safe bet to use the
GCP project name as the bucket name. We also set versioning on so that we can
revert Terraform state back if we need to:

```bash
gsutil mb -p ${TF_ADMIN} gs://${TF_ADMIN}
gsutil versioning set on gs://${TF_ADMIN}
~~~ bash

Finally, we enable some GCP services that we may need in later projects. This
is because we cannot use the service account to enable a service in another
project if that service is not also enabled in the admin project:

~~~ bash
gcloud services enable appengine.googleapis.com
gcloud services enable artifactregistry.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable cloudresourcemanager.googleapis.com
gcloud services enable cloudbilling.googleapis.com
gcloud services enable firebase.googleapis.com
gcloud services enable firestore.googleapis.com
gcloud services enable iam.googleapis.com
```

Now that we've completed GCP setup, we can leverage GitHub actions to provision
new GCP projects and run Terraform commands!
