---
layout: post
title:  "GitOps with Flux, the basics"
date:   2023-02-11 09:00:10 +0000
categories: homelab gitops
tags: homelab gitops flux
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1072817422555418694/Fredrik999_djungle_party_d1c37ca1-d3b6-4445-b321-b4d802431346.png
---
Flux is a GitOps agent which e.g. can ensure that applications are consistency deployed according to a "wanted state" defined in a runtime git-repository. This post describe the basics for how to set this up

References:
* [Manage Helm releases with Flux](https://fluxcd.io/flux/guides/helmreleases/)
* [Setup Flux helm controllers](https://fluxcd.io/flux/use-cases/helm/)

# Setting up Flux
Best is to go to and follow the latest and greatest instructions on fluxcd.io.

Below a condensed version of [Flux getting started guide](https://fluxcd.io/flux/get-started/) which I used

## General preparations
* Prepare a kubernetes cluster (see other post)
* This example expects that a namespace "my-nginx" has been created (even though that could also be handled by flux if taking the example one more step)
```shell
kubectl create namespace my-nginx
```


## GitHub preparations
Depending on where you have your runtime repo, the steps are slightly different. Below is if your repo is in GitHub.

### Create a GitHub personal access token as follows
See [Create github token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

* Go to Settings->Developer settings->Personal access tokens->Fine-grained tokens
* Generate a new token
* Make sure to set as little access as possible including to only give access to the specific reuntime repository you will use
* Note: If you want the repo to be created by flux, you can enable repo-creation at first and then lock it down once repo has been created (this is what I did)
* Needed repository permissions:
** Administration: R/W (initially, then only R)
** Commit statuses: R/W
** Contents: R/W
** Metadata: R
** Pull requests: R/W
Probably it can be locked down more, please try

* Generate and keep the token somewhere safe-ish (like in bitbucket)
* Export the access token as an environment variable at the host from which you will use flux, kubectl etc.
```shell
export GITHUB_TOKEN=<your-token>
```

## Install Flux CLI
[Install Flux CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli)
In linux:
```shell
curl -s https://fluxcd.io/install.sh | sudo bash
```
(alternatively use container: docker.io/fluxcd/flux-cli:<version> / ghcr.io/fluxcd/flux-cli:<version>)

## Bootstrapping Flux
Execute the following to have flux creating a repo in GitHub for your cluster. (Of course you need to use your own username and the name you want on your runtime-repo + the what you want to name your cluster)
```shell
flux bootstrap github \
  --owner=my-github-username \
  --repository=my-repository \
  --path=clusters/my-cluster \
  --personal
```
Now you can also drop the access for the token to no-longer be allowed to create repos by changing Administration from R/W

# Deploy nginx from helmchart using flux
## Define the "wanted state" in runtime git repo
In the repo, create files in the cluster-folder which was created in the bootstrap step above (e.g. my-repository/clusters/my-cluster).

* In this example, I create a nginx service using the bitnami helmchart. It is created in the namespace "my-nginx" with the name my-nginx

The two files to create are:

* One file defining a source for helmcharts
* The source can be a HelmRepository, GitRepository or a Bucket. In this case we point to an external HelmRepository from Bitnami
``` yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: my-nginx
spec:
  interval: 1m0s
  url: https://charts.bitnami.com/bitnami
```
{: file="my-nginx-source.yaml" }

* One file defining the wanted state for your application
* We specify in what namespace it is to be deployed and with what name
* We also specify which is the HelmChart to deploy. Here we specify an exact version, but it is also possible to specify a semver range  e.g. >=13.2.0 <14.0.0.
* In the example we also show that we can set values. Here we use that to instruct nginx to clone down static html from another git-repo at an interval
``` yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: my-nginx
  namespace: my-nginx
spec:
  chart:
    spec:
      chart: nginx
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: bitnami
      version: 13.2.23
  interval: 1m0s
  values:
    cloneStaticSiteFromGit:
      enabled: true
      repository: "https://github.com/fredrikp999/static-html"
      branch: "main"
      interval: 3600
  ```
  {: file="my-nginx-helmrelease.yaml" }

## Make it happen
Now you just have to commit and merge the files to gitlab repo and wait for reconciliation to take place

* Check that the helmchart has been installed
``` shell
helm list -n my-nginx
```
``` shell
kubectl get all -n my-nginx
```
If you also have ingress seup properly (either manually or by the magics of the custom k3s-installation described in other post), you should also be able to access the nginx webpage from the browser. The port and IP would be visible by looking at the service
```
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
service/my-nginx   LoadBalancer   10.43.165.124   192.168.1.80   80:31945/TCP   61m
```
# Things to try
You can now modify the file in git e.g. changing the helm chart version to 13.2.21 to see it changing in the cluster after a short wait

When you have this working, you can move on to the next post which goes into more details e.g. on how to customize the deployment + later on also how to integrate the events with argo etc.

# More References
[Nginx from bitnami](https://artifacthub.io/packages/helm/bitnami/nginx)