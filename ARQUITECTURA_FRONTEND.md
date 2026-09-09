# Arquitectura del Frontend — mav-rd-frontend

> Refleja el estado REAL del código al 07/09/2026. Reemplaza la versión
> anterior de este mismo archivo. Para el historial de cómo se llegó aquí,
> ver HISTORIAL_MODIFICACIONES.md.

Stack: Next.js 16 (App Router) + React 19 + Tailwind CSS v4 + lucide-react
(íconos) + `qrcode` (QR generado en el navegador, diploma compartible) +
despliegue en Vercel, ahora con dominio propio: **www.muvordvial.com**.

## Infraestructura y despliegue

- Backend: https://mav-rd-backend.onrender.com/api (variable de entorno
  NEXT_PUBLIC_API_URL — sin cambios, el backend se sigue desplegando
  aparte en Render).
- **Dominio propio (27/08/2026):** la fundadora compró `muvordvial.com`
  directo en su cuenta de Vercel (Settings → Domains del proyecto). URL
  principal de producción: `https://www.muvordvial.com` (con `www` —
  decisión explícita del usuario). `muvordvial.com` (sin www) redirige
  308 hacia la versión con `www`. El dominio original de Vercel,
  `muvo-rd.vercel.app`, se dejó activo como fallback.
- CORS: el backend ya no depende de una sola variable `FRONTEND_URL`
  para esto — `origenesPermitidos` en `app.js` es ahora una lista fija
  que incluye `www.muvordvial.com`, `muvordvial.com` y
  `muvo-rd.vercel.app` a la vez (ver ARQUITECTURA_BACKEND.md).

## ⚠️ Corrección pendiente (sigue abierta, sin cambios esta sesión)

Al pasar `app/(estudiante)/diploma/page.tsx` se confirmó que esa página
vive bajo un grupo de ruta `(estudiante)` que **no estaba documentado**
en el árbol de carpetas. **Sigue sin saberse** si `dashboard`,
`aula-virtual/[sesion]`, `examen/[intentoId]`, `inscripcion` y
`perfil/cambiar-password` también están bajo ese mismo grupo, o si
`diploma` es la única excepción. Tratar el árbol de abajo como
aproximado para esas rutas específicas.

## Audiencia del curso (cambio de alcance, 06/08/2026)

El curso **ya no es exclusivo para mujeres** — incluye también
adolescentes de ambos sexos. Lenguaje neutral aplicado en lo que se ha
tocado hasta ahora; **siguen pendientes de revisar** `testimonios/page.tsx`,
`registro/page.tsx`, correos transaccionales del backend, y las fotos de
`public/inscripcion/` (de clases anteriores, probablemente solo mujeres).

## Estructura real de planes (reestructurado 07/09/2026)

Un solo curso teórico (igual para todas dentro del programa `estandar`),
con **3 variantes de práctica de manejo**: Fundación (grupal, la más
accesible), Estándar (antes "Normal") e VIP. La diferencia entre los 3
es exclusivamente cuánta atención personalizada y tiempo con el
instructor se recibe en la práctica — no hay diferencia en el contenido
teórico. Reemplaza la versión de 2 planes (Normal/VIP) descrita antes en
este documento — ver la sección "Home — Planes y Precios" más abajo para
el detalle del cambio.

## Estructura de carpetas (real)

mav-rd-frontend/
├── app/
│ ├── page.tsx # Inicio — Planes/precios (13/08) + banner Empresas (13/08) + promo libro de la fundadora, colores brand-yellow/brand-mamey nuevos (sesión sin documentar, confirmado 28/08/2026)
│ ├── sitemap.ts # NUEVO (documentado 28/08/2026, existía desde antes sin registrar) — SITE_URL corregido al dominio propio
│ ├── robots.ts # NUEVO (documentado 28/08/2026, existía desde antes sin registrar) — SITE_URL corregido, ya no bloquea /inscripcion por error
│ ├── empresas/page.tsx # NUEVO (13/08/2026) — programa empresarial, informativo + formulario
│ ├── acerca-de-nosotros/page.tsx
│ ├── kit-preparacion/page.tsx
│ ├── noticias/page.tsx
│ ├── noticias/[id]/page.tsx
│ ├── testimonios/page.tsx # sin revisar — lenguaje de género pendiente
│ ├── faq/page.tsx # confirmado sin cambios necesarios
│ ├── verificar-diploma/page.tsx
│ ├── login/page.tsx # redirige por rol: coordinadora/admin -> /panel/pagos, conductor -> /practica, estudiante -> /dashboard (05/09/2026)
│ ├── registro/page.tsx # sin revisar — lenguaje de género pendiente
│ ├── olvide-password/page.tsx
│ ├── restablecer-password/page.tsx
│ ├── verificar-email/page.tsx
│ ├── dashboard/page.tsx # SESIONES = [1,2,3,4] + gate del test psicológico + pantalla de "lista para práctica"/instructor aprobado tras completar teoría (05/09/2026)
│ ├── inscripcion/page.tsx # lee precios de /api/configuracion
│ ├── test-psicologico/page.tsx # NUEVO (05/09/2026) — consentimiento + cuestionario de 54+5 preguntas, una sola vez
│ ├── aula-virtual/[sesion]/page.tsx # PDF con enlace firmado (13/08) + UI de contenido (pdf/enlace/video/texto) unificada y botón "Marcar como visto" centrado para los 4 tipos (28/08/2026)
│ ├── examen/[intentoId]/page.tsx
│ ├── (estudiante)/
│ │ └── diploma/page.tsx # tarjeta compartible: bug de dominio corregido (05/09/2026, ver sección abajo)
│ ├── perfil/cambiar-password/page.tsx
│ ├── (coordinadora)/
│ │ ├── panel/layout.tsx
│ │ ├── panel/page.tsx # MODULOS_ADMIN: 2 tarjetas nuevas, Choferes y Notif. de práctica (05/09/2026)
│ │ ├── panel/pagos/page.tsx
│ │ ├── panel/estudiantes/page.tsx
│ │ ├── panel/aula-virtual/page.tsx # subida de PDF real como archivo (13/08/2026, ver detalle abajo)
│ │ ├── panel/examenes/page.tsx
│ │ ├── panel/diplomas/page.tsx
│ │ ├── panel/test-psicologico/page.tsx # NUEVO (05/09/2026) — lista + detalle de respuestas, sin puntaje calculado
│ │ ├── panel/noticias/page.tsx
│ │ ├── panel/testimonios/page.tsx
│ │ └── panel/faq/page.tsx
│ ├── (admin)/
│ │ ├── admin/layout.tsx
│ │ ├── admin/page.tsx
│ │ ├── admin/contabilidad/page.tsx
│ │ ├── admin/contenido-pagina/page.tsx
│ │ ├── admin/notificaciones/page.tsx
│ │ ├── admin/choferes/page.tsx # NUEVO (05/09/2026) — CRUD de instructores de práctica
│ │ ├── admin/notificaciones-practica/page.tsx # NUEVO (05/09/2026) — destinatarios aparte, solo avisos de práctica
│ │ └── admin/asistente/page.tsx # chatbot con Gemini, solo admin (04/09/2026)
│ ├── (conductor)/
│ │ └── practica/layout.tsx, page.tsx # NUEVO (05/09/2026) — dashboard del chofer: aprobar estudiantes listas para práctica
│ ├── layout.tsx # Metadata SEO completa (title/description/OG/Twitter/JSON-LD Schema.org) — existía desde una sesión sin documentar (comentario interno fecha 13/08/2026), SITE_URL corregido al dominio propio el 28/08/2026
│ └── globals.css
├── lib/
│ └── bancoPreguntasTest.ts # NUEVO (05/09/2026) — 54 preguntas de escala + 5 de reflexión, compartidas entre el formulario del estudiante y la vista de la coordinadora
├── components/
│ ├── ui/Paginacion.tsx
│ ├── layout/Navbar.tsx, Footer.tsx # Navbar: link a "Empresas" agregado (13/08/2026)
│ ├── noticias/NoticiaAcciones.tsx, CompartirBotones.tsx
│ ├── auth/RutaProtegida.tsx # tipo Rol ahora incluye "conductor" (05/09/2026)
│ ├── dashboard/ProgresoCarretera.tsx # la parada "Práctica" ahora refleja practicaAprobada (05/09/2026)
│ └── contabilidad/
├── contexts/AuthContext.tsx # tipo Rol ahora incluye "conductor" (05/09/2026)
├── public/
│ ├── logo-mav-rd.png
│ ├── diploma-compartir.jpg
│ ├── og-image.png
│ ├── libro-maria-diaz.jpg # NUEVO — portada del libro de la fundadora, promocionado en el home (sesión sin documentar, confirmado 28/08/2026)
│ └── inscripcion/
│ ├── teoria-1.jpg, teoria-2.jpg, teoria-3.jpg
│ ├── practica-vip.jpg
│ └── practica-normal-ilustracion.jpg
├── app/favicon.ico
├── tailwind.config.ts
├── .env.local.example
└── package.json

## Tokens de color (Tailwind) — 2 tokens nuevos sin confirmar (28/08/2026)

```js
colors: {
  brand: {
    blue: '#1B3A6B',
    blueLight: '#4A7FC9',
    pink: '#D6336C',
    pinkLight: '#FBE4EC',
    // NUEVOS, en uso real en app/page.tsx (bg-brand-yellow,
    // border-brand-mamey) pero sin confirmar su valor hex exacto —
    // pendiente revisar tailwind.config.ts directamente. "Mamey" es el
    // naranja que se usa en señales de tránsito de precaución.
    yellow: '#PENDIENTE_CONFIRMAR',
    mamey: '#PENDIENTE_CONFIRMAR',
  },
  neutral: { bg: '#F7F8FA', text: '#1F2937' },
  status: { success: '#2F9E44', warning: '#F0A500' },
}
```

Tipografía: Poppins (títulos), Inter (cuerpo).

⚠️ Nota técnica encontrada esta sesión, no corregida a propósito (bajo
riesgo, no bloqueante): el código usa dos convenciones distintas para
las mismas variantes claras de marca según el archivo — `bg-brand-blue-light`/
`bg-brand-pink-light` (con guión) en `app/page.tsx`, vs.
`text-brand-blueLight`/`bg-brand-pinkLight` (camelCase) en las páginas
del panel de coordinadora y aula virtual. Ambas funcionan hoy en
producción (Tailwind v4 debe estar generando ambos alias), pero es
inconsistente. Unificar en algún momento, sin urgencia.

## Autenticación — 4to rol agregado (05/09/2026)

`AuthContext.tsx` y `RutaProtegida.tsx`: el tipo `Rol` pasó de 3 a 4
valores (`"estudiante" | "coordinadora" | "admin" | "conductor"`).
`login/page.tsx` redirige a `/practica` si `rol === "conductor"`, antes
de los casos ya existentes de coordinadora/admin y estudiante.

## Sesiones — 4, ya recreadas en la base de datos (13/08/2026)

`dashboard/page.tsx`: `const SESIONES = [1, 2, 3, 4]`. La nota de la
versión anterior de este documento ("el dashboard va a marcar Sesión 1
disponible pero al entrar muestra 'Sesión no encontrada'") **ya no
aplica** — `scripts/crearSesionesIniciales.js` se ejecutó y las 4
sesiones existen en Atlas (ver DATABASE.md).

El panel de coordinadora (`panel/aula-virtual/page.tsx`,
`panel/examenes/page.tsx`) ya puede listar y usar esas 4 sesiones con
normalidad. Sigue sin haber forma de **crear** sesiones nuevas desde el
panel (ver ARQUITECTURA_BACKEND.md) ni de **renombrarlas** — eso sigue
pendiente de un formulario dedicado.

## Barra de progreso ilustrada — extendida a la etapa de práctica (05/09/2026)

`components/dashboard/ProgresoCarretera.tsx` — la parada "Práctica" ya
existía visualmente, pero antes siempre se quedaba neutra ("no se
rastrea en la app"). Ahora recibe `progreso.practicaAprobada` y, cuando
es `true`, cambia de color y muestra el mismo check verde que un libro
de sesión aprobada. El mensaje motivacional distingue "contacta a tu
instructor para la práctica en carretera" (teoría completa, práctica
sin aprobar) de "tu instructor aprobó tu práctica — tu diploma está en
camino" (ambas cosas completas, diploma aún no generado).

## Diploma compartible en redes sociales — bug de dominio corregido (05/09/2026)

`app/(estudiante)/diploma/page.tsx` generaba la tarjeta con Canvas
(formato historia 9:16) usando `URL_INICIO` para el QR, pero el texto
visible debajo del QR estaba escrito **literal y aparte**
(`"muvo-rd.vercel.app"`) — al migrar al dominio propio (ver
"Infraestructura y despliegue"), el QR se corrigió pero el texto se
quedó apuntando al dominio viejo. Corregido: `URL_INICIO` ahora es
`https://www.muvordvial.com`, y el texto del `fillText` sale de esa
misma constante (`DOMINIO_VISIBLE = URL_INICIO.replace(/^https?:\/\//, "")`)
en vez de estar escrito aparte — QR y texto ya no pueden
desincronizarse otra vez.

## Contenido de estudio en PDF — subida real de archivo (13/08/2026)

`panel/aula-virtual/page.tsx` — cuando el tipo de material es "pdf", el
formulario ya no pide pegar una URL a mano: hay un selector de archivo
(`subirPDFContenido`, mismo patrón que ya existía para
`subirImagenContenido`) que sube a `POST /api/uploads/pdf` y guarda tanto
`url` como `publicIdCloudinary` en el formulario.

`aula-virtual/[sesion]/page.tsx` — el enlace "Abrir PDF ↗" apunta al
endpoint firmado del backend (`/contenido-sesion/:id/archivo?token=...`)
cuando el material tiene `publicIdCloudinary`; si no lo tiene (contenido
viejo con URL pegada a mano), usa `url` directo como antes. Ver
ARQUITECTURA_BACKEND.md para el porqué de la URL firmada (Cloudinary
bloquea la entrega pública de recursos `raw` sin firmar).

**Actualización 28/08/2026 — contenido cargado, pero hay que rehacerlo:**
en una sesión sin documentar sí se cargó contenido real (títulos reales
por material, ej. "1.1 Bienvenida a Muvo RD Vial"), pero los PDFs
subidos tienen errores de codificación (texto con símbolos extraños) y
no son presentables. El flujo técnico en sí funciona bien — es la
calidad de los archivos lo que hay que corregir. Ver ARQUITECTURA_BACKEND.md
y DATABASE.md para el plan de borrar vía `curl` y recargar.

Pendiente, sin empezar: texto enriquecido con imágenes incrustadas
dentro de `contenidoTexto` (hoy es un string plano de HTML/Markdown sin
editor visual) — se identificó como pedido aparte, más grande, no
incluido en este bloque de trabajo.

## NUEVO: UI de contenido en aula virtual unificada entre los 4 tipos (28/08/2026)

Bug real detectado por el usuario en `aula-virtual/[sesion]/page.tsx`:
para contenido tipo `video` y `texto`, el botón "Marcar como visto"
caía en su propia línea de forma natural (el contenido vivía dentro de
un `<div>` de bloque). Para `pdf` y `enlace`, el link era un `
className="inline-block">` — al ser `inline-block`, el navegador lo
ponía en la misma línea que el botón siguiente si había espacio,
quedando visualmente pegados y desordenados.

Fix aplicado: los 4 tipos de contenido ahora tienen tratamiento visual
consistente.

- `pdf` y `enlace` pasaron de ser un link de texto suelto a una tarjeta
  de bloque completo (borde + fondo `bg-neutral-bg`, `flex
items-center justify-center`, mismo peso visual que el recuadro de
  video).
- El botón "Marcar como visto" ahora vive dentro de un `<div
className="flex justify-center">` para los 4 tipos por igual, en vez
  de quedar alineado a la izquierda solo en video/texto.

No se tocó la lógica de `marcarVisto()`, `intentarDesbloquear`, ni
ningún endpoint — cambio 100% visual, confirmado probado en producción
con los 4 tipos de contenido.

## NUEVO: SEO real — sitemap, robots, metadata y Search Console con el dominio propio (28/08/2026)

Trabajo de SEO que en parte ya existía de una sesión sin documentar
(comentarios internos fechados 13/08/2026 y 16/08/2026 en el código),
descubierto y corregido en esta sesión al migrar al dominio propio.

**`app/sitemap.ts`** (Next.js App Router, genera `/sitemap.xml`
dinámicamente): listaba 10 páginas públicas con prioridad y frecuencia
de cambio — `SITE_URL` estaba quemado al dominio viejo
(`muvo-rd.vercel.app`), corregido a `https://www.muvordvial.com`.

**`app/robots.ts`** (genera `/robots.txt`): bloquea rutas privadas
(`/dashboard`, `/panel`, `/admin`, `/aula-virtual`, `/examen`,
`/perfil`) y apunta al sitemap. Tenía dos problemas: mismo `SITE_URL`
del dominio viejo, y **bloqueaba `/inscripcion` por error** — es una
página pública de marketing, no debía estar en el `disallow` (el
usuario confirmó que fue sin querer). Ambos corregidos.

**`app/layout.tsx`** (metadata global): ya tenía, de la sesión sin
documentar, una configuración de SEO bastante completa — título con
template, descripción orientada a búsqueda real ("escuela de manejo
Santo Domingo", "licencia de conducir INTRANT", etc.), array de
`keywords` (sin efecto real en ranking desde que Google lo dejó de usar
en 2009, pero inofensivo dejarlo), Open Graph, Twitter Card, y un bloque
JSON-LD de Schema.org (`EducationalOrganization`) con dirección,
fundadora y fecha de fundación. El único problema real era, otra vez,
`SITE_URL` quemado al dominio viejo — corregido.

**Google Search Console:** la propiedad vieja (`https://muvo-rd.vercel...`,
tipo "Prefijo de URL") seguía activa con 9 páginas indexadas y un
sitemap funcionando desde mayo/2026 — nada de esto se perdió, solo
quedó huérfana del dominio nuevo. Se creó una propiedad nueva tipo
**"Dominio"** para `muvordvial.com` (cubre `www`/sin `www`/http/https a
la vez), verificada por DNS (registro TXT agregado en Vercel → Domains
→ DNS Records, mismo lugar que los registros de Resend). Sitemap
reenviado y confirmado ("Correcto", 10 páginas descubiertas). Se
solicitó indexación manual de home, `/empresas` y `/registro` para
acelerar el rastreo en vez de esperar el orgánico.

**Nota de negocio (28/08/2026):** se evaluó agregar "escuela de
choferes" como término de búsqueda objetivo, pero se descartó — en RD
ese término sugiere formación de conductores profesionales, no coincide
con lo que Muvo ofrece (curso para principiantes) y podría atraer
tráfico que rebota rápido. Se mantuvo el lenguaje ya usado en el sitio
("aprender a manejar", "escuela de manejo", "examen del INTRANT").

## Home — sección de Planes y Precios (rediseñada 07/09/2026)

Reemplaza la versión del 13/08/2026 (2 tarjetas, precio leído de
`GET /api/configuracion`). Ahora:

- `app/page.tsx` sigue siendo componente servidor async, pero hace
  `fetch` a `GET /api/planes` (no `/configuracion`) con `cache: "no-store"`.
- Renderiza **3 tarjetas** (Fundación/Estándar/VIP), generadas
  dinámicamente con `.map()` sobre la respuesta — agregar o quitar un
  plan, o cambiar su nombre/precio/frase, no requiere tocar este
  archivo, solo editar el dato en `/admin/planes`.
- El Home ahora es **solo resumen**: nombre, precio y una frase
  destacada corta por plan (`Plan.fraseDestacada`), con un botón "Ver
  detalles del plan" hacia `/inscripcion` (antes el CTA iba directo a
  `/registro` y el Home mostraba la lista completa de características —
  esa lista ahora vive solo en `/inscripcion`, decisión explícita del
  usuario para no duplicar el detalle en dos páginas).
- VIP se marca visualmente como destacado (`plan.codigo === "vip"`) en
  vez de un campo `destacado` hardcodeado como antes.

`/inscripcion` (`app/inscripcion/page.tsx`) también se actualizó: el
`fetch` inicial pasó de `/configuracion` a `/planes`, el tipo `Precios`
se reemplazó por un tipo `Plan` completo, las 2 tarjetas de comparación
hardcodeadas pasaron a 3 tarjetas generadas con `.map()` (mostrando
duración/cantidad de sesiones de práctica y costo de combustible por
plan, además de `caracteristicas`), y el `<select>` del formulario de
inscripción ahora itera sobre los planes recibidos en vez de tener 2
`<option>` fijas.

## NUEVO: Panel de admin — Planes y precios (07/09/2026)

`app/(admin)/admin/planes/page.tsx` — pantalla protegida con
`RutaProtegida rolesPermitidos={["admin"]}`, accesible desde una tarjeta
nueva en `panel/page.tsx` (grupo "Solo fundadora", ícono `DollarSign`).
Carga los 3 planes vía `GET /api/planes/admin/todos` (incluye inactivos,
a diferencia del endpoint público) y muestra un formulario editable por
plan: nombre, precio, frase destacada, modalidad de práctica, duración y
cantidad de sesiones, costo de combustible, características (textarea,
una por línea) y un checkbox de "Visible en el sitio" (`activo`). Cada
plan se guarda de forma independiente vía
`PATCH /api/planes/:codigo`. Resuelve el pendiente de tener que editar
precios a mano en Atlas o vía Postman.

Inspirado en el patrón visual ya existente de
`(conductor)/practica/page.tsx` (tarjetas blancas, mensajes de
éxito/error inline) — no se introdujo ningún componente ni librería
nueva.

**Nota histórica preservada:** el banner hacia `/empresas` entre
Testimonios y el CTA final del Home (agregado 13/08/2026) sigue igual,
sin cambios en este rediseño.

## NUEVO: Home — promoción del libro de la fundadora + colores nuevos (sesión sin documentar, confirmado 28/08/2026)

`app/page.tsx` gana una sección entre "Planes y Precios" y
"Testimonios" promocionando el libro de María Díaz ("Cómo protegerte de
un conductor temerario"), con portada (`public/libro-maria-diaz.jpg`),
cita del libro, y dos botones reales hacia Amazon (versión física y
Kindle). Usa dos clases de color nuevas no documentadas antes en los
tokens de Tailwind: `bg-brand-yellow` y `border-brand-mamey` — ver nota
en "Tokens de color" arriba, valores hex pendientes de confirmar
directo en `tailwind.config.ts`.

## NUEVO: Página `/empresas` (13/08/2026)

Página pública nueva, sección "programa empresarial" — primera versión
deliberadamente simple: contenido informativo + formulario de contacto,
**sin** modelo de precios escalonado ni inscripción grupal (decisión
explícita del usuario, para validar demanda antes de construir esa
lógica).

Secciones: hero, "¿Por qué capacitar a tu equipo con nosotros?" (4
beneficios con ícono, estilo tomado de academiavial.com pero con copy
propio), "Cómo funciona" (3 pasos), y un formulario (`nombreEmpresa`,
`contacto`, `cargo` opcional, `telefono`, `email`,
`cantidadEstudiantes`, `mensaje` opcional) que hace
`POST /api/empresas/contacto`. El precio por persona **no se muestra en
el sitio** — se comunica que se cotiza según el tamaño del grupo, se
resuelve por correo.

`components/layout/Navbar.tsx` — se agregó `{ href: "/empresas", label:
"Empresas" }` al array `enlaces` (alimenta tanto el menú de escritorio
como el móvil desde un solo lugar).

## NUEVO: Panel de admin — Asistente (chatbot) (04/09/2026)

`app/(admin)/admin/asistente/page.tsx` — pantalla de chat simple para
que la fundadora pregunte por cifras reales de la app (inscripciones,
pagos, estudiantes, balance, solicitudes de Empresas, resultados de
examen). Ver ARQUITECTURA_BACKEND.md para el detalle completo del
backend (Gemini 3.6 Flash + function calling + 7 herramientas de solo
lectura).

- Burbujas de mensaje (azul a la derecha para la fundadora, gris a la
  izquierda para el asistente), auto-scroll al último mensaje.
- 4 preguntas de ejemplo como botones, visibles solo antes del primer
  mensaje — para que no tenga que pensar qué escribir la primera vez.
- Estados de error (fallo de red o del backend) se muestran en el
  mismo color rosa de error que ya usa el resto del panel (mismo
  patrón que `admin/contabilidad/page.tsx`).
- Llama a `POST /api/chatbot/preguntar` con el token de `useAuth()`,
  mismo patrón de autenticación que el resto del panel.
- El texto de la interfaz se mantuvo deliberadamente breve — se quitó
  una primera versión que repetía "nunca inventa datos" dos veces
  (sonaba más a advertencia que a descripción de producto). La
  garantía real de que no invente cifras vive en la instrucción de
  sistema del backend, no hace falta repetirla en la UI.
- Acceso: tarjeta nueva "Asistente" (ícono `Bot` de lucide-react) en
  `panel/page.tsx`, grupo "Solo fundadora" — mismo array `MODULOS_ADMIN`
  donde ya estaban Contabilidad, Contenido de página y Notificaciones.

## NUEVO: Test psicológico de perfil conductual (05/09/2026)

Pedido directo de la fundadora: entre el pago confirmado y el acceso al
contenido, un cuestionario de perfil psicológico/conductual — basado en
un instrumento en papel que ya usaba. Ver ARQUITECTURA_BACKEND.md para
el detalle completo de la decisión de alcance (solo se digitalizaron
las secciones que llena el estudiante, no las que requieren un
evaluador humano; tampoco se calcula ningún puntaje).

- **`app/test-psicologico/page.tsx`** — tres estados: (1) si ya lo
  completó, mensaje de agradecimiento y botón a "Ir a mi panel", sin
  poder volver a editarlo; (2) si no, pantalla de consentimiento con el
  mismo disclaimer del documento original y un checkbox obligatorio;
  (3) el formulario en sí — 7 secciones (A-G) con las 54 preguntas de
  escala (botones Nunca/Casi nunca/A veces/Casi siempre/Siempre) más la
  sección H con 5 preguntas de reflexión abierta (opcionales). Envío
  único — el backend rechaza un segundo intento con 409.
- **`app/dashboard/page.tsx`** — ahora también consulta
  `GET /test-psicologico/mi-respuesta` en paralelo con el progreso.
  Si el pago está confirmado pero el test no, se muestra una pantalla
  de aviso ("Antes de empezar, completa tu cuestionario de perfil") en
  vez de las tarjetas de sesión — la lógica real de bloqueo vive en el
  backend (`sesionController.js`), esto es solo para que la estudiante
  no llegue a un error 403 confuso al hacer clic en una sesión.
- **`app/(coordinadora)/panel/test-psicologico/page.tsx`** — lista de
  quiénes lo han completado (nombre, cédula, fecha), con acceso de
  coordinadora y admin por igual. Cada fila se expande para mostrar las
  54 respuestas (pregunta + etiqueta de la escala elegida, no el número
  crudo) organizadas por sección, más las 5 reflexiones. **Sin ningún
  promedio ni indicador calculado** — deliberado, ver ARQUITECTURA_BACKEND.md.
- **`lib/bancoPreguntasTest.ts`** — única fuente de verdad para el
  texto de las 54+5 preguntas, usado tanto por el formulario del
  estudiante como por la vista de detalle de la coordinadora. El índice
  de cada pregunta en este archivo es exactamente el índice esperado en
  el array `respuestas` que guarda el backend — si se edita el texto de
  una pregunta, no pasa nada; si se reordena o se agrega/quita una
  pregunta, hay que migrar los datos ya guardados o el índice deja de
  coincidir.
- Tarjeta de acceso nueva en `panel/page.tsx`, grupo "Gestión del
  curso" (no "Solo fundadora" — la coordinadora también tiene acceso).

## NUEVO: Seguimiento de práctica de manejo (05/09/2026)

Ver ARQUITECTURA_BACKEND.md para el detalle completo del backend
(Instructor, DestinatarioPractica, gate del diploma). Aquí, lo nuevo en
el frontend:

- **`app/dashboard/page.tsx`** — cuando `progreso.cursoCompletado` es
  `true`, ya no se muestran las tarjetas de sesión. En su lugar: si
  `!practicaAprobada`, una pantalla de felicitación con la lista de
  choferes activos (nombre, teléfono, correo, horarios, vía
  `GET /instructores/activos`) para que la estudiante los contacte
  directamente — sin asignación automática. Si `practicaAprobada` pero
  el diploma aún no existe, un mensaje de "tu diploma está en camino".
  Si el diploma ya existe, el botón "Ver mi diploma" de siempre.
- **`components/dashboard/ProgresoCarretera.tsx`** — la parada "Práctica"
  ahora cambia de color y muestra el check cuando `practicaAprobada` es
  `true`, igual que un libro más de sesión aprobada. El mensaje
  motivacional distingue "contacta a tu instructor" de "tu instructor
  ya te aprobó, tu diploma está en camino".
- **`app/(admin)/admin/choferes/page.tsx`** (NUEVO) — CRUD de choferes:
  formulario de creación (llama a `POST /usuarios/conductor`, crea el
  `User` y el `Instructor` en un paso) y edición de horarios/activo por
  chofer existente (`PATCH /instructores/:id`).
- **`app/(admin)/admin/notificaciones-practica/page.tsx`** (NUEVO) —
  copia exacta del patrón de `admin/notificaciones/page.tsx`, apuntando
  a `/destinatarios-practica` — lista separada de quién recibe avisos
  cuando una estudiante termina la teoría.
- **`app/(conductor)/practica/layout.tsx` + `page.tsx`** (NUEVO) —
  dashboard del chofer: lista de estudiantes con `cursoCompletado` pero
  sin `practicaAprobada`, con botón "Aprobar práctica" por cada una.
- **`app/(coordinadora)/panel/page.tsx`** — dos tarjetas nuevas en
  `MODULOS_ADMIN` ("Solo fundadora"): "Choferes" (ícono `Car`) y
  "Notif. de práctica" (ícono `Send`).

Probado de punta a punta en esta sesión: crear chofer → login como
chofer → estudiante termina teoría → notificación + lista de choferes
visible → chofer aprueba → diploma generable.

## Testing antes de cada commit importante — sin cambios

## Pendiente real (frontend)

- Agregar la sección "Lo que aprendiste" (temas reales) a la imagen del
  diploma compartible — ya se puede hacer, los 4 temas reales existen
  (ver `ContenidoSesion` en ARQUITECTURA_BACKEND.md), aunque están
  pendientes de recrearse por el problema de PDFs corruptos.
- Confirmar y corregir el alcance real del grupo de ruta `(estudiante)`.
- Revisar lenguaje de género en `testimonios/page.tsx` y
  `registro/page.tsx`.
- Reemplazar las fotos de `public/inscripcion/` cuando haya material
  nuevo que refleje la audiencia ampliada.
- Construir un formulario en `panel/aula-virtual/page.tsx` para
  renombrar sesiones desde el panel (sigue sin existir; hoy es solo vía
  `PATCH /sesiones/:numero` a mano).
- **ALTA PRIORIDAD (28/08/2026):** recargar `ContenidoSesion` (PDFs) y
  `Examen` desde cero — el contenido actual tiene errores de
  codificación y un bug de examen (respuesta siempre en A). Ver
  ARQUITECTURA_BACKEND.md y DATABASE.md para el detalle y el plan
  (borrar vía `curl`, recrear).
- Texto enriquecido con imágenes incrustadas en `contenidoTexto` — pedido
  identificado, no empezado.
- Confirmar los valores hex reales de `brand-yellow` y `brand-mamey` en
  `tailwind.config.ts` y actualizar la tabla de tokens de color de este
  documento (hoy están marcados como pendiente de confirmar).
- **NUEVO:** unificar la convención de nombres de color de Tailwind
  (`brand-blue-light` vs `brand-blueLight`) — inconsistente entre
  archivos, sin urgencia.
- **NUEVO:** si se decide agregar animaciones más elaboradas al home
  (inspirado en academiavial.com), evaluar `framer-motion` — no se
  agregó en este bloque de trabajo, quedó fuera de alcance.
- "Me gusta" en comentarios individuales de noticias.
- **NUEVO:** si en el futuro se decide automatizar la asignación de
  instructor (hoy la estudiante contacta directo, sin asignación),
  revisar `PantallaListaParaPractica` en `dashboard/page.tsx` — hoy
  asume que siempre se muestra la lista completa de choferes activos.
- **NUEVO (07/09/2026), ALTA PRIORIDAD PARA LA PRÓXIMA SESIÓN:** todo el
  frontend de Escolar/Empresarial/Motorista — formularios de creación de
  grupo y roster, cuestionario informativo de Escolar,
  dashboard/aula virtual condicionados por `programa` (hoy asumen un
  solo currículo). Diseño completo en
  `ESPECIFICACION_PROGRAMAS_NUEVOS.md`, nada de esto empezado en código
  todavía.
