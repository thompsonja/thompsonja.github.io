---
layout: post
title:  "Discord Bot Pipeline"
date:   2023-03-18 12:04:35 +0000
categories: discord infrastructure cicd
permalink: /discordbots/
---
A few years back Discord [announced support for slash commands](https://discord.com/blog/slash-commands-are-here).
For a while I had been running private discord bots for my friends' servers
using the standard pattern of listening to all messages and reacting on any that
start with an exclamation point. For example, I had a bot that we used to easily
rename each other that would listen to any messages that begin with `!rename`.

This works really well for a lot of bots but had a clear downside. Bots had to
persistently be running, which meant either running them on a persistent VM on
the cloud, which costs money, or running them on a spare server at home, like a
raspberry pi. Although the latter works well and is cheap, I generally want to
avoid running services on my local network if I don't have to. Also, my network
has a tendency to falter from time to time, leading to an occasional downtime.

The solution is to leverage serverless cloud computing infrastructure. I'm the
type of developer who likes to spend massive amount of times working on the
infrastructure pipeline for a personal project, even if it's something pretty
lightweight like a Discord bot. I wanted a pipeline that would be automatically
integrated with GitHub, such that updates to the bot are tested and deployed
automatically. Further, I wanted setup of multiple bots to be a breeze. Finally,
I wanted to leverage GCP features like logging and alerting to know when the bot
was having issues. This series of blog posts will cover all of these, leveraging
several key pieces of technology in order to create a fully-fledged
infrastructure pipeline for Discord bot development. I will be breaking this
down into several posts, each covering different aspects of the pipeline. Feel
free to jump around as your interest dictates.

As a summary, I am using the following technologies and libraries, although you
can mix and match depending on your own preferences.

* Bot development - Golang using the [discordgo](https://github.com/bwmarrin/discordgo)
  library.
    * Any other language can be chosen here, libraries exist for a plethora of
      languages.
* Infrastructure as Code - [Terraform](https://terraform.io/)
    * For simple self-hosted projects, this is unnecessary, but if you want to
      simplify the process for hosting on a cloud provider, this may be useful.
* Cloud Computing - [GCP](https://cloud.google.com/)
    * AWS or Azure can easily be used here instead
* Continuous Integration / Continuous Deployment (CICD) - [GitHub Actions](https://github.com/features/actions).
    * Since the repos are already hosted on GitHub this is a natural fit.

Below are quick links to all of the posts in this series:
* [Part 1](part1-gcp) - Setting up your GCP organization and project
* [Part 2](part2-terraform) - Terraform Setup
* [Part 3](part3-github-actions) - GitHub Actions
* [Part 4](part4-discordgo) - Intro to discordgo
* [Part 5](part5-bot-creation) - Building a simple bot
* [Part 6](part6-bot-deploy) - Deploying the bot