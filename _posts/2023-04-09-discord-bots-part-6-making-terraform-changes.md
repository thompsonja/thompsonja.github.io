---
title: "Discord Bot Pipeline - Making Terraform Changes"
excerpt: "How do we make changes to our shiny new GCP project?"
date: 2023-04-09 12:50:00 -0400
categories:
  - discord
tags:
  - infrastructure
  - cicd
  - gcp
  - terraform
permalink: discordbots/part6-terraform-changes
series: discordbots
---

{% include series.html %}

With our new GCP project created, the next logical step is to learn how to make
changes to it. Using the standard "Click Ops" model, we can easily make changes
directly using the GCP management console site by clicking through GCP's
graphical frontend (hence "Click Ops"). However, as we have covered earlier,
this is prone to issues, including irreversibility and lack of review process or
auditability oversight.

## Viewing the project

If you navigate to the [GCP Console page](https://console.cloud.google.com/),
you can find an organization/project drop down in the top menu:

![Main Console](/assets/images/discordbots/terraform_changes/1%20-%20Main%20Console.png)

Select your newly created `discord-bots` project:

![Select Project](/assets/images/discordbots/terraform_changes/2%20-%20Changing%20Projects.png)

One of the first things you may notice is that if you attempt to view any part
of the project you will see an insufficient permissions error. For example, if
we try to look at Compute resources (which we'd expect to be empty), you'll see
the following image:

![Insufficient Access](/assets/images/discordbots/terraform_changes/3%20-%20Insufficient%20Access.png)

This is because we did not set any permissions in our Terraform configs! Let's
remedy that, which will also showcase the process for using GitHub to propose,
check, and make changes to our Terraform configs.

## Making Terraform changes

First let's make sure that we've fetched the latest git branches, since the
previous step resulted in a new remote branch, `discord-bots_deploy`, being
created.

```bash
# Pull everything from remote
git fetch

# Set yourself on top of the discord-bots_deploy branch
git checkout origin/discord-bots_deploy

# Create a new working branch
git checkout -b viewer
```

Let's go ahead and make a few changes to `terraform/project.tf`:

- Add a few more GCP services
  - Cloud Build
  - Container Registry
  - Cloud Run
- Add a google_project_iam_member role (roles/viewer)

This looks like the following diff:

![Terraform Change](/assets/images/discordbots/terraform_changes/4%20-%20Terraform%20Change.png)

Once you've made the change locally, changing the user to yourself of course,
commit it and push it to GitHub:

```bash
git add terraform/project.tf
git commit -m "Add services. Make self a viewer."
git push
```

Navigate to your GitHub Terraform repo, and you'll see the following:

![Create PR](/assets/images/discordbots/terraform_changes/5%20-%20Create%20a%20PR.png)

Go ahead and click the "Compare & Pull Request" button. It will bring up the
GitHub Pull Request page. Set the `base` branch to `discord-bots_deploy`. Our
Workflows are configured to run on branches ending in `_deploy`, so it's
critical that the `base` branch is the project you want to update:

![Set Base Branch](/assets/images/discordbots/terraform_changes/6%20-%20Set%20Base%20Branch.png)

Click `Create pull request`, which will route you to a new Pull Request page:

![Set Base Branch](/assets/images/discordbots/terraform_changes/7%20-%20Pull%20Request.png)

The Pull Request page will show the check for Terraform Plan is still in
progress. The GitHub Workflow was set up to automatically comment on the Pull
Request once it finishes. An example comment is shown below:

![Terraform Status](/assets/images/discordbots/terraform_changes/8%20-%20Terraform%20Status.png)

Some of these checks, like `Terraform Format`, are optional, but checks like
`Terraform Plan` will fail the GitHub Workflow if they do not succeed. Expand
the `Show plan` accordion to see Terraform's output:

![Terraform Plan](/assets/images/discordbots/terraform_changes/9%20-%20Terraform%20Plan.png)

Notice that the output shows correctly that we're adding 3 new GCP services and
the `roles/viewer` role to yourself. Merge the Pull Request. This will kick off
another GitHub Workflow for `Terraform Apply` that you can view similar to how
you viewed it for project creation. Verify it succeeded, and once it has, you
can now navigate back to the GCP Console page and refresh it:

![Sufficient Access](/assets/images/discordbots/terraform_changes/10%20-%20Sufficient%20Access.png)

## Adding branch protections

One thing that you may have noticed is that you could have merged the Pull
Request prior to having `Terraform Plan` run. This is not desired, so let's fix
that. Go to your repo's `Settings` page and select `Branches`. There will be an
option to `Add branch protection rule`:

![Add Branch Protection Rule](/assets/images/discordbots/terraform_changes/11-%20Add%20Branch%20Protection%20Rule.png)

I use the following configuration, but feel free to modify it as you see fit.
In particular, you may want to require a review, especially if you have
multiple collaborators on the project. If it's just you, feel free to not
require it, as you cannot review your own Pull Requests:

![Branch Protection Config](/assets/images/discordbots/terraform_changes/12%20-%20Protection%20Rule%20Config.png)

Apply the branch protection rule, and it will affect all `_deploy` branches for
this project and any future ones you create.

## Next up

Congratulations! You've now performed the entire GitHub based workflow for
applying Terraform to your Cloud Infrastructure! In the next section we'll
create some Discord bot specific Terraform configurations. Stay tuned!
