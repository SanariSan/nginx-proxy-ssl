# Nginx auto ssl

I have another project based on this one, which allows you to configure v2ray-wmess-tls proxy on vps -> 👉 [HERE](https://github.com/SanariSan/v2ray-ws-tls) 👈

## Table of Contents

- [About](#about)
- [Getting Started](#getting_started)
- [Usage](#usage)

## About <a name = "about"></a>

Nginx-proxy container + acme ssl container + control script.

Allows you to easily launch entry router on your vps 80/443 ports which will automatically reroute requests, pin AND renew ssl certs for each subsequent container you launch with according ENV variables ([detailed here](https://github.com/nginx-proxy/nginx-proxy)), but basically just these:

- VIRTUAL_HOST
- LETSENCRYPT_HOST

* and connect to `inbound` network, more on that later

So you get this:

- (browser) https://subdomain.yourdomain.com =>
- (your vps) nginx-proxy =>
- auto-pin letsencrypt ssl =>
- reroute to container with project =>
- your container, with ssl 😊

Repo presents:

- [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) container which will receive all requests on your vps 80/443 ports
- [nginx-proxy-acme](https://github.com/nginx-proxy/acme-companion) container which is in charge of issuing certificates for any deployed container + auto renewing them
- control script for user-friendy tweaks
- compose file for oneline launch as an alternative

---

## Getting Started <a name = "getting_started"></a>

### Prerequisites

1. You should own a domain, which has **A** record pointing to your vps ip
2. On your vps with Ubuntu system, you should install git and docker (commands from official site, pick by your own if need) [docker](https://docs.docker.com/engine/install/ubuntu/) | [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

```
sudo apt install git-all

sudo apt remove docker docker-engine docker.io containerd runc
sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

3. Stop apache on your server, it takes port 80 which prevents nginx from taking its' place.

```
sudo systemctl disable apache2
sudo systemctl stop apache2
```

---

### Installing

1. Clone this repo `git clone https://github.com/SanariSan/nginx-proxy-ssl`
2. Cd into directory `cd nginx-proxy-ssl`
3. Make script executable `chmod 755 ./start.sh`
4. Copy rename .env.copy to .env and replace values with your own, use **nano** or other editor `nano ./env`
5. Run script with `/bin/bash ./start.sh`
6. Or just run `docker-compose up --build --detach --force-recreate` for oneline setup

---

## Usage <a name = "usage"></a>

**Pictures from [v2ray-wmess-tls](https://github.com/SanariSan/v2ray-ws-tls) project but menu is almost the same**

0. Main menu

![Main menu](https://github.com/SanariSan/v2ray-ws-tls/blob/master/assets/general.png?raw=true)

1. Go to **1)** section (containers) and run test certificate request (it's dry run, no cert generated)
2. If that went fine start all the containers
3. Make sure all containers up and running, you will see **green circles**

![Containers menu](https://github.com/SanariSan/v2ray-ws-tls/blob/master/assets/containers_.png?raw=true)

4. Check out Logs in section **3)** if need to.

![Logs menu](https://github.com/SanariSan/v2ray-ws-tls/blob/master/assets/logs.png?raw=true)

5. If you wish to enable autostart on boot proceed to **2)** section.

![Autostart menu](https://github.com/SanariSan/v2ray-ws-tls/blob/master/assets/autostart.png?raw=true)

6. To enable BBR optimisation proceed to option **6)**.

.

---

#### Side note

Project uses [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) and [acme-companion](https://github.com/nginx-proxy/acme-companion) containers. To make them work not only within this project, but also for proxying other projects, I assigned network `inbound` to both containers. Left more info about that [here](https://github.com/nginx-proxy/nginx-proxy/issues/1081#issuecomment-1372296950).

---

### So how do you connect your projects after launching this router?

#### Example with `docker-compose.yaml`

```
...
networks:
    inbound:
        name: inbound
        external: true
services:
    app:
        image: ...
        networks:
            - inbound
            - default
        environment:
          VIRTUAL_HOST: 'subdomain.yourdomain.com'
          LETSENCRYPT_HOST: 'subdomain.yourdomain.com'
     postgres:
        image: ...
        networks:
            - default
        environment:
          NETWORK_ACCESS: 'internal'
...
```

---

#### Or, example with default `docker run`

If you have multiple containers which communicate with each other you have to start your containers with `inbound` network and then run `connect` to add local network for containers to communicate (to not trash inbound network)

```
docker run -d --rm \
--name app-container \
--net inbound \
--env VIRTUAL_HOST='subdomain.yourdomain.com' \
--env LETSENCRYPT_HOST='subdomain.yourdomain.com' \
app

docker run -d --rm \
--name postgres-container \
--net inbound \
--env NETWORK_ACCESS='internal' \
postgres

docker network create local-net
docker network connect local-net app-container
docker network connect local-net postgres-container
```

**SO, it's better to use docker-compose if you have more than one container!**
