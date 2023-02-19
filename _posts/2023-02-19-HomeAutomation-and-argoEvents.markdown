---
layout: post
title:  "Connecting home automation with Argo Events and workflows"
date:   2023-02-18 06:00:10 +0000
categories: homelab gitops iot
tags: homelab gitops flux iot homeautomation
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1072083046347583569/Fredrik999_big_gaps_30f02586-bef7-489b-9e29-448ae9cab653.png
---
How to connect Home Automation on e.g. Homey and/or Home Assistant with Argo workflows using Argo events and MQTT

References:
* ...

# Introduction
JUST A DRAFT FOR NOW, TO BE DETAILED

# High level flow
* Sensor, e.g. temperature/motion/luminance sensor (philips hue / Aquara etc) measure something
* Sensor reports to automation hub (e.g. Philips Hue bridge) at changed value e.g. over zigbee or z-wave (new standards coming up)
* Hub is registered at main automation hub, e.g. Homey or Home Assistant and is by that reporting changes to it
* Main automation hub e.g. Homey has MQTT-client and MQTT-hub application installed
* MQTT-client connect to an MQTT-broker, e.g. mosquito installed in k8s-cluster or elsewhere and can by that subscribe to and publish events on specific topics
* The MQTT-hub ís configured to publish state of selected sensors, switches etc. to MQTT-broker following the homie-standard (defined schema, structures)
* Argo events "eventsource" is setup to subscribe to specific MQTT-topics, e.g. the luminance in a specific room say the office
* The eventsource transforms the MQTT-data to a cloud-event and publish this on the Argo events bus (Jetstream)
* Argo events "sensor" is setup to subscribe to a certain topic on the Argo event bus, potentially with a filter to only trigger when the data (message in the topic) matches some pattern
* The sensor will when there is a match submit an Argo workflow to the k8s-cluster
* As an example, the workflow can create an event with some new topic and message back to the MQTT-hub, e.g. "it is pitch-black in the office"
* The main automation hub (e.g. Homey) can subscribe to this MQTT-message and when matching certain criterias, it can trigger a flow
* As an example, Homey can with the help of an installed application send a message to a channel in Discord "Now the lights are off in the office. Perhaps someone is not working as they should"

The above is of course not the most efficient way to do this, but it show how communication between the different systems can be made using an event-based setup.
* More useful actions would be to do things which cannot be done already in Homey / Home Assistant, e.g. triggering CI/CD-pipelines which perhaps build, test, publish and auto-deploy a new version of a webpage when something happens in the house
* As we can integrate to discord, and as we can invite other bots to that conversation - we can e.g. invite the midjourney-bot so that it can generate AI-pictures for us. By this we could do things like generating an image based on the "status in my house", e.g. based on which lights are on, at what dimlevel, with what colour + the temperature in the rooms, how many people are home, what is in their calenders right now etc. This url to this picture could then probably be picked up and used for updating a webpage showing the "current mood" or similar. Not useful but fun :)
* A benefit is that using Argo Workflows, it is very easy to use any available docker-image and get something quite advanced done quickly compared to having to implement it is javascript in Homey
* Executing the workflow in Argo also means that it is possible to use the full capacity of a k8s-cluster instead of being bound to the minimal memory of Homey. (However, it would be possible to just have a service running in k8s with a REST-API consumed from Homey if that would be easier)

## Argo event files used

EventSource subscribing to luminance values from the office
```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: mqtt-fkont
  namespace: argo
spec:
  mqtt:
    fkont-luminance:
      # mqtt broker url
      url: tcp://192.168.4.133:1883
      # name of the popic
      topic: homie/homey/fkont-sensor/measure-luminance
      # jsonBody specifies that all event body payload coming from this
      # source will be JSON
      jsonBody: true
      # client id
      clientId: "argo-fkont"
      # optional backoff time for connection retries.
      # if not provided, default connection backoff time will be used.
      connectionBackoff:
        # duration in nanoseconds, or strings like "2s". following value is 10 seconds
        duration: 10s
        # how many backoffs
        steps: 5
        # factor to increase on each step.
        # setting factor > 1 makes backoff exponential.
        factor: 2
        jitter: 0.2
```

Argo Sensor triggering when it is dark in the office, executing "Whalesay" which prints the luminance value in the logs
```yaml
# Sensor to trigger a workflow when a luminance value is below a threshold
# Treshold is set in the filters section to 1, which is pitch black
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: mqtt-luminance-sensor
  namespace: argo
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: mqtt
      eventSourceName: mqtt-fkont
      eventName: fkont-luminance
      filters:
        data:
          - path: body
            type: number
            comparator: "<="
            value:
              - "1"
  triggers:
    - template:
        name: luminance-workflow-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: luminance-triggered-
              spec:
                entrypoint: whalesay
                arguments:
                  parameters:
                  - name: message
                    # the value will get overridden by event payload from dependency per parameters section
                    value: hello world
                templates:
                - name: whalesay
                  inputs:
                    parameters:
                    - name: message
                  container:
                    image: docker/whalesay:latest
                    command: [cowsay]
                    args: ["{{inputs.parameters.message}}"]
          parameters:
            - src:
                dependencyName: mqtt
                dataKey: body
              dest: spec.arguments.parameters.0.value
```

Argo Sensor triggering when it is dark in the office, publish a message to another MQTT-topic that it is dark
```yaml
# Sensor to trigger a workflow when a luminance value is below a threshold
# Treshold is set in the filters section to 1, which is pitch black
# Sensor to trigger a workflow to publish a message to an MQTT broker as a proofpoint
# The published message could then be picked up by homey/home assistant and trigger a flow

apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: pitchblack
  namespace: argo
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: mqtt
      eventSourceName: mqtt-fkont
      eventName: fkont-luminance
      filters:
        data:
          - path: body
            type: number
            comparator: "<="
            value:
              - "1"
  triggers:
    - template:
        name: pitchblack-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: pitchblack-triggered-
              spec:
                entrypoint: publish
                templates:
                - name: publish
                  container:
                  # MQTT broker is running at 192.168.1.86:1883
                  # MQTT topic is foobar
                  # MQTT message is "hello world"
                    image: eclipse-mosquitto:2.0.15
                    command: ["mosquitto_pub"]
                    args: ["-h", "192.168.4.133", "-p", "1883", "-t", "office-asleep", "-m", "Yes, it is pitchblack. Better turn on the lights"]
```

Argo Sensor triggering when it is dark in the office, publish a message to another MQTT-topic that it is bright
```yaml
# Sensor to trigger a workflow when a luminance value is below a threshold
# Treshold is set in the filters section to 1, which is pitch black
# Sensor to trigger a workflow to publish a message to an MQTT broker as a proofpoint
# The published message could then be picked up by homey/home assistant and trigger a flow

apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: niceandbright
  namespace: argo
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: mqtt
      eventSourceName: mqtt-fkont
      eventName: fkont-luminance
      filters:
        data:
          - path: body
            type: number
            comparator: ">"
            value:
              - "50"
  triggers:
    - template:
        name: niceandbright-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: niceandbright-triggered-
              spec:
                entrypoint: publish
                templates:
                - name: publish
                  container:
                  # MQTT broker is running at 192.168.1.86:1883
                  # MQTT topic is foobar
                  # MQTT message is "hello world"
                    image: eclipse-mosquitto:2.0.15
                    command: ["mosquitto_pub"]
                    args: ["-h", "192.168.4.133", "-p", "1883", "-t", "office-asleep", "-m", "Nope, nice and bright in here"]
```

