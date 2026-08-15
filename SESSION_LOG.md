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

---

## Sesión 2026-08-15

### Diagnóstico: `/watch/kiwi/209983/sub/animepahe-6` falla/tarda ~60s
**Investigación, sin cambios de código en esta parte.**

- Se descartó colisión de slugs: los IDs de episodios de `kiwi/sub` para el anime 209983 son únicos y bien formados (`animepahe:6772:77466:1` para el episodio 6).
- Se aisló el problema llamando directamente a `get_sources()`: **todo** episodio del provider `kiwi` (backend `animepahe`) falla con `503 Pipe unavailable` tras ~60s (dos intentos de `_pipe_get` de ~30s cada uno, sin timeout explícito configurado). Otros providers del mismo anime (`ally`, `pewe`) resuelven bien en <0.5s.
- Conclusión: no es un bug de `api.py` — la integración de Miruro con `animepahe` está caída/colgándose del lado de ellos. El código no tiene timeout explícito en `pipe_session.get()`, por lo que en vez de fallar rápido, el usuario espera ~60s.
- Pendiente (no implementado): agregar `timeout=` explícito a las llamadas del pipe en `_pipe_get` para que este tipo de fallas retornen en segundos, no en un minuto.

### Feature: `BLOCKED_EPISODE_PREFIXES` — ocultar providers rotos de `/episodes`
**Archivos modificados:** `api.py`, `.env`, `CLAUDE.md`

**Qué se implementó (no es un ajuste temporal, es una feature permanente):**
- Nueva env var `BLOCKED_EPISODE_PREFIXES` (comma-separated), parseada como set en `api.py` (junto a la config de cache, ~línea 42).
- Nueva función `_filter_blocked_prefixes(data)` (justo antes de `_inject_source_slugs`): recorre `data["providers"][*]["episodes"][*]` y elimina cualquier episodio cuyo `id.split(":")[0]` esté en `BLOCKED_EPISODE_PREFIXES`. No-op si la env var está vacía.
- Se llama dentro de `get_episodes()` (`GET /episodes/{anilist_id}`), justo después de `_fetch_raw_episodes` y antes de `_inject_source_slugs` (tiene que ir antes porque `_inject_source_slugs` reescribe `ep["id"]` al formato slug `watch/...`, perdiendo el prefijo original).
- **Nota:** el provider (ej. `kiwi`) sigue apareciendo en la respuesta, pero con la lista de episodios vacía si todos sus IDs matchean el prefijo bloqueado. No se oculta el provider completo, solo los episodios cuyo prefijo esté bloqueado — así si un provider mezcla varios backends, solo se filtra el roto.

**Cómo usarlo (agregar más providers rotos en el futuro):**
```bash
# En .env, separar por coma:
BLOCKED_EPISODE_PREFIXES=animepahe,otroprovider
```
No requiere tocar código ni reiniciar nada más que el proceso (para recargar el `.env`).

**Estado actual:** `BLOCKED_EPISODE_PREFIXES=animepahe` activo en `.env`, por el problema de arriba. Quitar `animepahe` de esa lista (o vaciar la variable) cuando Miruro arregle su integración con animepahe — verificado en local que con la variable seteada, `kiwi/sub` en `/episodes/209983` devuelve 0 episodios y el resto de providers no se ven afectados.

**Nota:** este cambio NO toca `/watch/{provider}/.../{slug}` — si alguien pega directamente una URL vieja `/watch/kiwi/.../animepahe-X`, ese endpoint sigue intentando resolverla y tardará los ~60s de antes. Solo se filtra en el listado de `/episodes`.

### Fix: `timeUntilAiring` desactualizado en `/recent-episodes`
**Archivo modificado:** `api.py`

**Motivo:** la app Android (Kotlin) del usuario consume `/recent-episodes` y filtra episodios "recién salidos" comparando el campo `timeUntilAiring` contra un umbral muy chico (`api_miruro_recents_timestamp` en Remote Config, default 50 segundos). Se detectó que el `timeUntilAiring` que devuelve el pipe de Miruro (`path: "schedule"`) viene de un snapshot que Miruro cachea de su lado y no recalcula seguido — se verificó pidiendo `/recent-episodes` dos veces separadas por horas reales y la respuesta fue **byte-idéntica** (mismo MD5), con `timeUntilAiring` de un episodio que ya había salido hace horas todavía en positivo. Con un umbral de 50s, el filtro de la app casi nunca se cumplía hasta que el cache de Miruro se refrescaba (aparentemente 1 vez al día), por eso los episodios "recién salidos" solo aparecían casi al final del día.

**Qué se implementó:**
- Se duplicó el endpoint original tal cual estaba en `/recent-episodes-old` (función `get_recent_episodes_old`), sin ningún cambio — referencia/rollback rápido si hiciera falta comparar comportamiento.
- Se agregó `_recompute_time_until_airing(data)` (api.py, antes de `get_recent_episodes`): recorre la lista top-level de `/recent-episodes` y sobreescribe `item["timeUntilAiring"] = item["airingAt"] - int(time.time())` para cada item. `airingAt` sí es estable (viene de AniList, no se mueve), así que el recálculo queda exacto al segundo sin depender de qué tan viejo esté el snapshot de Miruro.
- Se llama **al servir la respuesta**, tanto si viene de cache Redis como si es fetch fresco — nunca se guarda el valor recalculado en cache, solo se cachea la data cruda (que sí puede quedarse cacheada 2h sin problema, ya que `airingAt`/título/imagen no cambian seguido).
- Se agregó `import time` a los imports de `api.py`.
- **No cambia el shape del JSON** (mismo campo, mismo nombre, mismo tipo int, puede quedar negativo si ya pasó — la app Kotlin lee `getInt("timeUntilAiring")` y compara con `<=`, funciona igual sin ningún cambio de la app). Verificado en local: `/recent-episodes-old` sigue devolviendo el valor viejo congelado (positivo), `/recent-episodes` devuelve el valor correcto (negativo, ~-20000s para un episodio que ya salió hace ~5.7h) — se comparó campo por campo que ningún otro dato cambió entre ambos endpoints, solo `timeUntilAiring`.

**Pendiente / no incluido en este cambio:** no se tocó `media.nextAiringEpisode.timeUntilAiring` (campo anidado) — la app Kotlin no lo usa para el filtro, solo lee `nextAiringEpisode.episode` para el número de episodio, así que no hacía falta.

### Filtro server-side: solo formato `TV` en `/recent-episodes`
**Archivo modificado:** `api.py`

**Motivo:** el único consumidor de `/recent-episodes` es la app Android, que ya descarta client-side todo lo que no sea `format == "TV"` (ver Kotlin del usuario). Confirmado con el usuario que no hay otro cliente (web, etc.) usando este endpoint que necesite los demás formatos. De 124 items que trae Miruro, solo 72 son `TV` — el resto (`ONA`, `TV_SHORT`, `MOVIE`, `SPECIAL`, `MUSIC`) se descartaban igual del lado del cliente, era payload/parseo desperdiciado.

**Qué se implementó:**
- Se agregó `_filter_tv_format(data)` (api.py, antes de `get_recent_episodes`): list comprehension que se queda solo con items donde `media.format == "TV"`.
- Se aplica dentro de `get_recent_episodes()`, después de leer de cache o de fetch fresco, antes de `_recompute_time_until_airing`. **No se filtra en `/recent-episodes-old`** — ese endpoint sigue devolviendo los 124 items con todos los formatos, intacto, como copia de referencia.
- No afecta el cache en Redis: se sigue cacheando la data cruda completa (los 124 items sin filtrar), el filtro se aplica solo al servir la respuesta — así si en el futuro se necesita otro formato desde otro endpoint/consumidor, la data completa sigue disponible sin re-pedirle a Miruro.

**Verificado en local:** `/recent-episodes` → 72 items, todos `format: "TV"`. `/recent-episodes-old` → 124 items, formatos mixtos (`TV`, `ONA`, `TV_SHORT`, `MOVIE`, `SPECIAL`, `MUSIC`), sin cambios.

### Fix: la app siempre mostraba "último episodio - 1"
**Archivo modificado:** `api.py`

**Motivo:** el usuario reportó que su app Android siempre muestra el episodio anterior al real. Su Kotlin calcula el episodio a mostrar así:
```kotlin
episodeAux = episode.getJSONObject("nextAiringEpisode").getInt("episode")
if (episodeAux > 1) episodeAux -= 1
```
Asume que `media.nextAiringEpisode.episode` siempre apunta al episodio siguiente al que ya salió (de ahí el `-1`). Se verificó sobre los 72 items `TV` de `/recent-episodes-old` que esto **no siempre es cierto**: 37/72 sí traían `nextAiringEpisode.episode == episode + 1` (la resta da bien), pero **34/72 traían `nextAiringEpisode.episode == episode`** (mismo valor, sin avanzar) — ahí la resta deja a la app mostrando un episodio menos del real. 1 item no tenía `nextAiringEpisode` (show `FINISHED`), ese caso ya lo maneja bien el Kotlin sin restar nada.

Es la misma raíz que el bug de `timeUntilAiring`: el snapshot que cachea Miruro no avanza `nextAiringEpisode` en sincronía con la hora real de emisión — para shows cuyo episodio salió después de que Miruro tomó su snapshot, ese campo se queda pegado en el mismo número que `episode` en vez de avanzar al siguiente. Ejemplo verificado: BLACK TORCH ep7, `nextAiringEpisode.episode` venía en `7` (igual, no `8`) — la app mostraba episodio 6.

**Qué se implementó:**
- Se agregó `_fix_next_airing_episode(data)` (api.py, antes de `get_recent_episodes`): para cada item, fuerza `media.nextAiringEpisode.episode = item["episode"] + 1`, sin importar el valor que traiga Miruro. No toca items sin `nextAiringEpisode` (los deja como vienen).
- Se llama al final de la cadena en `get_recent_episodes()` (después de `_filter_tv_format` y `_recompute_time_until_airing`), tanto en el camino de cache-hit como en el de fetch fresco. `/recent-episodes-old` no se toca.

**Verificado en local:** de los 72 items `TV`, los 72 quedan con `nextAiringEpisode.episode == episode + 1` (0 incorrectos). BLACK TORCH: `episode: 7`, `nextAiringEpisode.episode: 8` → la app calcularía `8 - 1 = 7`, correcto.
