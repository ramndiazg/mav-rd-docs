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
  error (el código seguía siendo válido) pero las rutas nuevas daban
  404. Diagnosticado comparando el `app.js` real en GitHub contra lo
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
  (`brand-blue-light` vs `brand-blueLight`) en `ARQUITECTURA_FRONTEND.md`.
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
