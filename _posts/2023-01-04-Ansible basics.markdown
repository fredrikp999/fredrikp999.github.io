---
layout: post
title:  "Ansible basics"
date:   2023-01-04 07:15:00 +0000
categories: ansible homelab
---

# Ansible introduction
# Ansible installation
# Ansible script development
## Ansible playbooks
## Ansible roles
Playbooks can contain all the actions needed, or they can refer to one or more roles

---
***Example of playbook "install-heimdall.yaml"***
which use a local role (role created by me)
```yaml
---
# Install heimdall using the local role
- hosts: heimdalls
  become: no
  roles:
    - role: heimdall-role
```
---
**Example of playbook "install-standardhost.yaml"** 
which use both a local and a community role. The community role have values which can be defined in the playbook. Default values are defined in the role itself.

For community provided roles, see https://galaxy.ansible.com/

```yaml
---
# Install portainer agent using the local role
- hosts: dockers
  become: yes
  roles:
    - role: geerlingguy.docker
      docker_edition: 'ce'
      docker_package: "docker-{{ docker_edition }}"
      docker_package_state: present
      docker_users:
      - ansibleguy

- hosts: dockers
  become: no
  roles:
    - role: portaineragent-role
```


### Create own role
In your playbooks/roles-folder, create an ansible role for installing some service (e.g. heimdall in the example)
```console
ansible-galaxy init heimdall-role
```
This creates a folder for the role with a number of sub-folders.
The main task is defined in .../roles/portainer-role/tasks/main.yml

Define the task here which for me normally is to use "community.docker.docker_container" to deploy a container, create some folder, perhaps copy some files etc.

.../roles/heimdall-role/tasks/main.yml
```yaml
# tasks file for heimdall-role
- name: Create data folder for heimdall
  become: no
  file:
    path: ~/services/heimdall/data
    state: directory

- name: Create heimdall container
  become: no
  community.docker.docker_container: 
    name: heimdall
    image: lscr.io/linuxserver/heimdall
    state: started
    recreate: yes
    restart_policy: always
    published_ports:
      - 80:80
      - 443:443
    volumes:
      - ~/services/heimdall/data/config:/config
```
