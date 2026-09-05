# Arquitectura del Backend — mav-rd-backend

> Refleja el estado REAL del código al 05/09/2026. Reemplaza la versión
> anterior de este mismo archivo. Para el historial de cómo se llegó aquí,
> ver HISTORIAL_MODIFICACIONES.md.

Stack: Node.js + Express + Mongoose (MongoDB Atlas, cluster compartido
mujeresalvolante.rd4sofa.mongodb.net, versión real 8.0.29) + JWT (sin
cookies) + Cloudinary (archivos) + Resend (email) + Telegram Bot API
(avisos internos) + despliegue en Render (mav-rd-backend.onrender.com).

## Infraestructura y despliegue

- **Dominio propio en producción (resuelto 27/08/2026):** la fundadora
  compró `muvordvial.com` directo en su cuenta de Vercel. Se conectó al
  proyecto frontend con `www.muvordvial.com` como destino de Production
  y `muvordvial.com` (sin www) redirigiendo 308 hacia `www.muvordvial.com`
  — decisión explícita del usuario de que la versión con `www` sea la
  URL principal. El dominio de Vercel original, `muvo-rd.vercel.app`,
  se dejó activo a propósito como fallback.
- CORS: `origenesPermitidos` en `app.js` **ya no depende de una sola
  variable de entorno** — se cambió a una lista fija con los 4 orígenes
  válidos, para no perder acceso desde ningún dominio activo:
  ```js
  const origenesPermitidos = [
    "http://localhost:3000",
    "https://www.muvordvial.com",
    "https://muvordvial.com",
    "https://muvo-rd.vercel.app",
  ];
  ```
- Variables de entorno relevantes: JWT_SECRET, JWT_EXPIRES_IN, Cloudinary
  (cloud name/api key/secret), RESEND_API_KEY, RESEND_FROM,
  TELEGRAM_BOT_TOKEN, MONGODB_URI (o MONGO_URI, según el .env real — los
  scripts de mantenimiento en `scripts/` prueban ambos nombres),
  FRONTEND_URL — sigue existiendo, pero ya **no** se usa para CORS (ver
  arriba); solo la usa `utils/notificaciones.js` para armar los links y
  el logo dentro de los correos. Actualizada a `https://www.muvordvial.com`
  (sin `/` al final). **NUEVAS (04/09/2026):** `GEMINI_API_KEY` (chatbot,
  ver sección más abajo), `GEMINI_MODEL` (opcional, default
  `gemini-3.6-flash` en el código), `CRON_SECRET` (protege el endpoint
  del resumen diario, debe coincidir con el mismo secreto en GitHub
  Actions).
- **Bloqueante de Resend resuelto (27/08/2026):** `muvordvial.com` se
  verificó en Resend (DKIM/SPF/DMARC en verde vía "Auto configure",
  conectado directo a la cuenta de Vercel — no hizo falta copiar
  registros DNS a mano). `RESEND_FROM` pasó de
  `onboarding@resend.dev` a `Muvo RD Vial <hola@muvordvial.com>`.
  Confirmado en producción: correos transaccionales (verificación de
  cuenta) y el correo del formulario de Empresas ya salen desde el
  dominio propio y llegan a la bandeja real, ya no solo a la cuenta
  dueña de Resend. Esto también desbloquea el formulario de Empresas
  (ver más abajo) — el correo real ya llega, aunque **hoy el único
  destinatario activo en `DestinatarioNotificacion` sigue siendo una
  cuenta personal de pruebas** (`ramndiaz@gmail.com`), no el correo
  institucional de la fundadora — pendiente que ella confirme cuál
  quiere usar (ver Pendiente real más abajo).

## Audiencia del curso (cambio de alcance, 06/08/2026)

El curso **ya no es exclusivo para mujeres** — a partir de esta fecha
también está dirigido a adolescentes de ambos sexos. El backend en sí
nunca tuvo lógica específica de género (roles, validaciones y modelos son
neutros desde el inicio), así que este cambio no tocó ninguna colección
ni controller. El impacto real está en el frontend (copy, textos de
marketing) — ver ARQUITECTURA_FRONTEND.md.

## Estructura real de planes (aclarado 13/08/2026)

Un solo curso teórico (4 sesiones, igual para todas) con **dos variantes
de práctica de manejo**: `normal` y `vip` — la diferencia es solo en la
práctica (más personalizada, más tiempo con el instructor en el plan
VIP), no en el contenido teórico. Esto ya estaba modelado así desde antes
en `Inscripcion.tipoPlan` (`enum: ["normal", "vip"]`); lo nuevo es que
ahora también se muestra en el home público (ver
ARQUITECTURA_FRONTEND.md), leyendo los mismos precios de
`GET /api/configuracion` que ya usaba `/inscripcion`.

## Autenticación y roles

- JWT propio (sin cookies) — cada request protegido manda
  `Authorization: Bearer <token>` a mano desde el frontend.
- 3 roles: `estudiante`, `coordinadora`, `admin` (la fundadora, María Díaz
  Guzmán — cuenta real: `maria@test.com`).
- `middleware/auth.js`: `protegerRuta` (requiere token válido) y
  `permitirRoles(...roles)` (restringe por rol).
- Login rechaza con 403 si `usuario.activo` es `false`.

### Auth (/api/auth)

Sin cambios desde la versión anterior — registro, verificación de email,
recuperación de contraseña, login con rechazo por cuenta desactivada.

## Usuarios (/api/usuarios)

Sin cambios de esquema ni de endpoints desde la versión anterior.

## Sesiones, contenido y exámenes

### `Sesion` — límite de 4, colección ya recreada (06-13/08/2026)

```js
numero: { type: Number, required: true, unique: true, min: 1, max: 4 },
```

El script `scripts/crearSesionesIniciales.js --confirmar` (documentado
como pendiente en la versión anterior de este archivo) **ya se ejecutó**
— las 4 sesiones existen en Atlas con títulos provisionales ("Sesión
1"..."Sesión 4"). Sigue sin haber un `POST /sesiones` — `sesionController.js`
solo expone `listarSesiones`, `obtenerSesionParaEstudiante` y
`actualizarSesion` (PATCH). Ver DATABASE.md para el estado real de la
colección.

**ACTUALIZADO (05/09/2026):** `obtenerSesionParaEstudiante` ahora exige
también haber completado `TestPsicologico` antes de devolver el
contenido de **cualquier** sesión (no solo la 1) — ver la nueva sección
"Test psicológico de perfil conductual" más abajo. Si falta, responde
403 con `codigo: "TEST_PSICOLOGICO_PENDIENTE"` para que el frontend
pueda distinguirlo de "sesión no desbloqueada" y redirigir a la
estudiante al lugar correcto.

**Sigue pendiente:** definir los 4 temas reales con la fundadora y
renombrar las sesiones — el backend ya soporta el rename vía
`PATCH /sesiones/:numero` (`titulo`), pero el panel de coordinadora
todavía no tiene un formulario para eso (ver ARQUITECTURA_FRONTEND.md).

### `ContenidoSesion` — ahora soporta subir PDF como archivo, no solo pegar URL (13/08/2026)

Antes, `tipo: "pdf"` solo aceptaba una URL pegada a mano en `url`. Ahora
se puede **subir el archivo real** desde el panel de coordinadora, con
entrega firmada — mismo patrón que ya existía para los diplomas.

```js
{
  sesionId: ObjectId,
  titulo: String,
  tipo: "video" | "pdf" | "enlace" | "texto",
  url: String,               // video/pdf/enlace
  publicIdCloudinary: String, // NUEVO — solo si el pdf se subió como archivo
  contenidoTexto: String,    // texto
  imagenUrl: String,         // portada opcional, cualquier tipo
  orden: Number,
  activo: Boolean,
}
```

Flujo completo:

1. **Subida** — `POST /api/uploads/pdf` (coordinadora/admin, multipart,
   campo `pdf`, límite 15MB, `middleware/upload.js` → `uploadPDF`).
   Sube a Cloudinary con `resourceType: "raw"` (carpeta
   `mav-rd/contenido-sesion`) y devuelve `{ url, publicId }`.
2. **Guardado** — `crearContenido`/`editarContenido` en
   `contenidoSesionController.js` aceptan y guardan `publicIdCloudinary`
   igual que cualquier otro campo.
3. **Entrega** — `GET /api/contenido-sesion/:id/archivo` (NUEVO). Como
   Cloudinary bloquea la entrega pública de recursos `raw` sin firmar
   (mismo motivo por el que existe `generarUrlDescargaFirmada` para los
   diplomas), este endpoint genera una URL firmada al momento, hace
   fetch a Cloudinary y sirve el PDF inline. Vive **fuera** de
   `protegerRuta` en `routes/contenidoSesion.js` porque un `<a href>` de
   descarga no puede mandar headers — verifica el token manualmente
   (header o `?token=`), mismo patrón exacto que
   `diplomaController.js#obtenerUsuarioDesdeToken`. A diferencia del
   diploma, aquí sí valida que la estudiante tenga la sesión
   desbloqueada (`sesion.numero <= progreso.sesionActualDesbloqueada`)
   antes de entregarle el archivo — coordinadora/admin tienen acceso
   libre.
4. **Fallback** — si el material tiene `url` pero no
   `publicIdCloudinary` (porque alguien pegó un link externo a mano en
   vez de subir un archivo), el frontend usa `url` directo. Contenido
   viejo (de antes de este cambio) sigue funcionando así.

**Actualización 28/08/2026 — contenido cargado, pero con errores
graves, hay que rehacerlo:** en una sesión sin documentar (entre el
13/08 y el 27/08) sí se cargó contenido real para las 4 sesiones —
títulos reales por material ("1.1 Bienvenida a Muvo RD Vial", etc.) —
pero los **PDFs tienen errores de codificación** (marcas y símbolos
extraños en el texto, no presentables a una estudiante real). El flujo
técnico en sí (subida → Cloudinary → URL firmada) sigue funcionando
bien — el problema es la calidad de los archivos originales, no el
código. Plan acordado con el usuario: **borrar todo vía `curl`** contra
los endpoints reales (no a mano en Atlas) y volver a subir los PDFs
corregidos. Ver también DATABASE.md (sección `contenidoSesion`) y el
mismo problema en paralelo con `Examen` (ver más abajo).

### `intentarDesbloquear()` y `entregarIntento()` — sin cambios en esta sesión

Sin cambios de comportamiento desde la versión anterior de este archivo.

### `Examen` — creados, pero con un bug grave (28/08/2026)

Las 4 versiones (una por sesión) sí se llegaron a crear en una sesión
sin documentar, pero el usuario detectó que **la respuesta correcta
cae siempre en la opción A**, en todas las preguntas — patrón
predecible que cualquier estudiante puede explotar sin siquiera leer la
pregunta. Ver el detalle completo en DATABASE.md (sección `examenes`).
Mismo plan que `ContenidoSesion`: borrar vía `curl` y recrear, esta vez
con las opciones en orden aleatorio por pregunta antes de guardar.

## NUEVO: Test psicológico de perfil conductual (05/09/2026)

Pedido directo de la fundadora: entre el pago confirmado y el acceso al
contenido, agregar un cuestionario de perfil psicológico/conductual —
partió de un instrumento en papel ya existente y usado por la
fundadora ("Test de Perfil Psicológico y Conductual del Conductor",
54 preguntas de escala + 5 de reflexión abierta, agrupadas en 7
categorías: autocontrol, estrés/emociones, percepción del riesgo,
atención/concentración, actitud/responsabilidad, confianza,
presión social).

**Decisión de alcance, importante:** el documento en papel tiene una
segunda mitad (indicadores del evaluador, perfil orientativo, y
recomendación) que requiere criterio profesional humano — **se decidió
explícitamente NO digitalizar esa parte**. Solo se digitalizaron las
secciones que llena el propio estudiante. Tampoco se calcula ningún
promedio ni puntaje por sección en el sistema — el instrumento mezcla
preguntas en sentido positivo y negativo a propósito (diseño
profesional estándar de este tipo de tests), así que un promedio
simple daría un número que parece objetivo pero no lo es. La
coordinadora ve las respuestas crudas, igual que si leyera el papel —
la interpretación sigue siendo 100% humana.

- **`models/TestPsicologico.js`** — `userId` único (una sola vez por
  estudiante, igual que el diploma), `respuestas` (array de exactamente
  54 números entre 1 y 5), `reflexiones` (array de 5 strings,
  opcionales — son las preguntas más sensibles, incluyen preguntar por
  incidentes/accidentes previos).
- **`POST /api/test-psicologico/mi-respuesta`** (estudiante) — rechaza
  con 409 si ya existe una respuesta previa para ese usuario.
- **`GET /api/test-psicologico/mi-respuesta`** (estudiante) — solo
  devuelve `{ completado: boolean }`, nunca las respuestas — no hay
  pantalla de "ver mis respuestas anteriores" para la estudiante.
- **`GET /api/test-psicologico`** y **`GET /api/test-psicologico/:userId`**
  (coordinadora/admin — mismo nivel de acceso que Estudiantes, no
  exclusivo de admin) — lista y detalle completo para revisión humana.
- **Gate real**: `sesionController.js#obtenerSesionParaEstudiante`
  verifica `TestPsicologico.exists({ userId })` antes de devolver
  contenido de cualquier sesión — ver actualización en la sección de
  arriba.
- **Consentimiento**: el frontend muestra el mismo disclaimer del
  documento original ("no constituye diagnóstico psicológico...") con
  checkbox obligatorio antes de mostrar las preguntas — no se
  implementó a nivel de backend (es solo UI), así que técnicamente el
  backend no puede verificar que se mostró, pero no hay forma de
  llegar al formulario sin pasar por esa pantalla en el flujo normal.

**Pendiente real, no resuelto por Claude:** este cuestionario captura
información psicológica/conductual que probablemente califica como
"dato sensible" bajo la Ley 172-13 de Protección de Datos de República
Dominicana. Se agregó consentimiento explícito y se restringió el
acceso a coordinadora/admin, pero la fundadora debe confirmar con
asesoría legal si hace falta algo más (política de privacidad
específica, tiempo de retención de datos, etc.) antes de usarlo con
estudiantes reales — no es una recomendación legal, solo una alerta de
que el tema existe.

## Diplomas (/api/diplomas)

Sin cambios en esta sesión.

## Inscripciones y pagos

Sin cambios en esta sesión.

## Sistema de notificaciones (internas)

`utils/notificaciones.js` — se agregó `enviarSolicitudEmpresarial`
(13/08/2026), que reutiliza exactamente el mismo mecanismo que
`notificarNuevoVoucher`/`notificarBalancePendiente`: busca todos los
`DestinatarioNotificacion` con `activo: true` y notifica a cada uno por
su `tipo` (`email` vía Resend, `telegram` vía Bot API). No se agregó
ninguna colección ni configuración nueva — llega al mismo correo
institucional que ya recibe los demás avisos internos.

## Formulario empresarial (Empresas)

Primera versión (13/08/2026), deliberadamente simple — **solo
generación de leads**, sin modelo de precios escalonado ni inscripción
grupal real (decisión explícita: evaluar demanda real antes de
construir esa lógica).

- `routes/empresasRoutes.js` — `POST /api/empresas/contacto`, público
  (fuera de `protegerRuta`, es un formulario de contacto abierto en
  `/empresas`).
- `controllers/empresasController.js#enviarContactoEmpresarial` — valida
  que vengan `nombreEmpresa`, `contacto`, `telefono` y `email`.
- **ACTUALIZADO (04/09/2026): ahora sí persiste en Mongo.** Se agregó
  `models/SolicitudEmpresarial.js` (ver DATABASE.md) — el controller
  guarda el lead primero (`SolicitudEmpresarial.create`) y luego llama a
  `enviarSolicitudEmpresarial` como ya hacía antes. Si el correo falla,
  el registro ya quedó guardado — resuelve el pendiente que existía
  desde el 13/08 ("si Resend falla, no queda registro"). Necesario
  además para que el chatbot y el resumen diario puedan consultar
  solicitudes de Empresas (ver secciones nuevas más abajo).
- Montado en `app.js` como `app.use("/api/empresas", empresasRoutes)`.

## NUEVO: Chatbot interno para la fundadora — Gemini + function calling (04/09/2026)

Idea surgida de una sesión de brainstorm sobre automatización para
María, que tiene poco tiempo para revisar el panel seguido. Decisión:
chatbot con acceso de solo lectura a los datos reales de la app (no un
chatbot genérico) — ella pregunta en lenguaje natural, el modelo decide
qué consultar, y responde con cifras reales, nunca inventadas.

- **`POST /api/chatbot/preguntar`** (`routes/chatbotRoutes.js`) —
  exclusivo `admin` (mismo patrón que `destinatarioRoutes.js`,
  ni siquiera coordinadora tiene acceso). Body: `{ pregunta: string }`.
- **`controllers/chatbotController.js`** — orquesta un loop de function
  calling contra la API de Gemini (`generateContent`, formato REST, sin
  SDK — mismo estilo que `notificaciones.js`, fetch nativo). Hasta 5
  pasos antes de rendirse, para evitar loops infinitos.
- **`utils/geminiHerramientas.js`** — 7 herramientas de **solo lectura**
  (nunca pueden crear/editar/borrar nada): `contarInscripciones`,
  `contarEstudiantesActivos`, `balanceMes`, `vouchersPendientes`,
  `buscarEstudiante`, `solicitudesEmpresariales`, `resultadosExamenes`.
  Cada una es una consulta Mongoose directa contra los modelos reales.
- **Modelo**: `gemini-3.6-flash` (configurable vía `GEMINI_MODEL`, con
  ese valor como default). Capa gratuita de Google confirmada vigente
  (rate-limited, no ilimitada) — requiere API key propia en
  [aistudio.google.com](https://aistudio.google.com), proyecto de
  Google Cloud **sin facturación activada** (activarla mata la capa
  gratuita para ese proyecto).

### Turbulencia real al integrar (documentada para no repetir la

### investigación si Google vuelve a cambiar algo)

Google está iterando la API de Gemini muy rápido (3.6 → 3.7 → 3.8 Flash
en cuestión de semanas durante 2026). Dos problemas reales encontrados
al conectar, ambos resueltos:

1. El modelo original usado (`gemini-2.5-flash`) ya no está disponible
   para cuentas nuevas — el propio error 404 de Google recomendó migrar
   a `gemini-3.6-flash`. La API `generateContent` en sí sigue
   totalmente soportada (aunque Google la considera "legacy" frente a
   su nueva "Interactions API") — no hizo falta reescribir la
   integración, solo cambiar el nombre del modelo.
2. **Cambio de formato en function calling con Gemini 3.x**: el rol
   para devolver el resultado de una herramienta pasó de `"function"` a
   `"user"`, y cada `functionResponse` ahora debe incluir el mismo `id`
   que trajo la `functionCall` correspondiente (antes no era
   obligatorio) — sin esto, Gemini rechaza la request con 400.
3. Se agregaron reintentos automáticos (hasta 3, con espera creciente)
   ante errores 503 ("alta demanda") o 429 — comunes en la capa
   gratuita en picos de uso, y transitorios.

**Si en el futuro un error similar vuelve a aparecer** (Google
deprecando el modelo actual, o cambiando de nuevo el formato), el
primer paso es buscar el error exacto — Google documenta bien estos
cambios y suele indicar la migración exacta en el mensaje de error.

### Frontend

`app/(admin)/admin/asistente/page.tsx` (nuevo) — pantalla de chat
simple, burbujas de mensaje, 4 preguntas de ejemplo como botones para
la primera vez, auto-scroll. Accesible desde una tarjeta nueva en
`panel/page.tsx`, grupo "Solo fundadora" (icono `Bot` de lucide-react).
Ver ARQUITECTURA_FRONTEND.md.

## NUEVO: Resumen diario automatizado por correo y Telegram (04/09/2026)

Segunda pieza de automatización de la misma sesión de brainstorm. A las
9:00 PM hora de Santo Domingo, se calcula y envía un resumen de la
actividad del día — reutiliza el mismo mecanismo de envío que ya usan
vouchers/balance/empresas (Resend + Telegram Bot API,
`DestinatarioNotificacion`).

- **`utils/resumenDiario.js`** — `calcularResumenDelDia()` hace las
  consultas (nuevas inscripciones, pagos confirmados/rechazados,
  vouchers pendientes **acumulados** — no solo de hoy, nuevos
  registros, diplomas generados, solicitudes de Empresas, exámenes
  aprobados/reprobados) y arma el texto plano + HTML.
- **`POST /api/interno/resumen-diario`** (`routes/resumenRoutes.js` +
  `controllers/resumenController.js`) — **fuera de `protegerRuta` a
  propósito**: quien llama es un robot (GitHub Action), no una persona
  con sesión iniciada. Se verifica un secreto compartido en el header
  `x-cron-secret` contra `process.env.CRON_SECRET` — mismo espíritu que
  la verificación manual de token en
  `GET /contenido-sesion/:id/archivo`, pero aquí el secreto es fijo, no
  por usuario.
- **Disparador**: `.github/workflows/resumen-diario.yml` en el repo del
  backend — GitHub Action programado (`cron: "0 1 * * *"`, que es
  01:00 UTC = 9:00 PM AST) que hace `POST` al endpoint de arriba. Sin
  costo — corre en la infraestructura de GitHub, no en Render, así que
  también sirve para "despertar" a Render si estaba dormido por
  inactividad (tier free). También soporta disparo manual
  (`workflow_dispatch`) desde la pestaña Actions, útil para probar sin
  esperar a la hora programada.
- **Variable de entorno nueva**: `CRON_SECRET` — debe existir con el
  mismo valor exacto en Render (Environment) y en GitHub (Settings →
  Secrets and variables → Actions).

**Nota real de esta sesión, no relacionada al código:** al hacer el
primer push del archivo `.yml`, GitHub rechazó el push
(`refusing to allow a Personal Access Token to create or update
workflow ... without workflow scope`) — los tokens de acceso personal
necesitan el permiso `workflow` explícito para tocar archivos dentro de
`.github/workflows/`. Se resolvió subiendo ese archivo específico
directo desde la interfaz web de GitHub (que no tiene esa restricción)
y luego sincronizando con `git pull`. Si se vuelve a tocar un archivo
de Actions desde la terminal, regenerar el Personal Access Token con el
scope `workflow` incluido evita este paso extra.

## Scripts de mantenimiento (`scripts/`)

- **`purgarDatosPrueba.js`**: sin cambios, ya documentado. Corrido el
  06/08/2026.
- **`crearSesionesIniciales.js`**: **ya se ejecutó** (con `--confirmar`)
  — las 4 sesiones existen con títulos provisionales. Este cambio no se
  había registrado formalmente en la versión anterior de este documento;
  queda corregido aquí.

## Notas de diseño

- Ningún borrado es físico donde importa la integridad histórica:
  `Examen`, `ContenidoSesion` y `User` (vía `activo`) son siempre soft
  delete. `Diploma` no tiene ni necesita soft delete.
- Patrón de "entrega firmada al momento" para recursos `raw` de
  Cloudinary: nació con los diplomas, ahora también lo usa el PDF de
  material de estudio. Si en el futuro se necesita un tercer caso,
  replicar el mismo patrón (`generarUrlDescargaFirmada` +
  verificación manual de token) en vez de inventar uno nuevo.

## Pendiente real (backend)

- **Confirmar cumplimiento legal del test psicológico (Ley 172-13)**
  antes de usarlo con estudiantes reales — ver detalle en la sección
  "Test psicológico de perfil conductual" de arriba. Requiere que la
  fundadora consulte con asesoría legal, no es algo que Claude pueda
  resolver por su cuenta.
- **ALTA PRIORIDAD (28/08/2026): borrar y recrear `ContenidoSesion` +
  `Examen` desde cero.** Ambos se cargaron en una sesión sin documentar,
  pero con defectos serios — PDFs con codificación rota y exámenes con
  la respuesta correcta siempre en la opción A. Ver detalle completo en
  las secciones de arriba y en DATABASE.md. Plan: borrar vía `curl`
  contra los endpoints reales, recrear con PDFs corregidos y opciones de
  examen aleatorizadas.
- Renombrar las 4 `Sesion` (`Sesion.titulo` sigue en "Sesión
  1"..."Sesión 4", confirmado en captura real del 28/08/2026) — a mano
  vía `PATCH /sesiones/:numero` hasta que exista un formulario en el
  panel. Buen momento para hacerlo junto con el punto de arriba.
- **Definir el destinatario real de `DestinatarioNotificacion`:** hoy el
  único registro `activo: true` de tipo email sigue siendo una cuenta
  personal de pruebas (`ramndiaz@gmail.com`), no el correo institucional
  de la fundadora — confirmar con ella cuál correo quiere recibir los
  avisos (voucher nuevo, balance pendiente, solicitud empresarial) y
  actualizar/agregar el registro correspondiente.
- Terminar Telegram para el celular de la fundadora (`chat_id`) — sería
  el canal de respaldo si algún correo de Resend llegara a fallar.
- **NUEVO:** no existe una UI de admin para editar `Configuracion`
  (`precio_plan_normal`/`precio_plan_vip`) — hoy se cambian a mano en
  Atlas. Ahora que el precio también se muestra en el home público, un
  cambio de precio mal hecho ahí se refleja directo en el sitio; vale la
  pena construir un formulario simple en el panel de admin en algún
  momento.
- Decidir si vale la pena construir `POST /sesiones` (crear sesión desde
  el panel) o si el script de terminal es suficiente a largo plazo.
- Recordatorios por correo (examen disponible / voucher sin seguimiento):
  ideas a futuro, sin diseñar. **Actualización (04/09/2026): el
  disparador sin cron real ya no es un obstáculo** — el patrón
  GitHub Action programado → endpoint protegido por `CRON_SECRET`, ya
  construido para el resumen diario, se puede reutilizar tal cual para
  estos dos recordatorios. Sigue pendiente solo diseñar el contenido y
  la frecuencia de cada uno.
- **NUEVO:** monitorear la vigencia del modelo de Gemini usado en el
  chatbot (`gemini-3.6-flash`, ver sección de arriba) — Google está
  reemplazando modelos cada pocas semanas (3.6 → 3.7 → 3.8 Flash
  durante 2026). Si el chatbot empieza a fallar con error 404, es
  señal de que hay que revisar el modelo vigente y actualizar
  `GEMINI_MODEL` en Render.
- Seguridad/confiabilidad: rotar credenciales expuestas, rate limiting en
  login/verificar-diploma/**empresas/contacto** (formulario público
  nuevo, sin límite de envíos todavía), CORS dinámico, Sentry — al final,
  cuando la app esté más madura.
- Afinar el rol `backup_readonly` en Atlas de `readAnyDatabase@admin` a
  un rol Read específico sobre `mav_rd` (no urgente, es de solo lectura).
