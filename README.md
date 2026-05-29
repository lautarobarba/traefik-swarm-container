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

## Setup

### 1. Crear la red overlay

```bash
docker network create \
  --driver=overlay \
  --attachable \
  traefik-network
```

### 2. Configurar Let's Encrypt

```bash
mkdir -p letsencrypt
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

### 3. Certificados locales (opcional)

Los certificados para dominios locales (ej. `homelab.local`) van en `certs/`:

- `certs/homelab.local.crt`
- `certs/homelab.local.key`

El archivo `dynamic/tls.yml` referencia estos certificados.

### 4. Desplegar el stack

```bash
docker stack deploy -c compose.yaml traefik
```

### 5. Verificar

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

| Archivo/Dir | Propósito |
|-------------|-----------|
| `compose.yaml` | Stack de Docker Swarm para Traefik |
| `traefik.yml` | Configuración principal (entrypoints, providers, ACME) |
| `dynamic/` | Configuración dinámica (TLS, routers estáticos) |
| `dynamic/tls.yml` | Certificados locales y router del dashboard |
| `letsencrypt/` | Almacenamiento de certificados ACME |
| `certs/` | Certificados locales (.crt + .key) |

## Seguridad

### Dashboard

El dashboard **nunca** debe exponerse con `insecure: true` en producción.

Opciones recomendadas:

- **Basic Auth** en el router del dashboard
- **VPN** para restringir acceso
- **IP whitelist** en el entrypoint

### Certificados

- `letsencrypt/acme.json` debe tener permisos `600`
- Los certificados locales en `certs/` están en `.gitignore` (no commitear keys)

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
