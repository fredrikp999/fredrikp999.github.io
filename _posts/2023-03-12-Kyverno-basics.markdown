---
layout: post
title:  "Policy as code with Kyverno, the basics"
date:   2023-03-12 15:00:10 +0000
categories: homelab security
tags: homelab security kyverno policy
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1083715852727234600/Fredrik999_a_secrets_vault_27c083fd-e6d3-4930-b9c2-034e3416cb95.png
---
Applying the principle of "Policy as code" is an efficient way to add one layer of security to your kubernetes cluster. There are mainly two mature tools in CNCF, OPA Gatekeeper and Kyverno. Kyverno is the more modern and the one I will explore here.

References:
* [Kyverno.io](https://kyverno.io/)

# Introduction
Kyverno runs as a dynamic admission controller in a Kubernetes cluster.
* Kyverno receives validating and mutating admission webhook HTTP callbacks from the kube-apiserver and applies matching policies to return results that enforce admission policies or reject requests.
* Kyverno policies can match resources using the resource kind, name, label selectors, and much more.

# Installation
## Helm
Either install using helm, follow:
* [Kyverno helmchart](https://artifacthub.io/packages/helm/kyverno/kyverno)

## GitOps, Flux
Or if you want to handle using gitOps in Flux, create one repo and helmrelease file

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: kyverno
  namespace: system
spec:
  interval: 1m0s
  url: https://kyverno.github.io/kyverno/
```
{: file="./my-cluster/namespaces/system/apps/kyverno/kyverno-repo.yaml" }

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: kyverno
  namespace: system
spec:
  chart:
    spec:
      chart: kyverno
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: kyverno
      version: 2.7.1
  interval: 1m0s
```
{: file="./my-cluster/namespaces/system/apps/kyverno/kyverno-helmrelease.yaml" }

## GitOps, ArgoCD
Due to the way ArgoCD handles installation of Helm charts, there are some tweaks which needs to be done to get it working without becoming a mess in the cluster. Please see Kyverno documentation for how to do it properly.

# Policy examples
See: [Kyverno example policies](https://kyverno.io/policies/)
A couple of examples taken from above

## Disallow Latest tag
The ':latest' tag is mutable and can lead to unexpected errors if the image changes. A best practice is to use an immutable tag that maps to a specific version of an application Pod. This policy validates that the image specifies a tag and that it is not called `latest`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/minversion: 1.6.0
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      The ':latest' tag is mutable and can lead to unexpected errors if the
      image changes. A best practice is to use an immutable tag that maps to
      a specific version of an application Pod. This policy validates that the image
      specifies a tag and that it is not called `latest`.      
spec:
# "audit" will only log the violation
# validationFailureAction: audit
# "enforce" will block the action
  validationFailureAction: enforce

  background: true
  rules:
  - name: require-image-tag
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "An image tag is required."
      pattern:
        spec:
          containers:
          - image: "*:*"
  - name: validate-image-tag
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Using a mutable image tag e.g. 'latest' is not allowed."
      pattern:
        spec:
          containers:
          - image: "!*:latest"
```
{: file="disallow-latest-tag-cluster-policy.yaml" }

After having create the policy (see further down), test the policy with
```shell
kubectl create deployment nginx --image=nginx:latest
```

This would give something like
```
error: failed to create deployment: admission webhook "validate.kyverno.svc-fail" denied the request:

policy Deployment/default/nginx for resource violation:

disallow-latest-tag:
  autogen-validate-image-tag: 'validation error: Using a mutable image tag e.g. ''latest''
    is not allowed. rule autogen-validate-image-tag failed at path /spec/template/spec/containers/0/image/'
```


## Require Signed Tekton Pipeline
Require that the pipeline is signed
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: signed-pipeline-bundle
  annotations:
    policies.kyverno.io/title: Require Signed Tekton Pipeline
    policies.kyverno.io/category: Tekton
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: PipelineRun
    kyverno.io/kyverno-version: 1.7.2
    policies.kyverno.io/minversion: 1.7.0
    kyverno.io/kubernetes-version: "1.23"
    policies.kyverno.io/description: >- 
      A signed bundle is required
spec:
  validationFailureAction: enforce
  webhookTimeoutSeconds: 30
  rules:
  - name: check-signature
    match:
      resources:
        kinds:
        - PipelineRun
    imageExtractors:
      PipelineRun:
        - name: "pipelineruns"
          path: /spec/pipelineRef
          value: "bundle"
          key: "name"
    verifyImages:
    - imageReferences:
      - "*"
      attestors:
      - entries:
        - keys: 
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEahmSvGFmxMJABilV1usgsw6ImcQ/
              gDaxw57Sq+uNGHW8Q3zUSx46PuRqdTI+4qE3Ng2oFZgLMpFN/qMrP0MQQg==
              -----END PUBLIC KEY-----              
```
{: file="require-signed-pipeline-cpolicy.yaml" }

# Working with the policies
## In-cluster enforcement
### Creating policies
* The recommended way is to use GitOps to handle the policies. Just commit the policy.yaml to the by Flux monitored runtime-repo and Flux handles the rest.
* Alternatively, you can apply the policy "manually"
```shell
kubectl apply -f disallow-latest-tag-cluster-policy.yaml
```

### Check policies
Get all cluster-wide policies
```shell
kubectl get cpol
```

Get all policies in a namespace
```shell
kubectl get pol -n my-namespace
```

Get details of a policy
```shell
kubectl get cpol/disallow-latest-tag -o yaml
```

## Report policy violations
List policy violations
```shell
kubectl get polr
```

This can give something like
```
NAME                       PASS   FAIL   WARN   ERROR   SKIP   AGE
cpol-disallow-latest-tag   4      4      0      0       0      6h21m
```

For more details
```shell
kubectl get polr/cpol-disallow-latest-tag -o yaml 
```

This can give something like
```
apiVersion: wgpolicyk8s.io/v1alpha2
kind: PolicyReport
metadata:
  creationTimestamp: "2023-03-12T09:57:14Z"
  generation: 14
  labels:
    app.kubernetes.io/managed-by: kyverno
    cpol.kyverno.io/disallow-latest-tag: "14787030"
  name: cpol-disallow-latest-tag
  namespace: default
  resourceVersion: "14870967"
  uid: 2321fe98-fddc-4b5d-9f01-1faaf9fcc9ff
results:
- category: Best Practices
  message: 'validation error: Using a mutable image tag e.g. ''latest'' is not allowed.
    rule autogen-validate-image-tag failed at path /spec/template/spec/containers/0/image/'
  policy: disallow-latest-tag
  resources:
  - apiVersion: apps/v1
    kind: Deployment
    name: theapp-infoapp
    namespace: webservers
    uid: 5ac91218-fb3d-49ec-9768-b6e55b33c3b6
  result: fail
  rule: autogen-validate-image-tag
  scored: true
  severity: medium
  source: kyverno
  timestamp:
    nanos: 0
    seconds: 1678636644
```

## Enforcement outside of k8s
It is possible to validate against the kyverno policies also without doing it in the k8s-cluster.
* This can be very useful e.g. in CI or prior to accepting files to a GitOps runtime-repo.
* The CLI can also be used to apply policies to a running cluster, to see pre-check how a specific policy would hit would it be added
* To do this, the Kyverno CLI is used
[Kyverno CLI](https://kyverno.io/docs/kyverno-cli/)

After the CLI has been installed (see above reference), it is possible to e.g.
* Apply a policy to a resource
```shell
kyverno apply /path/to/policy.yaml --resource /path/to/resource.yaml
```

* Apply a policy to all matching resources in a cluster based on the current kubectl context
```shell
kyverno apply /path/to/policy.yaml --cluster
```

See more examples in the official documentation
