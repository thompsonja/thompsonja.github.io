---
title: "Discord Bot Pipeline - Intro"
excerpt: "Introduction to a new series on Discord Bot pipelines on GCP"
date: 2023-03-18 12:04:35 -0400
categories:
  - discord
tags:
  - discord
  - infrastructure
  - cicd
permalink: /discordbots/
series: discordbots
header:
  image: /assets/images/discordbots/headers/infra.jpeg
---

{% include series.html %}

A few years back Discord [announced support for slash commands](https://discord.com/blog/slash-commands-are-here).
For a while I had been running private discord bots for my friends' servers
using the standard pattern of listening to all messages and reacting to any that
start with an exclamation point. For example, I had a bot that was used to
easily rename each other that would listen to any messages that begin with
`!rename`.

This works really well for many classes of Discord bots but had clear downsides.
For one, bots had to run persistently, which meant either running them on a
persistent VM on a cloud provider which costs money, or running them on a spare
server at home, like a Raspberry Pi. Although the latter works well and is
cheap, I generally want to avoid running services on my local network that are
reachable from the internet if I don't have to. Also, my network isn't always
reliable, due to either internet outages or just issues with my router or cable
modem, so downtime using this approach was pretty common.

The solution is to leverage serverless cloud computing infrastructure. I'm the
type of developer who likes to spend a lot of time working on infrastructure
pipelines for a personal project before even writing any actual code for the
project itself, even if it's something pretty lightweight like a Discord bot.
My requirements for the pipeline are:

- Integration with GitHub
  - Infrastructure code is stored as a GitHub repo
    - Changes to the infrastructure must be done as tested GitHub Pull Requests
  - Bot code is stored as a GitHub repo
    - Updates to the bot's code are tested and then deployed automatically
    - The repo can store multiple bot projects
  - A single source for logging across all bots
  - Email alerting when bots are having issues

This series of blog posts will cover creation of a pipeline to satisfy all of
these requirements, leveraging several key pieces of technology in order to
create a fully-fledged infrastructure pipeline for Discord bot development. I
will be breaking this down into several posts, each covering different aspects
of the pipeline. I will use the following technologies and libraries, although
you can mix and match depending on your own preferences.

- Bot development - Golang using the [discordgo](https://github.com/bwmarrin/discordgo)
  library.
  - Any other language can be chosen here, libraries exist for a plethora of
    languages.
- Infrastructure as Code - [Terraform](https://terraform.io/)
  - For simple self-hosted projects, this is unnecessary, but if you want to
    simplify the process for hosting on a cloud provider, this may be useful.
- Cloud Computing - [GCP](https://cloud.google.com/)
  - AWS or Azure can easily be used here instead
- Continuous Integration / Continuous Deployment (CICD) - [GitHub Actions](https://github.com/features/actions).
  - Since the repos are already hosted on GitHub this is a natural fit.
