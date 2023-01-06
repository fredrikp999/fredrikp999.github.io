---
layout: post
title:  "Jekyll basics"
date:   2023-01-03 07:15:00 +0000
categories: homelab web
tags: homelab jekyll
image:
  path: https://media.discordapp.net/attachments/1059461993817448459/1060601198232748042/Fredrik999_jekyll_and_hyde_black_background_b1d729ab-d22d-4b26-a33e-6c4494282b69.png
---
# Introduction
Jekyll is a static site generator. It takes text written in your favorite markup language and uses layouts to create a static website. You can tweak the site’s look and feel, URLs, the data displayed on the page, and more.
See [https://jekyllrb.com](lhttps://jekyllrb.com/)

You can use Jekyll to
* Generate static html-pages with your own CI + publish on your self-hosted site
* Generate static html-pages with your own CI, build a docker-image with nginx and the html-pages + use image for your self-hosted site & as a simple to deploy website
* Maintain jekyll site code in github and use github actions to build (CI) and publish (CD) to github pages

Note that also if only publishing on github pages, you should make sure you can build and test the pages locally. It is not strictly nessecary, but doing so makes it a lot easier to quickly see the effect of your changes, preview, troubleshoot etc.

# Installation
## Local installation
On you homelab LXC or VM, install Jekyll per official instructions on [https://jekyllrb.com](lhttps://jekyllrb.com/)


## Local

# Building
# Service for test
Start jekyll site with live reload, expose on external IP
```shell
bundle exec jekyll serve --host 0.0.0.0 --livereload
```
