---
layout: post
title:  "minIO - get started"
date:   2023-01-15 07:00:10 +0000
categories: homelab storage
tags: homelab storage minio s3
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1064091017592184832/Fredrik999_a_flamingo_standing_on_one_leg_264919a5-9cd3-47fd-ad75-b6676e645607.png
---
Short introduction for installing, configuring and using minIO and minIO client (mc).

# Installation and setup of minIO
Install in k8s or simply using docker. You can even install it using a default app template in portainer. 

(Add refs)

## minIO configuration
Two ports to expose at some external port
- Port for WebUI (port 9000 internally)
- Port for API to be used e.g. by mc or argo workflows (port 9001 internally)

## minIO user and service account setup
* Create users (Identlty->Users)
* Create service accounts for the user (Identity->Users->ServiceAccounts)
* Create Access Key for the user

The access key will be created with
* access-Key
* secrets-key

Take note of these and use them in your clients (mc, k8s secrets for argo etc.)

## minIO bucket setup
* Create a bucket in minIO (Buckets->Create Bucket)
* Configure access, policy for user (Access->Users)

# Install and configure minIO client
See [minIO client quick start](https://min.io/docs/minio/linux/reference/minio-mc.html)

In short, to install on linux:
```shell
curl https://dl.min.io/client/mc/release/linux-amd64/mc \
  --create-dirs \
  -o $HOME/minio-binaries/mc

chmod +x $HOME/minio-binaries/mc
export PATH=$PATH:$HOME/minio-binaries/
```

Set up alias for minIO service
```shell
bash +o history
mc alias set ALIAS HOSTNAME ACCESS_KEY SECRET_KEY
bash -o history
```
example:
```shell
mc alias set myminio https://minioserver.example.net:9001 ACCESS_KEY SECRET KEY
```
Note to use the API-port where you exposed minIO, not the same as the WebUI-port

Now you can access minio using the minio client
Some examples
```shell
mc cp myartifact.txt myminio/frippe # Copy file to root of frippe-bucket
mc ls myminio/frippe #ls of buckel frippe
```
See [minio mc reference](https://min.io/docs/minio/linux/reference/minio-mc.html) for available commands (quite a lot)

# Use minIO from Argo workflows
See other posts for details on this, but the short description is
* Create a kubernetes secret with the md5-encoded access-key and secrets-key. Note that how the keys are named are a bit different from the way minIO name them (secretKey instead of secrets-key)
* Reference the kubernetes secrets from the workflow when defining input or output artifacts
* Even better is to create and submit a template where the artifacts are defined centrally instead of defining the details in every workflow