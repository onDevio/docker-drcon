# docker-drcon

Service discovery using Docker, Registrator, Consul and Nginx

# Introduction

This repo is based on the article "Scalable Architecture DR CoN: Docker, Registrator, Consul, Consul Template and Nginx" by Graham Jenson. Youn can find the article at http://www.maori.geek.nz/scalable_architecture_dr_con_docker_registrator_consul_nginx/

This repo provides a "template" to deploy [Registrator](https://github.com/gliderlabs/registrator), [Consul](https://www.consul.io/), [Nginx](http://nginx.org/) and a sample web app with just one command using Docker Compose.

Please note there's also other repos in GitHub about this topic. Just search for "drcon".

## Why Service Discovery

It's common to link containers together, so that container **A** can call services on container **B**.
Docker provides the `--link` argument to make this possible.

However, this is a very "static" way of linking containers. 
The containers need to be running in advance to link to them.

Some scenarios need a more dynamic approach, like **load balancing**.
The number of containers to dispatch traffic load will change over time, so it's not possible to "statically" link to containers that will be created/destroyed in the future. 
They need to get linked/unlinked as they come and go.

# Getting started

First, let's see all this in action. We'll talk about how it works later on.

You'll need this software:

1. [Git](https://git-scm.com/)
2. [Docker Engine](https://www.docker.com/products/docker-engine)
3. [Docker Compose](https://www.docker.com/products/docker-compose)

Step by step:

1. Clone this repo: `git clone https://github.com/onDevio/docker-drcon.git`
2. Go into repo dir: `cd docker-drcon`
3. Build images: `docker-compose build`
4. Create containers: `docker-compose up -d`
5. Check it's running: `docker-compose ps`. You should see a list of 5 containers running.

Now point a web browser to http://localhost/web. You should see a welcome page.

