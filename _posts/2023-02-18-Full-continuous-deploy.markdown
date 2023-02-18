---
layout: post
title:  "Full continuous deployment"
date:   2023-02-18 06:00:10 +0000
categories: homelab gitops
tags: homelab gitops flux jfrog
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1075053563501350964/Fredrik999_a_wonderful_and_magic_scene_with_a_cute_orange_squid_f4918337-1e2c-46c2-8a10-a1dec2622682.png
---
Full CI & CD pipeline including flux. Auto-triggering image build using GitHub actions and pushing to docker-hub. Helm-chart in jfrog-cloud. Deployed to k8s-cluster using Flux

References:
* ...

# Setup
* GitHub repo with micro service source code (python, flask)
* Docker hub with micro service image built and pushed by GitHub actions
* GitHub repo with app helmchart, using the micro service
* (Todo) GitHub action to build the csar and upload to Jfrog
* Jfrog helmchart repo hosting the app helmchart
* GitHub repo with runtime-repo, defining the wanted state of the kubernetes cluster
* Flux installed in the k8s-cluster, monitoring the above run-time repo
* k8s-cluster where the application is installed

## Jfrog manual (not needed if using flux, then specify in flux manifests instead)
* Create a helmchart repo in jfrog
* Add helmchart repo (e.g. with name "my-jfrog" in example below)
* Enable anonymous access (if you want that...) https://jfrog.com/knowledge-base/artifactory-how-to-grant-an-anonymous-user-access-to-specific-repositories/
* Config->Platform Security-> "Allow anonymous access"
* To restrict access for anonymous only to specific repos: User Managent->Permissions. Add/change permission for specific repos/patterns
* Add repo reference to helm client
```shell
helm repo add my-jfrog https://me.jfrog.io/artifactory/api/helm/charts-helm --username me@here.com --password SOMELONGTOKEN
```
* Package helmchart
```shell
helm package .
```
* Upload chart
```shell
curl -ume@here.com:SOMELONGTOKEN -T ./myapp-0.1.0.tgz "https://me.jfrog.io/artifactory/charts-helm/myapp-0.1.0.tgz"
```

# Build and push using GitHub
...

# ToDo
* Build (helm package .) and push to jfrog
* Setup with auto-updates for minor version updates only etc.
* Better structure with tags, not using latest
* Add more tests and tag at success etc.



