---
layout: post
title:  "Cheat-Sheets"
date:   2022-12-27 17:15:00 +0000
categories: cheatsheets
tags: cheatsheets
---

# Markdown
[Markdown cheat sheet](https://www.markdownguide.org/cheat-sheet)
```markdown
# H1
## H2
**bold text**
[title](https://www.example.com)
![alt text](image.jpg)
```
# Docker and docker-compose
[Docker cheat sheet](https://www.dockercheatsheet.com/)
```console
docker build -t my-image:1.0 .
docker push myrepo/my-image:2.0
docker run my-image:1.0
docker container exec -it ubuntu bash
docker-compose up -d
```

# Jekyll
[Jekyll cheat sheet](https://devhints.io/jekyll)
```console
bundle exec jekyll serve --host 0.0.0.0 --livereload
```

# Ansible, ansible-playbook and ansible-galaxy
[Ansible cheat sheet](https://www.svastikkka.com/2021/04/ansible-cli-cheatsheet.html)
```console
ansible all -m ping
ansible-playbook MYPLAYBOOK.yml -i MY_CUSTOM_INVENTORY
ansible-galaxy init my-new-role
```

# Helm