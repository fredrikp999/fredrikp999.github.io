---
layout: post
title:  "Argo events - get started"
date:   2023-01-08 16:00:10 +0000
categories: homelab CICD
tags: homelab kubernetes argo CD
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1061573699083444306/Fredrik999_a_happy_orange_squid_with_big_round_eyes._In_the_sun_33a17c50-9944-411e-8e31-86e075ca5e75.png
---

Argo events...

# Install argo events
## Prerequisites
* Kubernetes cluster (e.g. k3d if going really light)
* Helm

## Install argo events using helm
```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo-events argo/argo-events
```