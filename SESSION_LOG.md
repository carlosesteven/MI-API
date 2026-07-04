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
