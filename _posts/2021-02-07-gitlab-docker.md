---
layout: post
title:  "GitLab on Raspberry Pi 4"
excerpt_separator: <!--more-->
---

I try to run all my containers in swarm mode. Some containers that have web interfaces are exposed using [Traefik](https://traefik.io/) reverse proxy. I highly recommend Traefik as a reverse proxy. You get automatic SSL with [letsencrypt](https://doc.traefik.io/traefik/https/acme/) and expose your containers under a subdomain or some path. For example, if you own domain `example.com` that is served by Traefik, then you can expose NodeRED under `https://nodered.example.com` or `https://example.com/nodered/`. Some other containers, you can expose through other subdomain/path and you do not have to use different port numbers to access these services. Additionally you can protect some services with username/password that does not have any by application itself.

<!--more-->

## Running GitLab on arm64

I think GitLab supports arm64, but not officially. They have not released docker images for arm64, but if you build all required components, then it's possible to run it on arm64. Of course, I didn't want to build it myself, but luckily there is someone who has built it and published it. To run GitLab on Raspberry pi 4, I used prebuild docker image from `https://gitlab.com/gypsophlia/gitlab-build-arm64`. Basically, I took my existing docker-compose for GitLab from the old amd64 system and replaced `image:` with `registry.gitlab.com/gypsophlia/omnibus-gitlab/gitlab-ce:13.1.2-ce.0_arm64`. Starting GitLab takes a long time (about 7 minutes), but when it's done, then it runs quite ok without any issues. On idle it takes about 2.6GB of memory and about 50% of one CPU core (out of 4 cores). One thing to note is that you do not have to run GitLab in `privileged` mode, I have seen someone to recommend it and there are some permission related errors in logs, but it still runs without `privileged` mode. But unfortunatly this docker image does not work in swarm mode. This means you need to do Traefik routing rules in Traefik configs and create special docker network to have communication between swarm containers and non-swarm containers. This is needed only if Traefik is running on swarm, if not, then you can have your Traefik rules in GitLab compose file as labels.

## GitLab docker-compose

Here is an example GitLab `docker-compose.yml` file. This service can not be started as a swarm container, but has to be started as a regular docker-compose container with the command `docker-compose up -d`.

```yml
version: '3'

services:
  gitlab:
    container_name: gitlab
    image: registry.gitlab.com/gypsophlia/omnibus-gitlab/gitlab-ce:13.1.2-ce.0_arm64
    restart: always
    networks:
      - traefik
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://example.com/gitlab/'
        nginx['listen_https'] = false
        nginx['listen_port'] = 80
        nginx['redirect_http_to_https'] = false
        nginx['proxy_set_headers'] = { "X-Forwarded-Proto" => "http", "X-Forwarded-Ssl" => "on" }
        letsencrypt['enable'] = false
        registry_external_url 'https://registry.example.com'
        registry_nginx['proxy_set_headers'] = { "X-Forwarded-Proto" => "http", "X-Forwarded-Ssl" => "on" }
        registry_nginx['listen_port'] = 4567
        registry_nginx['listen_https'] = false
    volumes:
      - './gitlab/config:/etc/gitlab'
      - './gitlab/logs:/var/log/gitlab'
      - './gitlab/data:/var/opt/gitlab'

networks:
  traefik:
    external:
      name: traefik
```

In compose file you can see that there are no ports exposed. Also, SSL is disabled for GitLab. This is because Traefik is used to accept SSL connections, handles all the certs stuff, and based on path `/gitlab/` (in Traefik config) it knows that requests must be redirected to GitLab service. Requests from Traefik to GitLab are unencrypted. But this unencrypted traffic can be encrypted with docker network.

## Traefik config for GitLab

Traefik is started in swarm, this means that by default it does not see GitLab container that runs as regular container. To make swarm containers and regular non-swarm containers to be able to communicate with each-other, you have to create `attachable` network. We can create a network called `traefik` with this command - `docker network create --attachable -d overlay traefik`. If you would like to docker to encrypt all data within `traefik` network, then on network create command you can add `--opt encrypted` argument. All services/containers using this network are able to see and connect to eachother. If Traefik service and GitLab service is using this network, then Traefik is able to connect to GitLab using `gitlab` hostname and redirect trafic to this service.

To make Traefik redirect traffic to GitLab we have to enable [File provider](https://doc.traefik.io/traefik/providers/file/) and add [dynamic configuration](https://doc.traefik.io/traefik/getting-started/configuration-overview/#the-dynamic-configuration) that redirects `example.com` host and path `/gitlab/` to `gitlab` container. Example dynamic config:

```yml
http:
  services:
    gitlab:
      loadBalancer:
        servers:
        - url: "http://gitlab"
  routers:
    gitlab:
      entryPoints:
        - "http"
      rule: "Host(`example.com`) && PathPrefix(`/gitlab/`)"
      service: gitlab
```

## Migrating GitLab data

To migrate all GitLab data, repositories, users to the new GitLab instance, you need to have the same GitLab version. My existing GitLab was quite old, so I needed to update it to version `13.1.2` using [GitLab upgrade path](https://docs.gitlab.com/ce/update/#upgrade-paths). After that you can do [backup](https://docs.gitlab.com/ee/raketasks/backup_restore.html#back-up-gitlab) in old GitLab instance, copy backup `tar` file from `/etc/gitlab/config_backup/` folder into same folder in new GitLab instance and [restore](https://docs.gitlab.com/ee/raketasks/backup_restore.html#restore-gitlab) tar backup.

GitLab was my main concern when moving to arm64 architecture, but looks like it actually worked out quite well. I'm looking forward to when GitLab adds arm64 builds to official docker images.
