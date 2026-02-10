# VisionClaw Android — User Stories (Resumen Ejecutable)

> Checklist de user stories ordenadas por prioridad y dependencia.
> Cada story es validable de forma independiente.

---

## Fase 1: Fundación (Semana 1)

### US-1: Setup Base ✅
- **Prioridad:** P0 (bloqueante)
- **Dependencias:** Ninguna
- **Esfuerzo:** 0.5 días
- **Estado:** ✅ Completado — el proyecto compila y la estructura está lista

**Checklist de validación:**
- [x] `./gradlew assembleDebug` compila sin errores
- [x] La app se instala en emulador/dispositivo
- [x] `MainScreen` se muestra con opciones Phone/Glasses
- [x] Permisos de cámara y micrófono se solicitan correctamente

---

### US-2: Conexión WebSocket con Gemini Live ✅
- **Prioridad:** P0 (bloqueante)
- **Dependencias:** US-1
- **Esfuerzo:** 1-2 días
- **Archivo principal:** `GeminiLiveService.kt`
- **Estado:** ✅ Implementado

**Checklist de validación:**
- [x] Conexión WebSocket establece correctamente
- [x] Mensaje `setup` enviado automáticamente en `onOpen`
- [x] `setupComplete` recibido → estado `CONNECTED`
- [x] Errores → estado `ERROR`
- [x] Desconexión limpia

**Cómo validar:** Verificar en Logcat los mensajes de conexión del tag `GeminiLiveService`

---

### US-3: Captura de Audio del Micrófono ✅
- **Prioridad:** P0 (bloqueante)
- **Dependencias:** US-1
- **Esfuerzo:** 1-2 días
- **Archivo principal:** `AudioCaptureManager.kt`
- **Estado:** ✅ Implementado

**Checklist de validación:**
- [x] AudioRecord configurado: 16kHz, mono, PCM_16BIT
- [x] Source: `VOICE_COMMUNICATION` (AEC habilitado)
- [x] Chunks de 100ms (3200 bytes)
- [x] Mute/unmute funcional
- [x] Stop/cleanup sin leaks

---

### US-4: Captura de Frames con CameraX ✅
- **Prioridad:** P0 (bloqueante)
- **Dependencias:** US-1
- **Esfuerzo:** 1-2 días
- **Archivo principal:** `PhoneCameraManager.kt`, `ImageUtil.kt`
- **Estado:** ✅ Implementado

**Checklist de validación:**
- [x] CameraX cámara trasera
- [x] Conversión YUV_420_888 → NV21 → JPEG
- [x] Throttle ~1fps (1000ms intervalo)
- [x] Calidad JPEG 50%
- [x] `imageProxy.close()` siempre llamado

---

## Fase 2: Integración de Pipelines (Semana 2)

### US-5: Audio Bidireccional
- **Prioridad:** P0 (bloqueante)
- **Dependencias:** US-2 + US-3
- **Esfuerzo:** 2-3 días
- **Archivos:** `AudioCaptureManager.kt`, `AudioPlaybackManager.kt`, `GeminiLiveService.kt`

**Checklist de validación:**
- [ ] Audio capturado → base64 → enviado como `realtimeInput`
- [ ] Audio recibido → decodificado → reproducido por AudioTrack (24kHz)
- [ ] Micrófono se silencia durante reproducción de Gemini
- [ ] Micrófono se reactiva en `turnComplete`
- [ ] Conversación básica funciona: hablar → Gemini responde por voz

**Cómo validar:** Decir "Hola" → escuchar respuesta de voz de Gemini

---

### US-6: Envío de Video Frames a Gemini
- **Prioridad:** P1 (importante)
- **Dependencias:** US-2 + US-4
- **Esfuerzo:** 1-2 días
- **Archivos:** `PhoneCameraManager.kt`, `GeminiLiveService.kt`

**Checklist de validación:**
- [ ] Frames JPEG enviados como `realtimeInput` con `mimeType: "image/jpeg"`
- [ ] Throttle respetado (1fps)
- [ ] Gemini puede describir lo que ve la cámara

**Cómo validar:** Apuntar cámara a un objeto, decir "¿Qué ves?" → Gemini lo describe

---

## Fase 3: Sesión Funcional (Semana 3)

### US-7: Sesión Completa en Modo Teléfono 🎯 HITO
- **Prioridad:** P0 (bloqueante)
- **Dependencias:** US-5 + US-6
- **Esfuerzo:** 2-3 días
- **Archivos:** `SessionViewModel.kt`, `SessionScreen.kt`, `MainScreen.kt`

**Checklist de validación:**
- [ ] "Start on Phone" inicia todos los componentes
- [ ] Preview de cámara visible en la UI
- [ ] Estado de conexión mostrado (punto de color)
- [ ] "Stop" detiene todo limpiamente
- [ ] Sin crashes al rotar pantalla
- [ ] Sin crashes al entrar/salir de background

**Cómo validar:** Flujo completo end-to-end: tap Start → hablar + mostrar objetos → Gemini responde → tap Stop

---

## Fase 4: Tool Calls (Semana 4)

### US-8: Ejecución de Tool Calls via OpenClaw
- **Prioridad:** P1 (importante)
- **Dependencias:** US-7
- **Esfuerzo:** 2-3 días
- **Archivos:** `ToolCallRouter.kt`, `OpenClawBridge.kt`

**Checklist de validación:**
- [ ] Gemini genera `toolCall` cuando usuario pide una acción
- [ ] `ToolCallRouter` extrae `task` y llama a OpenClaw
- [ ] POST HTTP correcto a `/v1/chat/completions`
- [ ] Respuesta enviada como `toolResponse` a Gemini
- [ ] Gemini verbaliza el resultado

**Requisitos previos:** Servidor OpenClaw desplegado y accesible

**Cómo validar:** Decir "Añade leche a mi lista de compras" → Gemini ejecuta y confirma

---

### US-9: Validación End-to-End de OpenClaw
- **Prioridad:** P2 (deseable)
- **Dependencias:** US-8
- **Esfuerzo:** 1-2 días

**Checklist de validación:**
- [ ] Happy path funciona (voz → tool call → resultado → voz)
- [ ] Error handling: OpenClaw caído → mensaje de error graceful
- [ ] Timeout: OpenClaw lento → no bloquea la sesión
- [ ] Múltiples tool calls secuenciales funcionan
- [ ] Sin OpenClaw configurado → Gemini funciona solo con voz (sin tool calls)

---

## Fase 5: Gafas Meta Ray-Ban (Semana 5)

### US-10: Integración Meta DAT SDK
- **Prioridad:** P1 (importante)
- **Dependencias:** US-7
- **Esfuerzo:** 3-5 días
- **Archivo principal:** `GlassesCameraManager.kt`

**Checklist de validación:**
- [ ] Dependencia DAT SDK configurada (GitHub Packages token)
- [ ] `Wearables` SDK inicializado
- [ ] Permisos solicitados (Bluetooth, Camera wearable)
- [ ] `StreamSession` establecida con `videoStream`
- [ ] `VideoFrame` I420 → JPEG conversión funcional
- [ ] Frames enviados a Gemini a 1fps
- [ ] MockDeviceKit funciona para testing

**Requisitos previos:**
- Token GitHub con `read:packages`
- Gafas Meta Ray-Ban con Developer Mode (o MockDeviceKit)

---

### US-11: Sesión Completa con Gafas
- **Prioridad:** P1 (importante)
- **Dependencias:** US-10
- **Esfuerzo:** 2-3 días

**Checklist de validación:**
- [ ] Modo "Glasses" inicia sesión con gafas como fuente de vídeo
- [ ] Audio por micrófono/altavoz del teléfono
- [ ] Desconexión de gafas manejada correctamente
- [ ] Cambio Phone ↔ Glasses funciona

---

## Fase 6: Testing (Semana 6)

### US-12: Testing y Estabilización
- **Prioridad:** P1 (importante)
- **Dependencias:** US-7, US-9
- **Esfuerzo:** 3-5 días

**Checklist de validación:**
- [ ] Tests unitarios: GeminiLiveService, ImageUtil, AudioUtil, OpenClawBridge, ToolCallRouter
- [ ] Tests de UI: navegación, estados de sesión
- [ ] Sin memory leaks (verificar con profiler)
- [ ] Estable en API 29, 33, 35
- [ ] CI configurado con GitHub Actions

---

## Resumen de Prioridades

| Prioridad | Stories | Descripción |
|-----------|---------|-------------|
| **P0** | US-1→US-7 | Sesión funcional en modo teléfono (core del producto) |
| **P1** | US-8, US-10, US-11, US-12 | Tool calls + gafas + testing |
| **P2** | US-9 | Validación e2e de OpenClaw |

## Ruta Crítica

```
US-1 → US-2 → US-5 → US-7 (sesión funcional)
US-1 → US-3 ↗
US-1 → US-4 → US-6 ↗
```

La ruta crítica para el primer hito demo es: **Setup → WebSocket → Audio Bidireccional + Video → Sesión Completa**.
