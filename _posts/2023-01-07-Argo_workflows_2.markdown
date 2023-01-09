---
layout: post
title:  "Argo Workflows - working with artifacts, templates etc."
date:   2023-01-07 12:00:10 +0000
categories: homelab kubernetes
tags: homelab kubernetes argo CD
image:
  path: https://media.discordapp.net/attachments/1039418871024721970/1060972302667366560/Fredrik999_a_happy_orange_squid_with_big_rund_eyes._Clean_desig_f35792d0-1991-49bc-9b9f-e7a691a7e4b5.png
---
How to work with input- and output-artifacts. How to work with Templates, secrets and more. 

For longer explanations, see this workshop by argo-team talking about this and more
* [Argo workflows 101 Workshop 22 Sep 2020](https://www.youtube.com/watch?v=XySJb-WmL3Q&t=4634s)

Recommendation is also to checkout some example workflows on github:
* [argoworkflow examples](https://github.com/argoproj/argo-workflows/tree/master/examples)

# Definitions
## Main concepts
* Template - A "Workflow Template", base for creating Workflows (specs)
* Workflow (specification) - The definition of a workflow
* Workflow (live) - The submitted workflow instance being executed


## Concepts within a workflow
* input, output arguments
* input, output artifacts
* template (in a Workflow spec) - a "job/function definition"
* step (in a Workflow spec) - defines one or more "jobs"

# Creating Workflows
## Contents of a Workflow Specification
Within one and the same workflowspecification-file, you would have the following sections
### Start of workflow definition
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: whalesong-  # Name of this Workflow
spec:
  entrypoint: do-whalesong   # Defines "do-whalesongs" as the "main" template
```

### One or more "templates" - the functions which can be used in the workflow
#### Here one template (function) with only one "step".
```yaml
templates:
  - name: whalesay            # Defining the "whalesay" template
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["hello world"]   # This template runs "cowsay" in the "whalesay" image with arguments "hello world"
```
#### And a template with multiple templates + a template executing "steps"
* This is normally what you point to as your "entrypoint" and would be the "main workflow", which executes one or more other templates (functions) in the order defined
* Also notice how parameters are used when specifying
* - template arguments (input-parameters-name:XXX)
* - when calling templates (name:XXX, value:YYY)
* - when using within template (inputs.parameters.XXX)

```yaml
templates:
    - name: do-whalesong  # The "main"-template used as entrypoint
      steps:
      - - name: hello-nisse
          template: whalesay-dynamic
          arguments:
              parameters: [{name: message, value: "hello nisse"}]
      - - name: hello-olle
          template: whalesay-dynamic
          arguments:
              parameters: [{name: message, value: "hello olle"}]
      - - name: hello-world
          template: whalesay
    - name: whalesay            # Defining the "whalesay" template
      container:
        image: docker/whalesay
        command: [cowsay]
        args: ["hello world"]   # This template runs "cowsay" in the "whalesay" image with hard-coded argument "hello world"
    - name: whalesay-dynamic    # Defining the "whalesay-dynamic" template
      inputs:
        parameters:
        - name: message
      container:
        image: docker/whalesay
        command: [cowsay]
        args: ["{{inputs.parameters.message}}"]  # This template runs "cowsay" in the "whalesay" image with arguments as provided by input parameter 
```

# Working with input and output artifacts
...

## Passing artifacts directly from one step to another within the same workflow
It is possible to produce an artifact in one step and then consume it in the next step without having to specify any itermediate storage (like an s3-bucket) to store and retrieve it from
* In the **template** executed by the step **generating** an artifact, specify "outputs:artifacts + name and path"
* In the **template** executed by the step **consuming** an artifact, specify "inputs:artifacts + name and path"
* In the **step** calling the template consuming the artifact, specify "arguments:artifacts + name". Also specify from which step the artifact shall be taken (steps.STEP.outputs.artifacts.ARTIFACTNAME)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifact-passing-
spec:
  entrypoint: artifact-example
  templates:
  - name: artifact-example
    steps:
    - - name: generate-artifact
        template: whalesay
    - - name: consume-artifact
        template: print-message
        arguments:
          artifacts:
          - name: message
            from: "{{steps.generate-artifact.outputs.artifacts.hello-art}}"

  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["sleep 1; cowsay hello world | tee /tmp/hello_world.txt"]
    outputs:
      artifacts:
      - name: hello-art
        path: /tmp/hello_world.txt

  - name: print-message
    inputs:
      artifacts:
      - name: message
        path: /tmp/message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["cat /tmp/message"]
  ```

## Use artifactory to store artifacts
Below example demonstrates the use of artifactory as the store for artifacts.
* For input and/or output artifacts, you specify "artifactory" and the needed url, usernameSecret and passworkSecrets
* This config is used by Argo to download and provide the referenced artifacts to the container executing the step - or to upload the output artifact to

The same way can be used for other repository types. The structure is very similar. See e.g. how to use s3 further down.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifactory-artifact-
spec:
  entrypoint: artifact-example
  templates:
  - name: artifact-example
    steps:
    - - name: generate-artifact
        template: whalesay
    - - name: consume-artifact
        template: print-message
        arguments:
          artifacts:
          - name: message
            from: "{{steps.generate-artifact.outputs.artifacts.hello-art}}"

  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["cowsay hello world | tee /tmp/hello_world.txt"]
    outputs:
      artifacts:
      - name: hello-art
        path: /tmp/hello_world.txt
        artifactory:
          url: http://artifactory:8081/artifactory/generic-local/hello_world.tgz
          usernameSecret:
            name: my-artifactory-credentials
            key: username
          passwordSecret:
            name: my-artifactory-credentials
            key: password

  - name: print-message
    inputs:
      artifacts:
      - name: message
        path: /tmp/message
        artifactory:
          url: http://artifactory:8081/artifactory/generic-local/hello_world.tgz
          usernameSecret:
            name: my-artifactory-credentials
            key: username
          passwordSecret:
            name: my-artifactory-credentials
            key: password
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["cat /tmp/message"]
  ```

## Use s3-bucket to store artifacts
Example for how to retrieve artifacts from s3.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: input-artifact-s3-
spec:
  entrypoint: input-artifact-s3-example
  templates:
  - name: input-artifact-s3-example
    inputs:
      artifacts:
      - name: my-art
        path: /my-artifact
        s3:
          # Use the corresponding endpoint depending on your S3 provider:
          #   AWS: s3.amazonaws.com
          #   GCS: storage.googleapis.com
          #   Minio: my-minio-endpoint.default:9000
          endpoint: s3.amazonaws.com
          bucket: my-bucket-name
          key: path/in/bucket
          # Specify the bucket region. Note that if you want Argo to figure out this automatically,
          # you can set additional statement policy that allows `s3:GetBucketLocation` action.
          # For details, check out: https://argoproj.github.io/argo-workflows/configure-artifact-repository/#configuring-aws-s3
          region: us-west-2
          # accessKeySecret and secretKeySecret are secret selectors.
          # It references the k8s secret named 'my-s3-credentials'.
          # This secret is expected to have have the keys 'accessKey'
          # and 'secretKey', containing the base64 encoded credentials
          # to the bucket.
          accessKeySecret:
            name: my-s3-credentials
            key: accessKey
          secretKeySecret:
            name: my-s3-credentials
            key: secretKey
    container:
      image: debian:latest
      command: [sh, -c]
      args: ["ls -l /my-artifact"]
  ```

# Working with input and output arguments in a workflow

# Working with git-repos in a workflow

# Working with secrets

Secrets are provided to the template from Kubernetes secrets
* The normal way for a workflow to handle secrets is to use kubernetes secrets.
* It could of course also be that the kubernetes secrets are used as keys to access the encrypted secrets in e.g. a vault

```shell
kubectl create secret generic my-artifactory-credentials --from-literal=username=<YOUR-ARTIFACTORY-USERNAME> --from-literal=password=<YOUR-ARTIFACTORY-PASSWORD>
``` 