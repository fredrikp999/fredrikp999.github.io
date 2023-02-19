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

# Introduction
* This post describes how to setup a CI/CDD-flow which use a number of tools and repos to enable a completely automated flow
* Starting from a new commit to a micro service, ending with an updated application with the new service deployed in a k8s-cluster with zero-touch

# Overview of the flow
* GitHub repo with micro service source code (python, flask)
* Docker hub with micro service image built and pushed by GitHub actions
* GitHub repo with app helmchart, using the micro service
* (Todo) GitHub action to build the helm-chart-package and upload to Jfrog
* Jfrog helmchart repo hosting the app helmchart
* GitHub repo with runtime-repo, defining the wanted state of the kubernetes cluster
* Flux installed in the k8s-cluster, monitoring the above run-time repo
* k8s-cluster where the application is installed

# Pre-requisites
The following needs to be prepared
## Github repo with source code for the micro service
## Github repo with source code for the application helm chart
## Docker registry for the micro service image
## Helm-chart repository e.g. at Jfrog / AWS / GCP / Local
## K8s-cluster with Flux

# Setups
## GitHub actions for building and pushing micro service image
* Create an action (CI) looking something like this in the repo (or add action from the github GUI)

```yaml
name: Docker Image CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:

  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Build the Docker image
      run: |
        docker build . --file Dockerfile --tag my-microservice:latest
    - name: Push image to Docker Hub
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login --username ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker tag my-microservice:latest ${{ secrets.DOCKER_USERNAME }}/my-microservice:latest
        docker push ${{ secrets.DOCKER_USERNAME }}/my-microservice:latest
```
{: file=".github/workflows/docker-image.yml" }

(TODO:  Update so that image is tagged with a unique tag based on dateseconds as well as with latest. Also add tests, linting etc.)

* Add secrets for your dockerhub account (DOCKER_PASSWORD and DOCKER_USERNAME)

## Jfrog manual setup
(not needed if using flux, then specify in flux manifests instead)
* Create a helmchart repo in jfrog
* Add helmchart repo (e.g. with name "my-jfrog" in example below)
* Enable anonymous access (if you want that...) https://jfrog.com/knowledge-base/artifactory-how-to-grant-an-anonymous-user-access-to-specific-repositories/
* Config->Platform Security-> "Allow anonymous access"
* To restrict access for anonymous only to specific repos: User Managent->Permissions. Add/change permission for specific repos/patterns

## If not using Flux but instead executing helm "manually" (or for testing)
* Add repo reference to helm client
```shell
helm repo add my-jfrog https://me.jfrog.io/artifactory/api/helm/charts-helm --username me@here.com --password SOMELONGTOKEN
```

## Package and upload helmchart
(TODO: Automate this part in github actions for the helm chart repo. For now manual)
* Package helmchart
```shell
helm package .
```
* Upload chart
* Get SOMTLONGTOKEN from jfrog UI, then upload helmchart.tgz
```shell
curl -ume@here.com:SOMELONGTOKEN -T ./myapp-0.1.0.tgz "https://me.jfrog.io/artifactory/charts-helm/myapp-0.1.0.tgz"
```

# Build and push using GitHub
...

# ToDo
## Describe better in this post
* Expand the steps and/or reference other posts with the details
## Improve the implementation and flow
* Build (helm package .) and push to jfrog
* Setup with auto-updates for minor version updates only etc.
* Better structure with tags, not using latest
* Add more tests and tag at success etc.
* Use other service than Jfrog as the free account is very limited, not sure what hit the roof quickly but perhaps the amount of requests from Flux?


