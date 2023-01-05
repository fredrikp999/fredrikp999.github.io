---
layout: post
title:  "Wildcard DNS on local services"
date:   2023-01-04 11:00:10 +0000
categories: DNS homelab cloudflare certbot
---

# Introduction
Purpose is to enable using proper certificates inside the homelab. This is solved by the following steps
* registering a wildcard A-record at your DNS-provider (e.g. cloudflare)
* proving you are the owner with the help of certbot 
* configuring local dns for the local wildcard record, pointing to a reverse proxy
* setting up a reverse proxy e.g. traefik or NGNX with rules to route each hostname to the correct service

# Register wildcard DNS at DNS-provider
If using cloudflare, see: https://developers.cloudflare.com/dns/manage-dns-records/reference/wildcard-dns-records/

register an A-record for something like local.EXAMPLE.COM or l.EXAMPLE.COM, pointing to your fixed IP (if having dynamic IP, set that up)

# Prove you are the owner or the domain using certbot
## Install certbot on a local server, not exposed
This is the recommended, more safe(?) alternative.
On one server on the local network, install certbot
(Assuming DNS-provider is cloudflare below)

In short:
### 1) Install certbot

Follow: https://certbot.eff.org/instructions?ws=other&os=ubuntufocal
in short:
```console
sudo snap install core; sudo snap refresh core
sudo apt-get install python3-certbot-dns-cloudflare
sudo mkdir /root/.secrets/
sudo touch /root/.secrets/cloudflare.ini
```
Note1: Needs to be installed on VM. Does not work on Proxmox LXC due to a mount problem with snapd for the moment (seems to be a workaround but it has some drawbacks, see https://forum.proxmox.com/threads/ubuntu-snaps-inside-lxc-container-on-proxmox.36463/#post-230060)

Note2: There are multiple tutorials out there which are not up to date. Make sure to use the latest from certbot

### 2) Get API key from cloudflare
API-key is created from cloudflare dashboard
* https://dash.cloudflare.com/
* Go to my profile --> API Tokens
* Create token with permissions to edit zone DNS


### 3) Configure file for certbot with your credentials
```console
sudo vi /root/.secrets/cloudflare.ini
```
update file with your details
```yaml
# Cloudflare API token used by Certbot
dns_cloudflare_api_token = YOUR-TOKEN
```
### 4) Secure secrets files
```console
sudo chmod 0700 /root/.secrets/
sudo chmod 0400 /root/.secrets/cloudflare.ini
```

### 5) Request certificate
* Request "certificate only"

```console
sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials /root/.secrets/cloudflare.ini -d example.com,*.example.com,*.l.example.com --preferred-challenges dns-01
 ```
Note: *.l.example.com (or *.local.example.com if you prefer) is what will be used for local hosts and is what will make it possible to use real certs also for internal services / host names


### 6) Use the retrieved certificates
If successfully received certificate.
* Certificate is saved at: /etc/letsencrypt/live/example.com/fullchain.pem
* Key is saved at: /etc/letsencrypt/live/example.com/privkey.pem
(Note: These files will be updated when the certificate renews. Certbot has set up a scheduled task to automatically renew this certificate in the background.)

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


If service is on same server as the original pem-files, reference these instead.
Also make sure to apply least privilage approach, lock down the access to the pem-files as much as possible

### 7) Add DNS-entries to *.l.mydomain.com
 * Simple approach: add A-record for each service. Either in local DNS or in /etc/hosts for easy tests
 * Better approach: add wild-card A-record for *.l.example.com to reverse proxy e.g. nginx or traefik

### 8) Setup reverse proxy, traefik
 * See separate post on details for traefik or nginx
 * In short, the reverse proxy recieves all traffic to *.l.example.com and routes traffic on either to different servers or to local docker containers using local docker-network. The proxy can also apply actions before forwarding the request, e.g. filtering or stripping paths etc.

## Notes
The expiry date for the API-key is set when creating it in cloudflare. Take care when it expires, when the cloudflare.ini needs to be updated with new API-key (or API key extended) + cert requested again per #5 above

## Alternative: Set up server with port 80 exposed 
Using a server in you DMZ which is isolated from the rest of the network, install certbot: https://certbot.eff.org/ and e.g. https://certbot.eff.org/instructions?ws=other&os=ubuntufocal
This require that you open up and forward port 80 to the server running certbot. Works well when using other DNS providers than Cloudflare which does not have the API-key method