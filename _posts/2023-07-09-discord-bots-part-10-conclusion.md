---
title: "Discord Bot Pipeline - Conclusion"
excerpt: "Wrapping up the Discord bot pipeline series!"
date: 2023-07-09 12:10:35 -0400
categories:
  - discord
tags:
  - discord
  - golang
  - gcp
  - terraform
permalink: discordbots/part10-conclusion
series: discordbots
series_finished: true
published: true
---

{% include series.html %}

That's a wrap for this series! Let's recap what we've accomplished:

- Used Google Cloud Identity to create a GCP organization to manage our
  infrastructure using Terraform
- Created a GitHub repository to contain our Terraform configurations
- Created Terraform configurations to manage GCP resources
- Created GitHub workflows to run `terraform plan` and `terraform apply`
  whenever we make Terraform changes
- Created Discord bot specific Terraform configs
- Created a bot in Discord
- Wrote a bot that connects to OpenAI to generate Dalle images

That's an impressive pipeline! The best part is that this pipeline can be easily
extended to additional Discord bots, as well as other non-Discord related
projects in the future. Subsequent blog series will build off of this one, so
we'll see you then!
