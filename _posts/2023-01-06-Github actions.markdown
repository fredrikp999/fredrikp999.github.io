---
layout: post
title:  "Github actions (CI/CD)"
date:   2023-01-06 09:00:10 +0000
categories: homelab CICD
tags: homelab CI CD github
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1060812446534746122/Fredrik999_snakes_all_over_the_place_scary_4ec4da86-dc83-40f0-bf55-6c76fe7f772a.png
---

# Introduction
Github offers integrated CICD pipeline called github actions. This is free up to 2000 execution hours/month.

# Basics
## Environment
* Pipelines are always executed in a VM. The VM flavor can be defined e.g. to ubuntu
* One VM is used for a full pipeline
* Each task within the pipeline is executed as separate jobs using docker

## Definitions
For more details, see [Understanding GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions)
### Workflow
* The pipeline, executing one or more jobs
* Defined in .github/workflows at the root of the repository. Triggered by an event / manually / by REST API

### Events
* An activity which triggers a workflow run
* Only events connected to the repository is supported (pull request, commit pushed etc.)

### Jobs
* One or more steps used a workflow
* All steps in a job is executed on the same runner
* Data can be shared between steps in a job as sharing same runner
* Dependencies between jobs can be defined if needed

### Steps
* A step defines the specific actions to take
* Always part of a Job
