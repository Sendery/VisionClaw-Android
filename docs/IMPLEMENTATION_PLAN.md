# VisionClaw Android — Plan de Implementación

> Plan de trabajo desglosado en User Stories validables, con dependencias, tareas incrementales y criterios de aceptación.

---

## Índice

1. [Visión General del Proyecto](#visión-general-del-proyecto)
2. [Estado Actual del Repositorio](#estado-actual-del-repositorio)
3. [Mapa de Dependencias](#mapa-de-dependencias)
4. [Epic 1: Gemini Live WebSocket](#epic-1-gemini-live-websocket)
5. [Epic 2: Pipeline de Audio](#epic-2-pipeline-de-audio)
6. [Epic 3: Captura de Cámara (Teléfono)](#epic-3-captura-de-cámara-teléfono)
7. [Epic 4: Integración OpenClaw](#epic-4-integración-openclaw)
8. [Epic 5: ViewModel y UI](#epic-5-viewmodel-y-ui)
9. [Epic 6: Meta Ray-Ban (DAT SDK)](#epic-6-meta-ray-ban-dat-sdk)
10. [Epic 7: Testing y Estabilización](#epic-7-testing-y-estabilización)
11. [Costes Recurrentes](#costes-recurrentes)
12. [Cronograma Resumen](#cronograma-resumen)

---

## Visión General del Proyecto

VisionClaw Android es el port Android de [VisionClaw iOS](https://github.com/sseanliu/VisionClaw): un asistente de IA en tiempo real para gafas inteligentes Meta Ray-Ban que usa:

- **Gemini Live API** (WebSocket bidireccional) para conversación de voz + visión
- **Meta DAT SDK Android** (`meta-wearables-dat-android` v0.4.0) para streaming de vídeo desde las gafas
- **OpenClaw** (opcional) como gateway HTTP a 56+ skills de acción en el mundo real

### Flujo Principal

```
Gafas Meta Ray-Ban (o cámara del teléfono)
       │
       │ Bluetooth / CameraX
       ▼
📱 App Android (Kotlin + Jetpack Compose)
  ├── DAT SDK (StreamSession → VideoFrames) o CameraX (Phone mode)
  ├── AudioRecord (micrófono 16kHz PCM)
  ├── GeminiLiveService (OkHttp WebSocket)
  │      │
  │      ├── wss://generativelanguage.googleapis.com ← Gemini Live API
  │      │         │
  │      │    Gemini procesa voz + visión
  │      │         │
  │      ├── Audio respuesta (24kHz PCM) → AudioTrack → altavoz
  │      └── Tool calls → OpenClawBridge
  │                              │
  │                         HTTPS POST
  │                              ▼
  │                    https://tu-dominio.com
  │                     (servidor OpenClaw)
  │                              │
  │                        OpenClaw Gateway
  │                              │
  │                      56+ skills ejecutándose
  └──────────────────────────────┘
```

---

## Estado Actual del Repositorio

| Módulo | Archivo | Estado | Notas |
|--------|---------|--------|-------|
| **Gemini** | `GeminiConfig.kt` | ✅ Completo | API key, modelo, URLs, system prompt |
| | `GeminiModels.kt` | ✅ Completo | Data classes Moshi para protocolo WebSocket |
| | `GeminiLiveService.kt` | ✅ Completo | Cliente WebSocket OkHttp implementado |
| **Audio** | `AudioCaptureManager.kt` | ✅ Completo | AudioRecord 16kHz + AEC |
| | `AudioPlaybackManager.kt` | ✅ Completo | AudioTrack 24kHz streaming |
| **Cámara** | `PhoneCameraManager.kt` | ✅ Completo | CameraX back camera, throttle 1fps |
| | `GlassesCameraManager.kt` | ⚠️ Stub | Requiere Meta DAT SDK |
| **OpenClaw** | `OpenClawBridge.kt` | ✅ Completo | HTTP client OkHttp |
| | `ToolCallModels.kt` | ✅ Completo | Declaración de tool "execute" |
| | `ToolCallRouter.kt` | ✅ Completo | Routing de tool calls |
| **UI** | `MainScreen.kt` | ✅ Completo | Permisos + selección de modo |
| | `SessionScreen.kt` | ✅ Completo | Preview + controles de sesión |
| | `SessionViewModel.kt` | ✅ Completo | Orquestación de ciclo de vida |
| **DI** | `AppModule.kt` | ✅ Completo | Hilt providers (OkHttp + Moshi) |
| **Utils** | `ImageUtil.kt` | ✅ Completo | YUV_420_888 → NV21 → JPEG |
| | `AudioUtil.kt` | ✅ Completo | Base64 encode/decode |
| **App** | `MainActivity.kt` | ✅ Completo | Navigation setup |
| | `VisionClawApp.kt` | ✅ Completo | Hilt Application |

**Resumen**: 15 de 16 archivos están implementados. Solo `GlassesCameraManager.kt` queda como stub pendiente de integración con el Meta DAT SDK.

---

## Mapa de Dependencias

```
                    ┌──────────────────────┐
                    │  US-1: Setup Base    │
                    │  (Build + Emulador)  │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │ US-2: Gemini │  │ US-3: Audio  │  │ US-4: Cámara │
   │  WebSocket   │  │  Captura     │  │  Teléfono    │
   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
          │                 │                 │
          ├─────────────────┼─────────────────┤
          ▼                 ▼                 ▼
   ┌──────────────┐  ┌──────────────┐
   │ US-5: Audio  │  │ US-6: Video  │
   │ Bidireccional│  │ → Gemini     │
   └──────┬───────┘  └──────┬───────┘
          │                 │
          └────────┬────────┘
                   ▼
          ┌──────────────────┐
          │ US-7: Sesión     │
          │ Completa (Phone) │
          └────────┬─────────┘
                   │
          ┌────────┼────────────┐
          ▼                     ▼
   ┌──────────────┐    ┌──────────────┐
   │ US-8: OpenClaw│    │ US-10: DAT   │
   │ Tool Calls    │    │ SDK Gafas    │
   └──────┬───────┘    └──────┬───────┘
          │                   │
          ▼                   ▼
   ┌──────────────┐    ┌──────────────┐
   │ US-9: OpenClaw│    │ US-11: Sesión│
   │ End-to-End    │    │ con Gafas    │
   └──────────────┘    └──────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ US-12: Testing   │
                    │ + Estabilización │
                    └──────────────────┘
```

**Leyenda de dependencias:**
- Las flechas indican que una user story requiere que la anterior esté completada
- US-2, US-3 y US-4 son paralelizables (sin dependencias entre sí)
- US-8 y US-10 son paralelizables (sin dependencias entre sí)

---

## Epic 1: Gemini Live WebSocket

### US-1: Setup Base — Build y Verificación en Emulador

**Como** desarrollador, **quiero** que el proyecto compile y se ejecute en un emulador Android, **para** tener una base funcional sobre la que iterar.

**Criterios de aceptación:**
- [ ] El proyecto compila sin errores con `./gradlew assembleDebug`
- [ ] La app se instala y abre en un emulador API 29+
- [ ] La pantalla principal (`MainScreen`) se muestra con los dos modos (Phone / Glasses)
- [ ] Los badges de configuración reflejan el estado real (Gemini configurado / OpenClaw no)

**Tareas:**
1. Clonar el repositorio y abrir en Android Studio
2. Configurar `GeminiConfig.kt` con un API key válido de Gemini
3. Ejecutar `./gradlew assembleDebug` y verificar que compila
4. Instalar en emulador/dispositivo y verificar la pantalla principal
5. Verificar que los permisos se solicitan correctamente

**Esfuerzo estimado:** 0.5 días  
**Dependencias:** Ninguna

---

### US-2: Conexión WebSocket con Gemini Live

**Como** usuario, **quiero** que la app establezca una conexión WebSocket con Gemini Live API, **para** poder iniciar sesiones de IA conversacional.

**Criterios de aceptación:**
- [ ] La app conecta al WebSocket de Gemini Live usando la URL correcta
- [ ] El mensaje `setup` se envía automáticamente al abrir la conexión
- [ ] Se recibe `setupComplete` y el estado cambia a `CONNECTED`
- [ ] Los errores de conexión se capturan y el estado cambia a `ERROR`
- [ ] La desconexión limpia el WebSocket correctamente

**Tareas:**
1. Verificar la implementación de `GeminiLiveService.connect()`
2. Verificar el manejo de `onOpen`, `onMessage`, `onFailure`, `onClosing`
3. Probar conectando con un API key válido
4. Verificar en Logcat que el flujo setup → setupComplete funciona
5. Probar desconexión y reconexión

**Validación:**
```
Logcat esperado:
GeminiLiveService: Connecting to Gemini Live...
GeminiLiveService: WebSocket opened, sending setup...
GeminiLiveService: Setup complete - connected to Gemini Live
```

**Esfuerzo estimado:** 1-2 días  
**Dependencias:** US-1

---

## Epic 2: Pipeline de Audio

### US-3: Captura de Audio del Micrófono

**Como** usuario, **quiero** que la app capture audio de mi micrófono, **para** poder hablar con Gemini.

**Criterios de aceptación:**
- [ ] `AudioCaptureManager` captura audio PCM Int16 a 16kHz mono
- [ ] Los chunks de audio son de 100ms (3200 bytes)
- [ ] Se utiliza `VOICE_COMMUNICATION` como source (echo cancellation)
- [ ] `AcousticEchoCanceler` se aplica si está disponible
- [ ] El mute/unmute funciona correctamente
- [ ] La captura se detiene limpiamente al llamar `stopCapture()`

**Tareas:**
1. Verificar la implementación de `AudioCaptureManager.startCapture()`
2. Verificar la configuración de AudioRecord (source, rate, channel, format)
3. Probar la captura en dispositivo real (el emulador puede no tener AEC)
4. Verificar que los chunks se envían al callback `onChunk`
5. Probar mute/unmute

**Validación:**
- Log de chunks de audio siendo capturados cada 100ms
- Sin crashes por `SecurityException` (permisos RECORD_AUDIO concedidos)

**Esfuerzo estimado:** 1-2 días  
**Dependencias:** US-1

---

### US-5: Audio Bidireccional — Envío y Recepción via WebSocket

**Como** usuario, **quiero** hablar con Gemini y escuchar sus respuestas por el altavoz, **para** tener una conversación en tiempo real por voz.

**Criterios de aceptación:**
- [ ] El audio capturado se codifica en base64 y se envía como `realtimeInput` al WebSocket
- [ ] El audio recibido (`serverContent.modelTurn.parts[].inlineData`) se decodifica y se reproduce
- [ ] `AudioPlaybackManager` reproduce audio PCM 24kHz por el altavoz
- [ ] El micrófono se silencia automáticamente durante la reproducción de Gemini (prevenir feedback)
- [ ] El micrófono se reactiva cuando Gemini termina de hablar (`turnComplete`)

**Tareas:**
1. Conectar `AudioCaptureManager.onChunk` → `GeminiLiveService.sendAudio()`
2. Conectar `GeminiLiveService.audioOutputChannel` → `AudioPlaybackManager`
3. Implementar coordinación de mute: `AudioPlaybackManager.onPlaybackStateChanged` → `AudioCaptureManager.setMuted()`
4. Conectar `turnCompleteChannel` para reactivar micrófono
5. Probar conversación básica: decir algo → escuchar respuesta

**Validación:**
- Decir "Hola, ¿cómo estás?" → Gemini responde por voz
- No hay eco ni feedback audible
- El micrófono se silencia visualmente durante la respuesta de Gemini

**Esfuerzo estimado:** 2-3 días  
**Dependencias:** US-2, US-3

---

## Epic 3: Captura de Cámara (Teléfono)

### US-4: Captura de Frames con CameraX

**Como** usuario, **quiero** que la app use la cámara trasera de mi teléfono, **para** que Gemini pueda ver lo que veo.

**Criterios de aceptación:**
- [ ] `PhoneCameraManager` usa CameraX con la cámara trasera
- [ ] Los frames se capturan en formato YUV_420_888
- [ ] La conversión YUV → NV21 → JPEG funciona correctamente (`ImageUtil.imageToJpeg`)
- [ ] El throttle limita los frames a ~1fps (intervalo de 1000ms)
- [ ] La calidad JPEG es del 50% (optimización de ancho de banda)
- [ ] El preview de cámara se muestra en la UI (opcional)

**Tareas:**
1. Verificar la implementación de `PhoneCameraManager.startCapture()`
2. Verificar `ImageUtil.imageToJpeg()` — conversión YUV → JPEG
3. Probar captura en dispositivo/emulador con cámara virtual
4. Verificar throttle: solo 1 frame por segundo máximo
5. Verificar que `imageProxy.close()` se llama siempre (evitar memory leaks)

**Validación:**
- Log mostrando frames capturados a ~1fps
- JPEG generados con tamaño razonable (< 100KB a 50% calidad)
- Preview visible en la pantalla de sesión

**Esfuerzo estimado:** 1-2 días  
**Dependencias:** US-1

---

### US-6: Envío de Video Frames a Gemini

**Como** usuario, **quiero** que los frames de la cámara se envíen a Gemini en tiempo real, **para** que Gemini pueda ver y describir lo que estoy mirando.

**Criterios de aceptación:**
- [ ] Los frames JPEG se codifican en base64 y se envían como `realtimeInput` con `mimeType: "image/jpeg"`
- [ ] El envío respeta el throttle de 1fps
- [ ] Gemini responde describiendo lo que ve cuando se le pregunta
- [ ] No hay degradación de rendimiento por el envío de frames

**Tareas:**
1. Conectar `PhoneCameraManager.onFrame` → `GeminiLiveService.sendVideoFrame()`
2. Verificar formato del mensaje JSON enviado
3. Probar: apuntar la cámara a un objeto y preguntar "¿Qué ves?"
4. Verificar que Gemini describe el objeto correctamente

**Validación:**
- Decir "¿Qué estoy mirando?" con la cámara apuntando a un objeto → Gemini lo describe
- Los logs muestran frames enviados a ~1fps

**Esfuerzo estimado:** 1-2 días  
**Dependencias:** US-2, US-4

---

## Epic 4: Integración OpenClaw

### US-8: Ejecución de Tool Calls via OpenClaw

**Como** usuario, **quiero** que Gemini pueda ejecutar acciones en el mundo real delegando a OpenClaw, **para** poder hacer cosas como añadir items a mi lista de compras o enviar mensajes.

**Criterios de aceptación:**
- [ ] Gemini detecta cuándo el usuario solicita una acción y genera un `toolCall`
- [ ] `ToolCallRouter` recibe el tool call y extrae el campo `task`
- [ ] `OpenClawBridge` envía un POST HTTP a la URL de OpenClaw con formato chat/completions
- [ ] La respuesta de OpenClaw se envía de vuelta a Gemini como `toolResponse`
- [ ] Gemini verbaliza el resultado al usuario

**Tareas:**
1. Verificar `ToolCallRouter.handleToolCall()` — routing correcto
2. Verificar `OpenClawBridge.executeTask()` — formato de request/response
3. Configurar un servidor OpenClaw accesible (o mock)
4. Probar: pedir a Gemini "Añade leche a mi lista de compras"
5. Verificar en logs el flujo completo: toolCall → HTTP → toolResponse

**Validación:**
```
Flujo esperado:
1. Usuario: "Añade leche a mi lista de compras"
2. Gemini genera toolCall: { name: "execute", args: { task: "Add milk to shopping list" } }
3. OpenClawBridge POST → OpenClaw gateway
4. Respuesta: "Added milk to your shopping list"
5. Gemini dice: "He añadido leche a tu lista de compras"
```

**Esfuerzo estimado:** 2-3 días  
**Dependencias:** US-7 (sesión completa con voz)

**Requisito adicional:** Servidor OpenClaw desplegado y accesible desde la red del teléfono.

---

### US-9: Validación End-to-End de OpenClaw

**Como** QA/desarrollador, **quiero** probar el flujo completo de tool calls desde voz hasta ejecución, **para** asegurar que la integración funciona de extremo a extremo.

**Criterios de aceptación:**
- [ ] El flujo voz → Gemini → toolCall → OpenClaw → toolResponse → voz funciona sin errores
- [ ] Los errores de OpenClaw se manejan gracefully (timeout, server down, respuesta inválida)
- [ ] Si OpenClaw no está configurado, Gemini responde sin intentar tool calls
- [ ] Múltiples tool calls en secuencia funcionan correctamente

**Tareas:**
1. Probar con OpenClaw funcionando: flujo happy path
2. Probar con OpenClaw apagado: verificar manejo de error
3. Probar con OpenClaw con timeout: verificar que no bloquea la sesión
4. Probar múltiples peticiones seguidas
5. Documentar resultados y bugs encontrados

**Esfuerzo estimado:** 1-2 días  
**Dependencias:** US-8

---

## Epic 5: ViewModel y UI

### US-7: Sesión Completa en Modo Teléfono

**Como** usuario, **quiero** tener una sesión de IA completa usando solo mi teléfono (cámara + micrófono), **para** poder usar VisionClaw sin necesitar las gafas Meta Ray-Ban.

**Criterios de aceptación:**
- [ ] `SessionViewModel` orquesta todos los componentes: Gemini + Audio + Cámara
- [ ] Pulsar "Start on Phone" inicia la sesión completa
- [ ] El estado de conexión se muestra visualmente (punto de color: gris/amarillo/verde/rojo)
- [ ] Pulsar "Stop" detiene todos los componentes limpiamente
- [ ] La UI muestra el preview de cámara durante la sesión
- [ ] Los permisos de cámara y micrófono se solicitan antes de iniciar

**Tareas:**
1. Verificar `SessionViewModel.startSession(PHONE)` — inicio secuencial de componentes
2. Verificar `SessionViewModel.stopSession()` — limpieza de recursos
3. Verificar `SessionScreen` — UI de sesión activa
4. Probar flujo completo: Main → Permisos → Session → Conversación → Stop
5. Verificar que no hay memory leaks al iniciar/detener sesiones repetidamente

**Validación:**
- Flujo end-to-end: iniciar sesión → hablar con Gemini → ver preview → detener sesión
- Sin crashes al rotar pantalla o poner la app en background

**Esfuerzo estimado:** 2-3 días  
**Dependencias:** US-5, US-6

---

## Epic 6: Meta Ray-Ban (DAT SDK)

### US-10: Integración de Meta DAT SDK para Streaming desde Gafas

**Como** usuario con gafas Meta Ray-Ban, **quiero** que la app reciba el streaming de vídeo desde mis gafas, **para** que Gemini vea lo que yo veo a través de ellas.

**Criterios de aceptación:**
- [ ] El Meta DAT SDK (`meta-wearables-dat-android` v0.4.0) está integrado como dependencia
- [ ] `GlassesCameraManager` conecta con las gafas via `StreamSession`
- [ ] Los `VideoFrame` (I420) se convierten a JPEG
- [ ] Los frames se envían a Gemini con throttle a 1fps
- [ ] Los permisos de Bluetooth y Camera (wearable) se solicitan correctamente
- [ ] La conexión con las gafas se gestiona (connect/disconnect)

**Tareas:**
1. Configurar acceso a GitHub Packages (token con `read:packages`)
2. Descomentar dependencias DAT SDK en `app/build.gradle.kts`
3. Implementar `GlassesCameraManager.kt`:
   - Inicializar `Wearables` SDK
   - Solicitar permisos con `Wearables.RequestPermissionContract()`
   - Establecer `StreamSession` con `videoStream`
   - Convertir `VideoFrame` (I420) → NV21 → JPEG
   - Throttle a 1fps
4. Conectar `GlassesCameraManager.onFrame` → `GeminiLiveService.sendVideoFrame()`
5. Actualizar `SessionViewModel` para modo GLASSES
6. Probar con gafas físicas o `MockDeviceKit`

**Validación con MockDeviceKit:**
```kotlin
// Test sin gafas físicas
val mockDevice = MockDeviceKit.pairRaybanMeta()
// Verificar que el streaming de vídeo mock funciona
```

**Validación con gafas reales:**
- Parear gafas via Meta AI app (Developer Mode habilitado)
- Pulsar "Start Streaming" → vídeo de las gafas llega a la app
- Preguntar "¿Qué ves?" → Gemini describe lo que las gafas capturan

**Esfuerzo estimado:** 3-5 días  
**Dependencias:** US-7 (sesión funcional en modo teléfono)

**Requisitos previos:**
- Token de GitHub con permisos `read:packages` para acceder al SDK
- Gafas Meta Ray-Ban con Developer Mode (o usar MockDeviceKit)

---

### US-11: Sesión Completa con Gafas Meta Ray-Ban

**Como** usuario, **quiero** tener una sesión de IA completa usando mis gafas Meta Ray-Ban como fuente de vídeo, **para** tener la experiencia VisionClaw hands-free.

**Criterios de aceptación:**
- [ ] El modo "Glasses" en MainScreen inicia una sesión con gafas como fuente de vídeo
- [ ] El audio sigue usando el micrófono/altavoz del teléfono (o de las gafas si disponible)
- [ ] La conversión de frames de gafas a JPEG funciona sin degradación
- [ ] La sesión se puede detener y reiniciar limpiamente
- [ ] Si las gafas se desconectan durante la sesión, se muestra un error apropiado

**Tareas:**
1. Actualizar `SessionViewModel.startSession(GLASSES)` para usar `GlassesCameraManager`
2. Gestionar estados de conexión de las gafas
3. Manejar desconexión inesperada de las gafas
4. Probar flujo completo con gafas (o MockDeviceKit)
5. Verificar que el cambio entre modo Phone y Glasses funciona

**Esfuerzo estimado:** 2-3 días  
**Dependencias:** US-10

---

## Epic 7: Testing y Estabilización

### US-12: Testing Integral y Estabilización

**Como** equipo de desarrollo, **queremos** tener tests automatizados y estabilidad validada, **para** asegurar la calidad del producto antes de release.

**Criterios de aceptación:**
- [ ] Tests unitarios para `GeminiLiveService` (parsing de mensajes, formato de envío)
- [ ] Tests unitarios para `ImageUtil` (conversión YUV → JPEG)
- [ ] Tests unitarios para `AudioUtil` (encode/decode base64)
- [ ] Tests unitarios para `OpenClawBridge` (formato de request, parsing de response)
- [ ] Tests unitarios para `ToolCallRouter` (routing, manejo de errores)
- [ ] Test de integración: conexión WebSocket con Gemini (requiere API key)
- [ ] Test de UI: navegación MainScreen → SessionScreen
- [ ] Sin memory leaks en sesiones repetidas (verificar con LeakCanary o profiler)
- [ ] Sin crashes al rotar pantalla, background/foreground, o cambio de modo

**Tareas:**
1. Escribir tests unitarios con JUnit + MockK/Mockito
2. Escribir tests instrumentados con Espresso/Compose Testing
3. Configurar CI con GitHub Actions para build + test
4. Ejecutar tests en emulador API 29, 33, 35
5. Profiling de memoria con Android Studio Profiler
6. Documentar bugs encontrados y correcciones aplicadas

**Esfuerzo estimado:** 3-5 días  
**Dependencias:** US-7 (sesión funcional), US-9 (OpenClaw e2e)

---

## Costes Recurrentes

| Concepto | Coste | Notas |
|----------|-------|-------|
| **Gemini API** | Gratis (con límites) o desde $0.075/1M tokens | Tier gratis disponible en Google AI Studio |
| **Servidor OpenClaw** | €5-20/mes (VPS) o $0 (local) | Solo necesario para tool calls |
| **Google Play Developer** | $25 (pago único) | Para publicar en Play Store |
| **Meta Developer Account** | Gratis | Para acceso al DAT SDK |

---

## Cronograma Resumen

| Semana | User Stories | Entregable |
|--------|-------------|------------|
| **Semana 1** | US-1, US-2, US-3, US-4 | Build funcional + componentes individuales verificados |
| **Semana 2** | US-5, US-6 | Audio bidireccional + vídeo → Gemini |
| **Semana 3** | US-7 | Sesión completa en modo teléfono (demo-ready) |
| **Semana 4** | US-8, US-9 | Integración OpenClaw end-to-end |
| **Semana 5** | US-10, US-11 | Integración Meta Ray-Ban con DAT SDK |
| **Semana 6** | US-12 | Testing, estabilización, preparación para release |

### Hitos clave:
- **Fin de Semana 3**: 🎯 **Demo funcional en modo teléfono** — conversación con Gemini + visión
- **Fin de Semana 4**: 🎯 **Tool calls funcionando** — Gemini ejecuta acciones vía OpenClaw
- **Fin de Semana 5**: 🎯 **Gafas Meta Ray-Ban integradas** — experiencia VisionClaw completa
- **Fin de Semana 6**: 🎯 **Release candidate** — testado y estable

---

## Equivalencia iOS ↔ Android

| Componente VisionClaw (iOS) | Equivalente Android (implementado/pendiente) |
|---|---|
| `meta-wearables-dat-ios` | ✅ `meta-wearables-dat-android` v0.4.0 (disponible) |
| `AVCaptureSession` (cámara) | ✅ `PhoneCameraManager` (CameraX) / ⚠️ `GlassesCameraManager` (DAT SDK) |
| `videoFramePublisher` (frames) | ✅ `VideoFrame` I420 → JPEG (en DAT SDK sample) |
| `capturePhoto` | ✅ `streamSession.capturePhoto()` (DAT SDK) |
| Permisos cámara gafas | ✅ `Wearables.RequestPermissionContract()` |
| `MockDevice` (testing) | ✅ `MockDeviceKit.pairRaybanMeta()` |
| `GeminiLiveService.swift` | ✅ `GeminiLiveService.kt` (OkHttp WebSocket) |
| `AudioManager.swift` | ✅ `AudioCaptureManager.kt` + `AudioPlaybackManager.kt` |
| `GeminiConfig.swift` | ✅ `GeminiConfig.kt` |
| `GeminiSessionViewModel.swift` | ✅ `SessionViewModel.kt` |
| `OpenClawBridge.swift` | ✅ `OpenClawBridge.kt` |
| `ToolCallRouter.swift` | ✅ `ToolCallRouter.kt` |
| `ToolCallModels.swift` | ✅ `ToolCallModels.kt` |

---

> **Nota**: Este plan asume que el desarrollador tiene acceso a un dispositivo Android físico para pruebas de audio y cámara. El emulador es suficiente para verificar la UI y la lógica de conexión, pero las pruebas de audio en tiempo real y la integración con gafas Meta Ray-Ban requieren hardware real.
