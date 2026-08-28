# Arquitectura del Frontend — mav-rd-frontend

> Refleja el estado REAL del código al 27/08/2026. Reemplaza la versión
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
│   ├── page.tsx                          # Inicio — sección de Planes/precios + banner a Empresas (13/08/2026)
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
│   ├── aula-virtual/[sesion]/page.tsx    # PDF ahora usa enlace firmado cuando aplica (13/08/2026)
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
│   │   └── admin/notificaciones/page.tsx
│   ├── layout.tsx
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
│   └── inscripcion/
│       ├── teoria-1.jpg, teoria-2.jpg, teoria-3.jpg
│       ├── practica-vip.jpg
│       └── practica-normal-ilustracion.jpg
├── app/favicon.ico
├── tailwind.config.ts
├── .env.local.example
└── package.json
```

## Tokens de color (Tailwind) — sin cambios

```js
colors: {
  brand: {
    blue: '#1B3A6B',
    blueLight: '#4A7FC9',
    pink: '#D6336C',
    pinkLight: '#FBE4EC',
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

## NUEVO: Contenido de estudio en PDF — subida real de archivo (13/08/2026)

`panel/aula-virtual/page.tsx` — cuando el tipo de material es "pdf", el
formulario ya no pide pegar una URL a mano: hay un selector de archivo
(`subirPDFContenido`, mismo patrón que ya existía para
`subirImagenContenido`) que sube a `POST /api/uploads/pdf` y guarda tanto
`url` como `publicIdCloudinary` en el formulario.

`aula-virtual/[sesion]/page.tsx` — el enlace "Abrir PDF ↗" ahora apunta
al endpoint firmado del backend
(`/contenido-sesion/:id/archivo?token=...`) cuando el material tiene
`publicIdCloudinary`; si no lo tiene (contenido viejo con URL pegada a
mano), usa `url` directo como antes. Ver ARQUITECTURA_BACKEND.md para el
porqué de la URL firmada (Cloudinary bloquea la entrega pública de
recursos `raw` sin firmar).

**Todavía no se cargó contenido real** — solo se confirmó que el flujo
completo (subir → guardar → abrir) funciona en producción.

Pendiente, sin empezar: texto enriquecido con imágenes incrustadas
dentro de `contenidoTexto` (hoy es un string plano de HTML/Markdown sin
editor visual) — se identificó como pedido aparte, más grande, no
incluido en este bloque de trabajo.

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

## Testing antes de cada commit importante — sin cambios

## Pendiente real (frontend)

- Agregar la sección "Lo que aprendiste" (temas reales) a la imagen del
  diploma compartible, cuando estén definidos los 4 temas.
- Confirmar y corregir el alcance real del grupo de ruta `(estudiante)`.
- Revisar lenguaje de género en `testimonios/page.tsx` y
  `registro/page.tsx`.
- Reemplazar las fotos de `public/inscripcion/` cuando haya material
  nuevo que refleje la audiencia ampliada.
- Construir un formulario en `panel/aula-virtual/page.tsx` para
  renombrar sesiones desde el panel.
- Cargar contenido real (PDFs, videos, texto) en las 4 sesiones —
  el flujo ya está listo, falta el contenido.
- Texto enriquecido con imágenes incrustadas en `contenidoTexto` — pedido
  identificado, no empezado.
- **NUEVO:** unificar la convención de nombres de color de Tailwind
  (`brand-blue-light` vs `brand-blueLight`) — inconsistente entre
  archivos, sin urgencia.
- **NUEVO:** si se decide agregar animaciones más elaboradas al home
  (inspirado en academiavial.com), evaluar `framer-motion` — no se
  agregó en este bloque de trabajo, quedó fuera de alcance.
- "Me gusta" en comentarios individuales de noticias.
- **NUEVO (27/08/2026): Re-confirmar registro en Google Search Console
  con el dominio nuevo.** En algún momento antes de esta sesión se había
  empezado a registrar el sitio en Google (verificar propiedad en
  Search Console + enviar un `sitemap.xml`), pero **con el dominio
  viejo** (`muvo-rd.vercel.app` o `muvordvial.com` antes del cambio de
  DNS) — nunca quedó confirmado ni documentado, y ahora que la URL de
  producción es `www.muvordvial.com` hay que asumir que ese registro
  quedó huérfano o inválido. No hay certeza de qué método de
  verificación se usó (DNS/TXT vs. archivo/meta tag) ni si el sitemap
  se generó desde código o se subió a mano — **antes de rehacer nada,
  la próxima sesión debe auditar el estado real:**
  1. Entrar a Google Search Console (search.google.com/search-console)
     y revisar qué propiedades existen hoy — es probable que solo
     aparezca `muvo-rd.vercel.app` o el dominio sin `www`.
  2. Revisar el repo del frontend por un archivo `sitemap.xml`, una ruta
     `app/sitemap.ts`/`sitemap.xml/route.ts` (convención de Next.js App
     Router), o un `next-sitemap.config.js` — hoy no hay nada de esto
     documentado en la estructura de carpetas de este archivo, así que
     puede que el sitemap se haya generado/subido manualmente y no viva
     en código.
  3. Con eso claro, agregar `www.muvordvial.com` como propiedad nueva en
     Search Console (verificación recomendada por DNS ya que el dominio
     vive en Vercel — mismo patrón que se usó para verificar el dominio
     en Resend), generar/confirmar el sitemap real, enviarlo, y pedir
     indexación de las páginas públicas principales (home, `/empresas`,
     `/inscripcion`, etc.).
