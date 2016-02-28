# docker-drcon

Service discovery using Docker, Registrator, Consul and Nginx

# Introduction

This repo is based on the article "Scalable Architecture DR CoN: Docker, Registrator, Consul, Consul Template and Nginx" by Graham Jenson. Youn can find the article at http://www.maori.geek.nz/scalable_architecture_dr_con_docker_registrator_consul_nginx/

This repo provides a "template" to deploy [Registrator](https://github.com/gliderlabs/registrator), [Consul](https://www.consul.io/), [Nginx](http://nginx.org/) and a sample web app with just one command using Docker Compose.

Please note there's also other repos in GitHub about this topic. Just search for "drcon".

## Why Service Discovery

It's common to link containers together, so that container **A** can call services on container **B**.
Docker provides the `--link` argument to make it possible.

Like so:

```
docker run -ti --name B some_img some_cmd
```

```
docker run -ti --name A --link B some_img some_cmd
```

However, this is a very "static" way of linking containers: containers to link to (B in this example) need to be **running in advance**, must have a **fixed name**, and there must be a **fixed number** of them.

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
5. Check it's running: `docker-compose ps`. 

You should see a list of 5 containers, 4 of them running.

```
                  Name                                     Command                                     State                                      Ports                   
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------
dockerdrcon_consul_1                       /bin/start -server -bootst ...             Up                                         53/tcp, 0.0.0.0:8600->53/udp, 8300/tcp,  
                                                                                                                                 8301/tcp, 8301/udp, 8302/tcp, 8302/udp,  
                                                                                                                                 8400/tcp, 0.0.0.0:8500->8500/tcp         
dockerdrcon_nginx_1                        /bin/sh -c /usr/sbin/nginx ...             Up                                         0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp 
dockerdrcon_registrator_1                  /bin/registrator -internal ...             Up                                                                                  
dockerdrcon_simple_1                       /bin/sh -c DEBUG=myapp:* n ...             Up                                         3000/tcp                                 
dockerdrcon_stress_1                       echo Usage docker-compose  ...             Exit 0                              
```

**A Note for Mac users**: If you are using Docker in Mac, replace "localhost" with Docker's IP for the rest of the article.
You can find what's Docker's IP by running `docker-machine ip`.

Now point a web browser to http://**localhost**/. You should see a welcome page.

Try this before getting into how it works:

1. Scale up the web to 3 containers: `docker-compose scale simple=3`
2. Open another terminal to watch the logs: `docker-compose logs simple`
3. Go back to the first terminal and stress test the app: `docker-compose run stress ab -n 1000 -c 10 http://localhost/`
4. Watch the logs again. See how traffic is dispatched to every `simple` container?
 
Enough fun. Let's get to how it works.

# How it works

This is just a summary of what's in Graham Jenson's article, with some extras regarding Docker Compose.

It boils down to this (deep breathe):

> When a new container is created or destroyed, [Registrator](https://github.com/gliderlabs/registrator) passes IP and exposed Ports to [Consul](https://www.consul.io/), which is frequently queried by [Consul Template](https://github.com/hashicorp/consul-template), which in turn writes that information into nginx configuration file and restarts [Nginx](http://nginx.org/)

Let's see how these components are weaved together in `docker-compose.yml`.

## Docker Compose

Docker Compose starts all the necessary containers and links them together according to `docker-compose.yml` file.

Here's a commented version of `docker-compose.yml`.

```yml
# A service named "simple", which is actually an Express application with a Welcome page.
# Note there are no links to other containers.

simple:
  build: ./web
  environment:
    - SERVICE_NAME=simple

# Consul service. It maps port 8500 and 8600. 
# Point a browser to http://localhost:8500 to see Consul`s control panel.  

consul:
  image: progrium/consul
  command: -server -bootstrap -log-level debug
  hostname: node
  ports:
    - 8500:8500
    - 8600:53/udp
  environment:
    - SERVICE_IGNORE=true

# Registrator service. It links to Consul to be able to call Consul's API 
# to de/register information about containers that come and go.

registrator:
  image: gliderlabs/registrator
  command: -internal consul://consul:8500
  volumes:
    - /var/run/docker.sock:/tmp/docker.sock
  links:
    - consul
  environment:
    - SERVICE_IGNORE=true

# Nginx + Consul Template. It links to Consul to be able to make queries to update nginx config file.
# It mounts a volume to map the template file (nginx-loadbalancer.conf) inside the container. 
# Note there are no link to containers other than Consul.
# Also, note it maps port 80, so you can hit http://localhost/

nginx:
  build: ./nginx
  links:
    - consul
  ports:
    - 80:80
    - 443:443
  volumes:
    - ./nginx/consul-templates/nginx-loadbalancer.conf:/etc/consul-templates/nginx.conf
  environment:
    - SERVICE_IGNORE=true

# A container to run Apache Benchmark to stress test a URL.

stress:
  build: ./ab
  command: echo 'Usage docker-compose run stress ab -n 1000 -c 10 http://[DOCKER_IP|localhost]/'
```

## Load Balancing

So you now by now how the pieces interact. Let's take a closer look into **load balacing** itself.

When you hit http://localhost/ nginx will get this request and will act upon according to its configuration file.

Consul Template rewrites nginx config file according to the template file `nginx-loadbalancer.conf`:

```
upstream app {
  least_conn;
  {{range service "simple"}}
  server  {{.Address}}:{{.Port}};
  {{else}}server 127.0.0.1:65535;{{end}}
}

server {
  listen 80 default_server;
  location / {
    proxy_pass http://app;
  }
}
```

In short, this template should be read like this:

> Query Consul for services named "simple", and write its Address and Port inside **upstream app** block. All calls to port 80 will then be proxy_pass'ed to that **upstream**.

The result of processing that template will be written to `/etc/nginx/conf.d/default.conf` inside nginx running container.
You can see its contents with this command (given `dockerdrcon_nginx_1` is the name of the nginx container):

```
docker exec -ti dockerdrcon_nginx_1 cat /etc/nginx/conf.d/default.conf
```

It should look something like this:

```
upstream app {
  least_conn;
  
  server  172.17.0.3:3000;
  
}

server {
  listen 80 default_server;
  location / {
    proxy_pass http://app;
  }
}
```

