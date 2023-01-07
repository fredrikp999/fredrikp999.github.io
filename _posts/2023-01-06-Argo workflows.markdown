---
layout: post
title:  "Argo Workflow basics"
date:   2023-01-06 11:00:10 +0000
categories: homelab kubernetes
tags: homelab kubernetes argo CD
image:
  path: https://media.discordapp.net/attachments/1039418871024721970/1060972305653702817/Fredrik999_a_happy_orange_squid_with_big_rund_eyes._Clean_desig_9db13728-f380-4975-aea5-ceafc6e43d69.png
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
Optionally, create rolebinding (not part of official guide)
```shell
kubectl create rolebinding default-admin \
  --clusterrole cluster-admin \
  --namespace argo \
  --serviceaccount=argo:default
```

Optionally, Port-forward the UI
```shell
kubectl -n argo port-forward deployment/argo-server 2746:2746
```

Optionally, check and wait for controller to be up
```shell
kubectl wait deployment workflow-controller \
  --for condition=Available \
  --namespace argo
```

## Install Argo CLI
Version 3.4.4 installed below, check and update link if later version is available
```shell
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.4.4/argo-linux-amd64.gz
gunzip argo-linux-amd64.gz
chmod +x argo-linux-amd64
sudo mv ./argo-linux-amd64 /usr/local/bin/argo
```
Test installation
```shell
argo version
```
# Using Argo Workflows
## Basic workflow examples
[Examples from codefresh](https://codefresh.io/learn/argo-workflows/learn-argo-workflows-with-8-simple-examples/)
### Create argo workflow manifests
Hello world prints some text using whalesay
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-  # Name of this Workflow
spec:
  entrypoint: whalesay        # Defines "whalesay" as the "main" template
  templates:
  - name: whalesay            # Defining the "whalesay" template
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["hello world"]   # This template runs "cowsay" in the "whalesay" image with arguments "hello world"
```
{: file="helloword-workflow.yaml" }

"Random Integer" generates and prints a random number 
```yaml
  - name: gen-random-int
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        i = random.randint(1, 100)
        print(i)
```
{: file="randomInt-workflow.yaml" }

### Execute the workflows
Execute "hello world"-workflow
```shell
argo submit helloworld-workflow.yaml
```
Expect something like
```
Name:                hello-world-hj7dl
Namespace:           default
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Pending
Created:             Sat Jan 07 07:25:22 +0000 (now)
Progress:
```
Check status with
```shell
argo list
```
Expect something like
```
NAME                STATUS      AGE   DURATION   PRIORITY   MESSAGE
hello-world-hj7dl   Succeeded   1m    10s        0
```
Inspect logs to see if something was executed
```shell
argo logs hello-world-hj7dl
```
Expect something like
```
hello-world-hj7dl:  _____________
hello-world-hj7dl: < hello world >
hello-world-hj7dl:  -------------
hello-world-hj7dl:     \
hello-world-hj7dl:      \
hello-world-hj7dl:       \
hello-world-hj7dl:                     ##        .
hello-world-hj7dl:               ## ## ##       ==
hello-world-hj7dl:            ## ## ## ##      ===
hello-world-hj7dl:        /""""""""""""""""___/ ===
hello-world-hj7dl:   ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
hello-world-hj7dl:        \______ o          __/
hello-world-hj7dl:         \    \        __/
hello-world-hj7dl:           \____\______/
hello-world-hj7dl: time="2023-01-07T07:25:27.568Z" level=info msg="sub-process exited" argo=true error="<nil>"
```

### Using kubectl instead of Argo CLI
It is possible to create and interact with argo workflows and using kubectl directly:
```shell
kubectl create -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml
kubectl get wf -n argo
kubectl get wf hello-world-xxx -n argo
kubectl get po -n argo --selector=workflows.argoproj.io/workflow=hello-world-xxx
kubectl logs hello-world-yyy -c main -n argo
```
However, using Argo CLI provides a number of features you do not get with kubectl

## More advanced workflow examples
[Example workflows from pipekit.io](https://pipekit.io/blog/top-10-argo-workflows-examples)
### Git-clone

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: input-artifact-git-
spec:
  entrypoint: git-clone
  templates:
  - name: git-clone
    inputs:
      artifacts:
      - name: argo-source
        path: /src
        git:
          repo: https://github.com/argoproj/argo-workflows.git
          revision: "v2.1.1"
          usernameSecret:
            name: github-creds
            key: username
          passwordSecret:
            name: github-creds
            key: password
    container:
      image: golang:1.10
      command: [sh, -c]
      args: ["git status && ls && cat VERSION"]
      workingDir: /src
```
{: file="git-clone-workflow.yaml" }

### Directed Acyclic Graph (DAG)
Using DAG, you can use dependencies between tasks in a nice way. [DAG definition](https://www.techopedia.com/definition/5739/directed-acyclic-graph-dag)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-target-
spec:
  entrypoint: dag-target
  arguments:
    parameters:
    - name: target
      value: E

  templates:
  - name: dag-target
    dag:
      target: "{{workflow.parameters.target}}"

      tasks:
      - name: A
        template: echo
        arguments:
          parameters: [{name: message, value: A}]
      - name: B
        depends: "A"
        template: echo
        arguments:
          parameters: [{name: message, value: B}]
      - name: C
        depends: "A"
        template: echo
        arguments:
          parameters: [{name: message, value: C}]
      - name: D
        depends: "B && C"
        template: echo
        arguments:
          parameters: [{name: message, value: D}]
      - name: E
        depends: "C"
        template: echo
        arguments:
          parameters: [{name: message, value: E}]

  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"] 
```
{: file="dag-workflow.yaml" }

Checking the logs of the dag workflow with:
```shell
argo logs dag-target-zdv48
```

Gives something like:
```
dag-target-zdv48-echo-2662937463: A
dag-target-zdv48-echo-2662937463: time="2023-01-07T07:44:34.105Z" level=info msg="sub-process exited" argo=true error="<nil>"
dag-target-zdv48-echo-2696492701: C
dag-target-zdv48-echo-2696492701: time="2023-01-07T07:44:40.333Z" level=info msg="sub-process exited" argo=true error="<nil>"
dag-target-zdv48-echo-2595826987: E
dag-target-zdv48-echo-2595826987: time="2023-01-07T07:44:50.329Z" level=info msg="sub-process exited" argo=true error="<nil>"
```
...