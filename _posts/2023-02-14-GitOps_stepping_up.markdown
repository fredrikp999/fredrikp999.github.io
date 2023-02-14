---
layout: post
title:  "GitOps, stepping up"
date:   2023-02-14 06:00:10 +0000
categories: homelab gitops
tags: homelab gitops flux
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1074708601979031592/Fredrik999_a_futuristic_pipeline_in_the_style_of_Simon_stalenha_6bcc9c3c-c4a3-4ec6-9534-4cfb23130711.png
---
More advanced use of Flux / GitOps. Troubleshooting tips and more

References:
* [Flux troubleshooting cheatsheet](https://fluxcd.io/flux/cheatsheets/troubleshooting/)

# Flux repository strategies
There are multiple options for how to setup your runtime repos for use with Flux. Which one work best depends on your use case, how many groups of people will work on the repos.

* See some thoughts on this here:
* And some more here: 

## Strategy for my homelab
In my homelab, it will only be one person making changes - but there will also be system accounts which make changes. So even thou I should (hopefully) be able to trust myself, I do trust the system account used from automation a bit less...
* One "Core repo" for "The core installation", especially important that thi includes the applications which enforce policies on what is allowed to deploy in the cluster, e.g. OPA Gatekeeper or Kyvernio (see other post)
* One repo for "Other". This is e.g. user added contents like Argo sensors, Argo event sources and related config maps but also applications and other resources which are created more temporary. This is the repo which I give access to from selected automation services e.g. the service accounts executing from argo workflows

Example of a setup below. The example is still in one repo and simulating the two repos like this:
* "namespaces"-folder simulates the "Core repo"
* "user-owned"-folder simulates the "Other repo"
Eventually I will separate these to two real repos, but during the early tests it is easier in one and the same repo.
Also notice the folder "temporary-removed-manifests". This is a folder which is not monitored by Flux why it is convenient to temporary move things there when you want to keep it for reference but not deployed

```
.
|-- clusters
|   `-- my-cluster
|       |-- flux-system
|       |   |-- gotk-components.yaml
|       |   |-- gotk-sync.yaml
|       |   `-- kustomization.yaml
|       |-- namespaces
|       |   |-- argo
|       |   |   |-- apps
|       |   |   |   |-- argo-events
|       |   |   |   |   `-- argo-events-helmrelease.yaml
|       |   |   |   `-- argo-workflows
|       |   |   |       `-- argo-workflows-helmrelease.yaml
|       |   |   |-- argo-namespace.yaml
|       |   |   |-- argo-repo.yaml
|       |   |   |-- configmaps
|       |   |   |   `-- artifact-references
|       |   |   |       `-- artifact-repositories.yaml
|       |   |   `-- rolebindings
|       |   |       |-- role-deployments-operator.yaml
|       |   |       |-- rolebinding-for-operate-workflow-sa.yaml
|       |   |       `-- serviceaccount-operate-workflow-sa.yaml
|       |   |-- iot
|       |   |   |-- apps
|       |   |   |   |-- homeassistant
|       |   |   |   |   |-- homeassistant-helmrelease.yaml
|       |   |   |   |   `-- homeassistant-repo.yaml
|       |   |   |   `-- mosquitto
|       |   |   |       |-- mosquitto-helmrelease.yaml
|       |   |   |       `-- mosquitto-repo.yaml
|       |   |   `-- iot-namespace.yaml
|       |   |-- monitoring
|       |   |   |-- apps
|       |   |   |   |-- grafana
|       |   |   |   |   |-- grafana-helmrelease.yaml
|       |   |   |   |   `-- grafana-repo.yaml
|       |   |   |   `-- prometheus
|       |   |   |       |-- prometheus-helmrelease.yaml
|       |   |   |       `-- prometheus-repo.yaml
|       |   |   `-- monitoring-namespace.yaml
|       |   |-- system
|       |   |   |-- apps
|       |   |   |   |-- keycloak
|       |   |   |   |   |-- bitnami-repo.yaml
|       |   |   |   |   `-- keycloak-helmrelease.yaml
|       |   |   |   `-- vault
|       |   |   |       |-- hashicorp-repo.yaml
|       |   |   |       `-- vault-helmrelease.yaml
|       |   |   `-- system-namespace.yaml
|       |   `-- webservers
|       |       |-- apps
|       |       |   `-- nginx-webserver
|       |       |       |-- nginx-webserver-helmrelease.yaml
|       |       |       `-- nginx-webserver-repo.yaml
|       |       `-- webservers-namespace.yaml
|       `-- user-owned
|           |-- argo-eventsources
|           |   |-- argo-eventsource-fkont-luminance.yaml
|           |   |-- argo-eventsource-fkont-motion.yaml
|           |   |-- argo-eventsource-foobar.yaml
|           |   `-- argo-eventsource-webhook.yaml
|           `-- argo-sensors
|               |-- argo-sensor-fkont-luminance.yaml
|               |-- argo-sensor-foobar.yaml
|               `-- argo-sensor-webhook.yaml
`-- temporary-removed-manifests
    |-- example-configmap.yaml
    |-- nats
    |   |-- nats-helmrelease.yaml
    |   `-- nats-repo.yaml
    |-- old-role.yaml
    `-- role-bindings
```

# Flux troubleshooting
## Check reconciliation status:
* See overall sync status
```shell
flux logs
```
Shows logs like:
```
2023-02-14T05:55:36.637Z info GitRepository/flux-system.flux-system - stored artifact for commit 'version 13.2.23'
2023-02-14T05:56:36.659Z info GitRepository/flux-system.flux-system - garbage collected 1 artifacts
2023-02-14T05:56:37.883Z info GitRepository/flux-system.flux-system - no changes since last reconcilation: observed revision 'main/1e446a271364cd15c9590d34574d9cfe6167f919'
```
* See changes done in a specific namespace (e.g. ns=webservers)
```shell
flux logs -n webservers
```
Shows logs like:
```
2023-02-14T05:55:03.333Z info HelmChart/webservers-webserver.webservers - artifact up-to-date with remote revision: '13.2.21'
2023-02-14T05:55:25.238Z info HelmRepository/bitnami.webservers - artifact up-to-date with remote revision: '722e4f549fa3ab42018481b74ed2993014dec0467bb2c7b901adac72f99c5279'
2023-02-14T05:55:25.239Z info HelmRepository/bitnami.webservers - stored fetched index of size 3.943MB from 'https://charts.bitnami.com/bitnami'
2023-02-14T05:55:38.317Z info HelmChart/webservers-webserver.webservers - pulled 'nginx' chart with version '13.2.23'
2023-02-14T05:56:03.352Z info HelmChart/webservers-webserver.webservers - garbage collected 1 artifacts
2023-02-14T05:56:04.042Z info HelmChart/webservers-webserver.webservers - artifact up-to-date with remote revision: '13.2.23'
2023-02-14T05:56:25.715Z info HelmRepository/bitnami.webservers - artifact up-to-date with remote revision: '722e4f549fa3ab42018481b74ed2993014dec0467bb2c7b901adac72f99c5279'
2023-02-14T05:56:25.716Z info HelmRepository/bitnami.webservers - stored fetched index of size 3.943MB from 'https://charts.bitnami.com/bitnami'
2023-02-14T05:57:04.675Z info HelmChart/webservers-webserver.webservers - artifact up-to-date with remote revision: '13.2.23'
2023-02-14T05:57:26.202Z info HelmRepository/bitnami.webservers - artifact up-to-date with remote revision: '722e4f549fa3ab42018481b74ed2993014dec0467bb2c7b901adac72f99c5279'
```

## Get basic information when something seems not to sync as expectes
* Show all Flux objects that are not ready
```shell
flux get all -A --status-selector ready=false
```
* Show flux warning events
```shell
kubectl get events -n flux-system --field-selector type=Warning
```

* Show deployed artifacts. (Note: Does only show resources which is handled specifically by Flux apis but not e.g. vanilla config maps, argo resources etc.)
```shell
flux get sources all -A
```
Example output:

```
NAMESPACE       NAME                            REVISION        SUSPENDED       READY   MESSAGE

flux-system     gitrepository/flux-system       main/1e446a2    False           True    stored artifact for revision 'main/1e446a271364cd15c9590d34574d9cfe6167f919'

NAMESPACE       NAME                                    REVISION                                                                SUSPENDED READY   MESSAGE
system          helmrepository/hashicorp                ca64107ca3ff6e89fdc2a9565bf8542dd90febf3a2b8facb9c7513bbca474b47        False     True    stored artifact: revision 'ca64107ca3ff6e89fdc2a9565bf8542dd90febf3a2b8facb9c7513bbca474b47'
monitoring      helmrepository/prometheus-community     52ffce03535284289e4ac86dbfe970b56d7c80156b81b8607fee29f30fe85056        False     True    stored artifact: revision '52ffce03535284289e4ac86dbfe970b56d7c80156b81b8607fee29f30fe85056'
iot             helmrepository/mosquitto                925e7778f3686d834dc9972b9e9682ee4c0802e23b0b1eecd5d374116d448a6f        False     True    stored artifact: revision '925e7778f3686d834dc9972b9e9682ee4c0802e23b0b1eecd5d374116d448a6f'
iot             helmrepository/homeassistant            7e1c9316975448f9a73b69e0d36c505fc99b414b60ae0e744017821aa9077544        False     True    stored artifact: revision '7e1c9316975448f9a73b69e0d36c505fc99b414b60ae0e744017821aa9077544'
argo            helmrepository/argo                     e3034dacf78ca083a11b6a8a087fadcaaaf9e9424e678ef27692eb6ac53dec29        False     True    stored artifact: revision 'e3034dacf78ca083a11b6a8a087fadcaaaf9e9424e678ef27692eb6ac53dec29'
monitoring      helmrepository/grafana                  a7742f6432d0894d6131ab32043e044ed4f6c8e03a07f80cd221b70148dec349        False     True    stored artifact: revision 'a7742f6432d0894d6131ab32043e044ed4f6c8e03a07f80cd221b70148dec349'
webservers      helmrepository/bitnami                  fa4d12da2f8f1e3fdbd478c003939b8419d887408d735727c93223ffea2a6af0        False     True    stored artifact: revision 'fa4d12da2f8f1e3fdbd478c003939b8419d887408d735727c93223ffea2a6af0'
system          helmrepository/bitnami                  fa4d12da2f8f1e3fdbd478c003939b8419d887408d735727c93223ffea2a6af0        False     True    stored artifact: revision 'fa4d12da2f8f1e3fdbd478c003939b8419d887408d735727c93223ffea2a6af0'

NAMESPACE       NAME                            REVISION        SUSPENDED       READY   MESSAGE

system          helmchart/system-vault          0.23.0          False           True    pulled 'vault' chart with version '0.23.0'

monitoring      helmchart/monitoring-prometheus 19.3.3          False           True    pulled 'prometheus' chart with version '19.3.3'
iot             helmchart/iot-homeassistant     0.1.4           False           True    pulled 'homeassistant' chart with version '0.1.4'
iot             helmchart/iot-mosquitto         0.1.0           False           True    pulled 'mosquitto' chart with version '0.1.0'
argo            helmchart/argo-argo-events      2.1.1           False           True    pulled 'argo-events' chart with version '2.1.1'
argo            helmchart/argo-argo-workflows   0.22.11         False           True    pulled 'argo-workflows' chart with version '0.22.11'
monitoring      helmchart/monitoring-grafana    6.50.7          False           True    pulled 'grafana' chart with version '6.50.7'
webservers      helmchart/webservers-webserver  13.2.23         False           True    pulled 'nginx' chart with version '13.2.23'
system          helmchart/system-keycloak       13.0.4          False           True    pulled 'keycloak' chart with version '13.0.4'
```