# Arquitectura del Frontend — mav-rd-frontend

> Refleja el estado REAL del código al 04/09/2026. Reemplaza la versión
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

## Estructura real de planes (aclarado 13/08/2026)

Un solo curso teórico (igual para todas), con dos variantes de práctica
de manejo: **Normal** y **VIP**. La diferencia entre ambos es
exclusivamente cuánta atención personalizada y tiempo con el instructor
se recibe en la práctica — no hay diferencia en el contenido teórico.
Este matiz ahora se refleja tanto en `/inscripcion` (ya existía) como en
la nueva sección de precios del home (ver abajo).

## Estructura de carpetas (real)

```
mav-rd-frontend/
├── app/
│   ├── page.tsx                          # Inicio — Planes/precios (13/08) + banner Empresas (13/08) + promo libro de la fundadora, colores brand-yellow/brand-mamey nuevos (sesión sin documentar, confirmado 28/08/2026)
│   ├── sitemap.ts                        # NUEVO (documentado 28/08/2026, existía desde antes sin registrar) — SITE_URL corregido al dominio propio
│   ├── robots.ts                         # NUEVO (documentado 28/08/2026, existía desde antes sin registrar) — SITE_URL corregido, ya no bloquea /inscripcion por error
│   ├── empresas/page.tsx                 # NUEVO (13/08/2026) — programa empresarial, informativo + formulario
│   ├── acerca-de-nosotros/page.tsx
│   ├── kit-preparacion/page.tsx
│   ├── noticias/page.tsx
│   ├── noticias/[id]/page.tsx
│   ├── testimonios/page.tsx              # sin revisar — lenguaje de género pendiente
│   ├── faq/page.tsx                      # confirmado sin cambios necesarios
│   ├── verificar-diploma/page.tsx
│   ├── login/page.tsx
│   ├── registro/page.tsx                 # sin revisar — lenguaje de género pendiente
│   ├── olvide-password/page.tsx
│   ├── restablecer-password/page.tsx
│   ├── verificar-email/page.tsx
│   ├── dashboard/page.tsx                # SESIONES = [1,2,3,4]
│   ├── inscripcion/page.tsx              # lee precios de /api/configuracion
│   ├── aula-virtual/[sesion]/page.tsx    # PDF con enlace firmado (13/08) + UI de contenido (pdf/enlace/video/texto) unificada y botón "Marcar como visto" centrado para los 4 tipos (28/08/2026)
│   ├── examen/[intentoId]/page.tsx
│   ├── (estudiante)/
│   │   └── diploma/page.tsx
│   ├── perfil/cambiar-password/page.tsx
│   ├── (coordinadora)/
│   │   ├── panel/layout.tsx
│   │   ├── panel/page.tsx
│   │   ├── panel/pagos/page.tsx
│   │   ├── panel/estudiantes/page.tsx
│   │   ├── panel/aula-virtual/page.tsx   # subida de PDF real como archivo (13/08/2026, ver detalle abajo)
│   │   ├── panel/examenes/page.tsx
│   │   ├── panel/diplomas/page.tsx
│   │   ├── panel/noticias/page.tsx
│   │   ├── panel/testimonios/page.tsx
│   │   └── panel/faq/page.tsx
│   ├── (admin)/
│   │   ├── admin/layout.tsx
│   │   ├── admin/page.tsx
│   │   ├── admin/contabilidad/page.tsx
│   │   ├── admin/contenido-pagina/page.tsx
│   │   ├── admin/notificaciones/page.tsx
│   │   └── admin/asistente/page.tsx      # NUEVO (04/09/2026) — chatbot con Gemini, solo admin
│   ├── layout.tsx                        # Metadata SEO completa (title/description/OG/Twitter/JSON-LD Schema.org) — existía desde una sesión sin documentar (comentario interno fecha 13/08/2026), SITE_URL corregido al dominio propio el 28/08/2026
│   └── globals.css
├── components/
│   ├── ui/Paginacion.tsx
│   ├── layout/Navbar.tsx, Footer.tsx     # Navbar: link a "Empresas" agregado (13/08/2026)
│   ├── noticias/NoticiaAcciones.tsx, CompartirBotones.tsx
│   ├── auth/RutaProtegida.tsx
│   ├── dashboard/ProgresoCarretera.tsx
│   └── contabilidad/
├── contexts/AuthContext.tsx
├── public/
│   ├── logo-mav-rd.png
│   ├── diploma-compartir.jpg
│   ├── og-image.png
│   ├── libro-maria-diaz.jpg               # NUEVO — portada del libro de la fundadora, promocionado en el home (sesión sin documentar, confirmado 28/08/2026)
│   └── inscripcion/
│       ├── teoria-1.jpg, teoria-2.jpg, teoria-3.jpg
│       ├── practica-vip.jpg
│       └── practica-normal-ilustracion.jpg
├── app/favicon.ico
├── tailwind.config.ts
├── .env.local.example
└── package.json
```

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

## Autenticación — sin cambios

## Sesiones — 4, ya recreadas en la base de datos (13/08/2026)

`dashboard/page.tsx`: `const SESIONES = [1, 2, 3, 4]`, sin cambios de
código esta sesión. La nota de la versión anterior de este documento
("el dashboard va a marcar Sesión 1 disponible pero al entrar muestra
'Sesión no encontrada'") **ya no aplica** — `scripts/crearSesionesIniciales.js`
se ejecutó y las 4 sesiones existen en Atlas (ver DATABASE.md).

El panel de coordinadora (`panel/aula-virtual/page.tsx`,
`panel/examenes/page.tsx`) ya puede listar y usar esas 4 sesiones con
normalidad. Sigue sin haber forma de **crear** sesiones nuevas desde el
panel (ver ARQUITECTURA_BACKEND.md) ni de **renombrarlas** — eso sigue
pendiente de un formulario dedicado.

## Barra de progreso ilustrada — sin cambios en esta sesión

## Diploma compartible en redes sociales — sin cambios en esta sesión

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
un `<div>` de bloque). Para `pdf` y `enlace`, el link era un `<a
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

## NUEVO: Home — sección de Planes y Precios (13/08/2026)

`app/page.tsx` pasó de ser un componente cliente-friendly estático a un
**componente servidor async** — hace `fetch` a `GET /api/configuracion`
con `cache: "no-store"` (el precio puede cambiar sin un nuevo deploy) y
renderiza dos tarjetas, Normal y VIP, con:

- Precio real (`precio_plan_normal`/`precio_plan_vip`), con "Consultar"
  como fallback si la llamada a `/configuracion` falla.
- Copy de venta explícito (no solo descriptivo) — iterado varias veces en
  esta sesión con el usuario hasta llegar a una versión más comercial:
  "Sal manejando con confianza. Tú eliges cómo llegar ahí" + un párrafo
  que vende teoría + instructores + atención personalizada en la
  práctica.
- Lista de características por plan y CTA "Empezar con este plan" hacia
  `/registro`.

También se agregó un banner (`bg-brand-blue`, sección completa) hacia
`/empresas` entre Testimonios y el CTA final.

Inspirado en el análisis de la competencia (academiavial.com) — se tomó
la idea de mostrar precios desde el inicio, pero **no** se replicó su
estructura de múltiples "programas": Muvo vende un solo curso con dos
variantes de práctica, no un catálogo.

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
