---
description: Network resource monitoring tool for quick analysis
---

# Munin

Munin is a networked resource monitoring tool that can help analyze resource trends and "what just happened to kill our performance?" problems. It is designed to be very plug and play. A default installation provides a lot of graphs with almost no work.

![Munin Screenshot](../images/munin.png)

Using Munin you can easily monitor the performance of your computers, networks, SANs, applications, weather measurements and whatever comes to mind. It makes it easy to determine "what's different today" when a performance problem crops up. It makes it easy to see how you're doing capacity-wise on any resources.

Munin uses the excellent ​RRDTool (written by Tobi Oetiker) and the framework is written in Perl, while plugins may be written in any language. Munin has a master/node architecture in which the master connects to all the nodes at regular intervals and asks them for data. It then stores the data in RRD files, and (if needed) updates the graphs. One of the main goals has been ease of creating new plugins (graphs).

--8<-- "recipe-standard-ingredients.md"

## Preparation

### Prepare target nodes

Depending on what you want to monitor, you'll want to install munin-node. On Ubuntu/Debian, you'll use `apt-get install munin-node`, and on RHEL/CentOS, run `yum install munin-node`. Remember to edit `/etc/munin/munin-node.conf`, and set your node to allow the server to poll it, by adding `cidr_allow x.x.x.x/x`.

On CentOS Atomic, of course, you can't install munin-node directly, but you can run it as a containerized instance. In this case, you can't use swarm since you need the container running in privileged mode, so launch a munin-node container on each atomic host using:

```bash
docker run -d --name munin-node --restart=always \
  --privileged --net=host \
  -v /:/rootfs:ro \
  -v /sys:/sys:ro \
  -e ALLOW="cidr_allow 0.0.0.0/0" \
  -p 4949:4949 \
  --restart=always \
  funkypenguin/munin-node
```

### Setup data locations

We'll need several directories to bind-mount into our container, so create them in /var/data/munin:

```bash
mkdir /var/data/munin
cd /var/data/munin
mkdir -p {log,lib,run,cache}
```

### Prepare environment

Create /var/data/config/munin/munin.env, and populate with the following variables. Set at a **minimum** the `MUNIN_USER`, `MUNIN_PASSWORD`, and `NODES` values:

```bash

MUNIN_USER=odin
MUNIN_PASSWORD=lokiisadopted
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USERNAME=smtp-username
SMTP_PASSWORD=smtp-password
SMTP_USE_TLS=false
SMTP_ALWAYS_SEND=false
SMTP_MESSAGE='[${var:group};${var:host}] -> ${var:graph_title} -> warnings: ${loop<,>:wfields  ${var:label}=${var:value}} / criticals: ${loop<,>:cfields  ${var:label}=${var:value}}'
ALERT_RECIPIENT=monitoring@example.com
ALERT_SENDER=alerts@example.com
NODES="node1:192.168.1.1 node2:192.168.1.2 node3:192.168.1.3"
SNMP_NODES="router1:10.0.0.254:9999"
```

### Setup Docker Swarm

Create a docker swarm config file in docker-compose syntax (v3), something like this:

--8<-- "premix-cta.md"

```yaml
version: '3'

services:

  munin:
    image: funkypenguin/munin-server
    env_file: /var/data/config/munin/munin.env
    networks:
      - traefik_public
    volumes:
      - /var/data/munin/log:/var/log/munin
      - /var/data/munin/lib:/var/lib/munin
      - /var/data/munin/run:/var/run/munin
      - /var/data/munin/cache:/var/cache/munin
    deploy:
      labels:
        # traefik common
        - traefik.enable=true
        - traefik.docker.network=traefik_public

        # traefikv1
        - traefik.frontend.rule=Host:munin.example.com
        - traefik.port=8080     

        # traefikv2
        - "traefik.http.routers.munin.rule=Host(`munin.example.com`)"
        - "traefik.http.services.munin.loadbalancer.server.port=8080"
        - "traefik.enable=true"

        # Remove if you wish to access the URL directly
        - "traefik.http.routers.wekan.middlewares=forward-auth@file"

networks:
  traefik_public:
    external: true
```

--8<-- "reference-networks.md"

## Serving

### Launch Munin stack

Launch the Munin stack by running `docker stack deploy munin -c <path -to-docker-compose.yml>`

Log into your new instance at https://**YOUR-FQDN**, with user and password password you specified in munin.env above.

[^1]: If you wanted to expose the Munin UI directly, you could remove the traefik-forward-auth from the design.

--8<-- "recipe-footer.md"
