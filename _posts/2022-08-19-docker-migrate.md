---
title: How to migrate a docker container to a different host, the easy way
date: 2022-08-19 10:00:00 -500
categories: [homelab, docker]
tags: [docker,homelab,portainer]
---

*tl;dr check out [docker migrate](https://github.com/acdoussan/docker-migrate)*

# The Problem

As my homelab continues to grow, I continue to learn and find out new things about the software that I use.
Recently, I discovered LXC containers in proxmox. I deciced to try them out and created a new container running
pihole as a service. This container consistently uses less than 100MB of ram, compared to the 1.5-2GB the VM I had
running pihole in a docker container used. This improvement is worth pretty much any tradeoff, and I set off to
migrate as many of my VMs as I could to LXC containers.

One of the VMs I have runs Portainer and serves as a docker host where I create containers from open source
projects. This was running as a VM, and I wanted to migrate everything running on it to an LXC container. I
created the new LXC container, installed docker, installed Portainer, and loaded my exported backup. This is where
I met my first surprise: Portainer only exports its own data, it doesn't recreate any continers on the new host.
This makes sense when you think about it, but meant that I had to find a way to migrate my containers another way.

# The Existing Solutions

One option is [docker-volumes.sh](https://github.com/ricardobranco777/docker-volumes.sh). This project lets you
export the volumes for your container as a tar file, and then import that data back in a new container. While
this is a good start, it expects you to create the new container yourself, including any options that are set
on the container, like forwarded ports. While I don't have very many containers, all of them were created manually
in the Portainer UI, and I don't trust myself to get everything right when creating the new container on my own.

Another project is [docker autocompose](https://github.com/Red5d/docker-autocompose). This project lets you
create a docker compose file from an existing container. This compose file includes all the configuration options
for the container, including networks, volumes, exported ports, etc.

I'm sure you can see where this is going...

# The New Solution

After a couple of pull requests to add support for volumes and default networks to docker autocompose ([#41](https://github.com/Red5d/docker-autocompose/pull/41),
[#42](https://github.com/Red5d/docker-autocompose/pull/42)), I created a new script.
[Docker migrate](https://github.com/acdoussan/docker-migrate) is a script that combines the data exporting features
of docker-volumes.sh with the configuration exporting features of docker autocompose. To migrate a container with
docker autocompose, you just run the following:

```bash
./docker-migrate.sh [-v|--verbose] <CONTAINER> <USER> <HOST>
```

Where `<CONTAINER>` is the name or hash of the container you want to migrate, and `<USER>` and `<HOST>` is
the username and host you want to use to migrate the container. For example, the following will migrate the
`uptime-kuma` container to host `10.0.0.0` with user `root`.

```bash
./docker-migrate.sh uptime-kuma root 10.0.0.0
```

There are several ssh connections that are made, therefore it is recommended to set up an SSH keypair before doing
this, otherwise you will have to enter your password several times. Also, make sure to take a backup before trying
this yourself. While it worked for my containers, there are too many different docker configurations for me to test
them all. Use this at your own peril.

# The Results

My new LXC container with all of my migrated docker containers uses about 1.25 GB of RAM, where the VM with the same
containers consumes 5.5 - 6 GB of RAM. While these docker containers are not really resource intensive, CPU
utilization also dropped from about 5% to 1.25% with the same configuration. I am completely floored with these
results, and while a LXC container is not the same as a full VM, this improvement in utilization is completely worth
it for my uses. The only thing left now is to add more services to use these new extra resources.

It is worth noting that proxmox recommends that docker be run in side of a full VM. From the
[official documentation](https://pve.proxmox.com/wiki/Linux_Container):

> If you want to run application containers, for example, Docker images, it is recommended that you run them inside a
> Proxmox Qemu VM. This will give you all the advantages of application containerization, while also providing the
> benefits that VMs offer, such as strong isolation from the host and the ability to live-migrate, which otherwise
> isn’t possible with containers.

I have had no issues so far, but If you are looking to run docker in an LXC container, know the tradeoffs before
commiting.
