---
title: "Discord Bot Pipeline - Terraform Setup"
excerpt: "This post covers setting up Terraform configs to handle Discord bot
creation."
date: 2023-03-18 12:06:35 -0400
categories:
  - discord
tags:
  - infrastructure
  - cicd
  - gcp
  - terraform
permalink: discordbots/part6-discord-terraform
series: discordbots
published: false
---

{% include series.html %}

Now that we have a pipeline that can generate new projects and apply Terraform
for us, let's customize our new `discord-bots` project and create some specific
Terraform configs for bot development.
