# Session Log

Historial de cambios realizados por Claude Code en este proyecto.

---

## Sesión 2026-05-22

### CLAUDE.md — Inicialización
- Se creó el archivo `CLAUDE.md` con documentación del proyecto para futuras sesiones: comandos de ejecución, arquitectura general, flujo de seguridad, encoding de IDs y variables de entorno.

### Redis Cache en `/recent-episodes`
**Archivos modificados:** `api.py`, `requirements.txt`, `.env`

**Qué se implementó:**
- Cache en Redis para el endpoint `GET /recent-episodes` con TTL de 2 horas (configurable).
- Mientras el cache esté vigente, no se ejecuta ningún request al Miruro pipe (`_fetch_raw_recents()`).
- Si Redis no está disponible, el endpoint sigue funcionando sin cache (fallos silenciosos con `try/except`).

**Variables de entorno agregadas a `.env`:**
| Variable | Valor por defecto | Propósito |
|---|---|---|
| `REDIS_HOST` | `localhost` | Host del servidor Redis |
| `REDIS_PORT` | `6379` | Puerto Redis |
| `REDIS_PASSWORD` | — | Contraseña Redis |
| `CACHE_RECENT_EPISODES_HOURS` | `2` | Duración del cache en horas (cambiar aquí para ajustar el TTL) |

**Key en Redis:**
```
miruro_api:cache:recent_episodes
```

Comandos para inspeccionar el cache:
```bash
# Ver si existe y su contenido
redis-cli -h <HOST> -p <PORT> -a <PASSWORD> GET miruro_api:cache:recent_episodes

# Ver segundos restantes de vida
redis-cli -h <HOST> -p <PORT> -a <PASSWORD> TTL miruro_api:cache:recent_episodes

# Invalidar manualmente (fuerza refetch en el próximo request)
redis-cli -h <HOST> -p <PORT> -a <PASSWORD> DEL miruro_api:cache:recent_episodes
```

**Dependencia agregada a `requirements.txt`:** `redis` (v4+ incluye asyncio de forma nativa; no se necesita `redis[asyncio]`).

**Estructura del código en `api.py`:**
```python
# Constantes (al inicio del archivo, junto a la configuración)
CACHE_RECENT_EPISODES_HOURS = int(os.getenv("CACHE_RECENT_EPISODES_HOURS", "2"))
CACHE_RECENT_EPISODES_TTL   = CACHE_RECENT_EPISODES_HOURS * 3600
REDIS_KEY_RECENT_EPISODES   = "miruro_api:cache:recent_episodes"

redis_client = aioredis.Redis(host=..., port=..., password=..., decode_responses=True)
```

---

## Sesión 2026-07-03

### Servicio systemd para auto-inicio en boot
**Archivos modificados:** `README.md` (nuevo item "Deploy (production — systemd service, auto-start on boot)")
**Archivo creado (fuera del repo):** `/etc/systemd/system/mi-api.service`

**Qué se implementó:**
- Unidad systemd (`mi-api.service`) que reemplaza el arranque manual con `nohup ... &` tras un reinicio de la máquina. Activa el venv (`source venv/bin/activate`) y ejecuta `python -m uvicorn api:app --host 0.0.0.0 --port 8848` con `exec`, redirigiendo salida a un log timestamped (`uvicorn-$(date +%F-%H%M%S).log`) dentro del repo, igual convención que el flujo manual del README.
- `Type=simple` + `exec` en vez de `nohup`/`&`: bajo systemd no hacen falta (systemd ya desacopla el proceso de la terminal) y romperían el tracking del PID si se usaran.
- `Restart=on-failure` + `RestartSec=5`: reinicio automático ante crash.
- `WantedBy=multi-user.target` + `systemctl enable`: arranque automático en cada boot.

**Incidente al activar:** al iniciar el servicio por primera vez, hubo ~9 reinicios en bucle por `Errno 98: address already in use` — un proceso `nohup` manual previo seguía ocupando el puerto 8848. Se resolvió al matar ese proceso; el servicio quedó estable con un solo proceso corriendo bajo systemd. Este caso ya quedó documentado como advertencia en el README.

**Comandos clave:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable mi-api.service
sudo systemctl start mi-api.service
sudo systemctl status mi-api.service
journalctl -u mi-api -f
```

---

## Sesión 2026-07-06

### [TEMPORAL] Forzar `type: mp4` → `type: hls` solo en `/watch/{provider}/{anilist_id}/{category}/{slug}`
**Archivo modificado:** `api.py`
**Estado:** ACTIVO. Pensado para eliminarse en el futuro — instrucciones exactas de reversión más abajo.

**Motivo:** ajuste puntual solicitado por el usuario para el caso `/watch/ally/178789/sub/allmanga-2` (y cualquier otra ruta `/watch/...`): algunos streams del pipe llegan con `"type": "mp4"` y se necesita que temporalmente se reporten como `"type": "hls"`. No se toca ningún otro valor de `type` (p. ej. `"embed"` se deja intacto).

**Qué se implementó:**
- Se agregó la función `_force_mp4_to_hls_temp(data)` en `api.py`, justo antes de `get_watch_sources` (línea ~916, antes del decorador `@app.get("/watch/{provider}/{anilist_id}/{category}/{slug}")`). Recorre `data["streams"]` y donde `stream["type"] == "mp4"` lo reemplaza por `"hls"`.
- Se modificó el `return` final de `get_watch_sources` (antes: `return await get_sources(...)`) para que capture el resultado en `result` y le aplique `_force_mp4_to_hls_temp(result)` antes de devolverlo.
- **Alcance:** el cambio vive únicamente dentro de `get_watch_sources`. El endpoint `/sources` (llamado directo, sin pasar por el slug de `/watch/...`) NO se ve afectado — se verificó en vivo que sigue devolviendo los streams sin modificar.

**Diff aplicado (para referencia exacta):**
```python
# ANTES (dentro de get_watch_sources, última línea de la función):
    return await get_sources(episodeId=target_id, provider=provider, anilistId=anilist_id, category=category)

# DESPUÉS:
def _force_mp4_to_hls_temp(data: dict) -> dict:
    """TEMP (ver SESSION_LOG.md): fuerza streams con type=mp4 a hls. Borrar esta función
    completa y su única llamada en get_watch_sources para revertir. No toca "embed" ni "hls"."""
    for stream in data.get("streams", []):
        if stream.get("type") == "mp4":
            stream["type"] = "hls"
    return data

@app.get("/watch/{provider}/{anilist_id}/{category}/{slug}")
async def get_watch_sources(...):
    ...
    result = await get_sources(episodeId=target_id, provider=provider, anilistId=anilist_id, category=category)
    return _force_mp4_to_hls_temp(result)  # TEMP: quitar esta línea para revertir (ver SESSION_LOG.md)
```

**Cómo revertir (cuando el usuario lo pida, ej. "elimina el ajuste temporal de mp4 a hls"):**
1. En `api.py`, borrar la función completa `_force_mp4_to_hls_temp` (las 6 líneas, desde `def _force_mp4_to_hls_temp(data: dict) -> dict:` hasta su `return data`).
2. En `get_watch_sources`, reemplazar las dos líneas:
   ```python
   result = await get_sources(episodeId=target_id, provider=provider, anilistId=anilist_id, category=category)
   return _force_mp4_to_hls_temp(result)  # TEMP: quitar esta línea para revertir (ver SESSION_LOG.md)
   ```
   por la línea original:
   ```python
   return await get_sources(episodeId=target_id, provider=provider, anilistId=anilist_id, category=category)
   ```
3. Borrar esta sección del `SESSION_LOG.md` (o marcarla como "revertido" con fecha).
4. No hay variables de entorno, dependencias ni otros archivos involucrados — es autocontenido en `api.py`.

**Verificación tras revertir:** `curl` a `/watch/ally/178789/sub/allmanga-2` con el `x-api-key` correcto debe volver a mostrar `"type": "mp4"` en los streams que originalmente lo traían así (ya no todos como `"hls"`).
