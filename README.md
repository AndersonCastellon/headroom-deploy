# headroom-deploy

Despliegue del proxy [Headroom](https://github.com/headroomlabs-ai/headroom) en un VPS, detrás de Traefik (edge).

- URL: `https://headroom.andersoncastellon.com`
- Contenedor: `headroom-proxy` (imagen `ghcr.io/headroomlabs-ai/headroom:code`), sin puertos publicados.
- Redes: `edge-headroom` (Traefik, interna) y `headroom_default` (salida a internet).
- Ruta de Traefik: `headroom.yml`, instalada en `/opt/edge/dynamic/` con `edge-ctl`.

## Archivos

| Archivo | Qué es |
|---|---|
| `docker-compose.yml` | Servicio del proxy |
| `headroom.yml` | Ruta de Traefik del dominio |
| `.env.example` | Plantilla del secreto (el `.env` real vive solo en el VPS) |

## Despliegue en el VPS

```bash
git clone https://github.com/AndersonCastellon/headroom-deploy.git ~/headroom
cd ~/headroom
echo "HEADROOM_PROXY_TOKEN=$(openssl rand -hex 32)" > .env && chmod 600 .env
/opt/edge/scripts/edge-ctl network add headroom          # una sola vez
docker compose up -d
cd /opt/edge
./scripts/edge-ctl route validate headroom ~/headroom/headroom.yml
./scripts/edge-ctl route install headroom ~/headroom/headroom.yml
```

Comprueba que el contenedor esté `healthy` con `docker ps`.

## Verificación

```bash
TOKEN=$(grep HEADROOM_PROXY_TOKEN ~/headroom/.env | cut -d= -f2)   # en el VPS
curl -s -o /dev/null -w "%{http_code}\n" https://headroom.andersoncastellon.com/dashboard                       # 401
curl -s -o /dev/null -w "%{http_code}\n" -H "X-Headroom-Proxy-Token: $TOKEN" https://headroom.andersoncastellon.com/dashboard  # 200
```

Sin token responde `401`. Con token, `200`.

## Configurar Claude Code en tu PC

Claude Code necesita dos variables. Ponlas en `~/.claude/settings.json`, bajo `env`, que tiene prioridad sobre la shell:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://headroom.andersoncastellon.com",
    "ANTHROPIC_CUSTOM_HEADERS": "X-Headroom-Proxy-Token: <TOKEN>"
  }
}
```

- `<TOKEN>` es el valor de `HEADROOM_PROXY_TOKEN` en `~/headroom/.env` del VPS. **No lo subas a git ni lo pegues en el README.**
- Usa `chmod 600 ~/.claude/settings.json` para que solo tu usuario lo lea.
- Si también defines `ANTHROPIC_BASE_URL` en `~/.bashrc` o `environment.d`, mantén los mismos valores o elimina los antiguos. `settings.json` gana, pero conviene que no haya dos fuentes distintas.

Para probar sin tocar `settings.json`, usa un archivo temporal con `--settings`:

```bash
claude -p --settings /ruta/temporal.json "responde solo: OK"
```

Después borra ese archivo, porque contiene el token.

## Cómo comprobar que pasa por el proxy

En el VPS:

```bash
docker logs -f headroom-proxy | grep -E "v1/messages|PERF"
```

Cada petición de Claude Code debe aparecer como `proxy_inbound_request ... path=/v1/messages` seguida de `status=200`, y una línea `PERF` con `tok_before`, `tok_after` y `tok_saved`.

## Problemas conocidos

- **Claude Code responde pero no aparece nada en el log:** la variable no llega al proceso. Revisa `settings.json`, porque sus valores ganan sobre la shell.
- **502 con `Temporary failure in name resolution`:** el contenedor no tiene red de salida. Debe estar en `headroom_default` además de `edge-headroom`.
- **401 `x-api-key header is required`:** la petición llegó a Anthropic sin credenciales. Revisa que Claude Code esté autenticado.
- **401 `unauthorized` del proxy:** falta el header `X-Headroom-Proxy-Token` o el token no coincide.

## Seguridad

- El proxy ve todo el contexto que envías a Claude. Solo úsalo en un VPS que controlas.
- El proxy exige token para tráfico que no sea loopback. No definas `HEADROOM_ALLOW_UNAUTHENTICATED_BIND`.
- Telemetría desactivada (`HEADROOM_TELEMETRY=off`).
- Si el token se expone, genera uno nuevo en el VPS (`openssl rand -hex 32`), actualiza `.env`, reinicia con `docker compose up -d` y actualiza `settings.json` en tu PC.
