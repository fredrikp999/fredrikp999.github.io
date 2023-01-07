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

# Working with input and output artifacts in a workflow
Each 

# Working with input and output arguments in a workflow

# Working with git-repos in a workflow