# Lab 11 — Offline‑first con Service Worker, cachés y colas

Este laboratorio extiende la entrega del Lab 10 para transformar la PWA en una app con enfoque offline‑first. Implementarás: indicador de conectividad “real” al backend, cacheo de datos críticos, cola de envíos cuando no hay conexión y validación de modelos con caché expirable. Mantén la misma infraestructura serverless del Lab 10 (S3 + CloudFront + API Gateway + DynamoDB) y el mismo dominio personalizado.

Al final tendrás una PWA que funciona bien sin red, re-intenta cuando vuelve la conectividad y comunica claramente al usuario qué viene de la red y qué viene de caché.

## Funcionalidades a implementar

- Botón/indicador de “online” que refleje conectividad contra el servidor (no solo `navigator.onLine`), haciendo polling con una petición OPTIONS a la raíz de la API (`/api`)
- Caché de la lista de naves (spaceships):
  - Si está online, trae del servidor y actualiza caché local
  - Si está offline, responde desde caché y muestra que el dato viene de caché
- Reporte retrasado (outbox):
  - Si se reporta una nave y no hay conexión, intercepta y acumula el request
  - Muestra cuántas acciones están pendientes
  - Al volver la conexión, envía uno por uno. Si se corta durante el envío, conserva los no enviados
- Cacheo de “status” de nave en el modal de detalles:
  - Si está offline, mostrar el dato disponible que permitió imprimir la tabla, marcado como caché (p. ej. con un asterisco o badge)
- Caché para model validation con expiración de 5 minutos:
  - Lista “elegible” de modelos válidos conocida (ver más abajo). Para otros valores: “inválido asegurado”
  - Estados cuando offline: “inválido asegurado” (no elegible), “inválido pendiente” (elegible pero la última validación conocida fue inválida) y “válido pendiente” (válido pero proviene de caché)
  - Solo este caché expira (5 min). Los demás cachés son persistentes

Importante: las rutas y componentes de referencia están en la app del Lab 10 (`labs/lab10/app`).

## Requisitos previos

- Haber completado el Lab 10 con la infraestructura desplegada y el dominio funcionando bajo HTTPS
- Node.js y npm instalados en tu equipo
- Tener configurado `NEXT_PUBLIC_APP_DOMAIN` en `.env` de la app (ver Preparación)

## Preparación

1. Abre la carpeta de la app del Lab 10 (`lab10/app`)
2. Revisa `.env` y define el dominio de la app:
   - Para desarrollo local: `NEXT_PUBLIC_APP_DOMAIN=http://localhost:3000`
   - Para probar en remoto: `NEXT_PUBLIC_APP_DOMAIN=https://12345678k.www.exampledomain.cloud`
3. Arranca en local para desarrollo (APIs tienen CORS abiertos para localhost y http):
   - `npm ci` (o `npm install`)
   - `npm run dev`
4. Verifica que el Service Worker se registre (en DevTools → Application → Service Workers). La app incluye `public/sw.js` y `src/pwa/sw/Register.js`

Nota sobre alcance del Service Worker: intercepta requests solo del mismo origen. En desarrollo, si la API apunta a otro dominio, el SW no verá esos fetch. Aún así, todas las funcionalidades podrán comprobarse completamente en el entorno desplegado (mismo dominio detrás de CloudFront)

## Estructura base a tocar (referencia)

- Indicador de conectividad: `src/components/FooterBar.jsx`
- Lista de naves: `src/components/SpaceshipsListing.jsx`
- Modal de detalles/estado: `src/components/SpaceshipStatus.jsx`
- Reporte de naves (POST): `src/components/SpaceshipsAdd.jsx`
- Helpers de entorno: `src/lib/env.js`, `src/lib/auth.js`
- Service Worker: `public/sw.js` (extender interceptores, caché y cola)

## Procedimiento

Este procedimiento sugiere un orden lógico para implementar las funcionalidades, en orden incremental de dificultad y dependencia. Se recomienda hacer un nuevo deployment en remoto tras completar cada funcionalidad para probar el comportamiento con el Service Worker interceptando las rutas correctas. Revise al final de este documento las recomendaciones sobre el ciclo de publicación.

### 1. Estado online/offline con polling a OPTIONS /api

Objetivo: indicar conectividad al backend (no solo conectividad de red del dispositivo). Usa un estado compartido que otras piezas puedan consumir

Sugerencia técnica

- Crea un estado global “onlineStatus” con Context o una librería de estado (puede ser un simple React Context + `useReducer`). Ubicación sugerida: `src/hooks/useOnlineStatus.js` con `<OnlineStatusProvider>` en `src/app/layout.jsx` o `src/components/AppShell.jsx`
- El proveedor debe:
  - Inicializar con `navigator.onLine`
  - Suscribirse a eventos `window.online`/`window.offline` para cambios inmediatos
  - Cada 10–15s hacer `fetch(getApiUrl(), { method: 'OPTIONS' })` a la raíz `/api`
    - Si responde 2xx/3xx, considera “online contra servidor”
    - Si falla por red o timeout, considera “offline contra servidor”
  - Exponer `online` y `lastCheckedAt` en un hook `useOnlineStatus()`
- Reemplaza el uso directo de `navigator.onLine` en `FooterBar.jsx` por el hook `useOnlineStatus()`. Muestra el punto verde/rojo en base a este valor

Por qué OPTIONS: es un método liviano y universal para verificar capacidad de respuesta y CORS de la API en su raíz, sin efectos colaterales. El estudiante debe investigar más sobre su rol (preflight, descubrimiento de capacidades) y su pertinencia para “salud” del backend

### 2. Caché de lista de spaceships (GET /api/spaceships)

Objetivo: poder listar naves aunque estés offline y señalar cuando el dato vino de caché

Sugerencia técnica (Service Worker + IndexedDB)

- En `public/sw.js`, añade manejo de `fetch` para `GET` a `/api/spaceships` (mismo origen):
  - Estrategia network‑first:
    1. Intenta `fetch` a la red, si responde OK:
       - Clona la respuesta JSON y guarda la lista en IndexedDB (DB sugerida: `imperial-db`, store `spaceships`, con clave `spaceship_id`)
       - Devuelve la respuesta de red
    2. Si falla red/timeout:
       - Lee desde IndexedDB la lista completa y construye una `Response` JSON equivalente (respeta el shape que consume el front: array o `{ items: [...] }`)
       - Agrega un header `X-From-Cache: 1` para que el cliente pueda indicarlo en UI
- En `SpaceshipsListing.jsx`:
  - Lee `res.headers.get('X-From-Cache')` y, si existe, cambia el texto de “N ships reported” por “N ships reported (from cache)”
  - Mantén el botón de Refresh. Si offline, al hacer refresh seguirá viniendo de caché

Detalles prácticos

- Para IndexedDB usa una utilidad ligera (p. ej. `idb`) o un helper propio (p. ej. `src/pwa/utils/db.js`). Persistente y sin expiración para este caso
- Si el backend devuelve array directo o `{ items }`, normaliza al guardar para que el cliente siempre reciba el formato que espera

### 3. Reporte retrasado (cola de envíos) para POST /api/spaceships

Objetivo: si el usuario reporta una nave sin conexión, la app no pierde el dato; se encola y se envía automáticamente al volver la conectividad. La UI debe mostrar Reportes pendientes: X”

Sugerencia técnica (Service Worker + Outbox + BroadcastChannel/postMessage)

- En `public/sw.js` intercepta `POST` a `/api/spaceships`:
  - Si `onlineStatus` indica offline (o el `fetch` falla por red), clona el request (lee JSON) y guarda un item `{ body, headersNecesarios, createdAt }` en IndexedDB store `outbox`
  - Responde al cliente con `202 Accepted` y un body JSON `{ queued: true }` para que la UI informe “pendiente”
  - Emite un evento de “outbox-updated” a las páginas con `BroadcastChannel('imperial-outbox')` o `self.clients.matchAll()` + `client.postMessage()`
- Implementa un “reintento” cuando vuelve la conectividad:
  - El SW se entera al re‑evaluar el polling o escuchando `online` del navegador
  - Procesa la cola en orden de llegada: `for` secuencial
  - Si un envío falla por red → detén el proceso, conserva el resto
  - Si responde 4xx/5xx definitivos, decide si re‑intentas (opcional) o descartas con marca de error (para este lab, basta con reintentar solo ante errores de red)
  - Tras cada envío exitoso, remueve el item y emite “outbox-updated”
- En la UI (por ejemplo, la franja superior de `SpaceshipsListing.jsx` donde hoy se muestra “N ships reported”):
  - Crea un pequeño estado/contador `pendingCount` suscrito al canal `imperial-outbox`
  - Muestra “Pending actions: X” cuando `X > 0`, con el mismo estilo tipográfico de la línea de conteo
  - Cuando `POST` devuelve `{ queued: true }`, actualiza inmediatamente el contador (sin esperar al SW)

Nota: si prefieres, puedes escuchar mensajes del SW en un hook (`useEffect` global) y exponer `pendingCount` en el mismo `OnlineStatusProvider` para reutilizarlo

### 4. Detalles de nave offline en el modal (GET /api/spaceships/:id)

Objetivo: al abrir el modal (`SpaceshipStatus.jsx`) que se despliega desde la lista de naves, si no hay red, mostrar datos disponibles desde caché y marcarlos como tal

Sugerencia técnica

- Cuando el modal intenta cargar detalles:
  - Si online: hace `fetch` normal y, si responde, persiste los detalles en IndexedDB store `spaceship_details` (clave `spaceship_id`)
  - Si offline o la red falla: intenta recuperar de la caché de lista (store `spaceships`) el registro con ese `id` y úsalo para poblar campos (p. ej. `pilot`, `model`)
  - Muestra un indicador claro: “from cache” o un asterisco “\*” al lado del valor
- No necesitas expirar estos detalles (persistentes). Cuando vuelvas a estar online, una nueva apertura del modal refrescará desde la red y actualizará caché

### 5. Caché de model validation (TTL 5 minutos)

Objetivo: validar el `model` con la API externa `https://api.algundominio.link/spaceships/models/validator?model={model_name}` usando un caché con expiración y lógica específica para el modo offline

Lista de modelos elegibles (Modelos que pueden retornar un valor válido, si un modelo no está en la lista se está seguro que siempre será inválido):

- A/SF-01 B-Wing
- AT-AT
- BTL Y-Wing
- DS-1
- Delta-7
- Executor-class
- Firespray-31
- Imperial I-class
- Lambda-class
- MC80
- MC80A
- N-1 Starfighter
- Nebulon-B
- RZ-1 A-Wing
- T-65 X-Wing
- TIE Advanced x1
- TIE/IN Interceptor
- TIE/LN Fighter
- TIE/sa Bomber
- YT-1300

Sugerencia técnica

- Implementa un pequeño módulo `modelValidationCache` (p. ej. `src/pwa/utils/modelValidation.js`) que exponga:
  - `isEligible(model)` → true/false según la lista de arriba (normaliza mayúsculas/minúsculas y espacios)
  - `getCached(model)` → `{ status: 'valid'|'invalid', updatedAt } | null` desde IndexedDB store `model_validation`
  - `setCached(model, status)` → guarda con `updatedAt = Date.now()`
  - `isExpired(updatedAt)` → true si pasaron > 5 minutos
- En `SpaceshipStatus.jsx`, donde hoy se hace el `fetch` de validación:
  - Si `online`:
    - Llama a la API de validación y guarda el resultado (`valid`/`invalid`) en la caché con timestamp
  - Si `offline`:
    - Si `!isEligible(model)`: muestra “inválido asegurado”
    - Si es elegible y hay cache:
      - Si el último fue `valid`: muestra “válido pendiente” (badge “Valid (cache)”)
      - Si el último fue `invalid`: muestra “inválido pendiente”
    - Si es elegible y NO hay cache: muestra “pendiente” (sin decisión) o un estado neutro
- Solo este caché expira a los 5 minutos. Si el valor está expirado pero offline, sigue mostrando el estado “pendiente” correspondiente (no inventes nuevo resultado). Al recuperar conectividad, revalida

## Apéndice: ciclo de publicación y pruebas

### Trabajo iterativo sugerido

- Itera en local con `npm run dev` (las APIs ya fueron configuradas para enviar los headers CORS necesarios para trabajar en localhost). Cuando completes una etapa, publica para probar el comportamiento con SW interceptando la misma ruta de origen

### Publicar nueva versión

1. En el .env ajusta el valor de NEXT_PUBLIC_APP_VERSION para reflejar la nueva versión

- La notación X.Y.Z se denomina semver (semantic versioning). Incrementa:
  - Z (patch) para cambios menores o fixes
  - Y (minor) para nuevas funcionalidades compatibles
  - X (major) para cambios incompatibles o re-escrituras importantes

2. En la carpeta `app/`:
   - `npm ci` (o `npm install`)
   - `npm run build`
   - Se generará `out/` con `index.html` y assets
3. Sube el contenido de `out/` al bucket S3 del Lab 10 (reemplaza el contenido actual). Puedes usar la consola S3 o tu herramienta preferida
4. Abre tu aplicación en el celular y revisa que todo funcione correctamente

### Sobre invalidación de CloudFront

- Invalidation es el proceso de purgar objetos cacheados en los edge de CloudFront para forzar a traer una versión nueva desde el origen
- En entornos de producción, es común invalidar al publicar, es decir, avisar a CloudFront que el contenido cambió y debe refrescarse sin esperar a que expire el TTL de caché
- En este laboratorio, la distribución está configurada para no cachear el sitio (origin no cache y/o headers que deshabilitan caching), por lo que NO necesitas invalidar al subir una nueva build: bastará con reemplazar los objetos en S3

## Conclusiones

Con estas mejoras, tu PWA ofrece una experiencia offline‑first robusta: detecta conectividad real al backend, sirve datos desde caché cuando es necesario, encola envíos para no perder acciones del usuario y maneja validaciones de modelo con una política clara de expiración. Este patrón es reutilizable para muchas apps serverless modernas
