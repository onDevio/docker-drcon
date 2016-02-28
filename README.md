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

First, let's see an example of **load balancing**. We'll talk about how it works later on.

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

**A Note for Mac users**: If you are using Docker in Mac, replace "localhost" with Docker's IP for the rest of the article.
You can find what's Docker's IP by running `docker-machine ip`.

Now point a web browser to http://**localhost**/. You should see a welcome page.

Try this before getting into how it works:

1. Scale up the web to 3 containers: `docker-compose scale simple=3`
2. Open another terminal to watch the logs: `docker-compose logs simple`
3. Go back to the first terminal and stress test the app: `docker-compose stress ab -n 1000 -c 10 http://localhost/`
4. Watch the logs again. See how traffic is dispatched to every `simple` container?
 
Enough fun. Let's get to how it works.

# How it works
