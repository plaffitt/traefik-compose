# Traefik - How to: Multiple network-isolated compose stacks

This is a demo of how to use Traefik with multiple docker compose stacks, all network-isolated from each others and in an easy way. Normally you would have to manually add traefik to the networks of your each of your compose stacks, here we use a tool that automatically does this, allowing to deploy additionnal stacks on your host without needing to update your traefik configuration. And as a bonus, we will also expose a service running in the host directly (not in a container). 

## How it works?

We use the [docker provider](https://doc.traefik.io/traefik/providers/docker/) in order to discover containers based on traefik annotations. But this is not enough to make it work, traefik has to be able to reach those containers, which means they have to be in the same network. However, we want services to be isolated from each other. So they cannot share a common network. Instead, traefik has to be added to each individual networks. Since this is can be a bit repetitive and boring, we use [`obeoneorg/traefik_network_connector`](https://github.com/obeone/traefik_network_connector) to do the job automatically. It works out of the box without any additionnal configuration since we called the traefik container "traefik", which is the default name that traefik_network_connector searches for.

We then use the [file provider](https://doc.traefik.io/traefik/providers/file/) to configure services running in the host. But how to access them then? using `networkMode: host` will prevent us to reach containers that we configured earlier, and we don't want everything to run in the host network! The trick is to use the hostname `host.docker.internal`, which is provided by docker when you add the `extra_host` configuration to your container.

```yaml
services:
  reverse-proxy:
    image: traefik:v3.3
    container_name: traefik
    extra_hosts:
    - "host.docker.internal:host-gateway"
```

> According to some users the special DNS name only works within the Docker's default bridge network, not within custom networks.
> [stackoverflow answer](https://stackoverflow.com/a/43541732/5677103)

## Let's do it!

First, we start the docker compose nginx stacks and the local service:

```bash
# The first one
cd ./nginx1
docker compose up -d
cd ../
# And the second one
cd ./nginx2
docker compose up -d
cd ../
# Then the local service using http.server python module
cd ./python.http.server
./start.sh # you can also run `python -m http.server 8643`
cd ../
```

And finally, we start traefik and the network-connecter

```bash
docker compose up -d
```

Traefik is listening to `:80` and `:8080` for the dashboard. Now we can try to access our services:

```bash
$ curl nginx1.localhost
Nginx Service 1
$ curl nginx2.localhost
Nginx Service 2
$ curl python.localhost
<!DOCTYPE HTML>
[...]
</html>
```

Hosts are defined as annotation for the docker compose variants in respective compose files, and in a config file (`python.yaml`) for the local variant. On linux, `localhost` points to `127.0.0.1` as well as its subdomains (`*.localhost`), so you shouldn't need to edit your `/etc/hosts`.

## Check that evrything is well isolated

First, we can check that every container is in the expected network:

```bash
$ docker inspect traefik | jq '.[0].NetworkSettings.Networks | keys'
[
  "nginx1_default",
  "nginx2_default",
  "traefik-compose_default"
]
$ docker inspect nginx1-nginx-1 | jq '.[0].NetworkSettings.Networks | keys'
[
  "nginx1_default"
]
$ docker inspect nginx2-nginx-1 | jq '.[0].NetworkSettings.Networks | keys'
[
  "nginx2_default"
]
$ docker inspect traefik-compose-network-connector-1 | jq '.[0].NetworkSettings.Networks | keys'
[
  "traefik-compose_default"
]
```

Then we can also check from inside containers.

Traefik has access to both nginx containers:

```bash
$ docker exec -it traefik sh
/ # ping nginx1-nginx-1
PING nginx1-nginx-1 (172.22.0.2): 56 data bytes
64 bytes from 172.22.0.2: seq=0 ttl=64 time=0.043 ms
[...]
/ # ping nginx2-nginx-1
PING nginx2-nginx-1 (172.23.0.2): 56 data bytes
64 bytes from 172.23.0.2: seq=0 ttl=64 time=0.045 ms
[...]
```

But they cannot access each others:

```bash
$ docker exec -it nginx1-nginx-1 sh
# curl nginx2-nginx-1
curl: (6) Could not resolve host: nginx2-nginx-1
```
