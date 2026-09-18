# Historial de modificaciones — Muvo RD Vial

> Registro breve por sesión. El estado actual y detallado del sistema vive en
> ARQUITECTURA_BACKEND.md, ARQUITECTURA_FRONTEND.md y DATABASE.md; este
> archivo es un changelog de continuidad, no la fuente de verdad de cómo
> funciona nada.
>
> **Reorganizado el 13/09/2026 (cuarta sesión):** este archivo había crecido
> a ~1800 líneas con el detalle línea-por-línea de cada sesión desde julio.
> Se comprimió a lo que de verdad ayuda a la siguiente sesión a saber qué
> pasó y qué falta — el detalle técnico completo de "cómo funciona" sigue
> viviendo en los tres documentos de arquitectura, no aquí. Nada de código
> cambió en esta limpieza, solo documentación.

## 17/09/2026 (sesión paralela) — Cierre de 4 incidencias reportadas, bug nuevo en resumen diario, rediseño del Home y Footer

Sesión de trabajo en paralelo a la del sistema de reportes/soporte
(arriba). Punto de partida: reporte de incidencias de Ramon tras la
última ronda de pruebas.

- **4 de las 6 incidencias reportadas ya tenían el fix en el código**
  (fechado 17/09, sin documentar ni confirmado en producción todavía):
  precio quitado de las tarjetas del Home, botón "Ver detalles del
  plan" ya enlazando bien, banner escolar agregado, y el fix de
  `AuthContext.tsx` para el parpadeo de login. Confirmado por Ramon que
  el código ya estaba desplegado; quedaba pendiente solo el bug de
  "Sesión no encontrada" (ver abajo) y probar todo en vivo — **ambos
  confirmados funcionando en producción esta sesión.**
- **Bug corregido: "Sesión no encontrada"** para estudiantes de
  `estandar` (colegio y standard). Causa: `programaContenido` se
  agregó al esquema de `Sesion` el 11/09 con default, pero los 4
  documentos originales (de antes de esa fecha) nunca tuvieron el
  campo escrito de verdad — un default de Mongoose no reescribe
  documentos existentes. `scripts/corregirProgramaContenidoSesion.js`
  (ya existía, dry-run) corrido en producción con `--confirmar`. Ver
  ARQUITECTURA_BACKEND.md y DATABASE.md sección 4 para el detalle.
- **Bug nuevo encontrado en `utils/resumenDiario.js` — distinto del ya
  corregido el 16/09.** El fix del 16/09 arregló cómo se calculaba
  "hoy"; este es sobre qué fecha filtra cada métrica: "pagos
  confirmados"/"pagos rechazados" filtraban por `createdAt` de la
  inscripción (cuándo se creó) en vez de `fechaPago`/`updatedAt`
  (cuándo se confirmó o rechazó el pago) — con voucher, esas dos
  fechas casi siempre son días distintos, así que las confirmaciones
  reales nunca aparecían en ningún resumen. Corregido; pendiente
  confirmar con el correo real de esta noche/mañana.
- **Rediseño del Home + Footer,** usando como referencia un volante
  propio que compartió la fundadora (copy y estructura real, no solo
  estética): nueva sección "Manejo preventivo, manejo defensivo" (5
  pilares) justo después del Hero, testimonios movidos de casi el
  final a justo después de esa sección (prueba social antes de
  precios), sección de "Impacto" + banner de marca, CTA final
  corregido (tenía un párrafo vacío) y cerrado con una frase
  manuscrita del volante en una tipografía de acento nueva (`Caveat`,
  uso puntual). Color del banner escolar cambiado de rosa a mamey (a
  pedido — "el rojo está muy llamativo"). Footer reescrito: Muvo RD
  Vial y la Fundación Mujeres al Volante RD separados en dos bloques
  propios (antes mezclados en un párrafo, con la fundación pareciendo
  dueña de todo el sitio — riesgo real dado que solo subsidia un plan
  específico), copyright cambiado a Muvo RD Vial, y crédito del
  desarrollador agregado. Ver ARQUITECTURA_FRONTEND.md para el detalle
  completo.
- **Entrega:** archivos sueltos por descarga en cada paso (mismo
  patrón que la sesión de soporte/reportes).
- **Documentos de contexto actualizados** (esta misma tarea): los
  cuatro documentos de contexto, a partir de la versión ya actualizada
  por la sesión de reportes/soporte — sin pisar nada de esa sesión.

## 17-18/09/2026 (sexta sesión) — Sistema de reportes/soporte de estudiantes

Pedido de la fundadora: que las estudiantes puedan reportar una
incidencia y que coordinadora/admin la vean y respondan. Idea
original ("chat de soporte") evaluada y descartada a favor de un
sistema de tickets asíncrono (ver ARQUITECTURA_BACKEND.md y
ARQUITECTURA_FRONTEND.md para el detalle completo, DATABASE.md sección
25 para el schema). Resumen:

- **Construido:** colección `Reporte` (estudianteId, tipo, mensaje,
  estado, hilo de `respuestas`), `POST/GET /api/reportes` y variantes,
  toggle por tipo de reporte en `configuracion`
  (`reportes_notificaciones_activas`) para la notificación por correo/
  Telegram (reutiliza `DestinatarioNotificacion`, no es tabla nueva).
  Frontend: `/soporte` (estudiante, enlazado desde el dropdown del
  Navbar), `/panel/soporte` (coordinadora/admin, lista+detalle) con
  badge de reportes abiertos en `/panel`, y `/admin/notificaciones-reportes`
  (toggles, solo admin).
- **Decisiones cerradas con Ramon:** notificación por tipo (no
  todo-o-nada); categorías fijas + "otro" con texto libre; un reporte
  `resuelto` no se puede reabrir (se crea uno nuevo).
- **Bug corregido en el camino: `react-hooks/set-state-in-effect`** en
  las 3 páginas nuevas y en las 2 cargas nuevas de `panel/page.tsx` —
  mismo bug ya conocido de sesiones anteriores (`panel/estudiantes`,
  `panel/grupos/*`, `/inscripcion`). Fix idéntico:
  `queueMicrotask(...)` envolviendo la llamada, tanto para funciones
  nombradas como para IIFEs. De paso se corrigió el mismo defecto, sin
  que nadie lo hubiera reportado antes, en el efecto ya existente de
  `pendientesPago` en `panel/page.tsx`.
- **Entrega:** primero se probó pegando archivo por archivo en el
  proyecto local; una vez confirmado que corría sin el error de lint,
  se re-entregaron los mismos archivos corregidos como descarga
  (Ramon prefiere archivos sueltos para descargar en vez de pegados en
  el chat cuando ya está iterando sobre una entrega previa).
- **Documentos de contexto actualizados** (esta misma tarea):
  `ARQUITECTURA_BACKEND.md`, `ARQUITECTURA_FRONTEND.md` y `DATABASE.md`
  con el detalle completo de todo lo de arriba. De paso, al revisar el
  repo real (zip subido esta sesión) se confirmó que
  `mav-rd-frontend` **no tiene `tailwind.config.ts`** — la nota vieja
  de "ambas convenciones de color funcionan" en
  ARQUITECTURA_FRONTEND.md era optimista; corregida ahí.

## 16/09/2026 (quinta sesión) — Cobertura de práctica de manejo + 3 bugs de zona horaria corregidos

Construcción de punta a punta (backend + frontend) de "Cobertura de
práctica de manejo", siguiendo el análisis cerrado en
`ANALISIS_COBERTURA_PRACTICA.md` (ver banner al principio de ese
documento — ya implementado, el detalle completo vive ahora en
`ARQUITECTURA_BACKEND.md`, `ARQUITECTURA_FRONTEND.md` y `DATABASE.md`
sección 24). Resumen:

- **Construido:** colección `MunicipioPractica` (whitelist de
  cobertura), `src/data/municipiosRD.js` (32 provincias/~160
  municipios), `User.municipio`, `ProgresoEstudiante.tipoPlan`, plan
  `estandar`/`teorico`, validación real de cobertura en
  `inscripcionController.js`, panel `admin/cobertura-practica`, y los
  `<select>` en cascada de provincia→municipio en `/registro` e
  `/inscripcion`. Los dos scripts de siembra (`sembrarCoberturaPractica.js`,
  `sembrarPlanEstandarTeorico.js`) ya corrieron en producción con los 8
  municipios iniciales.
- **Bug de despliegue (no de código):** el primer push a GitHub dejó
  `app.js` sin las dos líneas que montan las rutas nuevas — todos los
  demás archivos sí llegaron, ese cambio puntual no. Render desplegó sin
  error (el código seguía siendo válido) pero las rutas nuevas daban 404. Diagnosticado comparando el `app.js` real en GitHub contra lo
  esperado.
- **Bug corregido en el camino: `react-hooks/set-state-in-effect` en
  `/inscripcion`.** Tres `setState` síncronos dentro de efectos
  (`cargandoPlanes`, `cobertura`, `tipoPlan`) causaban renders en
  cascada reales, no solo ruido del linter. Reescrito como estado
  derivado — ver ARQUITECTURA_FRONTEND.md.
- **Bug corregido: el resumen diario por correo llegaba prácticamente
  siempre en cero.** `utils/resumenDiario.js` calculaba "hoy" en UTC del
  servidor en vez de hora RD (UTC-4) — el cron corre a las 9PM RD, que
  en UTC ya es la madrugada del día siguiente, así que la ventana que se
  consultaba en Mongo estaba mayormente en el futuro. Corregido con un
  offset fijo de 4 horas.
- **Mismo bug encontrado en dos lugares más, corregido de paso:**
  `utils/geminiHerramientas.js` (rangos de fecha/mes del Asistente,
  corridos 4 horas en cada borde) y, más serio,
  `controllers/chatbotController.js` — el Asistente nunca supo qué día
  es hoy (`INSTRUCCION_SISTEMA` no mencionaba la fecha actual), así que
  cualquier pregunta con fecha relativa ("hoy", "este mes") dependía de
  que Gemini adivinara la fecha por su cuenta. Corregido inyectando la
  fecha real (hora RD) en cada pregunta.
- **Documentos de contexto actualizados** (esta misma tarea):
  `ARQUITECTURA_BACKEND.md`, `ARQUITECTURA_FRONTEND.md` y `DATABASE.md`
  con el detalle completo de todo lo de arriba; de paso se restauró un
  encabezado que faltaba en la lista de pendientes de
  `ARQUITECTURA_FRONTEND.md` (defecto de una sesión anterior, no de
  esta).

## 13/09/2026 (cuarta sesión) — Home de Motorizados/Pesados, fix de login, limpieza de documentos

- **Sembrado en producción:** 4 `Sesion` + plan "teorico" de Motorizados
  (RD$3,500) y Pesados (RD$4,500) — bloqueado por un índice viejo
  `numero_1` (único global) en Atlas; se dropeó desde la UI de Atlas y el
  script corrió limpio.
- **Home (`app/page.tsx`):** nueva sección "¿Qué licencia de conducir
  necesitas?" justo después del Hero, con 3 tarjetas de categoría
  (Livianos/Motocicletas/Pesados) reusando las imágenes de `/inscripcion`.
  La de Livianos ancla a la sección de sesiones existente; las otras dos
  van a `/inscripcion?programa=...`. La sección de "Planes y precios"
  ahora también trae y muestra los planes reales de Motorizados/Pesados
  en un bloque propio debajo de los 3 planes de `estandar`.
- **Bug corregido: login se quedaba en `/login`.** `router.push()` se
  llamaba justo después de que `login()`/`registro()` actualizaran
  `usuario` en el contexto — carrera entre esa actualización (asíncrona) y
  `RutaProtegida` leyendo el mismo contexto en la página destino. Se
  corrigió reemplazando el push inmediato por un `useEffect` que reacciona
  al `usuario` ya confirmado, en `login/page.tsx` y `registro/page.tsx`.
- **`RutaProtegida`/`login`/`registro` propagan `?redirect=`** — si alguien
  sin sesión hace clic en una tarjeta de Motorizados/Pesados, no pierde el
  programa elegido al pasar por login o registro.
- **Documentos de contexto reducidos** (esta misma tarea): `ARQUITECTURA_*`
  y `DATABASE.md` actualizados de "pendiente/en diseño" a "construido"
  para Motorizados/Pesados; `ANALISIS_MOTORISTA_PESADOS.md` (483 líneas,
  ya ejecutado) reemplazado por `PENDIENTE_MOTORIZADOS_PESADOS.md` (solo
  lo que sigue abierto); `Resumen sesion 08-09-2026...md` eliminado (ya
  fusionado, lo decía el propio archivo); este archivo comprimido.
- De paso se cerró una duda de documentación abierta desde el 04/08/2026:
  el grupo de ruta `(estudiante)` en el frontend **solo contiene
  `diploma`** — confirmado revisando el código real.

## 13/09/2026 (primera y segunda sesión) — Construcción de Motorizados/Pesados

Sesión de trabajo dedicada para construir Motorizados/Pesados de punta a
punta (backend + frontend), siguiendo el análisis aprobado el 11/09.
Ver `ARQUITECTURA_BACKEND.md` (sección "Motorizados y Pesados") y
`ARQUITECTURA_FRONTEND.md` para el detalle completo de lo construido:
campo `programa`/`programaContenido` de punta a punta, plan único
`"teorico"` sin niveles, gate de práctica centralizado, filtros por
programa en `/inscripcion`, `dashboard`, y los paneles de aula
virtual/exámenes/planes. Decisiones cerradas: 4 sesiones cada uno (igual
que `estandar`), un solo plan sin niveles, nombre **"Motorizados"** (no
"Motoristas", puede sonar despectivo). Además: deploy con dos rondas de
fixes (un rename de colección que no había llegado al repo real, y un
`require` suelto en `scripts/limpiarCuentasBot.js`), y purga de datos de
prueba corrida en real (queda solo `maria@test.com` en producción).

## 11/09/2026 — Análisis de Motorizados/Pesados + bug de Cuestionario Escolar

Documento de análisis aprobado (ver `PENDIENTE_MOTORIZADOS_PESADOS.md`
para lo que sigue abierto de ahí). Correcciones preparatorias aplicadas:
rename completo `InformacionComplementariaEscolar` → `CuestionarioEscolar`
en todo el código; gate de práctica de manejo centralizado en un solo
helper (`utils/elegibilidadPractica.js`, antes repetido en 4 archivos);
índice de `Sesion` corregido de único-global a compuesto
`{ programaContenido, numero }`. Bug corregido: las respuestas de
`CuestionarioEscolar` se guardaban bien pero no existía pantalla para
verlas en el panel — se construyó `/panel/cuestionario-escolar`.

## 10/09/2026 — Seguridad documentada, bug del Home, decisiones de programas nuevos

Se documentó formalmente (no se construyó en esta sesión) la protección
contra registro masivo de bots: Cloudflare Turnstile, rate limiting por
IP, honeypot. Cambios menores a pedido de la fundadora: contabilidad de
`Grupo` sin prorrateo (una sola entrada contable por grupo en vez de una
por estudiante), roster de grupo solo fila-por-fila (se quitó el modo
CSV), soft delete en lote de estudiantes, login de admin/coordinadora ya
no cae en Pagos sino en el dashboard, cédula opcional para estudiantes
sin cédula (menores). Bug corregido: el Home no reflejaba los cambios
guardados desde `/admin/contenido-pagina` (nunca leía `/api/contenido`).
Decisiones cerradas con la fundadora: nuevo programa **Pesados**
(camiones/trailers, currículo propio); `Sesion`/`Examen`/`ContenidoSesion`
no se duplican por programa (campo `programaContenido` en vez de eso).

## 08-09/09/2026 — Programa Escolar/Empresarial completo

Construido y desplegado: colección `Grupo` (tipo colegio/empresa,
roster, `precioAcordado`), gates de práctica/cuestionario condicionales
por `grupoId`, formularios de inscripción de grupo, prorrateo contable,
cron de reporte diario (`GitHub Action` → endpoint protegido por
secreto), badge/filtro de grupo en `/panel/estudiantes`. Pendiente real
que quedó de esta sesión: probar el cron en vivo, y una posible cuenta
huérfana de prueba (probablemente ya limpiada por la purga del
13/09/2026, sin confirmar).

## 07/09/2026 — Reestructuración de planes + purga de usuarios + diseño de programas nuevos

Planes de 2 niveles (Normal/VIP) reestructurados a 3 (Fundación/
Estándar/VIP) en una colección `Plan` nueva, con endpoints y pantalla de
admin propios (`admin/planes`) para editar precio/nombre/características
sin tocar código. Se agregó el campo `programa` a `Plan`/`Inscripcion`
(default `"estandar"`) anticipando Escolar/Empresarial/Motorizados.
Diseño completo de Escolar/Empresarial consolidado (sin construir
todavía en esta sesión). Purga de usuarios de prueba corrida en real.

## 04-06/09/2026 — Práctica de manejo, test psicológico, chatbot y automatización

- **06/09:** seguimiento de práctica de manejo — choferes creados desde
  el panel de admin (rol `conductor`), notificación al completar teoría,
  dashboard del conductor para aprobar práctica, gate del diploma
  extendido a exigir `practicaAprobada`. Sin asignación automática de
  instructor (decisión explícita: la estudiante contacta directo).
- **05/09:** test psicológico de perfil conductual (54 preguntas de
  escala + 5 reflexiones abiertas, digitalizando solo la mitad
  autoadministrable de un instrumento en papel de la fundadora) como
  gate obligatorio antes de cualquier sesión. Sin puntaje calculado a
  propósito (el instrumento mezcla preguntas en sentido positivo/negativo
  y promediar sin ese criterio daría una cifra falsamente objetiva).
  Pendiente real que quedó: confirmación legal bajo la Ley 172-13
  (resuelta después, ver entrada del 10/09).
- **04/09:** chatbot interno para la fundadora (Gemini + function
  calling, 7 herramientas de solo lectura) y resumen diario automatizado
  por correo/Telegram (GitHub Action a las 9pm RD). De paso, el
  formulario de Empresas empezó a persistir los leads en Mongo (antes
  solo notificaba, sin guardar nada).

## 27-28/08/2026 — Dominio propio, SEO, y hallazgo de contenido con defectos

Dominio propio (`muvordvial.com`) verificado en Vercel y Resend — correo
transaccional real desbloqueado. SEO (sitemap, robots.txt, metadata,
Search Console) migrado del dominio de Vercel al propio. Al revisar el
Home se destapó contenido cargado en una sesión sin documentar con dos
bugs serios: **PDFs con errores de codificación** y **exámenes con la
respuesta correcta siempre en la opción A** (patrón explotable) —
**pendiente de alta prioridad, sigue sin resolver**, ver "Pendientes
abiertos" al final de este documento. Fix de UI menor en aula virtual
(alineación del botón "Marcar como visto" en contenido pdf/enlace).

## Antes del 27/08/2026 — Construcción inicial y primeras iteraciones

Resumen muy comprimido de sesiones más antiguas, ya estables y sin
pendientes reales conocidos:

- **13/08:** subida de PDF como archivo en `ContenidoSesion` (antes solo
  URL), sección de Planes en el Home, formulario de Empresas (sin
  persistencia todavía, ver 04/09).
- **06-07/08:** diploma compartible en redes sociales, ampliación del
  curso a 4 sesiones, purga de datos de prueba (primera ronda), backup
  manual (Docker local + Dropbox cifrado).
- **04/08:** exámenes y contenido desactivables desde el panel, panel de
  estudiantes reorganizado en pestañas (Activas/Graduadas/Inactivas) con
  archivado en vez de crecer indefinidamente.
- **22/07-03/08:** notificaciones por email/Telegram, auto-inscripción
  con voucher, Open Graph, rediseño de `ProgresoCarretera`, y la
  construcción inicial del backend y frontend antes de eso.

---

## Pendientes abiertos

> Solo lo que sigue realmente abierto. Los pendientes de Motorizados/
> Pesados viven en `PENDIENTE_MOTORIZADOS_PESADOS.md`, no aquí.

### Alta prioridad

- **Recrear `ContenidoSesion` (PDFs) y `Examen` de `estandar` desde
  cero.** El contenido cargado tiene PDFs con errores de codificación y
  exámenes con la respuesta correcta siempre en la opción A (patrón
  explotable). Plan: borrar vía `curl` contra los endpoints reales (no a
  mano en Atlas), recrear con PDFs corregidos y opciones aleatorizadas.
  Buen momento para de paso renombrar `Sesion.titulo` (sigue en "Sesión
  1"..."Sesión 4", solo los materiales individuales tienen títulos
  reales).

### Sin bloqueo, no depende de la fundadora

- **NUEVO (18/09/2026):** desplegar a producción el sistema de
  reportes/soporte (construido y probado en local) — confirmar que
  `app.js` real en GitHub/Render incluye las rutas nuevas (misma
  lección operativa de la sesión del 16/09). Decidir con la fundadora
  si el aviso de reporte nuevo debe llegar también por Telegram.
- Terminar Telegram para el celular de la fundadora (`chat_id` pendiente
  de agregar en el panel de Notificaciones) — canal de respaldo, el
  correo real ya funciona.
- Permisos inconsistentes: `PATCH /api/usuarios/desactivar-lote` y
  `PATCH /api/usuarios/:id/estado` son `admin`-only, pero la UI que los
  consume vive en `/panel`, al que también entra `coordinadora`. Decidir
  si se le abre el permiso.
- Probar en vivo el cron de `/api/interno/reporte-grupos` (nunca se
  esperaron las 24h reales ni se disparó a mano). Revisar si quedó una
  cuenta de estudiante huérfana de una prueba anterior (probablemente ya
  limpiada por la purga del 13/09/2026).
- **NUEVO (16/09/2026):** verificar `src/data/municipiosRD.js` (32
  provincias/~160 municipios, ver ARQUITECTURA_BACKEND.md) contra una
  fuente oficial de la JCE/ONE — se compiló de fuentes públicas
  generales, sin verificación línea por línea.
- Construir un formulario en el panel para renombrar sesiones (hoy solo
  vía `PATCH /sesiones/:numero` a mano).
- Decidir si vale la pena `POST /sesiones` (crear sesión desde el panel)
  o si el script de terminal es suficiente a largo plazo.

### Ideas a futuro, sin diseñar

- Recordatorios por correo: examen disponible (pasadas las 24h de
  espera) y voucher sin seguimiento — el disparador (patrón GitHub
  Action → endpoint con secreto) ya existe, solo falta diseñar contenido
  y frecuencia.
- Texto enriquecido con imágenes incrustadas en `contenidoTexto`.
- "Me gusta" en comentarios individuales de noticias.
- Animaciones más elaboradas en el Home (`framer-motion`) — evaluado,
  no empezado.

### Mejoras menores, sin urgencia

- Confirmar los valores hex reales de `brand-yellow`/`brand-mamey` y
  unificar la convención de nombres de color Tailwind
  (`brand-blue-light` vs `brand-blueLight`) — **confirmado (18/09/2026)
  que no es solo cosmético:** no hay `tailwind.config.ts` en el repo,
  así que la variante camelCase probablemente no pinta nada. Auditar
  qué páginas del panel/aula virtual la usan.
- Afinar el rol `backup_readonly` en Atlas a un rol Read específico
  sobre `mav_rd` (hoy es `readAnyDatabase@admin`, de solo lectura de
  todas formas — no urgente).
- Monitorear si con más choferes hace falta paginación en
  `GET /instructores/activos`.
- Monitorear la vigencia del modelo de Gemini del chatbot
  (`GEMINI_MODEL` en Render) — Google reemplaza modelos cada pocas
  semanas.
- Revisar lenguaje de género pendiente en `testimonios/page.tsx` y
  `registro/page.tsx`.
- Agregar la sección "Lo que aprendiste" (temas reales) al diploma
  compartible, una vez esté resuelto el punto de "Alta prioridad".
- Seguridad/confiabilidad de fondo (rotar credenciales expuestas, CORS
  dinámico, Sentry): al final, cuando la app esté más madura.

### Decisiones cerradas (no volver a proponerlas)

- Pasarela de pago automática (Azul): no se hará.
- Asignación automática de instructor de práctica: no se hará — la
  estudiante contacta directo de una lista.
- Pedir testimonio automáticamente al graduarse: descartado.
- Foro/preguntas por sesión: pospuesto a una segunda etapa.
- Limpieza de datos de prueba en Mongo: se hace por sesión dedicada
  (última corrida: 13/09/2026), no de forma continua.
