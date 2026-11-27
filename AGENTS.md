# AGENTS.md — TTS

Servicio de texto a voz local del NAS. Contexto operativo para futuras sesiones.

## Qué es

- `docker-compose.yml`: dos contenedores.
  - `kokoro-tts` → `ghcr.io/remsky/kokoro-fastapi-cpu:latest`, puerto NAS **8890** → 8880 interno. API compatible con OpenAI.
  - `tts-web` → `nginx:alpine`, puerto NAS **8891** → 80. Sirve `web/` y hace de proxy `/api/` → `kokoro:8880` (evita CORS).
- `web/index.html`: interfaz propia, HTML+CSS+JS puro, sin build. Trocea el texto
  en frases de ≤1.200 caracteres y concatena los MP3.
- `nginx/nginx.conf`: proxy y timeouts largos (la CPU tarda en textos largos).
- `nginx/Dockerfile`: hornea `nginx.conf` + `index.html` en la imagen `tts-web:local`.
  **No** se usan bind mounts: las ACL del volumen de UGOS hacían que nginx no
  pudiera leer los ficheros (403 Permission denied), aunque estuvieran en 777.

## Reglas

- **No** exponer los puertos fuera de la LAN sin autenticación: la API no tiene
  clave y acepta cualquier petición.
- El NAS no tiene GPU: no cambiar a imagen `-gpu`. Kokoro CPU va sobrado; los
  modelos con clonación (XTTS-v2, Chatterbox) no son viables aquí.
- El límite de 3 GB (`mem_limit`) es deliberado: el NAS tiene 8 GB compartidos
  con Immich, Plex, n8n y opencode.
- Los textos largos se generan **en serie** desde el navegador; no paralelizar
  (el N100 tiene 4 núcleos y se satura).
- Cualquier cambio en `web/` o `nginx/` requiere reconstruir la imagen:
  `sudo docker compose up -d --build` (no hay montajes, es a propósito).
- Despliegue y verificación: `sudo docker compose up -d --build` y comprobar
  `curl -s http://192.168.1.41:8890/health`.
