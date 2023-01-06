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
Github offers integrated CICD pipeline called github actions. This is free up to 2000 execution hours/month. See [GitHub Actions](https://docs.github.com/en/actions)

# Basics
## Environment
* Workflows/Pipelines are always executed in VMs. The VM flavor can be defined to ubuntu-linux/mac/pc
* Each job within the pipeline is executed as separate jobs in it's own freshly installed VM - or using docker in a VM

## Definitions
For more details, see [Understanding GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions)
### Workflows
* The pipeline, executing one or more jobs
* Defined in .github/workflows at the root of the repository. Triggered by an event / manually / by REST API
* You can share actions and reusable workflows within your organization, without publishing them publicly

### Events
* An activity which triggers a workflow run
* Only events connected to the repository is supported (pull request, commit pushed etc.)

### Jobs
* One or more steps used within a workflow
* All steps in a job is executed on the same runner
* Data can be shared between steps in a job as sharing same runner
* Dependencies between jobs can be defined if needed

### Steps
* A step defines the specific actions to take
* Always part of a Job
* Either a shell-script or an "action"

### Runners
* A runner is the execution environment for a Job
* Always executed in a freshly installed VM
* Can execute directly in the VM or inside a container (?)

### Actions
* An action is a custom application for the GitHub Actions platform that performs a complex but frequently repeated task
* You can write your own actions, or you can find actions to use in your workflows in the GitHub Marketplace
* You can share actions and reusable workflows within your organization, without publishing them publicly
* Find actions on: [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)

# GitHub and Kubernetes
It is not possible to execute jobs directly in kubernetes. What you can do is to deploy a lightweight kubernetes cluster in the VM, e.g. K3d to test your application in.

## k3d
To spin up k3d as part of a workflow, use the GitHub action [AbsaOSS/k3d-action](https://github.com/marketplace/actions/absaoss-k3d-action), available in marketplace
AbsaOSS/k3d-action runs k3d which is a lightweight wrapper to run k3s (Rancher Lab’s minimal Kubernetes distribution) in containers. Thanks to that, we could spin up the test environment quickly with minimal memory requirements


# Examples
## Simple workflow example
```yaml
name: learn-github-actions
run-name: ${{ github.actor }} is learning GitHub Actions
on: [push]
jobs:
  check-bats-version:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '14'
      - run: npm install -g bats
      - run: bats -v
```
{: file=".github/workflows/myworkflow.yaml" }

# References
* [GitHub Actions](https://docs.github.com/en/actions)
* [Github Actions Review and Tutorial by DevOps Toolkit](https://www.youtube.com/watch?v=eZcAvTb0rbA&t=294s)