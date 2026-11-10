# headroom-deploy

Despliegue del proxy [Headroom](https://github.com/headroomlabs-ai/headroom) en el VPS, detrás del edge (Traefik).

- URL: https://headroom.andersoncastellon.com
- Contenedor: `headroom-proxy` (imagen `ghcr.io/headroomlabs-ai/headroom:code`), sin puertos publicados.
- Red del edge: `edge-headroom` (alias `headroom-web`, puerto 8787).
- Ruta de Traefik: `headroom.yml` (se instala en `/opt/edge/dynamic/` vía `edge-ctl`).

## Archivos

| Archivo | Qué es |
|---|---|
| `docker-compose.yml` | Servicio del proxy |
| `headroom.yml` | Ruta de Traefik para el dominio |
| `.env.example` | Plantilla del secreto (el `.env` real vive solo en el VPS) |

## Despliegue

En el VPS:

```bash
git clone git@github.com:<usuario>/headroom-deploy.git ~/headroom
cd ~/headroom
echo "HEADROOM_PROXY_TOKEN=$(openssl rand -hex 32)" > .env && chmod 600 .env
/opt/edge/scripts/edge-ctl network add headroom   # una sola vez
docker compose up -d
cd /opt/edge
./scripts/edge-ctl route validate headroom ~/headroom/headroom.yml
./scripts/edge-ctl route install headroom ~/headroom/headroom.yml
```

## Uso desde Claude Code

```bash
ANTHROPIC_BASE_URL=https://headroom.andersoncastellon.com \
ANTHROPIC_CUSTOM_HEADERS="X-Headroom-Proxy-Token: <TOKEN>" claude
```

El token está en `~/headroom/.env` en el VPS. Claude Code manda `Authorization` con tu login, por eso el token va en `X-Headroom-Proxy-Token`.

## Seguridad

- El proxy ve todo el contexto que envías a Claude. Solo úsalo en un VPS que controlas.
- Sin `HEADROOM_ALLOW_UNAUTHENTICATED_BIND`: el proxy exige token para tráfico que no sea loopback.
- Telemetría desactivada (`HEADROOM_TELEMETRY=off`).
