---
layout: post
title:  "Using Hashicorp Packer to create Proxmox VM templates"
date:   2023-01-22 09:00:10 +0000
categories: homelab IaC
tags: homelab IaC packer proxmox
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1060871516931235930/Fredrik999_a_big_old_wooden_ship_steering_wheel._focus_on_the_s_5797a714-a166-4809-978a-55558182f9b7.png
---
Packer is an open source tool that lets you create identical machine images for multiple platforms from a single source template. This post describes how to create image template for use in Proxmox VE

References:
* [Packer Proxmox builder plugin](https://developer.hashicorp.com/packer/plugins/builders/proxmox/clone)
* [Packer template boilerplates from ChristianLempa](https://github.com/christianlempa/boilerplates/tree/main/packer/proxmox)

# Installing packer
See official [Installation instructions](https://developer.hashicorp.com/packer/downloads)

For Ubuntu (at time of writing this post)
```shell
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install packer
```
# Preparations in Proxmox
* Optionally create a new user with least privilages needed
** Datacenter -> Users
** Assign to correct group with needed roles

* Create an API token and secret in Proxmox
** Datacenter -> API Tokens
** Add an API token for your user (root or other)
** IMPORTANT: Uncheck "Privilage Separation"!

* Copy and paste API token ID and API token secret to credentials file
** e.g. "credentials.pkt.hcl"
** Make sure to NOT include this file in files tracked in git!
** (add the filename to .gitignore)

# Decide on iso image to use
* Use a cloud-image if available, e.g. from [ubuntu releases](https://releases.ubuntu.com/). Either download and store e.g. in ./iso/... or reference the http-location. If refering to http-location, packer will cache the download so you do not have to download again
* Configure the location of the iso-image + it's checksum in pkr.hcl.file under "iso_url" and "iso_checksum"
* Also configure the name you want for the VM etc. in the same file

# Configure credentials
* In [something].pkr.hcl-file, configure "ssh-username" as the user which will be created and used during VM template creation
* In user-data-file, add the same user-name
* In user-data-file, add the public key from the host executing packer (~/.ssh/id_rsa.pub)

# Execute packer
```shell
packer build -var-file="credentials.pkr.hcl" ./ubuntu-server-focal-docker.pkr.hcl
```
You should now have a new VM template available in Proxmox

# Create a VM - by cloning the new template
## From Proxmox GUI
If you prefer creating VM from the Proxmox GUI:
* Right click on the VM template and select clone
* Choose "full clone" (so that new VM can exist even if template is removed)
* Choose an ID and a name + click "clone"
* Wait for new VM to be created
* On the new VM, click "cloud-init" and define user and password
* Click "regenerate image"
* Start VM

## Using Terraform
If you want to fully automate also the creation of VMs from the template, see other post on how to do this

# What is going on?
* Packer is uploading and starting up the cloud-init image in Proxmox
* During installation process, packer selects "autoinstall" and points back to itself
* Packer starts a server on a port as configured and provides input to the installation from that server according to user-data file
* user-data file instructs e.g. what locales to use, what extra packages to install etc.
* NOTE: I had problems installing qemu-agent, which I disabled in the example until solved the problem


