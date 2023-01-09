---
layout: post
title:  "Argo events - get started"
date:   2023-01-08 16:00:10 +0000
categories: homelab CICD
tags: homelab kubernetes argo CD
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1061753604135976990/Fredrik999_a_warm_summer_day_in_the_mountains._An_orange_squid__e8622bcd-4249-4102-8512-9ef02fd8dc08.png
---

Argo events... [TO BE ADDED]

# Install argo events
## Prerequisites
* Kubernetes cluster (e.g. k3d if going really light)
* Helm

## Install argo events using helm
```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo-events argo/argo-events
```