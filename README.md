# Traefik

El primer paso es crear la red para tener conexión con todos los nodos del swarm

```bash
docker network create \
    --driver=overlay \
    --attachable \
    traefik-network
```

Crear acme.json

```bash
mkdir letsencrypt
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

Levantar el stack

```bash
docker stack deploy -c compose.yaml traefik
docker service ls
docker service logs -f traefik_traefik
```

# Publicar apps en la red

```bash
services:

  app:
    image: nginx

    networks:
      - traefik-public

    deploy:
      labels:

        - traefik.enable=true

        - traefik.http.routers.app.rule=Host(`app.midominio.com`)
        - traefik.http.routers.app.entrypoints=websecure
        - traefik.http.routers.app.tls.certresolver=letsencrypt

        - traefik.http.services.app.loadbalancer.server.port=80

networks:
  traefik-public:
    external: true
```

## Dashboard protegido

Nunca dejar:

```yaml
insecure: true
```

en producción.

Idealmente:

- basic auth
- VPN
- whitelist IP
