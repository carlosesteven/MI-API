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
