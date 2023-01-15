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
{: file="multistep-workflow.yaml" }

# Working with input and output artifacts
Argo workflows can work with a number of different type of repositories
* Argo handles the communication towards them and provide a certain artifact / folder / bucket to the container executing the workflow.
* The artifacts is assigned a name and is provided to the container at a defined path
* Same goes in the other direction for output artifacts which are also defined by a name and a path + type
* Many different types are supported e.g. s3, http, artifactory, gcp artifact storage, aws, hdfs, git-repo etc.
* Reference to artifacts can be provided in a workflow, passing an artifact generated in one step to another step
* It is also possible to pass artifacts directly between steps

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
  {: file="artifactory-artifacts.yaml" }

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
  {: file="s3-artifacts.yaml" }

## Use HTTP URL to store artifacts
  Using artifact from HTTP URL
  ```yaml
  templates:
  - name: http-artifact-example
    inputs:
      artifacts:
      - name: kubectl
        path: /bin/kubectl
        mode: 0755
        http:
          url: https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl
  ```
  {: file="HTTP-URL-artifacts.yaml" }

## Use GCP artifact storage to store artifacts
  ```yaml
  templates:
    - name: input-artifact-gcs-example
      inputs:
        artifacts:
          - name: my-art
            path: /my-artifact
            gcs:
              bucket: my-bucket-name
              # key could be either a file or a directory.
              key: path/in/bucket
              # serviceAccountKeySecret is a secret selector.
              # It references the k8s secret named 'my-gcs-credentials'.
              # This secret is expected to have have the key 'serviceAccountKey',
              # containing the base64 encoded Google Cloud Service Account Key (json)
              # to the bucket.
              #
              # If it's running on GKE, and Workload Identity is used,
              # serviceAccountKeySecret is not needed.
              serviceAccountKeySecret:
                name: my-gcs-credentials
                key: serviceAccountKey
  ```
  {: file="gcp-artifacts.yaml" }

## Use a git-repo as input artifacts in a workflow
This example shows how to clone contents from a git-repo. Argo will provide it to the executing container on the path specified

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
          # For private repositories, create a k8s secret containing the git credentials and
          # reference the secret keys in the secret selectors: usernameSecret, passwordSecret,
          # or sshPrivateKeySecret.
          # NOTE: when authenticating via sshPrivateKeySecret, the repo URL should supplied in its
          # SSH format (e.g. git@github.com:argoproj/argo-workflows.git). Similarly, when authenticating via
          # basic auth, the URL should be in its HTTP form (e.g. https://github.com/argoproj/argo-workflows.git)
          # usernameSecret:
          #   name: github-creds
          #   key: username
          # passwordSecret:
          #   name: github-creds
          #   key: password
          # sshPrivateKeySecret:
          #   name: github-creds
          #   key: ssh-private-key
          # 
          # insecureIgnoreHostKey disables SSH strict host key checking during the git clone
          # NOTE: this is unnecessary for the well-known public SSH keys from the major git
          # providers (github, bitbucket, gitlab, azure) as these keys are already baked into
          # the executor image which performs the clone.
          # insecureIgnoreHostKey: true
          #
          # Shallow clones/fetches can be performed by providing a `depth`.
          # depth: 1
          #
          # Additional ref specs to fetch down prior to checkout can be
          # provided with `fetch`. This may be necessary if `revision` is a
          # non-branch/-tag ref and thus not covered by git's default fetch.
          # See https://git-scm.com/book/en/v2/Git-Internals-The-Refspec for
          # the refspec format.
          # fetch: refs/meta/*
          # fetch: refs/changes/*
          #
          # Single branch mode can be specified by providing a `singleBranch` and `branch` This mode 
          # is faster than passing in a revision, as it will only fetch the references to the given branch.
          # singleBranch: true
          # branch: my-branch
    container:
      image: golang:1.10
      command: [sh, -c]
      args: ["git status && ls && cat VERSION"]
      workingDir: /src
```
  {: file="git-workflow.yaml" }

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
  {: file="direct-artifact-passing.yaml" }

# Working with input and output arguments in a workflow
...


# Working with secrets

Secrets are provided to the template from Kubernetes secrets
* Secrets should be stored externally from the workflow as kubernetes secrets, and accessed using normal kubernetes facilities, such as volume mounting the secret, or as an environment variable.
* The kubernetes secrets could also be used as keys to access the encrypted secrets in e.g. a vault as one extra level of protection


This example demonstrates how to reference kubernetes secrets in a workflow, both as volume mounting and as environment variable.

 To run this example, first create the secret by running:
 ```shell
 kubectl create secret generic my-secret --from-literal=mypassword=S00perS3cretPa55word
 ``` 

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: secrets-
spec:
  entrypoint: print-secret
  # To use a secret as files, it is exactly the same as mounting secrets as in
  # a pod spec. First add an volume entry in the spec.volumes[]. Name the volume
  # anything. spec.volumes[].secret.secretName should match the name of the k8s
  # secret, which was created using kubectl. In any container template spec, add
  # a mount to volumeMounts referencing the volume name and a mount path.
  volumes:
  - name: my-secret-vol
    secret:
      secretName: my-secret
  templates:
  - name: print-secret
    container:
      image: alpine:3.7
      command: [sh, -c]
      args: ['
        echo "secret from env: $MYSECRETPASSWORD";
        echo "secret from file: `cat /secret/mountpath/mypassword`"
      ']
      # To use a secret as an environment variable, use the valueFrom with a
      # secretKeyRef. valueFrom.secretKeyRef.name should match the name of the
      # k8s secret, which was created using kubectl. valueFrom.secretKeyRef.key
      # is the key you want to use as the value of the environment variable.
      env:
      - name: MYSECRETPASSWORD
        valueFrom:
          secretKeyRef:
            name: my-secret
            key: mypassword
      volumeMounts:
      - name: my-secret-vol
        mountPath: "/secret/mountpath"
  ```
  {: file="secrets.yaml" }

# In-line code in workflows
Script templates provide a way to run arbitrary snippets of code in any language, to produce a output "result" via the standard out of the template.
* Results can then be referenced using the variable, {{steps.stepname.outputs.result}}, and used as parameter to other templates, and in 'when', and 'withParam' clauses.
* This example demonstrates the use of a python script to generate a random number which is printed in the next step.
* It is possible to write in-line code for python, bash and javascript
## Python
Below an example of using an in-line python script

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: scripts-python-
spec:
  entrypoint: python-script-example
  templates:
  - name: python-script-example
    steps:
    - - name: generate
        template: gen-random-int
    - - name: print
        template: print-message
        arguments:
          parameters:
          - name: message
            value: "{{steps.generate.outputs.result}}"

  - name: gen-random-int
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        i = random.randint(1, 100)
        print(i)
  - name: print-message
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo result was: {{inputs.parameters.message}}"]
  ```
  {: file="python-workflow.yaml" }