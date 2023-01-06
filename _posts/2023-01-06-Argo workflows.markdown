---
layout: post
title:  "Argo Workflows"
date:   2023-01-06 11:00:10 +0000
categories: homelab kubernetes
tags: homelab kubernetes argo CD
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1060871516931235930/Fredrik999_a_big_old_wooden_ship_steering_wheel._focus_on_the_s_5797a714-a166-4809-978a-55558182f9b7.png
---
Argo Workflows is an open source container-native workflow engine for orchestrating parallel jobs on Kubernetes. Argo Workflows is implemented as a Kubernetes CRD (Custom Resource Definition).

# Installation
## Install prerequisites
* Kubernetes cluster
* Host with kubectl

## Install Argo workflows
For more details and up-to-date description, see [Argo Workflows Quick Start](https://argoproj.github.io/argo-workflows/quick-start/)

Install Argo
Make sure to specify wanted argo release below. Find releases here: [argo workflow releases](https://github.com/argoproj/argo-workflows/releases). Example below use 3.4.4.
```shell
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.4.4/install.yaml
```
Patch argo-server authentication
```shell
kubectl patch deployment \
  argo-server \
  --namespace argo \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "server",
  "--auth-mode=server"
]}]'
```
Optionally, Port-forward the UI
```shell
kubectl -n argo port-forward deployment/argo-server 2746:2746
```

Optionally, wait for controller to be up
```shell
kubectl wait deployment workflow-controller \
  --for condition=Available \
  --namespace argo
```

Optionally, create rolebinding (not part of official guide)
```shell
kubectl create rolebinding default-admin \
  --clusterrole cluster-admin \
  --namespace argo \
  --serviceaccount=argo:default
```
...