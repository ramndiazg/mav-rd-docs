# Arquitectura del Backend — mav-rd-backend

> Refleja el estado REAL del código al 10/09/2026. Reemplaza la versión
> anterior de este mismo archivo. Para el historial de cómo se llegó aquí,
> ver HISTORIAL_MODIFICACIONES.md.

Stack: Node.js + Express + Mongoose (MongoDB Atlas, cluster compartido
mujeresalvolante.rd4sofa.mongodb.net, versión real 8.0.29) + JWT (sin
cookies) + Cloudinary (archivos) + Resend (email) + Telegram Bot API
(avisos internos) + despliegue en Render (mav-rd-backend.onrender.com).

## Seguridad — ataque de registro masivo (construido en sesión previa sin documentar, confirmado y documentado el 10/09/2026)

Se detectó un bot llenando repetidamente los formularios de registro
(`/registro`) y de contacto empresarial (`/empresas`) — cuentas y leads
falsos, cada uno disparando un correo real por Resend. Se construyeron
tres capas, confirmadas en el código de esta sesión:

- **Rate limiting por IP** (`middleware/rateLimiters.js`, paquete
  `express-rate-limit`): `limitadorRegistro` (5 registros/hora),
  `limitadorLogin` (15 intentos/15 min, fuerza bruta de contraseña),
  `limitadorCorreoTransaccional` (5/hora — olvidé mi contraseña /
  reenviar verificación), `limitadorContactoEmpresarial` (5/hora), y
  `limitadorInterno` (20/15 min, endpoints de cron protegidos por
  `x-cron-secret`). Depende de que Render mande bien
  `X-Forwarded-For` (`app.set("trust proxy", ...)` en `app.js`) — un
  atacante con muchas IPs distintas puede evadir esta capa, por eso no
  es la única.
- **Cloudflare Turnstile** (`utils/captcha.js`, función
  `verificarCaptcha`) — la capa que de verdad frena un ataque
  distribuido, porque no depende de la IP. Verifica contra
  `https://challenges.cloudflare.com/turnstile/v0/siteverify` con la
  variable de entorno `TURNSTILE_SECRET_KEY`. Solo está wireado en
  `POST /api/auth/registro` (`authController.js`) — el formulario de
  `/empresas` **no** lo usa, solo tiene rate limit + honeypot (ver
  abajo). Si `TURNSTILE_SECRET_KEY` no está configurada, se deja pasar
  con advertencia en consola (pensado para desarrollo local); en
  producción (Render) **siempre debe estar puesta**. Si Cloudflare
  mismo está caído, también se deja pasar — mismo criterio que el
  resto del proyecto para dependencias externas no críticas.
- **Honeypot** (`sitioWeb`) — campo oculto por CSS en el formulario
  (invisible para una persona real, pero un bot simple que no ejecuta
  CSS lo rellena igual). Presente en `POST /api/auth/registro` y
  `POST /api/empresas/contacto`. Si llega con contenido, se responde
  201 "éxito" falso (no un error) para no darle al script ninguna
  pista de qué lo detectó, sin crear nada real. Se registra en consola
  con IP y el correo que intentó usar.

**Variable de entorno nueva:** `TURNSTILE_SECRET_KEY` (Render). El
frontend necesita su propia clave pública, `NEXT_PUBLIC_TURNSTILE_SITE_KEY`
— ver ARQUITECTURA_FRONTEND.md.

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

## Reestructuración de planes: Fundación/Estándar/VIP + colección `Plan` (07/09/2026)

Reemplaza la sección anterior (2 planes, precios en `Configuracion`).
Ahora son **3 planes** dentro del programa `estandar` (Fundación, Normal
→ renombrado a Estándar, VIP), cada uno con datos reales de práctica, no
solo precio — nueva colección `Plan` (ver DATABASE.md para el schema
completo y la tabla de precios/sesiones/costos actual).

- **`GET /api/planes?programa=estandar`** (público) — lista los planes
  activos de un programa, default `"estandar"`. Lo consumen tanto el
  Home (resumen: nombre + precio + frase destacada) como `/inscripcion`
  (detalle completo: sesiones de práctica, duración, costo de
  combustible, características).
- **`GET /api/planes/:codigo?programa=estandar`** (público) — detalle de
  un plan.
- **`GET /api/planes/admin/todos?programa=estandar`** (solo admin) —
  igual que el listado público pero incluye los planes con
  `activo: false`, para que la UI de admin pueda reactivarlos.
- **`PATCH /api/planes/:codigo?programa=estandar`** (solo admin) — edita
  cualquier campo del plan excepto `codigo`/`programa`. La consume la
  pantalla nueva `admin/planes/page.tsx` (ver ARQUITECTURA_FRONTEND.md).
- `Inscripcion.tipoPlan` pasó de `["normal","vip"]` a
  `["fundacion","normal","vip"]`.
- **Campo `programa`** en `Plan` e `Inscripcion` (`String, default:
  "estandar"`, sin enum cerrado): se agregó en esta misma sesión, antes
  de que existiera ningún documento real de `Plan`, específicamente para
  no tener que migrar después. `programa` (currículo) queda separado de
  `tipoPlan` (nivel de práctica/precio dentro de ese currículo).
  **Decisión cerrada (10/09/2026):** `Plan` solo necesita entradas para
  programas públicos/individuales (`estandar`, y desde el 13/09/2026
  también Motorizados/Pesados) — Escolar/Empresarial nunca se muestran
  al público (las instituciones no entran a la app), su precio vive
  solo en `Grupo.precioAcordado`, no necesitan fila en `Plan`.
- `scripts/migrarPlanes.js` (nuevo) — siembra/actualiza los 3 planes
  (upsert por `{programa, codigo}`) y reetiquetó las inscripciones viejas
  `tipoPlan: "normal"` a `"fundacion"` (0 documentos migrados en la
  práctica, no quedaba ninguna inscripción real al momento de correrlo).
- `configuracionController.js`: `DEFAULTS` ya no incluye
  `precio_plan_normal`/`precio_plan_vip` — esos registros quedan
  huérfanos en Atlas (no se borraron) pero ningún endpoint los lee.

### Decisión cerrada (10/09/2026), campo agregado en código (11/09/2026): modelo de contenido para programas con currículo propio

`Sesion`/`Examen`/`ContenidoSesion` **no se duplican por programa**.
`Sesion` tiene el campo `programaContenido` (`String`, default
`"estandar"`, sin enum cerrado — mismo espíritu que `Plan.programa`);
`Examen`/`ContenidoSesion` no necesitan su propio campo porque
referencian `sesionId` y quedan filtrados transitivamente. El índice
único de `Sesion` es `{ programaContenido, numero }` compuesto — ver
detalle en la sección `Sesion` más abajo. Escolar y Empresarial no
entran en este campo porque no lo necesitan: sus estudiantes consumen
las `Sesion` con `programaContenido: "estandar"` sin crear nada propio.

### Motorizados y Pesados — construidos (13/09/2026)

Dos programas 100% teóricos (sin práctica de manejo), cada uno con su
propio plan único `codigo: "teorico"` (sin niveles Fundación/Estándar/
VIP) y sus propias 4 `Sesion` (`programaContenido: "motorizados"` /
`"pesados"`). Nombre en copy y en código: **"Motorizados"**, no
"Motoristas" (la fundadora lo señaló como un término que puede sonar
despectivo).

- `Plan.codigo` acepta `"teorico"`; `modalidadPractica`,
  `duracionSesionMinutos` y `costoPorSesion` son `required: false` (un
  plan teórico no tiene práctica, no hay con qué llenarlos).
- `Inscripcion.tipoPlan` acepta `"teorico"`.
- `utils/elegibilidadPractica.js` centraliza el criterio de "no requiere
  práctica" (Motorizados/Pesados, igual que estudiantes con `grupoId`);
  usado en `diplomaController.js`, `practicaController.js` e
  `intentoExamenController.js`.
- `sesionController.js` y `diplomaController.js` filtran por
  `programaContenido` en sus consultas de `Sesion`.
- `inscripcionController.js` acepta `programa` del body (default
  `"estandar"`), valida `tipoPlan` contra la lista válida de ese
  programa, y lo propaga a `ProgresoEstudiante` al confirmar el pago.
- `scripts/sembrarMotorizadosPesados.js` — ya corrido en producción
  (13/09/2026): 4 `Sesion` + plan "teorico" de cada programa
  (RD$3,500 Motorizados, RD$4,500 Pesados).
- **Pendiente real:** cargar contenido de estudio y exámenes reales
  para las 4 sesiones de cada programa (hoy son sesiones vacías, solo
  con título provisional) — ver "Pendiente real (backend)" al final de
  este documento. El nombre a mostrar en el PDF del diploma (¿debe
  decir "Motorizados"/"Pesados" explícito, o alcanza el diseño genérico
  de hoy?) tampoco está resuelto.

## Autenticación y roles

- JWT propio (sin cookies) — cada request protegido manda
  `Authorization: Bearer <token>` a mano desde el frontend.
- 4 roles: `estudiante`, `coordinadora`, `admin` (la fundadora, María Díaz
  Guzmán — cuenta real: `maria@test.com`) y `conductor` (NUEVO,
  05/09/2026 — instructor de práctica, ver sección propia más abajo).
  Ninguno de los 4 tiene registro público — `estudiante` es el único que
  se crea vía `/registro`; los otros 3 los crea un admin desde el panel
  (coordinadora y admin siguen haciéndose a mano en Atlas; `conductor` es
  el primero con un endpoint real, ver más abajo).
- `middleware/auth.js`: `protegerRuta` (requiere token válido) y
  `permitirRoles(...roles)` (restringe por rol).
- Login rechaza con 403 si `usuario.activo` es `false`.

### Auth (/api/auth)

Sin cambios desde la versión anterior — registro, verificación de email,
recuperación de contraseña, login con rechazo por cuenta desactivada.

## Usuarios (/api/usuarios)

**ACTUALIZADO (05/09/2026):** se agregó `POST /api/usuarios/conductor`
(admin) — ver sección "Seguimiento de práctica de manejo" más abajo.

**ACTUALIZADO (10/09/2026):** `User.cedula` dejó de ser `required` a
nivel de esquema — pasó a `unique + sparse` (mismo patrón que ya se usa
en `Inscripcion.numeroReferencia`). Motivo: los menores de un `Grupo`
tipo colegio no tienen cédula, y escribir "N/A" a mano en cada fila
chocaba como cédula duplicada en la segunda estudiante sin cédula (un
índice `unique` normal sí compara dos "N/A" como iguales). Sigue siendo
obligatoria donde corresponde — `authController.js` (autoregistro) la
exige ella misma antes de crear el `User`; en `grupoController.js`
(roster de `Grupo`) ahora es explícitamente opcional, y cualquier valor
vacío o alguna variante de "n/a" se guarda como `undefined` de verdad
(no como el texto "N/A"), para que el índice `sparse` los excluya a
todos correctamente. Ver DATABASE.md.

**NUEVO (10/09/2026): `PATCH /api/usuarios/desactivar-lote`** (admin) —
soft delete en lote: recibe `{ ids: [...] }` y pone `activo: false` a
todos con `User.updateMany`. Mismo mecanismo de siempre (nunca se borra
el `User`), pensado para cuando el roster de una institución cambia y
hay que desactivar a varias estudiantes de un `Grupo` de una sola vez
en vez de ir una por una desde `/panel/estudiantes`. Mismo permiso que
el toggle individual (`PATCH /:id/estado`, solo `admin`) — **nota
pendiente:** la UI que lo consume vive en `/panel/grupos/[id]`, a la
que también entra `coordinadora`; si ella necesita usarlo, hay que
decidir si se abre a `coordinadora` en ambos endpoints (ver "Pendiente
real" más abajo).

El resto del controller (`listarUsuarios`, `crearCoordinadora`,
`cambiarEstado`, `cambiarRol`) sin cambios de comportamiento.

## Sesiones, contenido y exámenes

### `Sesion` — límite de 4, colección ya recreada (06-13/08/2026)

```js
numero: { type: Number, required: true, min: 1, max: 4 },
programaContenido: { type: String, default: "estandar" },
```

```js
sesionSchema.index({ programaContenido: 1, numero: 1 }, { unique: true });
```

**CORREGIDO (11/09/2026):** el índice único era solo sobre `numero`
(global), lo que habría chocado en cuanto existiera una segunda
`Sesion { numero: 1 }` para Motorizados/Pesados. Ahora la unicidad es por
`{ programaContenido, numero }`. El índice viejo `numero_1` ya se
dropeó en Atlas (13/09/2026) antes de sembrar Motorizados/Pesados.

`obtenerSesionParaEstudiante`/`listarSesiones`/`actualizarSesion` ya
filtran por `programaContenido` (13/09/2026) — necesario desde que
existe más de un programa con sesiones reales.

El script `scripts/crearSesionesIniciales.js --confirmar` (documentado
como pendiente en la versión anterior de este archivo) **ya se ejecutó**
— las 4 sesiones de `estandar` existen en Atlas con títulos
provisionales ("Sesión 1"..."Sesión 4"). Sigue sin haber un
`POST /sesiones` — `sesionController.js` solo expone `listarSesiones`,
`obtenerSesionParaEstudiante` y `actualizarSesion` (PATCH). Ver
DATABASE.md para el estado real de la colección.

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

### `intentarDesbloquear()` y `entregarIntento()` — NUEVO disparador de notificación (05/09/2026)

`intentarDesbloquear()` sin cambios. `entregarIntento()` (en
`intentoExamenController.js`) sí cambió: en el mismo bloque donde ya se
ponía `progreso.cursoCompletado = true` al aprobar la sesión 4, ahora se
captura el valor anterior de `cursoCompletado` (`completadoAntes`) antes
de modificarlo, y si pasó de `false` a `true` en esta misma llamada, se
dispara `notificarEstudianteListaParaPractica` (sin `await`, no bloquea
la respuesta). Ver detalle completo en "Seguimiento de práctica de
manejo" más abajo.

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

## NUEVO: Seguimiento de práctica de manejo (05/09/2026)

Pedido directo del usuario, en dos partes: (1) avisar a un instructor
real cuando una estudiante termina la teoría, con todos sus datos, para
que se le dé seguimiento; (2) que ese mismo instructor confirme la
práctica antes de que se pueda generar el diploma. Deliberadamente
**sin asignación automática** de instructor a estudiante — el chofer se
crea con sus datos y horarios, se le muestran a la estudiante, y es
ella quien lo contacta directamente.

- **`models/Instructor.js`** (NUEVO) — perfil extendido de un `User` con
  `rol: "conductor"`: `diasDisponibles` (array de `{ dia, horario }`,
  texto libre en `horario`) y `activo`. Nombre/teléfono/correo ya viven
  en `User`, no se duplican.
- **`models/DestinatarioPractica.js`** (NUEVO) — mismo esquema que
  `DestinatarioNotificacion`, **colección separada a propósito**: nunca
  se mezcla con los avisos de vouchers/balance/empresas.
- **`User.rol`** — se agregó `"conductor"` al enum.
- **`ProgresoEstudiante`** — 3 campos nuevos: `practicaAprobada`,
  `fechaAprobacionPractica`, `practicaAprobadaPor`. Ver DATABASE.md.
- **`POST /api/usuarios/conductor`** (admin, en `usuarioController.js`)
  — crea el `User` y su `Instructor` asociado en un solo paso. No hay
  registro público, igual que coordinadora/admin (hoy también se crean
  a mano en Atlas — este es el primer rol con un endpoint real para
  crearlo desde el panel, en "Solo fundadora").
- **`GET /api/instructores`** (admin) y **`GET /api/instructores/activos`**
  (estudiante/conductor/coordinadora/admin — solo nombre, teléfono,
  correo y horarios) + `PATCH /api/instructores/:id` (admin, editar
  horarios/activo).
- **`controllers/destinatarioPracticaController.js`** +
  `routes/destinatarioPracticaRoutes.js` — copia 1:1 del patrón de
  `destinatarioController.js`, montado en `/api/destinatarios-practica`,
  exclusivo admin.
- **Disparador**: en `intentoExamenController.js#entregarIntento`, en el
  mismo bloque donde ya se ponía `progreso.cursoCompletado = true`, se
  captura el valor anterior (`completadoAntes`) para notificar **solo
  la primera vez** que se completa — evita reenvíos si algo más
  recalcula el progreso después.
- **`utils/notificaciones.js#notificarEstudianteListaParaPractica`**
  (NUEVA) — va a dos destinos a la vez: cada `Instructor` activo,
  directo a su correo (vía `User.email`), y los `DestinatarioPractica`
  activos (visibilidad para fundadora/coordinadora). Sin `await` a
  propósito, mismo patrón que `enviarCorreoDiplomaListo`.
- **`controllers/practicaController.js`** + `routes/practicaRoutes.js`
  (rol `conductor`/`admin`) — `GET /api/practica/pendientes` (estudiantes
  con `cursoCompletado: true` y `practicaAprobada: false`) y
  `POST /api/practica/:userId/aprobar`.
- **Gate del diploma**: ver sección "Diplomas" más abajo.

Probado de punta a punta en esta sesión — crear chofer, login como
conductor, notificación al completar teoría, aprobación de práctica y
generación de diploma condicionada — todo funcionando.

**CORREGIDO (11/09/2026): gate de práctica centralizado.** El criterio
"¿esta estudiante cursa práctica de manejo?" (antes `!usuario.grupoId`
repetido de forma independiente en `diplomaController.js` x2,
`practicaController.js` e `intentoExamenController.js`) ahora vive en
un solo lugar: `utils/elegibilidadPractica.js#requierePracticaDeManejo`.
Los 4 call sites lo importan en vez de repetir el `if`. Sin cambio de
comportamiento — mismo criterio de hoy (`grupoId`), solo se ordenó
antes de que Motorizados/Pesados (que tampoco tienen práctica, ver
sección "Motorizados y Pesados" más arriba) obligaran a tocar una
cuarta copia.

## Diplomas (/api/diplomas)

**ACTUALIZADO (05/09/2026):** `listarElegibles` y `generarDiploma` ahora
exigen también `progreso.practicaAprobada`, no solo `cursoCompletado` —
ver la sección "Seguimiento de práctica de manejo" de arriba para el
flujo completo. Sin este campo en `true`, `generarDiploma` responde 400
aunque la teoría esté completa.

## NUEVO: Programa Escolar/Empresarial — Grupo, roster, prorrateo y reporte diario (08-09/09/2026)

Diseño completo consolidado en `ESPECIFICACION_PROGRAMAS_NUEVOS.md` tras
varias sesiones de conversación con la fundadora. Construido en dos
sesiones: 08/09 (colección `Grupo`, campo `grupoId` en `User`, gates de
práctica/cuestionario) y 09/09 (formularios de grupo, prorrateo contable
—**reemplazado el 10/09/2026, ver más abajo**—, cron de reporte diario).

- **`models/Grupo.js`** — `tipo: "colegio" | "empresa"`,
  `nombreInstitucion`, datos de contacto, `precioAcordado` (total
  negociado), `cantidadEstudiantesEstimada` (solo referencia, del primer
  formulario), `pendienteRoster` (true hasta cargar el roster real),
  `activo` (false cuando todos completan el curso o a mano), `fechaInicio`
  (se fija al confirmar el roster, no al crear el grupo — de ahí cuentan
  las 24h del primer reporte).
- **`User.grupoId`** — ref a `Grupo`, `null` para autoregistro (sin
  cambios en ese flujo). Determina si se exige práctica para el diploma y
  cuál cuestionario previo aplica.
- **`CuestionarioEscolar`** (colección aparte, NO reusa
  `TestPsicologico`; nombre confirmado con la fundadora el 10/09/2026,
  **código renombrado el 11/09/2026** — hasta entonces el modelo/
  controller/rutas en disco seguían llamándose
  `InformacionComplementariaEscolar` pese a que este documento ya decía
  "renombrado"; quedó corregido: `models/CuestionarioEscolar.js`,
  `controllers/cuestionarioEscolarController.js`, mount en
  `POST/GET /api/cuestionario-escolar`) — 14 preguntas (12 escala 1-5 +
  2 abiertas), gate en
  `sesionController.js#obtenerSesionParaEstudiante`: si `Grupo.tipo ===
"colegio"` exige esta colección; para todo lo demás (incluido
  Empresarial) sigue exigiendo `TestPsicologico`. **Revisión legal del
  set de preguntas (Ley 172-13) ya hecha y aprobada (confirmado
  10/09/2026)** — puede usarse con estudiantes reales.
- **Gate de práctica condicional** — `diplomaController.js`:
  `requierePractica = !usuario.grupoId`; elegible si `cursoCompletado &&
(!requierePractica || practicaAprobada)`. Efectos en cascada ya
  cubiertos: `intentoExamenController.js#entregarIntento` no notifica
  "lista para práctica" si hay `grupoId`; `ProgresoCarretera.tsx`
  (frontend) oculta el paso de práctica. **Cascada adicional encontrada
  el 09/09 que la sesión del 08/09 no cubrió:**
  `practicaController.js#listarPendientes` mostraba para siempre a
  estudiantes de Grupo en la lista de "esperando práctica" (nunca les
  llega `practicaAprobada: true` porque no aplica) — ya filtrado con el
  mismo criterio `!usuario.grupoId`.
- **`controllers/grupoController.js`** + `routes/grupoRoutes.js`
  (`/api/grupos`, exclusivo `coordinadora`/`admin`, las instituciones
  nunca entran a la app):
  - `POST /api/grupos` — Formulario 1: crea el `Grupo` con
    `pendienteRoster: true`. No crea cuentas ni movimientos contables.
  - `GET /api/grupos` — lista con `cantidadEstudiantesReal` calculada al
    vuelo (`User.aggregate` por `grupoId`, no es un campo guardado).
  - `GET /api/grupos/:id` — detalle + roster actual.
  - `PATCH /api/grupos/:id` — editar; `precioAcordado`/
    `cantidadEstudiantesEstimada` solo editables mientras
    `pendienteRoster` sigue `true` (después ya hay contabilidad calculada
    a partir de esos números).
  - `POST /api/grupos/:id/roster` — Formulario 2, la pieza central: - Crea una cuenta `User` por fila (`rol: "estudiante"`, `grupoId`
    seteado, `emailVerificado: true` de entrada — estas cuentas nunca
    pasan por el link de verificación). Una fila con error (cédula/
    correo duplicado, campo faltante) no tumba el resto del lote — se
    reporta en `errores[]` con el número de fila y sigue con las demás. - Por cada cuenta creada: `Inscripcion` con `programa: "escolar"` o
    `"empresarial"` (mapeado desde `Grupo.tipo`), `tipoPlan: "grupo"`
    (**valor nuevo en el enum**, ver DATABASE.md), `estadoPago:
"pagado"` directo (el pago se da por hecho en el Formulario 1, sin
    pasar por la cola de verificación de voucher), `monto` = su parte
    prorrateada — **solo de referencia interna desde el 10/09/2026,
    ver nota de contabilidad abajo, ya no alimenta ningún
    `MovimientoContable`.** - `ProgresoEstudiante` con `sesionActualDesbloqueada: 1` (mismo
    upsert que `confirmarPago` en el flujo individual). - Correo de credenciales (`enviarCorreoCredencialesGrupo`, contraseña
    generada con `crypto.randomBytes`, sin `await` igual que el resto
    de correos transaccionales). - **Cédula opcional (NUEVO, 10/09/2026):** ver la nota de
    `User.cedula` en la sección "Usuarios" arriba — pensado para
    grupos tipo colegio con estudiantes menores sin cédula. - **Contabilidad — CAMBIO (10/09/2026), reemplaza el diseño
    original de "prorrateo contable":** antes se creaba un
    `MovimientoContable` por estudiante, cada uno con su monto
    prorrateado (`Math.floor(precioAcordado / cantidadTotal)`, residuo
    en la primera estudiante del lote). La fundadora pidió eliminarlo
    — muchas entradas pequeñas por el mismo grupo distorsionaban el
    balance y lo hacían difícil de leer de un vistazo. Ahora: se crea
    **una sola** `MovimientoContable` por grupo, por el monto TOTAL
    (`precioAcordado`), y solo en la primera confirmación del roster
    (el primer pago real). `Inscripcion.monto` de cada estudiante
    sigue calculándose igual que antes (prorrateado, residuo en la
    primera del lote) pero ya es puramente informativo — no se suma a
    contabilidad. Las adiciones tardías (ver abajo) **ya no generan
    ningún movimiento contable nuevo**: no hay cobro adicional real
    que registrar, `precioAcordado` ya cubrió al grupo completo por
    delante. - **Adiciones tardías:** el mismo endpoint soporta llamarlo otra vez
    después de la primera confirmación, para agregar estudiantes a un
    grupo que ya inició. `Inscripcion.monto` de las nuevas se sigue
    calculando como referencia (usando el total existentes + nuevos),
    pero — a diferencia del diseño original — no crea ningún
    `MovimientoContable`, por la misma razón de arriba. - Si el conteo real difiere del estimado (solo en la primera
    confirmación), la respuesta trae `discrepancia: true` — aviso no
    bloqueante, se crea igual con la cantidad real. - Al terminar la primera confirmación: `pendienteRoster: false`,
    `fechaInicio: ahora`.
- **`Inscripcion.tipoPlan`** — enum ampliado de `["fundacion","normal","vip"]`
  a `["fundacion","normal","vip","grupo"]`. `"grupo"` es exclusivo de
  estudiantes de un `Grupo`: no tienen nivel individual de plan, el precio
  vive solo en `Grupo.precioAcordado` (nunca pasa por la colección `Plan`
  — resuelve el punto que quedaba abierto en
  `ESPECIFICACION_PROGRAMAS_NUEVOS.md` sección 5, punto 5, a favor de la
  opción que ahí se marcaba como "probable"; **decisión reconfirmada el
  10/09/2026**, ver nota en la sección de `Plan` arriba).
- **`utils/reporteGrupos.js`** + endpoint
  `POST /api/interno/reporte-grupos` (mismo archivo `resumenRoutes.js`/
  `resumenController.js` del resumen diario general, mismo mecanismo de
  `x-cron-secret`) — cron aparte, **10:00 AM RD** (`cron: "0 14 * * *"` en
  `.github/workflows/reporte-grupos.yml`, UTC). Por cada `Grupo` con
  `activo: true` y `fechaInicio` de hace más de 24h: calcula el progreso
  de cada estudiante (sesiones aprobadas, curso completado, diploma) y
  manda un correo aparte a `Grupo.contactoEmail` (fan-out, no un solo
  correo para todos los grupos). Si **todas** las estudiantes del grupo
  tienen `cursoCompletado: true`, ese envío se marca como reporte final y
  `Grupo.activo` pasa a `false` en la misma pasada (no vuelve a entrar
  mañana). El toggle manual de `activo` en `PATCH /api/grupos/:id` sigue
  disponible como respaldo.
- **NUEVO (10/09/2026): soft delete en lote** — ver
  `PATCH /api/usuarios/desactivar-lote` en la sección "Usuarios" arriba.
  Pensado específicamente para cuando el roster de una institución
  cambia (estudiantes que ya no pertenecen al grupo).
- **UI de coordinadora/admin** (ver ARQUITECTURA_FRONTEND.md):
  `/panel/grupos` (listado + Formulario 1), `/panel/grupos/[id]`
  (detalle + Formulario 2 — **solo fila por fila desde el 10/09/2026,
  se eliminó el modo CSV/pegado**, ver nota abajo — más checkboxes para
  desactivar estudiantes en lote), y `/panel/estudiantes` (ahora muestra
  de qué institución es cada estudiante y permite filtrar por grupo —
  ver más abajo).
- **BUG CORREGIDO (11/09/2026): no existía ninguna pantalla para ver
  las respuestas de `CuestionarioEscolar`.** El backend
  (`GET /api/cuestionario-escolar`) siempre devolvió los datos bien —
  lo reportó la fundadora como "los test psicológicos de dos
  estudiantes de un grupo no aparecen", pero no era un bug de datos:
  nunca se construyó la pantalla, solo existía `/panel/test-psicologico`
  (que lista `TestPsicologico`, una colección distinta). Se creó
  `/panel/cuestionario-escolar` (mismo patrón que la de perfil
  conductual) y su tarjeta en `panel/page.tsx`.

**Desplegado y probado en producción (09/09/2026)** — se creó un grupo
real, se cargó un roster, y salieron dos bugs que no aparecían en la
revisión de código del sandbox (ninguno relacionado con la lógica de
Grupo en sí, los dos eran fallas preexistentes que este flujo nuevo puso
en evidencia por ser el primer camino del código que crea varias
`Inscripcion` seguidas sin pasar por el flujo de voucher):

1. **Bug: `numeroReferencia` con `default: null` + índice
   `unique + sparse`.** Un índice `sparse` en Mongo solo excluye
   documentos donde el campo está _ausente_, no donde vale `null`
   explícito. Con `default: null` en el schema, cada `Inscripcion`
   creada sin voucher (el flujo "efectivo" del admin, y ahora cada
   estudiante de un `Grupo`) quedaba con `numeroReferencia: null`
   _guardado de verdad_, así que la segunda de esas chocaba contra la
   primera como si fuera un duplicado — error real:
   `"Ya existe una cuenta registrada con ese numeroReferencia."` al
   agregar una segunda estudiante a un grupo. Corregido en
   `models/Inscripcion.js`: se quitó el `default: null` (queda
   `undefined` cuando no se manda, que el índice `sparse` sí excluye
   correctamente). No requirió tocar el índice en Mongo, solo lo que la
   app escribe al crear el documento.
2. **Mejora pedida tras la prueba: `/panel/estudiantes` no distinguía
   individuales de estudiantes de un grupo, ni agrupaba compañeras de la
   misma institución.** `controllers/usuarioController.js#listarUsuarios`
   ahora acepta `?grupoId=` como filtro y hace
   `.populate("grupoId", "nombreInstitucion tipo")`. El frontend
   (`/panel/estudiantes`) agregó un dropdown "Filtrar por grupo" (poblado
   desde `GET /api/grupos`) y cada fila muestra un badge con el nombre de
   la institución (o "Plan individual" si `grupoId` es `null`). Ver
   ARQUITECTURA_FRONTEND.md.

**Pendiente de esta sesión, no bloqueante:** cuando el bug de
`numeroReferencia` ocurrió, la fila que falló ya había pasado por
`User.create()` exitosamente antes de que `Inscripcion.create()` fallara
— es decir, puede haber quedado una cuenta de estudiante "huérfana" (sin
`Inscripcion`/`MovimientoContable`/`ProgresoEstudiante`) de esa prueba en
la base de datos real. No se limpió todavía — revisar en Mongo Atlas
antes de reintentar con la misma cédula/correo, o simplemente borrar esa
cuenta a mano si no se necesita.

Sigue faltando: probar el cron de `/api/interno/reporte-grupos` en vivo
(esperar 24h reales desde `fechaInicio`, o ajustar la fecha a mano en
Mongo para forzarlo, y dispararlo desde la pestaña Actions de GitHub con
`workflow_dispatch`) y confirmar que el correo de reporte le llega
correctamente al contacto de la institución.

## Inscripciones y pagos

Sin cambios en esta sesión (fuera del bug de "fundacion" corregido más
abajo).

### Bug corregido (09/09/2026): plan "Fundación" rechazado al auto-inscribirse

Cuando se reestructuraron los planes el 07/09 (ver sección de arriba),
`Inscripcion.tipoPlan` pasó a aceptar `"fundacion"` y se creó la colección
`Plan` para reemplazar los precios sueltos en `Configuracion`, pero
`inscripcionController.js` se quedó con el código viejo en dos lugares:

- `crearInscripcion` (admin) y `crearOReenviarInscripcionPropia`
  (autoinscripción) seguían validando `tipoPlan` contra
  `["normal", "vip"]` — cualquier intento de inscribirse en el plan
  Fundación era rechazado con 400.
- `crearOReenviarInscripcionPropia` buscaba el precio con
  `Configuracion.findOne({ clave: "precio_plan_normal" | "precio_plan_vip" })`,
  que ya no existe (esas llaves se migraron a `Plan`, ver nota en
  `configuracionController.js`).

Corregido: validación ahora acepta `["fundacion", "normal", "vip"]` en
ambos lugares, y el precio se busca con
`Plan.findOne({ codigo: tipoPlan, programa: "estandar", activo: true })`
— mismo patrón que ya usaba `planController.js`. `Configuracion` ya no se
importa en este archivo.

## Sistema de notificaciones (internas)

`utils/notificaciones.js` — se agregó `enviarSolicitudEmpresarial`
(13/08/2026), que reutiliza exactamente el mismo mecanismo que
`notificarNuevoVoucher`/`notificarBalancePendiente`: busca todos los
`DestinatarioNotificacion` con `activo: true` y notifica a cada uno por
su `tipo` (`email` vía Resend, `telegram` vía Bot API). No se agregó
ninguna colección ni configuración nueva — llega al mismo correo
institucional que ya recibe los demás avisos internos. **NUEVO
(05/09/2026):** se agregó `notificarEstudianteListaParaPractica`, que
usa un mecanismo aparte (`DestinatarioPractica` + `Instructor`) — ver
sección "Seguimiento de práctica de manejo" arriba.

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

### Turbulencia real al integrar (documentada para no repetir la investigación si Google vuelve a cambiar algo)

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

- **`crearSesionesIniciales.js`**: **ya se ejecutó** (con `--confirmar`)
  — las 4 sesiones existen con títulos provisionales. Este cambio no se
  había registrado formalmente en la versión anterior de este documento;
  queda corregido aquí.
- **`purgarUsuariosPrueba.js`** — único script de purga del proyecto
  (borra todos los usuarios excepto `maria@test.com`, sin importar rol,
  y su cascada asociada). **CORREGIDO Y AMPLIADO (11/09/2026):**
  - Se eliminó `purgarDatosPrueba.js` — era un archivo duplicado con
    contenido idéntico a este, sobrante de una purga anterior
    (06/08/2026); ya no existe ningún otro script de purga.
  - La cascada ahora también borra `CuestionarioEscolar` (bug: nunca se
    había agregado cuando se creó esa colección), y borra por completo
    (sin filtrar por usuario) `Grupo`, `MovimientoContable` y
    `BalanceMensual` — a pedido explícito de la fundadora, para dejar
    la base lista para producción con solo la cuenta admin de prueba.
  - Sigue sin tocar `Sesion`/`Examen`/`ContenidoSesion`/`Plan` (contenido
    del curso, no datos de prueba de una persona) ni
    `SolicitudEmpresarial` (leads del formulario de `/empresas` — no se
    incluyó por no estar explícitamente pedido; agregar a mano si
    también se quiere limpiar).
  - Uso sin cambios: `node scripts/purgarUsuariosPrueba.js` (dry-run) /
    `--confirmar` (real, pide escribir `BORRAR`).
- **`limpiarCuentasBot.js` — CORREGIDO (13/09/2026), se había pasado por
  alto en el rename del 11/09.** Tenía un `require("../src/models/
InformacionComplementariaEscolar")` activo (no un comentario, código
  real) que habría hecho crashear el script apenas se ejecutara. Se
  detectó porque, al desplegar en Render, el `require` equivalente en
  `sesionController.js` sí crasheó el servidor entero (`scripts/` no se
  ejecuta al bootear, por eso este no se notó hasta que se corrió a
  mano). Confirmado con una búsqueda de todo el repo (no solo `src/`)
  que no queda ningún otro caso.

- **`migrarPlanes.js` (NUEVO, 07/09/2026)**: siembra/actualiza los 3
  documentos de `Plan` y reetiqueta inscripciones viejas
  `tipoPlan: "normal"` → `"fundacion"`. Correrlo de nuevo (con
  `--confirmar`) es la forma correcta de actualizar precios/nombres de
  plan en bloque — aunque para cambios puntuales ya es más simple usar
  la UI de admin (`admin/planes`, ver ARQUITECTURA_FRONTEND.md).

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
  **Sigue siendo el único bloqueante de infraestructura abierto** —
  Resend (dominio verificado) y la UI de admin para precios de plan ya
  se cerraron, ver DATABASE.md.
- **NUEVO (10/09/2026): permisos inconsistentes en soft delete de
  estudiantes.** `PATCH /api/usuarios/desactivar-lote` (nuevo) y
  `PATCH /api/usuarios/:id/estado` (ya existía) son ambos `admin`-only,
  pero la UI que los consume vive en `/panel`, al que también entra
  `coordinadora` (`/panel/estudiantes` y, para el nuevo, `/panel/grupos/[id]`).
  Si la coordinadora necesita desactivar estudiantes ella misma, hay que
  decidir si se abre `permitirRoles("coordinadora", "admin")` en ambos
  endpoints.
- **PROGRAMA ESCOLAR/EMPRESARIAL: construido y desplegado
  (08-10/09/2026)** — ver la sección "NUEVO: Programa
  Escolar/Empresarial" más arriba para el detalle completo. Sigue
  pendiente:
  - Probar el cron de reporte diario en vivo (nunca se esperó a que
    pasaran las 24h reales, ni se disparó a mano con `workflow_dispatch`).
  - Limpiar la cuenta de estudiante "huérfana" que pudo haber quedado de
    la prueba donde salió el bug de `numeroReferencia` (ver esa sección).
- **MOTORIZADOS Y PESADOS — construidos y desplegados (13/09/2026).**
  Ver la sección "Motorizados y Pesados" más arriba para el detalle
  completo. Sigue pendiente:
  - Cargar contenido de estudio y exámenes reales para las 4 sesiones de
    cada programa (hoy están sembradas pero vacías, solo con título
    provisional) — con **varias versiones activas por sesión desde el
    día uno**, para no repetir el problema que tuvo `estandar` de quedar
    con una sola versión.
  - Decidir si el PDF del diploma debe mostrar explícitamente
    "Motorizados"/"Pesados" o alcanza el diseño genérico de hoy.
  - Para Pesados: confirmar si camión y trailer comparten exactamente
    las mismas 4 sesiones o en algún punto necesitan contenido distinto
    (si diverge, probablemente conviene un campo `subtipo` en vez de un
    tercer `programaContenido`).
  - Probar de punta a punta: inscripción → pago → cuestionario → 4
    sesiones → exámenes con selección aleatoria real → diploma sin
    pedir práctica.
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
- **NUEVO:** monitorear si con más choferes hace falta algún
  filtro/paginación en `GET /instructores/activos` — hoy devuelve todos
  los activos sin distinción, suficiente para la cantidad actual.
