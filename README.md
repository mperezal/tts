# TTS — Texto a voz local (Kokoro-82M)

Servicio de text-to-speech **gratuito y local** para el NAS (UGREEN DXP2800,
Intel N100, sin GPU). Nada de texto sale de la red de casa.

## Qué corre aquí

| Servicio | Contenedor | Puerto | Para qué |
|---|---|---|---|
| Kokoro-FastAPI | `kokoro-tts` | **8890** | API compatible con OpenAI + su propia web (`/web`) |
| Interfaz propia | `tts-web` | **8891** | Página en español para pegar texto y descargar MP3 |

- Modelo: [Kokoro-82M](https://huggingface.co/hexgrad/Kokoro-82M) (82 M de parámetros, Apache 2.0).
- Voces en español: `ef_dora` (mujer), `em_alex` (hombre), `em_santa` (hombre, grave).
- RAM en reposo: ~0,3 GB · generando: ~1,5 GB. Límite del contenedor: 3 GB.
- Tras 10 min sin uso libera el modelo (`MODEL_AUTO_UNLOAD_TIMEOUT_SECONDS`).
- Velocidad en N100: **más rápido que tiempo real**. Un módulo de 10 min de audio
  se genera en 1–3 min.

## Uso

### Interfaz web

http://192.168.1.41:8891

Pega el texto, elige voz y velocidad, «Generar audio», descarga el MP3.
Los textos largos se trocean en frases de ~1.200 caracteres, se sintetizan en
serie y se unen en **un único MP3**. `Ctrl+Enter` genera directamente.

Si pegas HTML (por ejemplo una entrega del taller formativo), marca
«Limpiar HTML del texto pegado» y se queda solo lo narrable.

> La web se sirve desde la imagen `tts-web:local` (ver `nginx/Dockerfile`), sin
> montajes: tras editar `web/index.html` hay que reconstruir con
> `sudo docker compose up -d --build`.

### API (compatible con OpenAI)

```bash
curl -X POST http://192.168.1.41:8890/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"model":"kokoro","input":"Hola, esto es una prueba.","voice":"ef_dora","response_format":"mp3","speed":1.0}' \
  --output voz.mp3
```

Python, con el SDK oficial:

```python
from openai import OpenAI

client = OpenAI(base_url="http://192.168.1.41:8890/v1", api_key="no-hace-falta")
with client.audio.speech.with_streaming_response.create(
    model="kokoro", voice="ef_dora", input="Hola, esto es una prueba."
) as r:
    r.stream_to_file("voz.mp3")
```

Formatos de salida: `mp3`, `wav`, `opus`, `flac`, `aac`, `pcm`.
Documentación interactiva del motor: http://192.168.1.41:8890/docs

### n8n

Nodo **HTTP Request**:

- Method: `POST`
- URL: `http://192.168.1.41:8890/v1/audio/speech`
  (desde el contenedor de n8n, si no resuelve la IP: `http://<nombre-contenedor-kokoro>:8880/v1/audio/speech`)
- Body (JSON): `{"model":"kokoro","input":"{{ $json.texto }}","voice":"ef_dora","response_format":"mp3"}`
- Response: *File* → luego subirlo a Telegram, Drive, etc.

## Gestión

```bash
cd /volume1/docker/opencode2/TTS
sudo docker compose ps
sudo docker compose logs -f kokoro
sudo docker compose up -d --build        # aplicar cambios en web/ o nginx/
sudo docker compose restart
sudo docker compose down
sudo docker compose pull && sudo docker compose up -d   # actualizar Kokoro
```

El primer arranque descarga las voces (~50 ficheros) y tarda un par de minutos;
hasta entonces `/health` responde pero la síntesis falla con 503.

## Notas

- Los MP3 se concatenan a nivel de bytes; funcionan en cualquier reproductor.
  Si algún día hace falta un MP3 perfectamente sellado (metadatos, duración
  exacta), se remuxa con ffmpeg en el NAS.
- La web se sirve con nginx y hace de proxy `/api/` → `kokoro:8880` para
  evitar problemas de CORS.
- **Ampliar a clonar voz**: Kokoro no clona. Para eso haría falta XTTS-v2
  (licencia no comercial, lento en CPU) o una GPU. Ver `nas/docs/`.
