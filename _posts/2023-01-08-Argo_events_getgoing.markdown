---
layout: post
title:  "Argo events - get started"
date:   2023-01-08 16:00:10 +0000
categories: homelab CICD
tags: homelab kubernetes argo CD
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1061617378733080616/Fredrik999_a_happy_orange_squid_with_big_round_eyes_4bdcf17d-b890-4edc-94cf-ca28c2c4245a.png
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