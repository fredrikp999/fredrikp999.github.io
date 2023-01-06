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
## Local base installation
On you homelab LXC or VM, install Jekyll per official instructions on [https://jekyllrb.com](https://jekyllrb.com/) for prerequisites and then either continue following the instructions for creating the site - or (recommended) follow the instructions for using the Chirpy theme.

## Install prerequisites on Ubuntu
```shell
sudo apt-get install ruby-full build-essential zlib1g-dev
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
gem install jekyll bundler
```

## Create the site from Chirpy theme
If you want a really quick start for a good looking github-hosted jekyll site:
Follow [Chirpy quick start](https://github.com/cotes2020/jekyll-theme-chirpy#quick-start)
This will create a repo with all the needed files in your github repo <GH_USERNAME>.github.io, where GH_USERNAME represents your GitHub username

If you want to use Chirpy for a local installation, just fork the repo to another repository.

## Clone gitrepo to local machine and serve it locally
Using Git clone, get the Jekyll files to your local machine which is prepared to execute Jekyll
```shell
git clone git@<YOUR-USER-NAME>/<YOUR-REPO-NAME>.git
```
Install dependencies
```shell
cd YOUR-REPO-NAME
bundle
```

## Serve the site locally for test
Start jekyll site with live reload, expose on external IP
```shell
bundle exec jekyll serve --host 0.0.0.0 --livereload
```
# Make changes and create posts
## Update _config.yml
Make needed changes to _config.yml. Examples are:
* title
* timezone
* avatar

## Create new posts
Create new files in the _posts folder, one for each post.
Filenames need to follow the structure YEAR-MONTH-DAY-NameOfPost.markdown
```markdown
---
layout: post
title:  "On networking"
date:   2023-01-05 10:15:00 +0000
categories: homelab networking
tags: DNS IP VIP PiHole pfSense
image:
  path: https://somepicture.png
---
# Heading 1
Some test
## Heading 1.1
Some more text
```
{: file=".../_posts_/2023-01-13-NameOfPost.markdown" }

## Test changes locally
On your local host prepared for Jekyll, start auto-build and auto-serve out the site. With this, you will get live updates as soon as you update and save a post. (Note: for changes to _config.yml, it needs to be restarted)
```shell
bundle exec jekyll serve --host 0.0.0.0 --livereload
```

# Push to production
## Github-pages hosted site
Simply push the changes to the remote github server (git push)
Github actions will build and publish to <GH_USERNAME>.github.io

## Self hosted site
### Manual way
Simply copy the whole _site-folder to your webserver
(The site-folder contains the generated static files)
### Proper way (CI/CD)
Push the changes to your private repo e.g. local gitlab.
Use CI and CD pipelines to deploy.
See separate post on this. Here is an example from TechnoTim: [TechnoTim Jekyll GitLab CICD](https://github.com/techno-tim/techno-tim.github.io/blob/master/.gitlab-ci.yml#L18)

# References
[Techno Tim post, which is the inspiration and base](https://docs.technotim.live/posts/jekyll-docs-site/)
