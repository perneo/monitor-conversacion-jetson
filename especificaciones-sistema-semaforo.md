# Sistema de Monitorización de Tono Conversacional — Especificaciones Técnicas Completas

**Versión:** 1.0  
**Estado:** Especificación para desarrollo  
**Fecha:** Febrero 2026

---

## Índice

1. [Visión general del sistema](#1-visión-general-del-sistema)
2. [Arquitectura global](#2-arquitectura-global)
3. [Componentes del sistema](#3-componentes-del-sistema)
4. [Modelo ASR — Voxtral](#4-modelo-asr--voxtral)
5. [Modelo de análisis — Qwen3-8B](#5-modelo-de-análisis--qwen3-8b)
6. [Backend de orquestación](#6-backend-de-orquestación)
7. [Frontend — Interfaz del semáforo](#7-frontend--interfaz-del-semáforo)
8. [Infraestructura y despliegue](#8-infraestructura-y-despliegue)
9. [Prompt del sistema de análisis](#9-prompt-del-sistema-de-análisis)
10. [Flujos de datos detallados](#10-flujos-de-datos-detallados)
11. [Gestión de estados y máquina de estados](#11-gestión-de-estados-y-máquina-de-estados)
12. [Parámetros de configuración](#12-parámetros-de-configuración)
13. [Consideraciones de privacidad y seguridad](#13-consideraciones-de-privacidad-y-seguridad)
14. [Requisitos de hardware](#14-requisitos-de-hardware)

---

## 1. Visión general del sistema

### 1.1 Propósito

El sistema monitoriza en tiempo real el tono de una conversación presencial captada por micrófono. A través de una interfaz visual minimalista muestra un semáforo que indica si la conversación se desarrolla con cordialidad (verde) o si está derivando hacia un tono inapropiado (rojo), con un estado intermedio de advertencia (amarillo) antes de que se produzca el deterioro.

El sistema no almacena ningún fragmento de conversación. Opera exclusivamente en memoria durante la sesión activa y descarta el audio y el texto transcrito una vez analizado.

### 1.2 Principios de diseño

- **No intrusivo.** Una vez iniciado, no requiere interacción de los participantes.
- **Periférico.** La interfaz está pensada para ser visible desde distancia, como una señal de sala, no para ser leída activamente.
- **Neutral.** No identifica hablantes, no registra, no acusa. Evalúa el tono colectivo.
- **On-premise total.** Ningún dato sale de la infraestructura propia. Sin APIs externas.
- **Sin persistencia.** No existe base de datos. La conversación no se guarda en ningún formato.

### 1.3 Caso de uso principal

Una sala de reuniones dispone de un micrófono ambiente conectado a un ordenador que ejecuta el sistema. En una pantalla visible para todos los participantes se muestra la interfaz del semáforo. Los participantes son conscientes de que el sistema está activo. El sistema monitoriza el tono durante toda la reunión sin intervención humana.

---

## 2. Arquitectura global

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SALA DE REUNIONES                            │
│                                                                     │
│   🎤 Micrófono ambiente                                             │
│        │                                                            │
│        ▼                                                            │
│   [ Navegador Web ]                                                 │
│   AudioWorklet (16kHz, PCM16, mono)                                 │
│        │                                                            │
│        │  WebSocket (WSS) — stream de audio                         │
└────────┼────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    SERVIDOR ON-PREMISE (GPU)                        │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │              Backend FastAPI (Orquestador)                  │   │
│   │                                                             │   │
│   │  • Gestión de sesiones                                      │   │
│   │  • Buffer de contexto (ventana deslizante 90s)              │   │
│   │  • Programador de análisis (cada ~20s o por .done)          │   │
│   │  • Máquina de estados del semáforo                          │   │
│   │  • Emisión de eventos al frontend                           │   │
│   │                                                             │   │
│   └────────────┬────────────────────────┬───────────────────────┘   │
│                │                        │                           │
│                ▼                        ▼                           │
│   ┌────────────────────┐   ┌────────────────────────────────────┐   │
│   │  vLLM instancia 1  │   │       vLLM instancia 2             │   │
│   │                    │   │                                    │   │
│   │  Voxtral Mini 4B   │   │  Qwen3-8B-Instruct                 │   │
│   │  Realtime 2602     │   │  (Non-Thinking Mode, FP8)          │   │
│   │                    │   │                                    │   │
│   │  /v1/realtime      │   │  /v1/chat/completions              │   │
│   │  (WebSocket)       │   │  (HTTP, JSON structured)           │   │
│   └────────────────────┘   └────────────────────────────────────┘   │
│                                                                     │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   │  WebSocket (WSS) — eventos
                                   ▼
                          [ Navegador Web ]
                          Interfaz semáforo (Svelte)
```

### 2.1 Separación de responsabilidades

| Componente | Responsabilidad única |
|---|---|
| AudioWorklet | Captura y conversión de audio a PCM16 |
| Backend FastAPI | Orquestación, buffer, estado, routing |
| Voxtral (vLLM #1) | Transcripción ASR en tiempo real |
| Qwen3-8B (vLLM #2) | Análisis de tono y emisión de juicio |
| Frontend Svelte | Visualización del estado del semáforo |

---

## 3. Componentes del sistema

### 3.1 Lista de componentes

| ID | Componente | Tecnología | Puerto |
|---|---|---|---|
| C1 | Captura de audio | Web Audio API + AudioWorklet | — |
| C2 | Frontend interfaz | Svelte (compilado, sin runtime) | 3000 |
| C3 | Backend orquestador | Python 3.12 + FastAPI + Uvicorn | 8080 |
| C4 | Servidor ASR | vLLM (Voxtral) | 8001 |
| C5 | Servidor análisis | vLLM (Qwen3-8B) | 8002 |
| C6 | Proxy inverso | Nginx | 443 / 80 |

### 3.2 Dependencias entre componentes

```
C1 (AudioWorklet)
  └──► C3 (Backend) via WebSocket /ws/audio
         ├──► C4 (Voxtral) via WebSocket /v1/realtime
         │      └── transcription.delta / .done ──► C3 (Buffer)
         │                                              └── cada trigger
         │                                                    └──► C5 (Qwen3) via HTTP POST
         │                                                               └── JSON juicio ──► C3
         └──► C2 (Frontend) via WebSocket /ws/events
                └── { type: "transcript.delta" | "semaphore.update" }
```

---

## 4. Modelo ASR — Voxtral

### 4.1 Especificación del modelo

| Parámetro | Valor |
|---|---|
| Modelo | `mistralai/Voxtral-Mini-4B-Realtime-2602` |
| Parámetros | ~4B (3.4B LM + 0.6B audio encoder) |
| Formato de pesos | BF16 |
| VRAM requerida | 16 GB mínimo |
| Idiomas soportados | 13 (árabe, alemán, inglés, español, francés, hindi, italiano, holandés, portugués, chino, japonés, coreano, ruso) |
| Latencia transcripción | < 500ms con delay de 480ms |
| Arquitectura | Causal + sliding window attention (streaming nativo) |

### 4.2 Comando de arranque vLLM

```bash
VLLM_DISABLE_COMPILE_CACHE=1 vllm serve mistralai/Voxtral-Mini-4B-Realtime-2602 \
  --port 8001 \
  --compilation_config '{"cudagraph_mode": "PIECEWISE"}' \
  --max-model-len 131072 \
  --dtype bfloat16
```

**Notas sobre los parámetros:**

- `--max-model-len 131072` soporta sesiones de hasta ~3 horas de audio continuo (1 token = 80ms).
- `VLLM_DISABLE_COMPILE_CACHE=1` requerido por la arquitectura de Voxtral en la versión actual.
- El endpoint `/v1/realtime` se activa automáticamente al servir este modelo.

### 4.3 Dependencias de sistema

Las siguientes librerías deben estar instaladas en el entorno Python del servidor vLLM de Voxtral:

```
vllm (nightly, desde https://wheels.vllm.ai/nightly)
mistral_common >= 1.9.0
soxr
librosa
soundfile
```

### 4.4 Formato de audio esperado

| Parámetro | Valor |
|---|---|
| Sample rate | 16.000 Hz |
| Canales | 1 (mono) |
| Codificación | PCM signed 16-bit little-endian |
| Transporte | Base64 dentro de JSON WebSocket |
| Tamaño de chunk | 7.680 samples (480ms a 16kHz) |

### 4.5 Protocolo WebSocket con Voxtral

Secuencia de mensajes que el backend mantiene con Voxtral:

```
Backend → Voxtral:  (conexión WebSocket)
Voxtral → Backend:  { "type": "session.created" }
Backend → Voxtral:  { "type": "session.update", "model": "mistralai/Voxtral-Mini-4B-Realtime-2602" }
Backend → Voxtral:  { "type": "input_audio_buffer.commit" }
Backend → Voxtral:  { "type": "input_audio_buffer.append", "audio": "<base64 PCM16>" }
  ... (se repite por cada chunk de 480ms)
Voxtral → Backend:  { "type": "transcription.delta", "delta": "texto parcial" }
Voxtral → Backend:  { "type": "transcription.done" }
  ... (ciclo continuo)
Backend → Voxtral:  { "type": "input_audio_buffer.commit", "final": true }  (al cerrar sesión)
```

### 4.6 Configuración del delay de transcripción

El delay de 480ms es el valor por defecto recomendado por Mistral como punto óptimo entre latencia y precisión. Si se requiere ajuste, se modifica el campo `transcription_delay_ms` en el archivo `params.json` (tekken.json) del modelo antes de servir:

| Delay | Latencia | Precisión | Caso de uso |
|---|---|---|---|
| 240ms | Muy baja | Menor | Casos donde la latencia es crítica |
| 480ms | Baja | Alta | **Valor recomendado para este sistema** |
| 960ms | Media | Mayor | Si se prioriza precisión sobre latencia |
| 2400ms | Alta | Máxima | Equivalente a modelo offline |

---

## 5. Modelo de análisis — Qwen3-8B

### 5.1 Especificación del modelo

| Parámetro | Valor |
|---|---|
| Modelo | `Qwen/Qwen3-8B-Instruct` |
| Parámetros | 8B |
| Formato de pesos | FP8 (si GPU Ada Lovelace/Hopper) / BF16 (resto) |
| VRAM requerida | ~8GB (FP8) / ~16GB (BF16) |
| Idiomas soportados | 100+ idiomas y dialectos |
| Modo de operación | **Non-Thinking Mode obligatorio** |
| Contexto máximo | 32.768 tokens (nativo) |
| Ventana de análisis | ~90 segundos de conversación (~500-800 tokens típicos) |

### 5.2 Comando de arranque vLLM

```bash
vllm serve Qwen/Qwen3-8B-Instruct \
  --port 8002 \
  --dtype float8 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.85
```

En GPU sin soporte FP8 (anterior a Ada Lovelace):

```bash
vllm serve Qwen/Qwen3-8B-Instruct \
  --port 8002 \
  --dtype bfloat16 \
  --max-model-len 32768
```

### 5.3 Configuración de la llamada de análisis

Cada llamada al modelo de análisis se realiza vía HTTP POST a `/v1/chat/completions` con los siguientes parámetros fijos:

```json
{
  "model": "Qwen/Qwen3-8B-Instruct",
  "temperature": 0.0,
  "max_tokens": 150,
  "response_format": { "type": "json_object" },
  "chat_template_kwargs": { "enable_thinking": false },
  "messages": [
    { "role": "system", "content": "<PROMPT_SISTEMA — ver sección 9>" },
    { "role": "user",   "content": "<BUFFER_CONTEXTO — ver sección 10.2>" }
  ]
}
```

**Parámetros críticos:**

- `temperature: 0.0` — eliminación de variabilidad. El juicio debe ser determinista para el mismo input.
- `enable_thinking: false` — desactiva el modo de razonamiento de Qwen3. Sin este parámetro el modelo genera bloques `<think>...</think>` que incrementan la latencia en cientos de tokens innecesarios.
- `max_tokens: 150` — el JSON de respuesta ocupa típicamente 60-80 tokens. 150 garantiza margen sin desperdiciar cómputo.
- `response_format: json_object` — fuerza output JSON válido, evita texto adicional alrededor de la respuesta.

### 5.4 Estructura del JSON de respuesta

El modelo devuelve siempre este JSON (garantizado por `response_format` y el prompt):

```json
{
  "estado": "verde",
  "puntuacion": 85,
  "tendencia": "estable",
  "razon": ""
}
```

| Campo | Tipo | Valores posibles | Descripción |
|---|---|---|---|
| `estado` | string | `"verde"` \| `"rojo"` | Juicio binario del modelo |
| `puntuacion` | integer | 0 — 100 | Cordialidad (100 = máxima) |
| `tendencia` | string | `"mejorando"` \| `"estable"` \| `"empeorando"` | Evolución dentro del fragmento |
| `razon` | string | Texto ≤ 15 palabras | Solo se muestra si estado es `"rojo"`. Vacío si verde |

**Nota:** El campo `razon` está en el idioma detectado de la conversación. El prompt instruye al modelo a responder en el mismo idioma que el fragmento analizado.

---

## 6. Backend de orquestación

### 6.1 Responsabilidades

El backend FastAPI es el núcleo del sistema. Gestiona todos los flujos de datos y es el único componente que conoce el estado global de la sesión.

Sus responsabilidades son:

- Aceptar la conexión WebSocket de audio desde el frontend
- Reenviar el audio a Voxtral y recibir los deltas de transcripción
- Mantener el buffer de contexto conversacional
- Programar y ejecutar las llamadas de análisis a Qwen3
- Mantener la máquina de estados del semáforo
- Emitir eventos de actualización al frontend
- Gestionar el ciclo de vida completo de la sesión

### 6.2 Endpoints

| Método | Endpoint | Descripción |
|---|---|---|
| `WebSocket` | `/ws/audio` | Recibe stream de audio PCM16 en base64 desde el frontend |
| `WebSocket` | `/ws/events` | Emite eventos de transcripción y semáforo al frontend |
| `GET` | `/health` | Health check para monitorización |
| `GET` | `/status` | Estado actual de la sesión (para diagnóstico) |

### 6.3 Buffer de contexto conversacional

El buffer es una estructura de datos en memoria (no persistida) que mantiene los fragmentos de texto transcritos con sus marcas temporales.

**Estructura lógica del buffer:**

```
Buffer {
  fragmentos: [
    { texto: "llevamos dos horas sin llegar a nada", timestamp: T-87s },
    { texto: "porque nadie escucha las propuestas",  timestamp: T-71s },
    { texto: "eso no es verdad, todo el mundo ha tenido turno", timestamp: T-54s },
    ...
  ],
  ventana_segundos: 90,
  ultimo_analisis: timestamp
}
```

**Reglas del buffer:**

- Los fragmentos más antiguos que `ventana_segundos` (90s por defecto) se eliminan automáticamente al añadir nuevos.
- El buffer no supera nunca los 90 segundos de conversación.
- Si el buffer está vacío (silencio prolongado), no se lanza análisis.
- El buffer se destruye completamente al finalizar la sesión.

### 6.4 Lógica de disparo del análisis

El análisis se lanza bajo dos condiciones, la primera que se cumpla:

**Condición A — Por evento:** Cada vez que Voxtral emite `transcription.done`, si han pasado más de 15 segundos desde el último análisis.

**Condición B — Por tiempo:** Si no ha habido `transcription.done` en los últimos 20 segundos pero el buffer contiene fragmentos nuevos no analizados.

Esta lógica evita tanto el análisis excesivo (por cada delta de Voxtral) como los períodos ciegos en conversaciones con frases muy largas.

### 6.5 Lógica de transición del semáforo

El backend aplica histéresis para evitar parpadeos del semáforo ante evaluaciones oscilantes:

```
puntuacion > 65  →  estado VERDE
puntuacion 40-65 →  estado AMARILLO (advertencia, sin razon visible)
puntuacion < 40  →  estado ROJO (con razon visible)
```

**Reglas adicionales de estabilización:**

- Para pasar de VERDE a ROJO se requieren **2 evaluaciones consecutivas** con puntuación < 40. Una sola evaluación negativa no cambia el estado (evita falsos positivos por frases aisladas).
- Para pasar de ROJO a VERDE se requiere **1 sola evaluación** con puntuación > 65 (recuperación inmediata cuando el tono mejora).
- El estado AMARILLO no tiene regla de confirmación: transiciona inmediatamente en ambas direcciones.

### 6.6 Eventos emitidos al frontend

El backend emite mensajes JSON por el WebSocket `/ws/events`:

```json
// Actualización de transcripción (alta frecuencia)
{
  "type": "transcript.delta",
  "text": "fragmento de texto recién transcrito",
  "timestamp_ms": 1234567890
}

// Cambio de estado del semáforo (baja frecuencia, solo cuando cambia)
{
  "type": "semaphore.update",
  "estado": "rojo",
  "puntuacion": 32,
  "tendencia": "empeorando",
  "razon": "Tono descalificativo sostenido en los últimos intercambios"
}

// Estado del sistema
{
  "type": "system.status",
  "connected": true,
  "session_seconds": 342
}

// Error
{
  "type": "system.error",
  "message": "Conexión con Voxtral perdida",
  "recoverable": true
}
```

### 6.7 Gestión del ciclo de vida de la sesión

**Al iniciar sesión:**
1. Frontend se conecta a `/ws/audio` y `/ws/events`
2. Backend inicia conexión WebSocket con Voxtral en `/v1/realtime`
3. Backend inicializa buffer vacío y estado VERDE
4. Backend emite `system.status` de confirmación al frontend

**Durante la sesión:**
- Audio fluye de Frontend → Backend → Voxtral de forma continua
- Deltas de texto fluyen de Voxtral → Backend → Frontend de forma continua
- Análisis de Qwen3 se ejecutan periódicamente según lógica de disparo
- Estado del semáforo se emite solo cuando cambia

**Al finalizar sesión (cierre limpio):**
1. Frontend envía señal de cierre
2. Backend envía `{ type: "input_audio_buffer.commit", final: true }` a Voxtral (libera KV cache)
3. Backend destruye el buffer en memoria
4. Backend cierra conexión con Voxtral
5. Sesión destruida, sin persistencia de ningún dato

**Al finalizar sesión (cierre abrupto — pestaña cerrada):**
1. Frontend envía `{ type: "input_audio_buffer.commit", final: true }` en `beforeunload`
2. Backend detecta desconexión del WebSocket de audio
3. Backend ejecuta limpieza idéntica al cierre limpio
4. Timeout de seguridad de 5 segundos antes de liberar recursos si no hay señal de beforeunload

---

## 7. Frontend — Interfaz del semáforo

### 7.1 Tecnología

| Componente | Tecnología | Justificación |
|---|---|---|
| Framework | Svelte | Bundle mínimo sin runtime, ideal para UI reactiva con actualizaciones de alta frecuencia |
| Audio capture | Web Audio API + AudioWorklet | Procesado en hilo dedicado de tiempo real, sin interferencia del hilo UI |
| Comunicación | WebSocket nativo del navegador | Sin dependencias adicionales |
| Estilos | CSS custom properties | Sin frameworks CSS externos |

El resultado final es un bundle de pocos kilobytes servido como HTML+JS estático por Nginx.

### 7.2 Arquitectura del frontend

El frontend se compone de dos piezas que operan en hilos separados:

**Hilo de audio (AudioWorklet):**
- Archivo independiente `audio-processor.worklet.js`
- Corre en Audio Worklet Global Scope (hilo de tiempo real)
- Recibe bloques de 128 samples de la Web Audio API
- Acumula hasta completar un chunk de 480ms (7.680 samples a 16kHz)
- Convierte Float32 → Int16 con clamp a [-1, 1] para evitar overflow
- Transfiere el ArrayBuffer al hilo principal via MessagePort (zero-copy)

**Hilo principal (Svelte):**
- Recibe el ArrayBuffer del worklet
- Codifica en Base64 y envía por WebSocket `/ws/audio`
- Recibe eventos por WebSocket `/ws/events`
- Actualiza el estado reactivo de la UI

### 7.3 Captura de audio

```
getUserMedia({ audio: { channelCount: 1, sampleRate: 16000,
                         echoCancellation: true,
                         noiseSuppression: true,
                         autoGainControl: true } })
  └── AudioContext({ sampleRate: 16000 })
        ├── createMediaStreamSource(stream)
        │     ├── connect(AnalyserNode)    ← para visualizador de volumen
        │     └── connect(AudioWorkletNode "voxtral-processor")
        │               └── port.onmessage → base64 → WebSocket
        └── audioWorklet.addModule("audio-processor.worklet.js")
```

El `AudioContext` a 16kHz hace que el motor de audio del navegador resamplee automáticamente desde el sample rate nativo del micrófono (típicamente 44.1kHz o 48kHz) usando su resampler interno de calidad WebRTC, superior a cualquier interpolación manual en JavaScript.

### 7.4 Pantalla única — tres zonas

#### Zona A — Semáforo (40-50% de la pantalla)

- Círculo grande, centrado, dominante visualmente
- Tres colores: verde `#22c55e`, amarillo `#f59e0b`, rojo `#ef4444`
- Transición animada entre estados: fundido de 1.5 segundos (evita parpadeo brusco)
- Debajo del círculo: texto de `razon` — visible solo en estado ROJO
- Texto de `razon` con tipografía grande y legible desde distancia

#### Zona B — Transcripción en tiempo real (35% de la pantalla)

- Scroll automático al contenido más reciente
- Texto con horizonte temporal: los fragmentos con más de 90 segundos de antigüedad se desvanecen gradualmente con transición CSS opacity
- Sin etiquetas de hablante (el sistema no identifica quién habla)
- Fuente monoespacio o sans-serif de alta legibilidad
- Cursor parpadeante al final del texto mientras Voxtral está produciendo output

#### Zona C — Estado del sistema (15% de la pantalla)

- Indicador de nivel de audio (barra de volumen reactiva)
- Tiempo de sesión transcurrido (formato MM:SS)
- Indicador de conectividad (punto verde/rojo discreto)
- Botón "Finalizar sesión" en posición discreta (esquina inferior)

### 7.5 Estados de la aplicación

| Estado | Descripción | Semáforo | Transcripción | Zona C |
|---|---|---|---|---|
| `IDLE` | Pantalla de inicio | No visible | No visible | Solo botón inicio |
| `CONNECTING` | Estableciendo conexión | Gris animado | No visible | Spinner |
| `ACTIVE` | Sesión en curso | Verde/Amarillo/Rojo | Visible y actualizable | Completa |
| `ERROR` | Conexión perdida | Icono de error | Congelada | Mensaje + reconexión |
| `FINISHING` | Cerrando sesión | Gris fundido | Congelada | "Cerrando..." |

### 7.6 Pantalla de configuración (pre-sesión)

Antes de iniciar, se muestra una pantalla mínima con:

- Campo URL del backend (default: `wss://localhost:8080`)
- Selector de sensibilidad del semáforo (Baja / Media / Alta), que mapea a los umbrales de puntuación
- Botón "Iniciar sesión"

Una vez iniciada la sesión, esta configuración no es accesible. No existe botón de pausa ni de silencio durante la sesión activa.

### 7.7 Gestión de reconexión en frontend

Si se pierde la conexión WebSocket con el backend:

1. El semáforo muestra estado de error (icono, no color)
2. El frontend intenta reconectar con backoff exponencial: 1s, 2s, 4s, 8s, 16s
3. Máximo 5 intentos de reconexión automática
4. Si se supera el límite, se muestra mensaje de error con botón de reconexión manual

### 7.8 Comportamiento en `beforeunload`

Cuando el usuario cierra la pestaña o el navegador:

```javascript
window.addEventListener("beforeunload", () => {
  // Señal de cierre ordenado a Voxtral via backend
  // Libera KV cache en vLLM, evita sesiones zombie
  ws_audio.send(JSON.stringify({ type: "session.close" }))
})
```

---

## 8. Infraestructura y despliegue

### 8.1 Configuración Nginx

Nginx actúa como proxy inverso gestionando TLS y enrutando las conexiones:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl;
    server_name semaforo.local;

    ssl_certificate     /etc/ssl/certs/semaforo.crt;
    ssl_certificate_key /etc/ssl/private/semaforo.key;

    # Frontend estático
    location / {
        root /var/www/semaforo;
        try_files $uri $uri/ /index.html;
    }

    # WebSocket de audio y eventos
    location /ws/ {
        proxy_pass         http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade    $http_upgrade;
        proxy_set_header   Connection $connection_upgrade;
        proxy_set_header   Host       $host;
        # Crítico: sin este timeout las conexiones largas se cortan a los 60s
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }

    # Health check del backend
    location /health {
        proxy_pass http://localhost:8080/health;
    }
}
```

**Por qué HTTPS es obligatorio:** El navegador solo permite acceso a `getUserMedia()` (micrófono) desde orígenes seguros (HTTPS o localhost). Sin HTTPS la captura de audio es imposible.

### 8.2 Estructura de puertos

| Servicio | Puerto interno | Expuesto externamente |
|---|---|---|
| Nginx | 443 (HTTPS) | Sí — punto de entrada único |
| FastAPI backend | 8080 | No — solo via Nginx |
| vLLM Voxtral | 8001 | No — solo accesible por backend |
| vLLM Qwen3 | 8002 | No — solo accesible por backend |

### 8.3 Orden de arranque de servicios

Los servicios deben iniciarse en este orden:

1. vLLM con Voxtral (tarda varios minutos en cargar el modelo)
2. vLLM con Qwen3 (tarda menos, puede iniciarse en paralelo)
3. FastAPI backend (hace health check de ambos vLLM antes de aceptar conexiones)
4. Nginx (puede estar activo desde el principio, responderá 503 hasta que el backend esté listo)

### 8.4 Variables de entorno del backend

```bash
# URLs de los servidores vLLM
VOXTRAL_WS_URL=ws://localhost:8001/v1/realtime
QWEN_HTTP_URL=http://localhost:8002/v1/chat/completions

# Modelos
VOXTRAL_MODEL=mistralai/Voxtral-Mini-4B-Realtime-2602
QWEN_MODEL=Qwen/Qwen3-8B-Instruct

# Configuración del buffer y análisis
BUFFER_WINDOW_SECONDS=90
MIN_ANALYSIS_INTERVAL_SECONDS=15
MAX_ANALYSIS_INTERVAL_SECONDS=20

# Umbrales del semáforo
THRESHOLD_GREEN=65
THRESHOLD_RED=40
CONFIRMATIONS_FOR_RED=2

# WebSocket
WS_MAX_RECONNECT_ATTEMPTS=5
SESSION_IDLE_TIMEOUT_SECONDS=300
```

---

## 9. Prompt del sistema de análisis

Este es el prompt completo que recibe Qwen3-8B como mensaje de sistema en cada llamada de análisis.

```
Eres un sistema de análisis de tono conversacional. Tu única función es evaluar 
si una conversación se está desarrollando de forma cordial o si está derivando 
hacia un tono inaceptable.

IDIOMA: Analiza el fragmento en el idioma en que esté escrito. No traduzcas.
Devuelve el campo "razon" en el mismo idioma que la conversación.

TAREA: Recibirás un fragmento de conversación reciente con marcas temporales 
relativas. Los fragmentos están ordenados cronológicamente del más antiguo al 
más reciente. Los fragmentos más recientes tienen mayor peso en tu evaluación.

---

CRITERIOS PARA TONO INACEPTABLE (rojo, puntuacion < 40):
- Insultos explícitos o implícitos dirigidos a personas presentes
- Descalificaciones personales ("eres un incompetente", "no sabes nada")
- Amenazas directas o veladas de cualquier tipo
- Tono despectivo sostenido durante más de una intervención consecutiva
- Sarcasmo agresivo o humillante dirigido a una persona
- Negaciones absolutas repetidas sobre la integridad del otro ("estás mintiendo", 
  "eso es una mentira")
- Escalada progresiva de agresividad aunque ninguna frase aislada sea grave

CRITERIOS PARA ZONA DE ADVERTENCIA (amarillo, puntuacion 40-65):
- Una frase de frustración o descalificación aislada sin escalada
- Tono tenso pero sin insulto directo
- Debate agresivo que no llega a descalificación personal
- Señales iniciales de escalada (interrupción, negación frecuente)

CRITERIOS PARA TONO ACEPTABLE (verde, puntuacion > 65):
- Desacuerdo expresado con respeto ("no estoy de acuerdo porque...")
- Debate técnico intenso sin descalificación personal
- Tono directo pero sin agresividad hacia personas
- Frustración expresada sin insulto ("esto no funciona", "no podemos seguir así")
- Fragmentos neutros o de silencio
- Una sola frase negativa aislada sin contexto de escalada

---

REGLAS CRÍTICAS:
1. Una frase aislada negativa NO es suficiente para rojo. Debe haber patrón o escalada.
2. El desacuerdo técnico enérgico NO es tono inaceptable.
3. La frustración sin insulto NO es tono inaceptable.
4. En caso de duda, devuelve verde. Es preferible un falso negativo a un falso positivo.
5. Evalúa la conversación colectiva, no a personas individuales.
6. Si el fragmento contiene silencio o es muy corto, devuelve verde con puntuacion 80.

---

FORMATO DE RESPUESTA:
Devuelve ÚNICAMENTE el siguiente JSON. Sin texto adicional. Sin markdown. 
Sin explicaciones fuera del JSON. El JSON debe ser válido y parseable.

{
  "estado": "verde" | "rojo",
  "puntuacion": <entero entre 0 y 100, donde 100 es máxima cordialidad>,
  "tendencia": "mejorando" | "estable" | "empeorando",
  "razon": "<frase de máximo 15 palabras explicando el estado, en el idioma 
             de la conversación. Vacío si estado es verde>"
}

El campo "tendencia" refleja cómo ha evolucionado el tono a lo largo del 
fragmento completo, no solo el último momento.
```

---

## 10. Flujos de datos detallados

### 10.1 Flujo de audio (captura → transcripción)

```
1. Micrófono captura audio continuo
2. getUserMedia() entrega stream al AudioContext (16kHz)
3. AudioWorkletNode "voxtral-processor" recibe bloques de 128 samples
4. Acumula hasta 7.680 samples (480ms)
5. Convierte Float32 → Int16 (con clamp [-1,1])
6. Transfiere ArrayBuffer (zero-copy) al hilo principal
7. Hilo principal codifica en Base64
8. Envía JSON { type: "input_audio_buffer.append", audio: "<b64>" }
   por WebSocket /ws/audio al backend
9. Backend reenvía a Voxtral manteniendo sesión activa
10. Voxtral procesa y emite transcription.delta y transcription.done
11. Backend recibe, añade al buffer con timestamp, reenvía al frontend
```

### 10.2 Formato del mensaje de usuario a Qwen3

En cada llamada de análisis, el backend construye el mensaje de usuario a partir del buffer:

```
[hace 87s] "llevamos dos horas sin llegar a nada concreto"
[hace 71s] "porque nadie escucha las propuestas que se hacen"
[hace 54s] "eso no es verdad, aquí todo el mundo ha tenido su turno"
[hace 38s] "pues yo no lo he visto así, francamente"
[hace 20s] "mira, si vamos a seguir así prefiero dejarlo para otro día"
[hace 6s]  "perfecto, como siempre, huir del problema"
```

Las marcas temporales relativas permiten al modelo detectar la tendencia de forma natural, sin lógica adicional en el backend.

### 10.3 Flujo de análisis (buffer → juicio → semáforo)

```
1. Trigger de análisis (por evento o por tiempo, ver sección 6.4)
2. Backend construye mensaje de usuario con fragmentos del buffer + timestamps
3. HTTP POST a Qwen3 /v1/chat/completions (con enable_thinking: false)
4. Qwen3 responde con JSON { estado, puntuacion, tendencia, razon }
5. Backend aplica lógica de histéresis (ver sección 6.5):
   a. puntuacion > 65 → estado VERDE interno
   b. puntuacion 40-65 → estado AMARILLO interno
   c. puntuacion < 40 → incrementa contador de confirmaciones
      - Si contador >= 2 → estado ROJO interno
      - Si contador < 2 → mantiene estado actual, espera siguiente análisis
6. Si el estado interno ha cambiado respecto al anterior:
   → Emite { type: "semaphore.update", ... } al frontend por /ws/events
7. Si el estado no ha cambiado: no se emite nada (evita tráfico innecesario)
```

---

## 11. Gestión de estados y máquina de estados

### 11.1 Máquina de estados del semáforo (backend)

```
                    ┌─────────────────────────────────┐
                    │                                 │
              ┌─────▼──────┐                          │
   inicio ───►│   VERDE    │◄─── puntuacion > 65      │
              └─────┬──────┘     (1 evaluación)       │
                    │                                 │
              puntuacion 40-65                        │
                    │                                 │
              ┌─────▼──────┐                          │
              │  AMARILLO  │◄──► transición inmediata │
              └─────┬──────┘     en ambas direcciones │
                    │                                 │
              puntuacion < 40                         │
              (primer aviso)                          │
                    │                                 │
              ┌─────▼──────┐                          │
              │ PENDIENTE  │  (estado interno)        │
              │    ROJO    │                          │
              └─────┬──────┘                          │
                    │                                 │
              puntuacion < 40                         │
              (segundo aviso)                         │
                    │                                 │
              ┌─────▼──────┐                          │
              │    ROJO    │──── puntuacion > 65 ─────┘
              └────────────┘     (1 evaluación suficiente
                                  para volver a verde)
```

### 11.2 Máquina de estados de la sesión (frontend)

```
IDLE ──► CONNECTING ──► ACTIVE ──► FINISHING ──► IDLE
              │              │
              │              └──► ERROR ──► CONNECTING (reconexión)
              │                       └──► IDLE (máx. intentos superados)
              └──► IDLE (fallo de conexión inicial)
```

---

## 12. Parámetros de configuración

### 12.1 Parámetros de sensibilidad del semáforo

Estos parámetros son configurables antes de iniciar sesión. La UI ofrece tres presets:

| Preset | `THRESHOLD_GREEN` | `THRESHOLD_RED` | `CONFIRMATIONS_FOR_RED` | Caso de uso |
|---|---|---|---|---|
| Baja | 55 | 30 | 3 | Entornos donde el debate intenso es normal (negociaciones, debates) |
| **Media (default)** | **65** | **40** | **2** | **Reuniones de trabajo estándar** |
| Alta | 75 | 50 | 1 | Entornos donde se requiere máxima cordialidad (atención al cliente, mediación) |

### 12.2 Parámetros del buffer y análisis

| Parámetro | Default | Rango | Descripción |
|---|---|---|---|
| `BUFFER_WINDOW_SECONDS` | 90 | 30-180 | Ventana de texto que se mantiene en el buffer |
| `MIN_ANALYSIS_INTERVAL_SECONDS` | 15 | 10-30 | Mínimo tiempo entre análisis |
| `MAX_ANALYSIS_INTERVAL_SECONDS` | 20 | 15-60 | Máximo tiempo sin análisis si hay contenido nuevo |

### 12.3 Parámetros de audio

| Parámetro | Valor | Modificable |
|---|---|---|
| Sample rate | 16.000 Hz | No |
| Canales | 1 (mono) | No |
| Chunk size | 480ms / 7.680 samples | Sí (80ms a 2400ms) |
| Echo cancellation | Activado | No |
| Noise suppression | Activado | No |
| Auto gain control | Activado | No |

---

## 13. Consideraciones de privacidad y seguridad

### 13.1 Modelo de privacidad

- **Sin persistencia.** Ningún fragmento de audio, texto transcrito, ni juicio de análisis se almacena en disco o base de datos.
- **Sin telemetría.** El sistema no envía datos a servicios externos. Toda la inferencia es local (on-premise).
- **Destrucción inmediata.** Al finalizar la sesión, el buffer en memoria se destruye y los KV caches de ambos modelos se liberan.
- **Sin identificación.** El sistema no identifica, etiqueta ni asocia texto a personas individuales.

### 13.2 Consentimiento

Se recomienda que los participantes de cualquier conversación monitorizada sean informados explícitamente de que el sistema está activo. La pantalla visible con el semáforo sirve como recordatorio permanente, pero no sustituye al consentimiento informado previo.

### 13.3 Seguridad de red

- Toda comunicación usa TLS (HTTPS/WSS). El certificado puede ser autofirmado en entornos de red local cerrada.
- Los puertos de vLLM (8001, 8002) no están expuestos fuera del servidor. Solo el backend accede a ellos.
- El puerto 8080 del backend tampoco está expuesto directamente. Solo Nginx hace proxy hacia él.
- No hay autenticación de usuario implementada en la especificación base. Si el sistema se despliega en red con múltiples usuarios, se debe añadir autenticación en Nginx antes del WebSocket.

---

## 14. Requisitos de hardware

### 14.1 Configuración mínima (GPU 24GB)

Una única GPU de 24GB VRAM puede alojar ambos modelos simultáneamente:

| Modelo | VRAM (FP8) | VRAM (BF16) |
|---|---|---|
| Voxtral Mini 4B | ~16GB (BF16, no soporta FP8) | 16GB |
| Qwen3-8B FP8 | ~8GB | — |
| **Total** | **~24GB** | **~32GB** |

GPU recomendada para configuración mínima: **NVIDIA L40S (48GB)** o **NVIDIA A100 40GB** si se usa BF16 para ambos, o **RTX 4090 (24GB)** con Qwen3 en FP8.

### 14.2 Configuración recomendada (2 GPUs)

Para mayor estabilidad y sin competencia de VRAM entre modelos:

| GPU | Modelo asignado | VRAM |
|---|---|---|
| GPU 0 | Voxtral Mini 4B (BF16) | 16GB usados |
| GPU 1 | Qwen3-8B (FP8 o BF16) | 8-16GB usados |

### 14.3 Requisitos de CPU y memoria RAM

| Recurso | Mínimo | Recomendado |
|---|---|---|
| CPU | 8 cores | 16 cores |
| RAM | 32GB | 64GB |
| Almacenamiento | 50GB SSD | 100GB NVMe |
| Red local | 100 Mbps | 1 Gbps |

### 14.4 Software de sistema

| Componente | Versión mínima |
|---|---|
| Ubuntu | 22.04 LTS |
| CUDA | 12.4+ |
| Python | 3.12 |
| NVIDIA Driver | 550+ |
| Docker (opcional) | 26+ |

---

*Fin del documento de especificaciones — versión 1.0*
