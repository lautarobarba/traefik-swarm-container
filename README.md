# Traefik Swarm Container

Configuración de Traefik v3 como reverse proxy en Docker Swarm con soporte para SSL (Let's Encrypt + certificados locales) y dashboard protegido.

## Overview

Traefik se ejecuta como servicio en Docker Swarm, actuando como punto de entrada para todas las aplicaciones del homelab. Soporta:

- **HTTP → HTTPS redirect** automático
- **Certificados SSL** vía Let's Encrypt (producción) y certificados locales (desarrollo)
- **Dashboard** con autenticación y TLS
- **Service discovery** automático mediante el provider de Docker Swarm

## Prerequisites

- Docker Swarm inicializado (`docker swarm init`)
- Acceso al nodo manager del swarm
- NFS (u otro storage compartido). Crear una carpeta disponible compartida /srv/nfs/traefik **NFS_PATH**.

## Estructura de datos en NFS

Todos los archivos de configuración viven en el NFS para que Swarm pueda reschedular el contenedor en cualquier nodo sin perder estado.

```
/srv/nfs/traefik/
└── data/
    ├── traefik.yml          # Configuración principal de Traefik
    ├── dynamic/
    │   └── tls.yml          # Configuración dinámica (TLS, dashboard)
    ├── certs/
    │   ├── homelab.local.crt
    │   └── homelab.local.key
    └── letsencrypt/
        └── acme.json        # Almacenamiento de certificados ACME
```

## Setup

### 1. Crear la red overlay

```bash
docker network create \
  --driver=overlay \
  --attachable \
  traefik-network
```

### 2. Crear la estructura de directorios en el NFS

```bash
mkdir -p /srv/nfs/traefik/data/{dynamic,certs,letsencrypt}
```

### 3. Configurar Let's Encrypt

```bash
touch /srv/nfs/traefik/data/letsencrypt/acme.json
chmod 600 /srv/nfs/traefik/data/letsencrypt/acme.json
```

> **Nota:** Con `replicas: 1` no hay escrituras concurrentes sobre `acme.json`, por lo que no se necesita un backend externo como Redis.

### 4. Copiar los archivos de configuración al NFS

Crear los archivos de configuraciones y ajustar según el entorno:

```bash
touch /srv/nfs/traefik/data/traefik.yml
nano /srv/nfs/traefik/data/traefik.yml
```

```yaml
# /srv/nfs/traefik/data/traefik.yml
api:
  dashboard: true
  # insecure: true

log:
  # level: INFO
  level: DEBUG

entryPoints:
  web:
    address: ":80"

    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: ":443"

providers:
  swarm:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false

  file:
    directory: /etc/traefik/dynamic
    watch: true

certificatesResolvers:
  letsencrypt:
    acme:
      email: email@gmail.com
      storage: /letsencrypt/acme.json

      httpChallenge:
        entryPoint: web
```

```bash
touch /srv/nfs/traefik/data/dynamic/tls.yml
nano /srv/nfs/traefik/data/dynamic/tls.yml
```

```yaml
# /srv/nfs/traefik/data/dynamic/tls.yml
tls:
  certificates:
    - certFile: /certs/example.local.crt
      keyFile: /certs/example.local.key

http:
  routers:
    traefik-dashboard:
      rule: Host(`traefik.local`)
      service: api@internal
      entryPoints:
        - websecure
      tls: {}
```

### 5. Certificados locales (opcional)

Para dominios locales (ej. `homelab.local`), copiar los certificados al NFS:

```bash
cp homelab.local.crt /srv/nfs/traefik/data/certs/
cp homelab.local.key /srv/nfs/traefik/data/certs/
```

El archivo `dynamic/tls.yml` los referencia desde `/certs/`.

### 6. Actualizar owner de archivos

```bash
chown -R nobody:nogroup /srv/nfs/traefik
```

### 7. Desplegar el stack

```bash
docker stack deploy -c compose.yaml traefik
```

### 8. Verificar

```bash
docker service ls
docker service logs -f traefik_traefik
```

## Publicar aplicaciones

Para exponer un servicio detrás de Traefik, agrega los labels correspondientes en tu `compose.yaml`:

```yaml
services:
  app:
    image: nginx

    networks:
      - traefik-network

    deploy:
      labels:
        - traefik.enable=true
        - traefik.http.routers.app.rule=Host(`app.midominio.com`)
        - traefik.http.routers.app.entrypoints=websecure
        - traefik.http.routers.app.tls.certresolver=letsencrypt
        - traefik.http.services.app.loadbalancer.server.port=80

networks:
  traefik-network:
    external: true
```

## Estructura del proyecto

**Repositorio** (solo configuración de Swarm):

| Archivo                | Propósito                                            |
| ---------------------- | ---------------------------------------------------- |
| `compose.yaml`         | Stack de Docker Swarm para Traefik                   |
| `examples/traefik.yml` | Plantilla de configuración principal de Traefik      |
| `examples/tls.yml`     | Plantilla de configuración dinámica (TLS, dashboard) |

**NFS** (`/srv/nfs/traefik/data/`):

| Archivo/Dir             | Propósito                                              |
| ----------------------- | ------------------------------------------------------ |
| `traefik.yml`           | Configuración principal (entrypoints, providers, ACME) |
| `dynamic/tls.yml`       | Certificados locales y router del dashboard            |
| `letsencrypt/acme.json` | Almacenamiento de certificados ACME (permisos `600`)   |
| `certs/*.crt`           | Certificados locales                                   |
| `certs/*.key`           | Claves privadas de certificados locales                |

## Seguridad

### Dashboard

El dashboard **nunca** debe exponerse con `insecure: true` en producción.

Opciones recomendadas:

- **Basic Auth** en el router del dashboard
- **VPN** para restringir acceso
- **IP whitelist** en el entrypoint

### Certificados

- `letsencrypt/acme.json` debe tener permisos `600` (`chmod 600`)
- Los certificados locales en `certs/` no deben commitearse; el repo solo contiene `compose.yaml`

## Troubleshooting

### Traefik no detecta servicios

- Verificar que el servicio tenga `traefik.enable=true`
- Confirmar que la red `traefik-network` existe y es externa
- Revisar logs: `docker service logs -f traefik_traefik`

### Certificados Let's Encrypt no se generan

- Verificar que el dominio apunta al servidor (DNS)
- Confirmar que el puerto 80 está accesible (HTTP challenge)
- Revisar logs de Traefik para errores de ACME

### Dashboard no accesible

- Verificar que `dynamic/tls.yml` tiene el router configurado
- Confirmar que los certificados locales existen en `certs/`
- El dashboard está en `traefik.homelab.local` (configurable en `tls.yml`)
