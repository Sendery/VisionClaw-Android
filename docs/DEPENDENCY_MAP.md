# VisionClaw Android — Mapa de Dependencias

> Referencia rápida de dependencias entre módulos, user stories y componentes externos.

---

## Dependencias entre User Stories

| User Story | Depende de | Bloquea a |
|------------|------------|-----------|
| US-1: Setup Base | — | US-2, US-3, US-4 |
| US-2: WebSocket Gemini | US-1 | US-5, US-6 |
| US-3: Audio Captura | US-1 | US-5 |
| US-4: Cámara CameraX | US-1 | US-6 |
| US-5: Audio Bidireccional | US-2, US-3 | US-7 |
| US-6: Video → Gemini | US-2, US-4 | US-7 |
| US-7: Sesión Teléfono | US-5, US-6 | US-8, US-10 |
| US-8: Tool Calls OpenClaw | US-7 | US-9 |
| US-9: E2E OpenClaw | US-8 | US-12 |
| US-10: DAT SDK Gafas | US-7 | US-11 |
| US-11: Sesión Gafas | US-10 | US-12 |
| US-12: Testing | US-7, US-9 | — |

---

## Dependencias de Módulos Internos

```
┌─────────────────────────────────────────────────────────────┐
│                        UI Layer                             │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────────┐    │
│  │MainScreen│  │SessionScreen │  │  SessionViewModel  │    │
│  └────┬─────┘  └──────┬───────┘  └─────────┬──────────┘    │
│       │               │                    │                │
│       └───────────────┼────────────────────┘                │
│                       │ usa                                  │
└───────────────────────┼─────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────────┐
        │               │                   │
        ▼               ▼                   ▼
┌──────────────┐ ┌──────────────┐  ┌──────────────────┐
│GeminiLive    │ │Audio         │  │Camera            │
│Service       │ │Capture/Play  │  │Phone/Glasses     │
├──────────────┤ ├──────────────┤  ├──────────────────┤
│ - connect()  │ │ - start()    │  │ - startCapture() │
│ - sendAudio()│ │ - stop()     │  │ - stopCapture()  │
│ - sendVideo()│ │ - mute()     │  │ - onFrame()      │
│ - disconnect │ │              │  │                  │
└──────┬───────┘ └──────────────┘  └──────────────────┘
       │
       │ usa
       ▼
┌──────────────────┐
│ToolCallRouter    │──── usa ────▶ OpenClawBridge
│ - handleToolCall │               │ - executeTask()
└──────────────────┘               └──────────────────┘
```

---

## Dependencias Externas (APIs y SDKs)

| Dependencia | Tipo | Requerido para | Configuración |
|-------------|------|---------------|---------------|
| **Gemini Live API** | WebSocket | US-2, US-5, US-6 | API key en `GeminiConfig.kt` |
| **Google AI Studio** | Web | Obtener API key | https://aistudio.google.com/apikey |
| **OpenClaw Gateway** | HTTP REST | US-8, US-9 | Host + port + token en `GeminiConfig.kt` |
| **Meta DAT SDK** | Maven (GitHub Packages) | US-10, US-11 | GitHub token con `read:packages` |
| **Meta Ray-Ban Glasses** | Bluetooth | US-11 | Developer Mode habilitado |

---

## Dependencias de Librerías (build.gradle.kts)

| Librería | Versión | Uso | User Story |
|----------|---------|-----|------------|
| `okhttp3:okhttp` | 4.12.0 | WebSocket Gemini + HTTP OpenClaw | US-2, US-8 |
| `okhttp3:logging-interceptor` | 4.12.0 | Debug de requests HTTP | US-2, US-8 |
| `moshi:moshi-kotlin` | 1.15.1 | JSON serialización/deserialización | US-2 |
| `moshi:moshi-kotlin-codegen` | 1.15.1 | Generación de adaptadores JSON | US-2 |
| `camera-core/camera2/lifecycle/view` | 1.3.x | Captura de cámara (Phone mode) | US-4 |
| `hilt-android` | 2.51.1 | Inyección de dependencias | Todos |
| `compose-bom` | 2024.x | UI framework | US-7 |
| `navigation-compose` | 2.8.x | Navegación entre pantallas | US-7 |
| `kotlinx-coroutines` | 1.8.x | Pipelines async audio/video | US-3, US-5 |
| `mwdat-core` | 0.4.0 | DAT SDK core (opcional) | US-10 |
| `mwdat-camera` | 0.4.0 | DAT SDK camera (opcional) | US-10 |

---

## Paralelización Posible

Estas stories se pueden trabajar en paralelo (sin dependencias entre sí):

### Bloque 1 (tras US-1):
- US-2: WebSocket Gemini
- US-3: Audio Captura
- US-4: Cámara CameraX

### Bloque 2 (tras US-7):
- US-8: Tool Calls OpenClaw
- US-10: DAT SDK Gafas

---

## Configuración Requerida por Entorno

### Desarrollo local (mínimo):
```
✅ Android Studio Flamingo+
✅ JDK 17
✅ Emulador API 29+ (o dispositivo físico)
✅ Gemini API key (gratis)
```

### Testing con audio/cámara:
```
✅ Dispositivo Android físico (emulador tiene limitaciones de audio)
✅ Gemini API key
```

### Testing con OpenClaw:
```
✅ Todo lo anterior
✅ Servidor OpenClaw desplegado (local o remoto)
✅ URL + token configurados en GeminiConfig.kt
```

### Testing con gafas Meta Ray-Ban:
```
✅ Todo lo anterior
✅ GitHub token con read:packages (para DAT SDK)
✅ Gafas Meta Ray-Ban con Developer Mode
✅ Meta AI app instalada en el teléfono
   ─ o ─
✅ MockDeviceKit para testing sin gafas físicas
```
