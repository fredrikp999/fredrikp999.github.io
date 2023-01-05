---
layout: post
title:  "Docker registry with certificates"
date:   2023-01-04 09:16:00 +0000
categories: registry docker homelab
---

#### Example usage of certs on another server for (docker) registry:
* create folder for certs
* copy fullchain.pem and provkey.pem to ./certs
* for simple test, use docker run
```console
docker run -d --restart=always --name registry -v "$(pwd)"/certs:/certs -e REGISTRY_HTTP_ADDR=0.0.0.0:443 -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/fullchain.pem -e REGISTRY_HTTP_TLS_KEY=/certs/privkey.pem -p 443:443 registry:2
```
Or better, use docker-compose + authentication

Create folders + Generate password and store in file
```console
mkdir data
mkdir auth
mkdir certs
docker run --entrypoint htpasswd httpd:2 -Bbn yourusername yourpassword > auth/htpasswd
```

Create docker-compose.yaml
```yaml
registry:
  restart: always
  image: registry:2
  ports:
    - 5000:5000
  environment:
    REGISTRY_HTTP_TLS_CERTIFICATE: /certs/fullchain.pem
    REGISTRY_HTTP_TLS_KEY: /certs/privkey.pem
    REGISTRY_AUTH: htpasswd
    REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
  volumes:
    - ./data:/var/lib/registry
    - ./certs:/certs
    - ./auth:/auth
```
Start docker container (detached)
```console
docker-compose up -d
```
Now login from somewhere where you want to use this docker registry
```console
docker login registry.l.example.com:5000
```
(enter username and password)

Verify that you can access the registry. First get the catalogue, then the available tags for the ubuntu image
```console
curl -u youruser:yourpass https://l.example.com:5000/v2/_catalog
curl -u youruser:yourpass https://l.example.com:5000/v2/ubuntu/tags/list
```

Now try pushing and pulling images. First build or pull an image to your local docker, then tag it for the new local registry and push it
```console
docker pull ubuntu:16.04
docker tag ubuntu:16.04 registry.l.example.com/my-ubuntu:1.0
docker push ubuntu:16.04 registry.l.example.com/my-ubuntu:1.0
```
NOTE: This is not working yet, I get following error message:
* Get "https://registry.l.example.com/v2/": x509: certificate is not valid for any names, but wanted to match registry.l.example.com
To be sorted out. Strange as login + curl is working fine with https + fails when cert does not match hostname