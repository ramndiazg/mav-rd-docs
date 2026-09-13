# Historial de modificaciones — Muvo RD Vial

> Registro breve por sesión. El estado actual y detallado del sistema vive en
> ARQUITECTURA_BACKEND.md, ARQUITECTURA_FRONTEND.md y DATABASE.md, este
> archivo es solo un changelog, no la fuente de verdad de cómo funciona nada.

## 13/09/2026 (segunda sesión) — Construcción de Motorizados/Pesados

Sesión de trabajo dedicada (la que se había pospuesto explícitamente en la
sesión anterior) para construir Motorizados/Pesados de punta a punta,
siguiendo el orden de trabajo de `ANALISIS_MOTORISTA_PESADOS.md`, sección 8.

**Backend:**

- `models/ProgresoEstudiante.js`: campo nuevo `programa` (espejo de
  `Inscripcion.programa`, seteado una sola vez al confirmar el pago).
- `models/Plan.js`: `codigo` acepta `"teorico"` (plan único de
  Motorizados/Pesados, sin niveles); `modalidadPractica`,
  `duracionSesionMinutos` y `costoPorSesion` pasan a `required: false` — un
  plan "teorico" no tiene práctica de manejo, no hay con qué llenarlos.
- `models/Inscripcion.js`: `tipoPlan` acepta `"teorico"`.
- `utils/elegibilidadPractica.js`: `requierePracticaDeManejo` ahora recibe
  un segundo parámetro opcional `programa` — `motorizados`/`pesados` no
  requieren práctica, igual que las estudiantes con `grupoId` (Escolar/
  Empresarial). Actualizados los 4 archivos que lo usan:
  `diplomaController.js` (x2), `practicaController.js`,
  `intentoExamenController.js`.
- `controllers/sesionController.js`: las 3 funciones (`listarSesiones`,
  `obtenerSesionParaEstudiante`, `actualizarSesion`) ahora filtran por
  `programaContenido` — sin esto, con `Sesion.numero` repetido entre
  programas (ya no es único global desde el 11/09), una estudiante de
  Motorizados podía terminar viendo la Sesión 1 de `estandar` por
  accidente. `listarSesiones` acepta `?programaContenido=` opcional (sigue
  devolviendo todo si no se manda, por compatibilidad); `actualizarSesion`
  acepta `?programaContenido=` con default `"estandar"` (endpoint sin uso
  real en el frontend todavía, pero corregido igual).
- `controllers/diplomaController.js`: `listarElegibles` y `generarDiploma`
  usan el nuevo criterio con `programa`; además, la consulta de `Sesion`
  para armar la lista de sesiones del PDF del diploma ahora también
  filtra por `programaContenido` (mismo motivo que arriba).
- `controllers/inscripcionController.js`: `crearInscripcion` y
  `crearOReenviarInscripcionPropia` aceptan `programa` del body (default
  `"estandar"`), validan `tipoPlan` contra la lista válida de ese
  programa, y lo guardan en la `Inscripcion` creada. `confirmarPago` copia
  `inscripcion.programa` al `ProgresoEstudiante` que crea.
- Script nuevo `scripts/sembrarMotorizadosPesados.js`: siembra las 4
  `Sesion` (títulos provisionales, sin contenido real) y el `Plan`
  "teorico" (precio provisional RD$0, o el que se pase con
  `--precio-motorizados=`/`--precio-pesados=`) de cada programa. Mismo
  patrón dry-run/`--confirmar` que los scripts existentes. **Sin correr
  todavía** — pendiente para cuando se despliegue este trabajo.

**Frontend:**

- `app/inscripcion/page.tsx`: nuevo paso "Elige tu curso" (3 tarjetas:
  Escolares/Motorizados/Pesados) antes de "Elige tu plan". Los planes se
  piden a `GET /api/planes?programa=...` según lo elegido; el formulario
  manda `programa` en `POST /inscripciones/mia`. Admite preselección vía
  `?programa=motorizados` en la URL (para un eventual botón "Inscríbete"
  desde una página de marketing propia — no construida todavía, ver
  pregunta 4 de `ANALISIS_MOTORISTA_PESADOS.md`). Envuelto en `<Suspense>`
  por el uso de `useSearchParams`.
- `app/dashboard/page.tsx`: `requierePractica` ahora también depende de
  `progreso.programa` (motorizados/pesados no tienen práctica), no solo de
  `grupoId`. El conteo fijo `SESIONES = [1,2,3,4]` se dejó **sin tocar** a
  propósito — la pregunta de "¿cuántas sesiones tiene cada programa?" se
  cerró el 13/09 (primera sesión) en las mismas 4 para los 3 programas, así
  que no hacía falta el fetch dinámico que el análisis original planteaba
  como alternativa.
- `app/(coordinadora)/panel/aula-virtual/page.tsx` y
  `panel/examenes/page.tsx`: pestañas de programa (Escolares/Motorizados/
  Pesados) para filtrar qué sesiones se gestionan — filtrado en el
  cliente sobre la misma lista completa que ya se traía de
  `GET /api/sesiones`, sin pegarle otra vez al backend por cada pestaña.
- `app/(admin)/admin/planes/page.tsx`: agregado el selector de programa
  (antes solo mostraba los planes de `estandar`, no tenía forma de llegar
  a los de Motorizados/Pesados). El fetch y el `PATCH` ahora mandan
  `?programa=`; el formulario oculta los 4 campos de práctica cuando
  `plan.codigo === "teorico"` (no aplican). Se corrigió también un `key`
  que hubiera colisionado (`plan.codigo` solo, y "teorico" existe en dos
  programas distintos) — pasó a `` `${plan.programa}-${plan.codigo}` ``.

**Verificación:** todos los `.js` del backend tocados pasan `node --check`.
El frontend se revisó a mano (balance de llaves/paréntesis por archivo) —
el proyecto no tiene `node_modules` instalado, así que no se pudo correr
`tsc --noEmit` ni `next build` completo. **Recomendado antes de
desplegar:** correr `npm run build` en el frontend real (con
`node_modules`) para atrapar cualquier error de tipos que esta revisión
manual no haya visto.

**Pendiente real para la próxima sesión (no bloqueante, pero sin hacer):**

1. Correr `node scripts/sembrarMotorizadosPesados.js --confirmar` en
   producción (con los precios reales vía `--precio-motorizados=`/
   `--precio-pesados=`, o ajustarlos después desde
   `/admin/planes`). Recordar el paso de despliegue del 11/09: confirmar
   que el índice viejo `numero_1` de `Sesion` ya no existe en Atlas antes
   de sembrar (si el modelo se desplegó después del 11/09, ya no aplica).
2. Cargar contenido real y exámenes (con **varias versiones activas por
   sesión desde el día uno**) para Motorizados y Pesados desde el panel.
3. `npm run build` real en ambos repos antes de desplegar (ver arriba).
4. Preguntas 3 y 5 de `ANALISIS_MOTORISTA_PESADOS.md` (sección 7) siguen
   abiertas: contenido del diploma (¿debe decir "Motorizados"/"Pesados"
   explícitamente?) y si Pesados necesita distinguir camión/trailer dentro
   del programa. No bloquean lo ya construido.
5. Página(s) de marketing propia por programa (`/motorizados`, `/pesados`)
   — opcional, ver sección 5 del análisis. El `?programa=` de
   `/inscripcion` ya está listo para recibirlas como destino de un botón
   "Inscríbete" si se deciden a construir.

## 13/09/2026 — Deploy de las correcciones del 11/09, bug adicional encontrado, decisiones finales de Motorizados/Pesados

Sesión corta de seguimiento tras desplegar los cambios del 11/09 a los
repos reales (Render + Vercel).

**Deploy con problemas, resuelto en dos rondas:**

1. Primer intento: Render tiró `Cannot find module
   '../models/InformacionComplementariaEscolar'` — el rename del 11/09
   nunca había llegado a pegarse en el repo real (solo se habían
   borrado los archivos viejos, sin reemplazo). Vercel no lo detectó
   porque un link roto a una ruta de Next.js no rompe el build, solo da
   404 en producción — por eso "en Vercel subió bien" no era señal de
   que el backend también estuviera bien.
2. Se recomendó reemplazar carpetas completas (`src/`, `scripts/` en
   backend; `app/`, `contexts/`, `lib/` en frontend) en vez de archivos
   sueltos, para evitar que quedara algo a medio pegar otra vez.
3. Tras aplicarlo, la fundadora encontró **un archivo que se me había
   pasado en el rename original**: `scripts/limpiarCuentasBot.js`
   tenía un `require("../src/models/InformacionComplementariaEscolar")`
   real (no comentario) — no crasheaba el servidor porque `scripts/` no
   se ejecuta al bootear, solo al correrlo a mano. Corregido. Se
   confirmó con una búsqueda de **todo** el repo (no solo `src/`) que
   no queda ningún otro caso — antes la búsqueda se había limitado a
   los archivos que ya se sabía que se habían tocado.

**Aclaración de expectativas:** la fundadora, tras varios días desde el
análisis original, esperaba ver Motorizados/Pesados ya funcionando
(tarjetas en `/inscripcion`, contenido gestionable desde el panel). Se
aclaró que el trabajo del 11/09 fue **solo la preparación técnica**
que el propio análisis pedía hacer antes de construir el programa — la
construcción real (backend + `/inscripcion` + panel + dashboard) nunca
se empezó. Documentado explícitamente en `ARQUITECTURA_BACKEND.md` para
que no se repita la confusión.

**Decisiones finales cerradas para Motorizados/Pesados** (cierran las
preguntas 1 y 2 de `ANALISIS_MOTORISTA_PESADOS.md`, sección 7):
- **4 sesiones** cada uno, igual que `estandar`.
- **Un solo plan por programa, sin niveles** (no hay práctica de manejo).

**Corrección al propio análisis:** se encontró que la pregunta de
"Estructura de precio" se había borrado por accidente en una edición
anterior del documento (un `str_replace` mal delimitado durante el
renumerado del 11/09) — restaurada con su respuesta ya incluida.

**Construcción de Motorizados/Pesados: queda para una sesión de trabajo
dedicada**, a pedido explícito de la fundadora, para no arrancarla y
dejarla a medias por falta de tiempo/espacio en el chat. Con las dos
últimas preguntas cerradas, el análisis queda completo — la próxima
sesión puede ir directo a construir siguiendo la sección 8.

**Purga de datos de prueba: corrida en real.** Dry-run y `--confirmar`
dieron los mismos conteos (consistentes, sin sorpresas): 10 usuarios
estudiante, 5 inscripciones, 4 progresoEstudiante, 3 grupos, 16
movimientosContables, 1 balanceMensual. Base de datos de producción
queda con una sola cuenta: `maria@test.com` (admin).

## 11/09/2026 — Análisis de Motorizados/Pesados, bug de Cuestionario Escolar corregido, 3 correcciones preparatorias

Sesión de dos partes: primero un documento de análisis
(`ANALISIS_MOTORISTA_PESADOS.md`) sobre el flujo completo de los
programas Motorista y Pesados (todavía sin construir), y después, ya
aprobado el análisis, las correcciones que ese mismo análisis identificó
como necesarias antes de tocar el código mayor.

**Decisiones cerradas con la fundadora en esta sesión:**

- Nombre: **"Motorizados"**, no "Motoristas" — puede sonar despectivo.
  Se usa así tanto en copy como en el valor real de `programa`/
  `programaContenido` cuando se construya el programa.
- Motorizados y Pesados usan el `TestPsicologico` completo (54+5), igual
  que el resto de los choferes — no necesitan un cuestionario corto
  propio.
- Se agrega una pestaña informativa nueva **"Educación Vial Escolar"**
  (`/escolar`, mismo patrón que `/empresas`) — Estándar, Motorizados y
  Pesados en cambio se explican dentro de `/inscripcion`, no como
  páginas propias.
- Énfasis explícito de la fundadora: dado el tamaño de los cambios que
  se vienen, cuidar especialmente que nada de lo que ya funciona se
  rompa — regresión, no solo feature nueva.

**Bug reportado y confirmado:** la fundadora reportó que las respuestas
de dos estudiantes de un Grupo no aparecían en el panel al revisar "los
test psicológicos". No era un bug de datos — `GET /api/cuestionario-
escolar` (entonces `/informacion-complementaria-escolar`) siempre
guardó y devolvió bien la información. Nunca se había construido una
pantalla para verla; solo existía `/panel/test-psicologico`, que lista
una colección distinta (`TestPsicologico`). Se construyó
`/panel/cuestionario-escolar`.

**Correcciones aplicadas (backend + frontend):**

1. Rename completo `InformacionComplementariaEscolar` →
   `CuestionarioEscolar` en código (modelo, controller, rutas, mount en
   `app.js` como `/api/cuestionario-escolar`, ruta de estudiante
   `/cuestionario-escolar`, comentarios en `authController.js`,
   `User.js`, `AuthContext.tsx`, `dashboard/page.tsx`). El documento ya
   decía "renombrado" desde el 10/09, pero el código en disco seguía
   con el nombre viejo — quedó sincronizado.
2. Gate de práctica de manejo centralizado en
   `utils/elegibilidadPractica.js#requierePracticaDeManejo` — antes el
   mismo `!grupoId` estaba repetido de forma independiente en
   `diplomaController.js` (x2), `practicaController.js` e
   `intentoExamenController.js`. Sin cambio de comportamiento.
3. Índice de `Sesion` corregido: `numero` dejó de ser único a nivel de
   campo (único global) y pasó a único compuesto
   `{ programaContenido, numero }`, con `programaContenido` (default
   `"estandar"`) agregado al esquema. **Pendiente de despliegue:**
   dropear a mano el índice viejo `numero_1` en Atlas (o correr
   `syncIndexes()`) antes de sembrar sesiones de un programa nuevo.
4. Nueva pantalla `app/(coordinadora)/panel/cuestionario-escolar/page.tsx`
   (mismo patrón que `panel/test-psicologico/page.tsx`) + su tarjeta en
   `panel/page.tsx`.
5. De paso, se corrigió un error de documentación en `DATABASE.md`: el
   campo real del modelo es `reflexiones`, no `respuestasAbiertas` como
   estaba escrito ahí; y se reubicó un párrafo sobre historial de
   precios de `Plan` que había quedado pegado por error bajo la sección
   de `CuestionarioEscolar`.

**Verificación:** todos los `.js` del backend tocados pasan
`node --check`. Los `.tsx`/`.ts` del frontend se revisaron a mano
(llaves balanceadas) — el proyecto no tiene `node_modules` instalado,
así que no se pudo correr `tsc --noEmit` completo.

**Sigue pendiente** (detalle completo en `ANALISIS_MOTORISTA_PESADOS.md`):
cantidad de sesiones de Motorizados/Pesados, estructura de precio/`Plan`
para un programa sin práctica, contenido del diploma, selector de
programa en `/inscripcion`, página `/escolar`, y dejar de hardcodear
`SESIONES = [1,2,3,4]` en `dashboard/page.tsx`.

### Continuación misma fecha: purga de datos de prueba ampliada

A pedido de la fundadora, para dejar la base lista para producción con
solo la cuenta `maria@test.com`:

- **Eliminado `purgarDatosPrueba.js`** — era un duplicado exacto de
  `purgarUsuariosPrueba.js`, sobrante de una purga anterior. Queda un
  solo script de purga en el proyecto.
- **`purgarUsuariosPrueba.js` ampliado:** la cascada ahora también borra
  `CuestionarioEscolar` (bug: nunca se había agregado a la cascada
  cuando se creó esa colección) y, sin filtrar por usuario,
  `Grupo`, `MovimientoContable` y `BalanceMensual` completos. Sigue sin
  tocar `Sesion`/`Examen`/`ContenidoSesion`/`Plan` ni
  `SolicitudEmpresarial` (no pedido explícitamente).
- Se explicó por qué esto se resuelve con un script Node conectado
  directo a Mongo y no con `curl` contra la API: no existe (ni debería
  existir) un endpoint HTTP de "borrar todo" — sería un riesgo de
  seguridad serio si algún día quedara mal protegido.
- Se documentó el procedimiento para dropear en Atlas el índice viejo
  `numero_1` de `sesiones` (paso manual, Mongoose no lo hace solo) antes
  de que el índice compuesto nuevo (`{ programaContenido, numero }`,
  ver más arriba) tome efecto del todo.

## 10/09/2026 — Seguridad documentada, cambios menores, bug del Home, y cierre de decisiones de Escolar/Empresarial/Motorista/Pesados

### Contexto de arranque

Entre la sesión del 09/09 y esta, hubo una sesión previa sin documentar
donde se construyó protección contra un ataque de registro masivo de
cuentas falsas (bot llenando `/registro` y `/empresas`). Esta sesión
arrancó auditando ese código contra ARQUITECTURA_BACKEND.md/
ARQUITECTURA_FRONTEND.md (no estaba registrado en ningún lado) y lo
documentó formalmente antes de tocar nada nuevo.

### Seguridad — auditada y documentada (construida en sesión previa)

Cloudflare Turnstile (`utils/captcha.js`, solo en `/registro`), rate
limiting por IP en 5 endpoints (`middleware/rateLimiters.js`) y
honeypot (`sitioWeb`) en `/registro` y `/empresas`. Nada de esto se
tocó en código esta sesión, solo se confirmó y se escribió en
ARQUITECTURA_BACKEND.md/ARQUITECTURA_FRONTEND.md por primera vez.

### Cambios menores pedidos por la fundadora

1. **Contabilidad de `Grupo` sin prorrateo.** Antes: un
   `MovimientoContable` por estudiante, cada uno con su parte
   prorrateada de `precioAcordado` — muchas entradas pequeñas por el
   mismo grupo distorsionaban el balance. Ahora: una sola
   `MovimientoContable` por grupo, por el monto TOTAL, creada solo en
   la primera confirmación del roster. Adiciones tardías ya no generan
   ningún movimiento nuevo. `Inscripcion.monto` por estudiante sigue
   prorrateado, pero ya es solo referencia interna.
2. **Roster de grupo — se eliminó el modo "CSV/pegado".** Quedó solo
   fila por fila (`panel/grupos/[id]/page.tsx`) — el modo CSV no se
   entendía bien (columnas por posición, sin encabezados visibles).
3. **Soft delete en lote de estudiantes.** `PATCH
/api/usuarios/desactivar-lote` (nuevo, admin) + checkboxes en
   "Estudiantes ya cargadas" (`panel/grupos/[id]/page.tsx`) para
   desactivar varias de una vez cuando el roster de una institución
   cambia.
4. **Login de admin/coordinadora ya no cae en Pagos.** `login/page.tsx`
   y `Navbar.tsx` redirigían a `/panel/pagos` — ahora van a `/panel`
   (el dashboard). El badge de "pago nuevo" ya avisa cuando hace falta
   revisar pagos, no había que forzar esa pantalla en cada sesión.
5. **Cédula opcional para estudiantes sin cédula (menores).** Al crear
   un grupo tipo colegio, escribir "N/A" en la cédula de un segundo
   menor chocaba como duplicado — `User.cedula` tenía `unique` normal.
   Se cambió a `unique + sparse` (mismo patrón que
   `Inscripcion.numeroReferencia`) y `grupoController.js` ahora guarda
   cualquier variante de "n/a"/vacío como `undefined` real, no como
   texto. Sigue siendo obligatoria en autoregistro individual.

Ver ARQUITECTURA_BACKEND.md, ARQUITECTURA_FRONTEND.md y DATABASE.md
para el detalle técnico completo de los 5 puntos.

### Bug corregido: el Home no reflejaba los cambios del dashboard de contenido

`app/(admin)/admin/contenido-pagina/page.tsx` permite editar el hero y
la tarjeta de la derecha del Home desde hace tiempo, y guardaba bien en
Mongo — pero `app/page.tsx` nunca leía `/api/contenido`, esos textos
estaban fijos en el JSX. Mismo bug que ya se había resuelto antes en
`acerca-de-nosotros/page.tsx`. Corregido replicando ese mismo patrón.
De paso, se renombró el título fijo de la tarjeta ("Desde 2017", sin
contexto suficiente) a **"Así empezamos"**, en el código y en la
etiqueta del campo correspondiente del dashboard.

### Decisiones cerradas con la fundadora (sin tocar código, para desbloquear el diseño de Motorista/Pesados)

- **Blockers de infraestructura cerrados:** dominio de Resend
  verificado, todo corriendo — y se confirmó en código que ya existe
  una UI de admin para editar precio/nombre/descripción de los planes
  (`admin/planes`, existía desde el 07/09, solo faltaba cerrarlo como
  pendiente). Solo queda abierto el `chat_id` de Telegram de la
  fundadora.
- **Nuevo programa: Pesados** — conductores de vehículos pesados,
  alcance inicial limitado a camiones y trailers, currículo propio
  distinto a "estandar" (igual que Motorista). Sin diseñar a detalle
  todavía — se suma a la lista de diseño pendiente con la fundadora.
- **Nombre de colección confirmado:** `CuestionarioEscolar` (no
  `InformacionComplementariaEscolar` — no había documentos reales
  creados, no hizo falta migrar nada).
- **Revisión legal (Ley 172-13)** de las 14 preguntas de
  `CuestionarioEscolar`: confirmada, aprobada.
- **Modelo de contenido para programas con currículo propio:**
  `Sesion`/`Examen`/`ContenidoSesion` no se van a duplicar por
  programa — se les agregará un campo `programaContenido`
  (`estandar`/`motorista`/`pesados`, todavía no implementado en
  código) cuando se construyan Motorista/Pesados. Escolar/Empresarial
  no necesitan nada nuevo aquí, siguen reusando `estandar`.
- **`Plan` no necesita entradas para Escolar/Empresarial** — confirmado,
  su precio vive solo en `Grupo.precioAcordado`. `Plan` sí las
  necesitará para Motorista/Pesados el día que se inscriban
  individualmente.

Ver la entrada "Ya resuelto" al final de este documento y
ARQUITECTURA_BACKEND.md/DATABASE.md para el detalle completo de cada
decisión.

### Pendiente nuevo, encontrado esta sesión

- **Permisos inconsistentes:** `PATCH /api/usuarios/desactivar-lote`
  (nuevo) y `PATCH /api/usuarios/:id/estado` (ya existía) son ambos
  `admin`-only, pero la UI vive en `/panel`, al que también entra
  `coordinadora`. Sin resolver todavía si se le abre el permiso a ella.
- **Diseñar Motorista y Pesados en detalle con la fundadora** — es lo
  único que falta de la lista de programas nuevos. Primer paso:
  la misma conversación de descubrimiento que ya se tuvo para
  Escolar/Empresarial, para cada uno.
- Actualizar los documentos de contexto quedó para el final del bloque,
  como de costumbre — hecho en esta misma sesión, después de probar
  todos los cambios de código.

## 08-09/09/2026 — Programa Escolar/Empresarial completo: Grupo, roster, prorrateo, reporte diario

### Contexto de arranque

Continuación directa del diseño consolidado en
`ESPECIFICACION_PROGRAMAS_NUEVOS.md` (acordado con la fundadora a lo
largo de varias conversaciones, ver entrada del 07/09/2026). Dos
sesiones seguidas: el 08/09 se construyó y probó la primera mitad
(colección `Grupo`, gates de práctica/cuestionario); el 08/09 se cortó
antes de terminar y dejó un resumen aparte
(`Resumen_sesion_08_09_2026_grupo___MD`) para que la sesión del 09/09
pudiera seguir sin repetir el análisis. El 09/09 arrancó revisando ese
bloque contra el código real (sin bugs encontrados, solo un efecto en
cascada que se había pasado por alto), corrigió un bug no relacionado
que salió en el camino (plan "Fundación" rechazado al inscribirse), y
construyó el resto: formularios de grupo, prorrateo contable, y el cron
de reporte diario.

### Construido — 08/09 (pasos 1-2 de la especificación)

Backend: `models/Grupo.js`, campo `User.grupoId`, colección
`CuestionarioEscolar` (14 preguntas, gate en
`sesionController.js` según `Grupo.tipo`), gate de práctica condicional
en `diplomaController.js` (`requierePractica = !usuario.grupoId`) con
sus efectos en cascada en `intentoExamenController.js` (no notifica
"lista para práctica") y `dashboard/page.tsx`/`ProgresoCarretera.tsx`
en el frontend (ocultan el paso de práctica). Ver ARQUITECTURA_BACKEND.md
y ARQUITECTURA_FRONTEND.md para el detalle completo.

### Construido — 09/09 (pasos 3-5 de la especificación)

- `controllers/grupoController.js` + `routes/grupoRoutes.js`
  (`/api/grupos`): Formulario 1 (crear grupo), Formulario 2 (confirmar
  roster — crea cuentas, inscripciones ya pagadas, prorrateo contable,
  `ProgresoEstudiante`, correo de credenciales), listar/detalle/editar.
  Soporta adiciones tardías al mismo grupo sin re-prorratear lo ya
  cobrado.
- `Inscripcion.tipoPlan` — nuevo valor `"grupo"`, exclusivo de
  estudiantes de un `Grupo` (precio vive solo en `Grupo.precioAcordado`,
  nunca pasa por `Plan`).
- `utils/reporteGrupos.js` + `POST /api/interno/reporte-grupos` +
  `.github/workflows/reporte-grupos.yml` — cron diario (10am RD) que
  manda un correo de avance por grupo al contacto de la institución, y
  apaga el grupo solo cuando todas sus estudiantes completan la teoría.
- Frontend: `panel/grupos/page.tsx` (listado + Formulario 1) y
  `panel/grupos/[id]/page.tsx` (Formulario 2, CSV/pegado + fila por
  fila), tarjeta nueva en el panel principal.
- Mejora en `panel/estudiantes/page.tsx` (pedida por el usuario después
  de probar): badge de institución por fila + filtro por grupo, para
  distinguir individuales de estudiantes de un `Grupo` y ver quiénes son
  compañeras de la misma institución. `usuarioController.js` ahora
  popula `grupoId` y acepta ese filtro.

Ver ARQUITECTURA_BACKEND.md y ARQUITECTURA_FRONTEND.md para el detalle
técnico completo de todo lo de arriba.

### Errores encontrados y corregidos en el camino

- **Efecto en cascada no cubierto el 08/09:**
  `practicaController.js#listarPendientes` mostraba para siempre a
  estudiantes de `Grupo` en la lista de "esperando práctica" (nunca les
  llega `practicaAprobada: true`, no aplica). Corregido con el mismo
  filtro `!usuario.grupoId` que ya tenían los otros tres puntos de la
  cascada.
- **Bug no relacionado con Grupo, encontrado al revisar el código:**
  desde la reestructuración de planes del 07/09,
  `inscripcionController.js` seguía validando `tipoPlan` contra
  `["normal","vip"]` (rechazaba `"fundacion"`) y buscando el precio en
  `Configuracion` (ya migrada a `Plan`). Corregido en los dos lugares
  afectados (`crearInscripcion` y `crearOReenviarInscripcionPropia`).
- **`react-hooks/set-state-in-effect` en las páginas nuevas de
  `panel/grupos/`** — se me olvidó aplicar el fix (`queueMicrotask`) ya
  usado en `panel/estudiantes/page.tsx` desde la sesión del 06/09.
  Corregido en ambas páginas nuevas.
- **Bug encontrado en producción, no en la revisión de código:**
  `Inscripcion.numeroReferencia` tenía `default: null` + índice
  `unique, sparse` — un `sparse` solo excluye documentos donde el campo
  está _ausente_, no donde vale `null` explícito, así que la segunda
  `Inscripcion` sin voucher (segunda estudiante de un grupo) chocaba
  como "duplicado" contra la primera. Bug preexistente (afectaba
  también el flujo "efectivo" del admin si se usaba más de una vez),
  que el flujo de Grupo expuso por ser el primero en crear varias
  inscripciones seguidas sin voucher. Corregido quitando el `default`
  en `models/Inscripcion.js`.

### Estado al cierre

Desplegado en Render/Vercel y probado por el usuario: crear grupo →
cargar roster → (salieron y se corrigieron los dos bugs de arriba) →
agregar una segunda estudiante funcionando. **Sin probar todavía:** el
cron de reporte diario (nunca se esperaron las 24h reales ni se disparó
a mano desde GitHub Actions). Puede haber quedado una cuenta de
estudiante huérfana (sin `Inscripcion`/`MovimientoContable`) de la
prueba donde salió el bug de `numeroReferencia` — no se limpió, revisar
en Mongo Atlas antes de reintentar con esa misma cédula/correo.
**Motorista sigue sin diseñar** — único punto de
`ESPECIFICACION_PROGRAMAS_NUEVOS.md` que no se tocó en ninguna de las
dos sesiones.

## 07/09/2026 — Reestructuración de planes (Fundación/Estándar/VIP), UI de admin, purga de usuarios, y diseño completo de Escolar/Empresarial/Motorista

### Contexto de arranque

Sesión larga con varios pedidos encadenados. Empezó con un análisis de
factibilidad: la fundadora quiere agregar dos líneas de formación nuevas
(Escolar y Empresarial) y, aprovechando el cambio, restructurar los
planes actuales de 2 (Normal RD$1,500 / VIP RD$7,000) a 3. El análisis
inicial completo (currículo distinto, multi-tenencia, legal de menores)
se simplificó varias veces por decisión del usuario hasta llegar a un
diseño mucho más chico de lo que parecía al principio — ver "Diseño de
Escolar/Empresarial" más abajo.

### Purga de usuarios de prueba (segunda purga)

Se detectó, revisando `DATABASE.md`, que quedaba pendiente sin resolver
desde antes: no había criterio definido para identificar qué datos
seguían siendo de prueba tras la purga del 06/08/2026. Se definió el
criterio con el usuario (todos los usuarios excepto `maria@test.com`,
sin importar rol) y se construyó `scripts/purgarUsuariosPrueba.js` —
igual patrón de seguridad que el script viejo (dry-run por defecto,
confirmación escrita), pero con cascada actualizada (`TestPsicologico`,
`Instructor`) y sin tocar `Sesion`/`Examen`/`ContenidoSesion` (esas ya
tenían contenido real). Corrido en modo real: se borraron 4 usuarios (3
estudiante + 1 conductor) y su cascada completa. Ver DATABASE.md para el
detalle exacto.

**Nota de proceso:** al copiar el script, el usuario sobrescribió por
accidente el `purgarDatosPrueba.js` original y generó un archivo con
typo (`purgaUsuariosPrueba.js`, sin la "r") — se corrigió renombrando,
sin pérdida de datos.

### Reestructuración de planes (Fundación/Estándar/VIP)

Se decidió separar esto del trabajo de Escolar/Empresarial por ser
autocontenido y no depender de nada más. La fundadora dio los datos
reales de cada plan (precio, duración y cantidad de sesiones de
práctica, costo de combustible por sesión, características de VIP)
durante la conversación. Se construyó:

- Colección nueva `Plan` (reemplaza `precio_plan_normal`/`precio_plan_vip`
  sueltos en `Configuracion`) — ver DATABASE.md para el schema completo.
- `GET /api/planes`, `GET /api/planes/:codigo`, `GET /api/planes/admin/todos`,
  `PATCH /api/planes/:codigo` — ver ARQUITECTURA_BACKEND.md.
- `scripts/migrarPlanes.js` — siembra los 3 planes, reetiqueta
  inscripciones viejas.
- Home (`app/page.tsx`) e `/inscripcion` actualizados para leer de
  `/api/planes` en vez de `/api/configuracion` — 3 tarjetas dinámicas en
  vez de 2 hardcodeadas. Decisión explícita del usuario: el Home muestra
  solo resumen (precio + frase corta), el detalle completo de sesiones
  de práctica y características vive únicamente en `/inscripcion`.
- Pantalla nueva `admin/planes/page.tsx` — edición de cualquier campo de
  cada plan sin tocar código ni Atlas. Cierra un pendiente que llevaba
  abierto desde el 13/08/2026.

**Corrección post-deploy:** el primer intento de montar `planRoutes` en
`app.js` usó una ruta relativa incorrecta (`./src/routes/...` desde un
archivo que ya vive dentro de `src/`, causando un `src/src` duplicado y
`MODULE_NOT_FOUND` en el deploy de Render) — corregido a `./routes/...`.

**Ajuste de precio post-lanzamiento:** horas después de construir esto,
el usuario recordó una conversación con la fundadora que no había
trasladado antes: el plan de entrada debía llamarse "Plan Estándar" (no
"Normal") y costar RD$1,000 (no RD$1,500). Se aplicó reeditando
`migrarPlanes.js` y corriéndolo de nuevo — el diseño ya soportaba este
tipo de cambio sin tocar ni un archivo de frontend, solo datos.

### Decisión de diseño importante: campo `programa`, agregado a tiempo

A media conversación, el usuario expresó preocupación real: cada
conversación con la fundadora revela una necesidad de contenido
diferenciado por tipo de curso que no estaba planeada al inicio (primero
Escolar, luego Empresarial, y el mismo día, un cuarto: un curso para
motoristas/motorizados). Se decidió, antes de que existiera ningún
documento real de `Plan`/`Inscripcion`, agregar un campo `programa`
(`String`, default `"estandar"`, sin enum cerrado) a ambos modelos —
gratis de hacer en ese momento porque no había nada que migrar. Separa
deliberadamente "qué currículo se enseña" (`programa`) de "qué nivel de
práctica/precio dentro de ese currículo" (`tipoPlan`). Ver
ARQUITECTURA_BACKEND.md y `ESPECIFICACION_PROGRAMAS_NUEVOS.md`.

### Diseño de Escolar/Empresarial — consolidado, no construido todavía

Se simplificó el alcance varias veces durante la conversación hasta
llegar a un diseño mucho más chico que el análisis inicial (que incluía
multi-tenencia completa, RBAC nuevo, etc.). El diseño final acordado:
sin currículo/examen distinto (se reusa el existente), sin práctica de
manejo, colegios/empresas nunca entran a la app (Muvo crea las cuentas),
reciben un reporte periódico por correo. **Todo el detalle — modelo de
`Grupo`, los dos formularios (grupo + roster separado), el cuestionario
informativo de Escolar con las 14 preguntas ya redactadas, el prorrateo
contable, y el cron de reporte a las 10am — está en
`ESPECIFICACION_PROGRAMAS_NUEVOS.md` (nuevo), para no repetir el análisis
en la próxima sesión.** Motorista se mencionó pero no se diseñó a este
nivel de detalle todavía.

## 06/09/2026 — Seguimiento de práctica de manejo (choferes) + fix de dominio en tarjeta compartible

### Contexto de arranque

Dos pedidos del usuario en la misma sesión. El primero, de análisis:
hoy, cuando una estudiante termina toda la teoría, nadie se entera para
darle seguimiento en la práctica de manejo — ni un instructor recibe
sus datos, ni ella sabe qué sigue. El segundo, más grande: la
propietaria quiere explorar "Movilidad Vial Escolar" (colegios pagando
el curso para sus estudiantes). Se decidió explícitamente **dejar la
parte escolar para otra sesión** y concentrar esta en que el
seguimiento de práctica funcione bien y sin errores.

### Decisiones del usuario, antes de construir

1. El chofer se crea desde el panel de admin ("Solo fundadora") con
   nombre, celular, correo y días/horarios de práctica — esos mismos
   datos se le muestran a la estudiante para que ella lo contacte
   directamente. **Sin asignación automática** de instructor a
   estudiante.
2. La notificación al completar la teoría va al correo de cada chofer
   activo, y por separado a una lista de destinatarios dedicada solo a
   avisos de práctica — nunca mezclada con la lista administrativa ya
   existente (vouchers/balance/empresas).
3. Solo se considera "lista para práctica" a una estudiante que termine
   los 4 módulos **con sus respectivos exámenes** aprobados.
4. Se agrega un rol/dashboard de conductor donde el chofer aprueba la
   práctica de cada estudiante — esa aprobación pasa a ser, junto con
   `cursoCompletado`, requisito para poder generar el diploma.
5. El carrito animado del dashboard (`ProgresoCarretera`) debía
   extenderse para reflejar también el avance en la etapa de práctica,
   no solo teoría.

### Construido

Backend: `models/Instructor.js` y `models/DestinatarioPractica.js`
(nuevos), `User.rol` con `"conductor"` agregado, `ProgresoEstudiante`
con `practicaAprobada`/`fechaAprobacionPractica`/`practicaAprobadaPor`,
`POST /api/usuarios/conductor` (crea `User` + `Instructor` en un paso),
`GET /api/instructores` y `/activos` + `PATCH /api/instructores/:id`,
CRUD de `destinatariosPractica` (copia 1:1 del patrón existente),
`GET /api/practica/pendientes` + `POST /api/practica/:userId/aprobar`,
disparador de notificación en `entregarIntento()` (detecta la
transición `cursoCompletado` false→true para notificar una sola vez), y
el gate nuevo en `generarDiploma`/`listarElegibles` exigiendo también
`practicaAprobada`. Ver ARQUITECTURA_BACKEND.md para el detalle
completo.

Frontend: 4to valor de `Rol` (`"conductor"`) en `AuthContext.tsx` y
`RutaProtegida.tsx`, redirección nueva en `login/page.tsx`,
`dashboard/page.tsx` con la pantalla de felicitación + lista de
choferes (o "diploma en camino" si ya aprobó), `ProgresoCarretera.tsx`
reflejando `practicaAprobada` en la parada de práctica,
`admin/choferes/page.tsx` y `admin/notificaciones-practica/page.tsx`
(nuevos), y `(conductor)/practica/layout.tsx` + `page.tsx` (dashboard
del chofer). Ver ARQUITECTURA_FRONTEND.md.

### Errores encontrados y corregidos en el camino

- Tres páginas nuevas (`admin/choferes`, `admin/notificaciones-practica`,
  `(conductor)/practica`) llamaban a una función `cargar()` directo
  dentro de un `useEffect`, disparando el lint/error de React "Calling
  setState synchronously within an effect" — corregido moviendo la
  lógica de fetch inline dentro del propio efecto (con su `cancelado`),
  en vez de invocar una función externa que hace `setState`.
- La tarjeta de diploma compartible (`app/(estudiante)/diploma/page.tsx`)
  seguía apuntando al dominio viejo (`muvo-rd.vercel.app`) en el texto
  visible bajo el QR, aunque el QR en sí ya usaba `URL_INICIO` — el
  texto estaba escrito literal y aparte. Corregido: ambos (QR y texto)
  ahora salen de la misma constante `URL_INICIO`
  (`https://www.muvordvial.com`), con `DOMINIO_VISIBLE` derivado de ella
  para el texto, para que no se puedan desincronizar otra vez.

### Estado al cierre

Probado de punta a punta por el usuario: crear chofer → login como
chofer → estudiante termina teoría → notificación + lista de choferes
visible → chofer aprueba → diploma generable. Confirmado funcionando.
La parte de Movilidad Vial Escolar queda pendiente de diseño para otra
sesión (ver más abajo).

## 05/09/2026 — Test psicológico de perfil conductual, entre el pago y el acceso al contenido

### Contexto de arranque

La fundadora, en conversación directa con el usuario (fuera de esta
herramienta), pidió agregar una prueba psicológica después del pago
confirmado y antes del acceso al contenido — quería capturar datos de
experiencia previa conduciendo y perfil conductual de cada estudiante.
Se entregó como PDF un instrumento en papel ya diseñado y en uso:
"Test de Perfil Psicológico y Conductual del Conductor" (Muvo RD
Vial), con disclaimer legal propio ("no constituye diagnóstico
psicológico... si corresponde, derivación a un profesional").

### Análisis antes de construir

Se leyó el PDF completo antes de proponer nada. Hallazgo importante:
el documento tiene dos mitades con roles distintos —

- **Secciones A-H**: 54 preguntas de escala (Nunca=1...Siempre=5) en 7
  categorías (autocontrol, estrés/emociones, percepción del riesgo,
  atención/concentración, actitud/responsabilidad, confianza, presión
  social) + 5 preguntas de reflexión abierta. Esto lo llena el
  estudiante.
- **Secciones I/J/K**: indicadores de atención del evaluador, perfil
  orientativo por área, y recomendación (incluye la opción "se
  recomienda evaluación psicológica profesional externa"). Esto lo
  llena un evaluador humano con criterio profesional — no es algo que
  un formulario web autoadministrado pueda generar sin perder el
  sentido del instrumento.

Se planteó esta disyuntiva al usuario antes de construir, junto con dos
preguntas más (qué tan bloqueante debe ser, y si debe avisar
automáticamente). Decisiones del usuario:

1. **Solo digitalizar A-H** — I/J/K se queda en papel o no se hace por
   ahora.
2. **Obligatorio** — no se puede entrar a la Sesión 1 (ni a ninguna
   otra) sin completarlo.
3. **Sin aviso automático** — la coordinadora lo revisa cuando quiera
   desde el panel, no hace falta notificación push.

### Decisión de diseño propia, no pedida explícitamente pero justificada

Se decidió **no calcular ningún promedio ni puntaje por sección** en el
sistema. El instrumento mezcla a propósito preguntas en sentido
positivo y negativo (técnica de diseño profesional estándar en este
tipo de tests, para detectar respuestas inconsistentes) — promediar
los números crudos sin ese criterio daría una cifra que aparenta ser
objetiva pero no lo es, reintroduciendo por la puerta trasera
exactamente la interpretación profesional que se decidió dejar fuera
(punto 1 de arriba). La coordinadora ve las respuestas tal cual las
llenó la estudiante, sin ninguna cifra resumen inventada.

También se agregó, sin que se pidiera explícitamente pero como buena
práctica dado que es información sensible:

- Pantalla de consentimiento obligatoria antes de mostrar las
  preguntas, con el mismo texto de advertencia del documento original.
- Acceso a las respuestas restringido a coordinadora/admin — ninguna
  estudiante puede ver las de otra, ni las propias después de enviarlas.
- Aviso explícito al usuario (no resuelto por Claude, señalado como
  pendiente real) de que este tipo de dato probablemente califica como
  "sensible" bajo la Ley 172-13 de Protección de Datos de RD, y que
  conviene confirmarlo con asesoría legal antes de usarlo con
  estudiantes reales.

### Construido

Backend: `models/TestPsicologico.js` (userId único, 54 respuestas
validadas 1-5, 5 reflexiones opcionales), `controllers/testPsicologicoController.js`
(enviar una vez con 409 si se repite, estado propio sin exponer
respuestas, listado y detalle para coordinadora/admin),
`routes/testPsicologicoRoutes.js`, y el gate real en
`sesionController.js#obtenerSesionParaEstudiante` (403 con
`codigo: "TEST_PSICOLOGICO_PENDIENTE"` si falta).

Frontend: `lib/bancoPreguntasTest.ts` (las 54+5 preguntas transcritas
del PDF, única fuente de verdad para el texto), `app/test-psicologico/page.tsx`
(consentimiento → formulario → envío único), `app/dashboard/page.tsx`
actualizado (pantalla de aviso si falta completarlo, en vez de las
tarjetas de sesión), `app/(coordinadora)/panel/test-psicologico/page.tsx`
(lista + detalle expandible, sin ningún puntaje calculado), y tarjeta
de acceso nueva en `panel/page.tsx` (grupo "Gestión del curso",
visible para coordinadora y admin).

Estado al cierre: construido y entregado; el usuario confirmó que
"quedó todo muy bien" tras probarlo. Pendiente real: la confirmación
legal sobre datos sensibles (Ley 172-13), señalada arriba.

## 04/09/2026 — Automatización para la fundadora: chatbot con Gemini + resumen diario + persistencia de Empresas

### Contexto de arranque

El usuario planteó un brainstorm: la fundadora tiene poco tiempo para
estar revisando el panel seguido, ¿qué se puede automatizar? Dos ideas
concretas sobre la mesa: un chatbot solo para ella (gratis, aunque sea
con preguntas limitadas) y un resumen diario de actividad por correo y
Telegram.

Antes de construir, se analizaron opciones reales:

- **Chatbot**: se confirmó que la API de Gemini de Google sigue
  teniendo una capa gratuita real vigente en 2026 (rate-limited, no
  ilimitada — hay que crear el proyecto de Google Cloud sin activar
  facturación, o se pierde la capa gratuita). Se plantearon dos
  versiones posibles: una simple (el modelo solo resume/explica datos
  ya calculados) y una completa (function calling — el modelo puede
  consultar la base de datos con preguntas libres, de solo lectura).
  **Decisión del usuario: la versión completa.**
- **Resumen diario**: se identificó que casi toda la infraestructura ya
  existía (Resend + Telegram Bot API + patrón `DestinatarioNotificacion`)
  — lo único que faltaba resolver era el disparador, ya que Render se
  duerme en el tier free y no hay cron real ahí. Se propuso un GitHub
  Action programado como solución 100% gratuita. **Decisión del
  usuario: enviarlo al final del día** (9:00 PM hora de Santo Domingo).

### Persistencia de Empresas (requisito previo, no una idea nueva)

Al confirmar el listado de 7 herramientas del chatbot, salió que una de
ellas ("solicitudes de Empresas por rango de fecha") no se podía
construir porque ese formulario **nunca guardaba nada en Mongo** — solo
enviaba una notificación (pendiente ya anotado desde el 13/08, pospuesto
a propósito en su momento). Se resolvió como parte de este bloque:
`models/SolicitudEmpresarial.js` nuevo, `empresasController.js`
actualizado para guardar primero y notificar después (si el correo
falla, el registro ya quedó guardado). Ver DATABASE.md y
ARQUITECTURA_BACKEND.md para el detalle.

### Chatbot construido: 7 herramientas de solo lectura

`utils/geminiHerramientas.js` — `contarInscripciones`,
`contarEstudiantesActivos`, `balanceMes`, `vouchersPendientes`,
`buscarEstudiante`, `solicitudesEmpresariales`, `resultadosExamenes`.
Todas de solo lectura a propósito — el chatbot nunca puede crear, editar
ni borrar nada, así que el peor caso ante una mala interpretación es una
respuesta rara, nunca un dato perdido.

`controllers/chatbotController.js` orquesta el loop de function calling
contra `generateContent` de Gemini (fetch nativo, sin SDK, mismo estilo
que `notificaciones.js`). `routes/chatbotRoutes.js` — exclusivo `admin`.

### Turbulencia real integrando con Gemini (Google itera muy rápido)

Al probar por primera vez, el modelo usado inicialmente
(`gemini-2.5-flash`) ya no estaba disponible para cuentas nuevas —
Google lanzó la familia Gemini 3.x y recomendó migrar a
`gemini-3.6-flash` directo en el mensaje de error 404. Cambio de una
sola línea, la API `generateContent` en sí seguía funcionando igual
("legacy" pero soportada).

Segundo problema, más sutil: Gemini 3.x cambió el formato de function
calling — el rol para devolver el resultado de una herramienta pasó de
`"function"` a `"user"`, y cada `functionResponse` ahora exige el mismo
`id` que trajo la `functionCall` original (antes no hacía falta). Sin
esto, Gemini rechazaba la request con 400. Corregido tras dos rondas de
prueba con `curl` real contra el endpoint desplegado en Render.

Se agregaron reintentos automáticos (hasta 3, con espera creciente) ante
errores 503/429 — confirmado con un caso real durante las pruebas
("alta demanda", típico de la capa gratuita en picos de uso).

Verificado funcionando de punta a punta con dos pruebas reales vía
`curl`: una pregunta simple (una sola herramienta) y una que combinó dos
herramientas en la misma respuesta (comparar inscripciones de una
semana contra total de estudiantes activos) — ambas respondieron con
cifras reales y correctas.

### Resumen diario automatizado

`utils/resumenDiario.js` calcula las cifras del día (nuevas
inscripciones, pagos confirmados/rechazados, vouchers pendientes
**acumulados**, nuevos registros, diplomas, solicitudes de Empresas,
exámenes aprobados/reprobados) y arma el mensaje. `POST
/api/interno/resumen-diario` — fuera de `protegerRuta` a propósito
(quien llama es un robot), protegido por un secreto compartido
(`CRON_SECRET`) en vez de JWT.

`.github/workflows/resumen-diario.yml` — GitHub Action programado
(`cron: "0 1 * * *"` UTC = 9:00 PM AST), con `workflow_dispatch` para
poder dispararlo a mano y probar sin esperar. Verificado funcionando:
el usuario corrió el workflow manualmente y confirmó que el correo/
Telegram llegó bien.

**Incidente real al hacer el primer push:** GitHub rechazó el push del
archivo `.yml` — los Personal Access Tokens necesitan el permiso
`workflow` explícito para tocar archivos dentro de
`.github/workflows/`, algo que el token del usuario no tenía. Se
resolvió subiendo ese archivo específico directo desde la interfaz web
de GitHub (sin esa restricción) y sincronizando después con `git pull`
— con un tropiezo adicional de `git` (archivo "untracked" bloqueando el
merge) resuelto borrando la copia local duplicada antes de traer la de
GitHub. Documentado en ARQUITECTURA_BACKEND.md por si se vuelve a tocar
un archivo de Actions desde la terminal.

### Frontend: pantalla del chat + tarjeta en el panel

`app/(admin)/admin/asistente/page.tsx` — chat con burbujas, preguntas
de ejemplo, auto-scroll. Construido replicando el estilo real de
`admin/contabilidad/page.tsx` (mismas clases de Tailwind, mismo patrón
de `useAuth()`) en vez de inventar un estilo nuevo. Tarjeta de acceso
agregada a `panel/page.tsx`, grupo "Solo fundadora" (ícono `Bot`).

Ajuste de texto pedido por el usuario: la primera versión repetía
"nunca inventa datos" dos veces en la interfaz (mensaje de bienvenida +
subtítulo) — sonaba más a advertencia que a descripción de producto.
Se simplificó a solo decir qué puede consultar, sin la repetición. La
instrucción real de "no inventes cifras" se queda donde importa: en el
system prompt del backend, no en la UI.

## 28/08/2026 — SEO con dominio propio + corrección de estado real de contenido + fix de UI en aula virtual

### Contexto de arranque

Continuación directa de la sesión del 27/08 (dominio + Resend). Al
revisar Google Search Console se confirmó que ya existía trabajo de SEO
real hecho en una sesión sin documentar entre el 13/08 y el 27/08 —
sitemap, robots.txt, metadata completa con JSON-LD, y una propiedad ya
verificada en Search Console — pero todo apuntando al dominio viejo. Se
migró todo al dominio propio y, en el proceso, se destapó que el
contenido real de las 4 sesiones (que también se había cargado en esa
misma sesión sin documentar) tiene defectos serios que obligan a
recrearlo desde cero.

### SEO migrado al dominio propio

- **Google Search Console**: la propiedad vieja (`muvo-rd.vercel.app`,
  tipo "Prefijo de URL") tenía 9 páginas indexadas y un sitemap activo
  desde mayo/2026 — confirmado con capturas reales, no se perdió nada,
  solo quedó desactualizada. Se creó una propiedad nueva tipo
  **"Dominio"** para `muvordvial.com`, verificada por DNS (registro TXT
  agregado en Vercel → DNS Records, mismo lugar que Resend) —
  verificó al primer intento.
- **`app/sitemap.ts`** y **`app/robots.ts`**: ambos tenían `SITE_URL`
  quemado al dominio viejo — corregidos a `https://www.muvordvial.com`.
  Además, `robots.ts` **bloqueaba `/inscripcion` por error** en el
  `disallow` (página pública de marketing, no debía estar ahí) — el
  usuario confirmó que fue un descuido, no intencional, y se quitó.
- **`app/layout.tsx`**: metadata SEO ya bastante completa desde antes
  (título, descripción con lenguaje de búsqueda real, Open Graph,
  Twitter Card, JSON-LD Schema.org `EducationalOrganization`) —
  mismo problema de `SITE_URL` quemado, corregido. Se evaluó agregar el
  código de verificación de Google como método de respaldo, pero
  propiedades tipo "Dominio" en Search Console solo ofrecen
  verificación por DNS, no por etiqueta HTML — no aplicaba, se descartó
  ese paso.
- Sitemap reenviado a la propiedad nueva (Correcto, 10 páginas) y se
  solicitó indexación manual de home, `/empresas` y `/registro`.
- Se evaluó agregar "escuela de choferes" como palabra clave — descartado
  por desalineado con el producto real (sugiere formación profesional,
  no un curso para principiantes); también se aclaró que la etiqueta
  meta `keywords` no tiene efecto real en Google desde 2009, así que el
  trabajo de SEO real está en el contenido/títulos/descripciones, no en
  una lista de palabras oculta.

Ver ARQUITECTURA_FRONTEND.md, sección "SEO real", para el detalle
técnico completo.

### Corrección de estado real de contenido (destapado, no resuelto)

Al revisar `app/page.tsx` para el trabajo de SEO, salieron a la luz
varias cosas hechas en la sesión sin documentar (comentarios internos
fechados 13/08 y 16/08/2026 en el código):

- Los 4 temas reales del curso sí se definieron y se usan como copy en
  el home ("Bienvenida y Cultura Vial", "Marco Legal y Señalización",
  "El Vehículo: Mecánica y Seguridad", "Técnicas de Conducción").
- Se cargó contenido real en `ContenidoSesion` con títulos reales por
  material (ej. "1.1 Bienvenida a Muvo RD Vial") — confirmado con
  captura real del aula virtual.
- Se crearon las 4 versiones de `Examen`.
- Se agregó una sección de promoción del libro de la fundadora en el
  home (portada, cita, links reales a Amazon física/Kindle), con dos
  clases de color nuevas (`brand-yellow`, `brand-mamey`) no
  documentadas en los tokens de Tailwind.

**Pero al confirmar con el usuario, salieron dos bugs serios que
invalidan ese contenido tal como está:**

1. **PDFs con errores de codificación** — texto con símbolos y marcas
   extrañas, no presentable a una estudiante real.
2. **Examen con la respuesta correcta siempre en la opción A**, en
   todas las preguntas — patrón predecible y explotable, probablemente
   porque el proceso que generó las preguntas no aleatorizó el orden de
   las opciones.

**Decisión del usuario:** borrar todo (`ContenidoSesion` + `Examen`)
vía `curl` contra los endpoints reales (no a mano en Atlas, para no
romper referencias) y recrear desde cero — PDFs corregidos, opciones de
examen aleatorizadas. Esto queda como pendiente de **alta prioridad**,
no resuelto — ver la lista de pendientes al final de este documento.

También se confirmó, revisando una captura real del aula virtual, que
**`Sesion.titulo` sigue en "Sesión 1"..."Sesión 4"** — los títulos
reales solo llegaron a los materiales individuales (`ContenidoSesion`),
no al nivel de la sesión misma. Ese rename sigue pendiente.

### Fix de UI: contenido de aula virtual desordenado según el tipo

El usuario reportó que el botón "Marcar como visto" se veía mal
alineado cuando el contenido era `pdf` o `enlace` (pegado justo al lado
del link "Abrir PDF ↗"/"Abrir enlace ↗"), mientras que en `video` y
`texto` se veía bien (en su propia línea, aunque no centrado).

Causa raíz identificada leyendo `aula-virtual/[sesion]/page.tsx`: los
bloques de `video` y `texto` envuelven su contenido en un `<div>` de
bloque, lo que empuja el botón siguiente a su propia línea de forma
natural. Los bloques de `pdf` y `enlace` eran un simple `<a
className="inline-block">` — al ser inline, el navegador lo colocaba en
la misma línea que el botón si había espacio.

Fix aplicado, confirmado probado en producción con los 4 tipos:

- `pdf` y `enlace` pasaron de link de texto suelto a una tarjeta de
  bloque completo (borde, fondo `bg-neutral-bg`, centrado), mismo peso
  visual que el recuadro de video.
- El botón "Marcar como visto" ahora está centrado (`flex
justify-center`) para los 4 tipos por igual, no solo alineado a la
  izquierda.

Cambio 100% visual — no se tocó lógica de `marcarVisto()`,
`intentarDesbloquear`, ni ningún endpoint.

## 27/08/2026 — Dominio propio en Vercel + Resend, envío de correos desbloqueado

### Contexto de arranque

La fundadora compró `muvordvial.com` directo en su cuenta de Vercel
(con otra tarjeta, en una sesión anterior no registrada formalmente).
Esta sesión fue 100% de configuración de infraestructura, sin tocar
lógica de negocio nueva — el objetivo era sacarle provecho real a esa
compra: conectar el dominio al frontend y desbloquear el envío de
correos reales vía Resend (Prioridad #1 desde el 25-26/07/2026).

### Conexión del dominio en Vercel

En Settings → Domains del proyecto frontend se agregaron `muvordvial.com`
y `www.muvordvial.com`. Vercel los configuró con `www.muvordvial.com`
como el que sirve Production y `muvordvial.com` (sin www) redirigiendo
308 hacia la versión con `www` — al revés de lo que se había sugerido
inicialmente (se había recomendado que la versión sin `www` fuera la
principal, por más corta), pero el usuario confirmó explícitamente que
prefiere quedarse con `www.muvordvial.com` como URL principal, así que
se mantuvo tal como Vercel lo dejó por defecto. El certificado SSL se
generó solo, sin intervención manual. `muvo-rd.vercel.app` se dejó
activo a propósito como fallback.

### Dominio verificado en Resend

En Resend → Domains → Add domain se usó la opción **"Auto configure"**
(en vez de copiar registros DNS a mano) porque Resend detecta que el
dominio vive en Vercel y puede escribir los registros él mismo vía
OAuth — evitó por completo el paso manual de copiar/pegar DKIM, SPF y
MX que se había planeado originalmente. Verificado en menos de 5
minutos (Domain added → DNS verified → Domain verified).

### Variables de entorno y código actualizados

- Render → `mav-rd-backend` → Environment:
  - `RESEND_FROM`: `onboarding@resend.dev` → `Muvo RD Vial <hola@muvordvial.com>`.
  - `FRONTEND_URL`: `https://muvo-rd.vercel.app` → `https://www.muvordvial.com`.
- `app.js`: `origenesPermitidos` dejó de depender de una sola variable
  (`process.env.FRONTEND_URL`) y pasó a una lista fija con los 4
  orígenes válidos (`localhost:3000`, `www.muvordvial.com`,
  `muvordvial.com`, `muvo-rd.vercel.app`) — para no perder acceso desde
  el dominio de Vercel ahora que `FRONTEND_URL` apunta al dominio
  propio. Commit + push a `main`, redeploy automático en Render.

### Pruebas realizadas en producción

1. Registro de cuenta nueva en `www.muvordvial.com/registro` → correo de
   verificación llegó con remitente `hola@muvordvial.com` y la
   plantilla visual completa (logo, colores, botón) — confirmado con
   captura de Gmail.
2. Formulario de `/empresas` → correo de notificación llegó
   correctamente formateado. **Nota:** llegó a la cuenta personal de
   pruebas del usuario, porque es el único registro `activo: true` en
   `DestinatarioNotificacion` hoy — no es un bug, solo falta que la
   fundadora decida/confirme qué correo institucional real debe
   recibir estos avisos.
3. Consola del navegador en `www.muvordvial.com` revisada (Network +
   Issues) → sin errores de CORS ni peticiones bloqueadas. El único
   "Issue" reportado por Chrome fue un aviso cosmético de accesibilidad
   (campo de formulario sin `id`/`name`), no relacionado.

### Estado al cierre

Prioridad #1 (dominio propio en Resend) queda **resuelta**. El correo
transaccional real ya funciona de punta a punta en producción. Pendiente
real: decidir el destinatario correcto de las notificaciones internas
(ver Pendiente real, backend) y Telegram para el celular de la
fundadora, que sigue como canal de respaldo sin terminar.

## 13/08/2026 — PDF de material de estudio, sección de Planes en el home, programa Empresas

### Contexto de arranque de la sesión

Se reinició la conversación desde cero (memoria del proyecto vive en
estos 3 archivos + los modelos/controllers reales). Se aclaró que
`scripts/crearSesionesIniciales.js --confirmar` ya se había ejecutado en
una sesión anterior pero nunca quedó registrado — corregido en
ARQUITECTURA_BACKEND.md y DATABASE.md.

### Material de estudio: PDF como archivo real, no solo URL pegada

Pedido: la coordinadora quería poder subir PDFs de verdad (no solo pegar
un link externo) para el material de estudio, con miras a eventualmente
tener texto enriquecido con imágenes también (esto último se identificó
como un pedido aparte, más grande, y se dejó fuera de este bloque).

Se revisó primero el patrón ya existente para PDFs en el proyecto: los
diplomas ya subían PDFs a Cloudinary como `resourceType: "raw"` y los
entregaban con una URL firmada generada al momento
(`generarUrlDescargaFirmada`, porque Cloudinary bloquea la entrega
pública de recursos `raw`). Se replicó exactamente ese patrón para
`ContenidoSesion` en vez de inventar uno nuevo:

- `middleware/upload.js`: se separó en `uploadImagen` (sin cambios de
  comportamiento) y `uploadPDF` (nuevo, 15MB, `application/pdf`).
- `POST /api/uploads/pdf` (nuevo, coordinadora/admin) sube el buffer a
  `mav-rd/contenido-sesion` y devuelve `{ url, publicId }`.
- `ContenidoSesion` ganó el campo `publicIdCloudinary` (opcional — solo
  se llena si el pdf se subió como archivo, no si se pegó una URL
  externa a mano).
- `GET /api/contenido-sesion/:id/archivo` (nuevo): genera la URL firmada
  al momento y sirve el PDF inline, verificando el token manualmente
  (header o `?token=`, mismo patrón que `diplomaController.js`) porque un
  `<a href>` de descarga no manda headers. A diferencia del diploma, sí
  valida que la sesión esté desbloqueada para la estudiante antes de
  entregarle el archivo.
- Frontend: `panel/aula-virtual/page.tsx` gana un selector de archivo
  para PDF (antes cualquier tipo no-video/no-texto caía en un `<input
type="text">` genérico); `aula-virtual/[sesion]/page.tsx` arma el link
  al endpoint firmado cuando hay `publicIdCloudinary`, con fallback a
  `url` directo para no romper contenido viejo.

Complicación durante la sesión: el usuario subió dos veces un archivo
llamado `page.tsx` (el de la vista de estudiante y luego el del panel de
coordinadora), y el segundo sobrescribió al primero en disco antes de
poder leerlo completo — hubo que pedir que lo resubiera con otro nombre
para poder devolver el archivo completo sin inventar el tramo que
faltaba (la lógica de la cuenta regresiva `disponibleEn`/`tiempoRestanteMs`).
Lección para sesiones futuras: pedir nombres de archivo distintos cuando
se suben varios `page.tsx` en la misma tanda.

Estado real al cierre: el flujo completo se probó en producción (deploy
en Render + Vercel) y funciona — **no se cargó contenido real todavía**,
solo se confirmó que subir/guardar/abrir un PDF funciona de punta a
punta.

### Análisis de la competencia (academiavial.com) y aclaración de la estructura de planes

El usuario pidió analizar academiavial.com como referencia de una
página de inicio más "comercial" (animaciones, precios visibles desde el
home, sección empresarial). Se navegó el sitio real (home + página de
servicios empresariales) antes de opinar.

Se identificó qué vale la pena adoptar vs. qué no, dado el stack real
(Next.js/Tailwind, sin el builder de animaciones que trae WordPress/
Elementor) y la restricción de infraestructura gratuita:

- **Sí adoptar:** precio visible desde el inicio (sin forzar a entrar a
  `/inscripcion`), sección de "por qué elegirnos", y una página
  empresarial — encaja con la misión de la fundación, es contenido +
  formulario, no requiere lógica nueva compleja.
- **No adoptar:** su estructura de múltiples "programas" con precios
  distintos (Muvo vende un solo curso, no un catálogo — forzar esa
  estructura habría sido inventar complejidad que no existe), ni el
  carrusel decorativo del hero (mucho esfuerzo visual para un solo
  curso).
- Animaciones: se sugirió `framer-motion` como opción de bajo costo si
  se quiere ir en esa dirección más adelante — no se implementó en este
  bloque, quedó fuera de alcance.

A mitad del análisis, el usuario aclaró un punto de negocio que no
estaba bien reflejado en el sitio: no son varios cursos, es **un solo
curso teórico con dos variantes de práctica** (Normal y VIP — la
diferencia es personalización/tiempo con el instructor, no contenido
teórico distinto). Esto ya vivía correctamente en el backend
(`Inscripcion.tipoPlan`), solo faltaba comunicarlo bien en el home.

### Sección de Planes y Precios en el home

`app/page.tsx` se convirtió en un componente servidor async que hace
`fetch` a `GET /api/configuracion` (`cache: "no-store"`) y muestra dos
tarjetas (Normal/VIP) con precio real, características, y CTA hacia
`/registro`. El copy se iteró varias veces en vivo con el usuario hasta
llegar a una versión explícitamente más comercial ("Sal manejando con
confianza. Tú eliges cómo llegar ahí...") en vez de solo descriptiva —
incluyó dos correcciones de redacción en español (tilde en "más", y
mayúscula indebida después de un guión largo).

Se agregó también un banner hacia `/empresas` entre Testimonios y el CTA
final del home.

### Programa Empresarial (`/empresas`) — primera versión

Se preguntó explícitamente qué tan lejos llegar en esta primera versión,
dando dos opciones: informativa (tabla/contenido + formulario, sin
cambios de backend más allá de un endpoint de envío) vs. inscripción
real con la empresa como unidad de pago. El usuario eligió la opción
informativa — sin precios fijos en el sitio, cotización por correo según
cantidad de estudiantes, curso completo (teoría + práctica) igual que el
individual.

Construido:

- `app/empresas/page.tsx` (nuevo): hero, 4 beneficios con ícono, "cómo
  funciona" en 3 pasos, formulario (`nombreEmpresa`, `contacto`, `cargo`
  opcional, `telefono`, `email`, `cantidadEstudiantes`, `mensaje`
  opcional).
- `POST /api/empresas/contacto` (nuevo, público): `controllers/empresasController.js`
  - `routes/empresasRoutes.js`, montado en `app.js`. Reutiliza
    `DestinatarioNotificacion` vía una función nueva en
    `utils/notificaciones.js` (`enviarSolicitudEmpresarial`) — mismo canal
    de correo/Telegram institucional que ya usan los avisos de voucher y
    balance pendiente, sin configuración nueva.
- **Decisión explícita:** no se persiste el lead en Mongo en esta
  primera versión — si Resend falla (sigue bloqueado por el dominio
  pendiente) o el mensaje se pierde, no queda registro. Anotado como
  pendiente a evaluar, no bloqueante para esta primera versión.
- `components/layout/Navbar.tsx`: se agregó el link "Empresas" al array
  `enlaces` compartido entre el menú de escritorio y el móvil.

Detalle de nomenclatura durante la sesión: el archivo de rutas se llamó
primero `routes/empresas.js`, pero al ver el `app.js` real del usuario
se confirmó que la convención del proyecto es `algoRoutes.js`
(`uploadRoutes.js`, `inscripcionRoutes.js`, etc.) — se corrigió a
`routes/empresasRoutes.js` antes de que el usuario lo desplegara.

### Pendiente real dejado para la próxima sesión

- Cargar contenido real (PDFs, videos, texto) en las 4 sesiones — el
  flujo técnico ya está listo.
- Evaluar si el formulario de Empresas necesita persistencia en Mongo.
- Construir una UI de admin para editar `configuracion` (precios) —
  ahora más visible al estar también en el home público.
- Texto enriquecido con imágenes en `contenidoTexto` — pedido
  identificado, no empezado.
- Animaciones más elaboradas en el home (framer-motion) — sugerido, no
  implementado.
- Unificar convención de nombres de color Tailwind (`brand-blue-light`
  vs `brand-blueLight`), inconsistente entre archivos — se detectó al
  editar el home, sin urgencia.

## 06-07/08/2026 — Diploma compartible, ampliación a 4 sesiones, audiencia inclusiva, purga de datos de prueba

### Diploma compartible en redes sociales — construido de principio a fin

Empezó como el pendiente #1 de la sesión anterior ("diseñado, sin
construir"). Primera versión: ícono de volante dibujado en canvas,
degradado azul/rosa, código de diploma visible. Feedback del usuario tras
verla: no se parecía a la marca real, faltaba el logo, y no había ningún
link para que contactos interesados llegaran a la app.

Se pidió el logo real (`public/logo-mav-rd.png` — azul marino `#08244B`,
dorado `#F8CB1A`, rojo `#D11523`) y se iteró el diseño con mockups en
vivo antes de tocar código (usando el visualizador de la conversación,
no archivos reales) hasta cerrar: foto real de fondo (buscada y elegida
de Unsplash, licencia libre — `person driving car during daytime` de
Stephan Mahlke — descargada a `public/diploma-compartir.jpg` porque el
`<canvas>` necesita imágenes del mismo origen o `toBlob()` falla en
silencio por contaminación CORS), logo real superpuesto, sin código de
diploma en la imagen (se decidió que ese dato se quede solo en la
tarjeta del PDF), mensaje motivador enmarcado como oportunidad, y un
bloque con QR + link a la página de inicio. Se agregó la dependencia
`qrcode` (+ `@types/qrcode`) para generar el QR 100% en el navegador.

Colores: se probó una paleta tomada directo del logo (azul marino/
dorado/rojo) pero se descartó — el degradado azul/rosa de marca que ya
se había mostrado antes gustó más, así que se mantuvo para el fondo,
reservando los colores del logo solo para el logo mismo.

Pendiente: agregar una sección "Lo que aprendiste" con los temas reales
del curso — se dejó fuera porque los 4 temas todavía no están definidos
(ver más abajo).

### Ampliación de 3 a 4 sesiones

El usuario preguntó qué tan difícil sería este cambio y dónde afectaría,
antes de tocar nada — se hizo un análisis archivo por archivo (10 en
total, backend y frontend) antes de escribir código.

Hallazgo principal: casi toda la lógica de negocio (`intentarDesbloquear`
en `examenController.js`, `ContenidoSesion`, `ProgresoEstudiante`,
`contenidoSesionController.js`, rutas dinámicas `aula-virtual/[sesion]` y
`examen/[intentoId]`) ya estaba escrita de forma genérica, sin el número
3 quemado. Los cambios reales quedaron acotados a 4 lugares:

1. `models/Sesion.js`: `numero` tenía `max: 3` a nivel de esquema de
   Mongoose — el verdadero candado del sistema, hubiera rechazado
   cualquier intento de crear una Sesión 4 sin importar que el resto del
   código ya generalizara bien. Cambiado a `max: 4`.
2. `intentoExamenController.js`, dentro de `entregarIntento`: tres
   números `3` quemados (avance de teoría, corte de "próxima sesión con
   espera", umbral de `cursoCompletado`) — cambiados a `4`.
3. `components/dashboard/ProgresoCarretera.tsx`: rediseño de 6 a 7
   paradas (se agregó Sesión 4), posiciones recalculadas manteniendo
   Inicio y Diploma en los mismos extremos para no tocar la carretera
   base; mismo patrón de "salto visual" que ya existía con la Sesión 3
   (el carrito no pausa en la última sesión de teoría, salta directo a
   Práctica).
4. `app/dashboard/page.tsx`: `SESIONES = [1,2,3,4]` + corrección de copy.

### Audiencia ampliada: el curso ya no es solo para mujeres

Decisión de negocio comunicada a mitad de la sesión: el curso pasa a
incluir adolescentes de ambos sexos, además de mujeres. Se aplicó
lenguaje neutral a todo lo que se tocó de aquí en adelante (instrucción
explícita del usuario: cambiar donde se toque, no hacer una pasada
retroactiva de todo el sitio en este momento).

Archivos corregidos: `app/page.tsx` (inicio — nombre actualizado a "Muvo
RD Vial", título principal y CTA sin adjetivos de género, "tres
sesiones" generalizado a "en sesiones" para no quedar desactualizado),
`kit-preparacion/page.tsx` (metadata title + "ya estarás lista" → "ya
tendrás lo necesario"), `inscripcion/page.tsx` ("embajadoras" →
"embajadores", coincidiendo con el logo real; "otras estudiantes" →
"tus compañeros de curso"; "el chofer" → "quien te instruye"). Los
testimonios existentes (Rosa M., Yolanda P.) se dejaron intactos a
propósito — son citas reales, reescribirlas cambiaría lo que esas
personas realmente dijeron.

Confirmado sin cambios necesarios: `aula-virtual/[sesion]/page.tsx`,
`examen/[intentoId]/page.tsx`, `faq/page.tsx`.

Pendiente: `testimonios/page.tsx`, `registro/page.tsx` (sin revisar
todavía), y las fotos de `public/inscripcion/` (probablemente muestran
solo mujeres, de clases anteriores al cambio de audiencia — reemplazo es
trabajo de contenido, no de código).

### Purga completa de datos de prueba

Ya estaba planeada de antes ("pospuesta hasta que la app esté lista para
producción", con instrucción de hacerse por terminal, nunca desde la
UI). Se aprovechó este momento porque, con el cambio a 4 sesiones, había
que reorganizar y recargar contenido real de todas formas.

Se construyó `scripts/purgarDatosPrueba.js` (dry-run por defecto,
requiere `--confirmar` + escribir `BORRAR` a mano) en vez de un
`deleteMany` genérico. Alcance confirmado con el usuario antes de
escribirlo: sobrevive únicamente `maria@test.com` (admin); se borran
todos los demás usuarios (estudiante y coordinadora, sin excepción),
todas las `Sesion` (con sus exámenes, para reconstruir desde cero según
el material real que hay que clasificar), y en cascada `Examen`,
`ContenidoSesion`, `IntentoExamen`, `ProgresoEstudiante`, `Inscripcion`,
`Diploma`.

Corrido en modo real el 06/08/2026. Conteos purgados: 17 usuarios, 3
sesiones, 15 exámenes, 22 contenidos de sesión, 30 intentos de examen,
11 progresos de estudiante, 13 inscripciones, 6 diplomas — ver
DATABASE.md para el detalle completo.

### Hueco descubierto: no existe forma de crear sesiones desde el panel

Al purgar y quedarse con 0 sesiones, se descubrió que
`sesionController.js` nunca tuvo un endpoint `POST /sesiones` — solo
`GET` (listar), `GET /:numero` (para la estudiante) y `PATCH /:numero`
(editar una que ya existe). Las 3 sesiones originales se habían creado
directo en Atlas, a mano. Esto explicaba tres síntomas reportados por el
usuario a la vez: "Sesión no encontrada" al entrar como estudiante nueva,
el botón de "agregar contenido" que no aparecía en el panel, y el examen
que "se creaba" pero sin dónde asignarlo (en realidad nunca se guardaba:
`sesionId` quedaba `null` porque no había pestañas de sesión que
seleccionar, y el mensaje de validación del formulario era engañoso —
decía "completa los campos" incluso con todo lleno).

Solución adoptada: `scripts/crearSesionesIniciales.js` (mismo patrón
dry-run + `--confirmar`) en vez de construir el endpoint — es una
operación de una sola vez. Creado pero **todavía sin ejecutar** al cierre
de esta sesión de trabajo.

También quedó anotado, pero pospuesto a propósito: el panel de
coordinadora tampoco tiene un formulario para _renombrar_ una sesión
(el backend sí lo permite vía `PATCH /sesiones/:numero`, el frontend no
tiene pantalla que lo use) — se retoma cuando haga falta.

### Pendiente real, en orden, para la próxima sesión

1. Correr `scripts/crearSesionesIniciales.js --confirmar`.
2. Definir los 4 temas del curso con la fundadora.
3. Clasificar y organizar el material real disponible por tema.
4. Crear las 4 `Sesion` con títulos finales (renombrar las provisionales).
5. Subir `ContenidoSesion` y crear versiones de `Examen` para cada una.
6. Probar de punta a punta con una cuenta de estudiante nueva.
7. Agregar la sección "Lo que aprendiste" al diploma compartible una vez
   estén los temas.
8. Revisar lenguaje de género pendiente en `testimonios` y `registro`.

## Corrección pendiente de documentación (detectada, no resuelta)

Al pasar el archivo de diploma en la sesión del 04/08/2026 se reveló que
vive en `app/(estudiante)/diploma/page.tsx`, no en `app/diploma/page.tsx`
como estaba documentado (ver ARQUITECTURA_FRONTEND.md, sección de
estructura de carpetas). Existe un grupo de ruta `(estudiante)` no
documentado hasta ahora. **Pendiente confirmar** si `dashboard`,
`aula-virtual`, `examen`, `inscripcion` y `perfil/cambiar-password`
también viven bajo ese mismo grupo o si `diploma` es la excepción —
corregir ARQUITECTURA_FRONTEND.md con la estructura real la próxima vez
que se toque cualquiera de esas páginas.

## 04/08/2026 — Cierre del anexo: exámenes/contenido desactivables + pestañas de estudiantes

- Se cerró el anexo de continuidad que había quedado pendiente de una
  sesión anterior (purga de usuarios de prueba + soft delete/versionado +
  panel de estudiantes menos cargado con el tiempo). Decisión tomada por
  Ramon: la pestaña **Graduadas** se queda **separada** de **Inactivas**
  (se descartó la idea de una pestaña "Historial" combinada).
- Confirmado por revisión de código real (no había que asumir nada):
  `Examen` y `ContenidoSesion` **ya tenían** soporte completo de
  `activo`/soft delete desde antes de esta sesión — no hizo falta tocar
  ningún modelo ni controlador de esos dos. `eliminarExamen` y
  `eliminarContenido` ya eran soft delete puro (nunca borran físico), y
  ambos listados ya separaban lo que ve la estudiante (`activo: true`) de
  lo que administra el panel (todo, incluidos inactivos).
- **Backend nuevo — `GET /api/diplomas`** (`listarTodos` en
  `diplomaController.js`): coordinadora/admin, devuelve todos los diplomas
  generados. Mismo patrón que ya se usaba con `GET /inscripciones` para
  cruzar estados en el frontend.
- **Backend — `usuarioController.js`:** `listarUsuarios` gana el query
  param `conDiploma` (true/false), que filtra a nivel de base de datos
  cruzando contra la colección `Diploma` — necesario para que la
  paginación (`totalPaginas`/`totalDocumentos`) salga exacta en cada
  pestaña del panel, en vez de resolverse en el frontend después de traer
  los datos.
- **Frontend — `panel/estudiantes/page.tsx` rediseñado** con 3 pestañas
  (Activas / Graduadas / Inactivas) y botón de Archivar/Reactivar cuenta
  en el detalle de cada estudiante (usa el `PATCH /usuarios/:id/estado`
  que ya existía desde antes, sin cambios de backend para eso).
- **Fix de React (2 rondas):** el patrón de carga inicial disparaba el
  warning nuevo `react-hooks/set-state-in-effect`. Se resolvió moviendo
  `cargarLista` a `useCallback([token])` (identidad estable entre
  renders) y dejando que el `useEffect` de carga dependa de
  `[token, pestana, cargarLista]` directamente, sin resetear estado
  (`setPagina`) de forma manual dentro del cuerpo del efecto — el reseteo
  de página/búsqueda ahora vive en `cambiarPestana()`, que es un
  manejador de evento, no un efecto.
- Subido a Render (backend) y Vercel (frontend) y probado — funcionando.
- El anexo de continuidad que traía todo este análisis ya puede borrarse
  — todo lo aplicable quedó integrado aquí y en ARQUITECTURA_BACKEND.md /
  ARQUITECTURA_FRONTEND.md.
- **Corrección de un error propio:** en esta misma sesión Claude había
  listado por error "badge de pendientes por verificar en la tarjeta de
  Pagos" como pendiente — ya estaba resuelto desde el 26/07/2026 (ver esa
  entrada más abajo), fue un dato tomado de una lista de pendientes vieja
  del propio historial que ya había sido reemplazada. Corregido, sin
  impacto en código.
- **Brainstorm de la próxima funcionalidad grande:** se evaluaron 4 ideas
  (diploma compartible, recordatorio de examen disponible, pedir
  testimonio automático al graduarse, recordatorio de pago
  pendiente/rechazado). Decisión de Ramon: diploma compartible es la
  prioridad inmediata (ver sección de arriba, "Próxima sesión"). Los
  recordatorios de examen disponible y de pago pendiente/rechazado
  interesan pero **solo por correo** (no Telegram, porque no se hace
  activación de Telegram por estudiante) — quedan como ideas a futuro,
  sin diseñar todavía, con una duda de diseño abierta y sin resolver: sin
  cron real (Render se duerme en el tier free), falta decidir un
  disparador confiable para ambos, ya que a diferencia del recordatorio
  de balance mensual (que un admin dispara sin querer al abrir el panel
  seguido), nadie abre la app justo cuando se cumple el plazo de una
  estudiante específica. Pedir testimonio automático se descartó — solo
  se necesitan algunos testimonios, no automatizar la captura.

## 03/08/2026 — Análisis de funcionalidades futuras

- Sesión de brainstorm sobre hacia dónde crecer la app, en tres direcciones
  posibles: escuela virtual genérica (catálogo de cursos, no solo manejo),
  escuela de choferes (cerrar la brecha de la práctica, que hoy vive 100%
  fuera del sistema — sin agenda, sin instructor/vehículo asignado, sin
  checklist), y página de contenido (categorías, buscador, newsletter,
  "me gusta" por comentario). Se sumó una cuarta idea no planteada
  originalmente: página de donaciones, reusando el patrón de pago con
  voucher que ya existe para inscripciones.
- Idea del foro/preguntas por sesión: descartada por ahora a propósito —
  la app aún no entra en producción con estudiantes reales, y no se sabe
  todavía si las alumnas van a interactuar con contenido más allá del
  curso. Se deja para una segunda etapa, después de validar con las
  primeras estudiantes reales.
- De las cuatro direcciones, la brecha de la práctica (escuela de
  choferes) quedó identificada como la más natural de cerrar a futuro —
  es la única parte del flujo completo "aprender a manejar" que hoy no
  vive en el sistema. No se empezó a construir nada de esto todavía, solo
  quedó el análisis.

## 01-03/08/2026 — Rediseño de ProgresoCarretera + cierre de pendientes cortos

- **Barra de progreso**: rediseño completo de
  components/dashboard/ProgresoCarretera.tsx — antes era estática, ahora
  tiene el tramo recorrido de la carretera pintado de color (el camino
  mismo muestra el avance), el carrito se desliza con transición suave en
  vez de saltar, mensaje motivacional debajo del contador que cambia según
  la etapa, y un destello de celebración cuando el diploma queda listo
  (único momento llamativo del componente, a propósito). Todo respeta
  prefers-reduced-motion. dashboard/page.tsx también ganó íconos por
  estado en las tarjetas de sesión y una entrada escalonada al cargar.
  Antes de construirlo se probó una demo interactiva simplificada fuera
  del código real, para validar la ubicación y el tono de los mensajes
  antes de tocar el componente de verdad.
- **Cuenta bancaria en inscripción**: confirmada correcta (Banco Popular
  Dominicano y Banco De Reservas). Se le agregó un botón "Copiar" por
  cuenta en app/inscripcion/page.tsx, con feedback visual de "Copiado".
  Se quitó el comentario TODO viejo, ya resuelto.
- **FRONTEND_URL en Render**: confirmado apuntando a
  https://muvo-rd.vercel.app.
- **Token del bot de Telegram**: regenerado (el original había quedado
  expuesto en texto plano durante la configuración) y actualizado en
  Render.
- Se evaluó automatizar un recordatorio de backup desde el panel de admin
  (mismo patrón que el de balance mensual) — se decidió no hacerlo, el
  backup se queda 100% manual (ver también la entrada del 31/07/2026).

## 31/07/2026 — Backup manual redundante (Docker local + Dropbox cifrado)

- Se armó un mecanismo de backup manual, fuera de Atlas por completo, para
  cubrir que el cluster es M0 (gratis) y Atlas no ofrece ningún backup
  automático en ese tier.
- Dos scripts, `backup-config.bat` (secretos, nunca se sube a git) y
  `backup-muvo.bat`, viven localmente en la PC de Ramon (Windows), fuera
  del repo, y se corren manualmente con doble clic cuando Ramon lo decide
  (ej. al recibir un aviso de Telegram de un voucher nuevo).
- Flujo del script: `mongodump` desde Atlas con un usuario de Atlas de
  solo lectura (`backup_readonly`, rol `readAnyDatabase@admin` — funciona
  pero es más amplio de lo necesario ya que el cluster es compartido;
  quedó pendiente afinarlo a un rol Read específico sobre `mav_rd`) →
  restaura a un MongoDB local en Docker (`mongodb://localhost:27018/mav_rd`,
  contenedor `mavrd-backup-db`, red `mavrd-backup-net`) → cifra el dump
  con 7-Zip y contraseña → copia el `.7z` cifrado a una carpeta local
  sincronizada con Dropbox → limpia backups locales viejos dejando los
  últimos 5.
- Imagen de Docker fijada a `mongo:8.0` (no `mongo:7` ni `mongo:8`
  genérico) para que coincida con la versión real de Atlas (`8.0.29`) y
  evitar advertencias de cross-version restore que podrían corromper la
  restauración. **Revisar este tag si Atlas sube de versión mayor.**
- Probado de punta a punta: dump real, restore de 179 documentos sin
  fallos, cifrado y copia a Dropbox exitosos.
- Decisión tomada: se queda 100% manual, sin botón ni automatización desde
  el panel de admin. Se evaluó un recordatorio automático (mismo patrón
  que el de balance mensual: marcador en Configuracion + aviso por
  Telegram al abrir la app) y se descartó por ahora a propósito — Render
  no puede ejecutar nada en la PC de Ramon de todas formas, así que la
  automatización real solo hubiera cubierto el recordatorio, no el backup
  en sí.

## 26/07/2026 — Barra de progreso ilustrada + fix de detección de diploma

- Nueva components/dashboard/ProgresoCarretera.tsx: camino horizontal con
  asfalto negro, línea central intermitente + zona de no rebasar, línea de
  arrancada, libros con check por examen aprobado, parada de práctica
  (siempre neutra, no se rastrea), y bandera de meta que se pinta de color
  cuando el diploma ya existe.
- Se exploraron 3 conceptos visuales distintos antes de decidir (autopista
  horizontal, camino serpenteante vertical, y la versión final horizontal
  compacta con detalles reales de carretera).
- Fix real encontrado por Ramon: el carrito se quedaba parado en "Práctica"
  aunque el diploma ya estuviera generado, porque dashboard/page.tsx nunca
  consultaba GET /diplomas/me. Se corrigió: ahora, cuando
  progreso.cursoCompletado es true, también se pregunta por el diploma y se
  le pasa ese dato al componente (diplomaListo).

## 25-26/07/2026 — Sistema de notificaciones (email + Telegram) + verificación

- Nueva colección/CRUD destinatariosNotificacion (admin), con avisos por
  Resend y Telegram Bot API cuando llega un voucher nuevo.
- Plantilla de correo compartida con logo + colores de marca en
  utils/notificaciones.js, extendida a 5 correos distintos: verificación de
  cuenta, pago confirmado, pago rechazado (con motivo), diploma listo, y
  recuperación de contraseña.
- Verificación de email al registrarse (emailVerificado, con link válido
  24h) — no bloquea el login, solo bloquea POST /inscripciones/mia.
- Recuperación de contraseña completa (olvide-password / restablecer-password).
- Rediseño de generarBalancePDF() (tarjetas de totales + tabla de
  categorías) y fix de la descarga sin extensión .pdf en Contabilidad,
  aplicando el mismo patrón que ya funcionaba en diplomas.
- Recordatorio automático de balance mensual pendiente: sin cron (Render se
  duerme en el tier free), se revisa cada vez que un admin abre la app
  (GET /auth/perfil), con un marcador en Configuracion para no repetir el
  aviso sobre el mismo mes.
- **Bug de build en Vercel:** app/verificar-email/page.tsx usaba
  useSearchParams() sin <Suspense>, lo que hace fallar next build al
  pre-renderizar. Corregido, y se aplicó <Suspense> desde el inicio en
  restablecer-password/page.tsx para no repetir el error.
- **Hallazgo crítico (prioridad #1 actual):** Resend con el dominio de
  pruebas (onboarding@resend.dev) solo permite enviar al correo del dueño
  de la cuenta de Resend — confirmado en el log real de Resend (403 en
  todos los envíos a otras direcciones). Hoy ningún correo dirigido a una
  estudiante real llega. Requiere comprar y verificar un dominio propio en
  Resend antes de invitar estudiantes reales. Telegram no tiene esta
  restricción y funciona bien para avisos internos.

## 25/07/2026 — Auto-inscripción con voucher + reorganización de documentación

- Nuevo flujo de auto-inscripción: la estudiante elige plan, sube su propio
  comprobante de depósito/transferencia sin que la coordinadora tenga que
  crear nada primero. Backend: Inscripcion gana 4 campos + 2 estados nuevos
  (pendiente_verificacion, rechazado); nuevos endpoints POST /inscripciones/mia
  y PATCH /:id/rechazar-pago. Frontend: app/inscripcion/page.tsx (con
  contenido de marketing y fotos reales del curso), dashboard/page.tsx
  maneja los 4 estados de pago, panel/pagos/page.tsx tiene cola de
  verificación con comprobante visible.
- Fix: POST /api/uploads/imagen estaba restringido a coordinadora/admin, se
  agregó el rol estudiante.
- Se consolidaron los 6 documentos de contexto en 4 (esta reorganización).
- Análisis de factibilidad de pasarela de pago en RD entregado. Decisión
  final tomada por la fundadora: NO se implementará Azul por ahora, la
  transferencia manual con auto-inscripción ya resuelve la necesidad real.

## 24-25/07/2026 — Panel de administración a tarjetas

- app/(coordinadora)/panel/page.tsx: pantalla de tarjetas con íconos
  (lucide-react) agrupadas en "Gestión del curso", "Contenido público" y
  "Solo fundadora" (solo admin). Reemplaza la barra de pills que existía antes.
- panel/layout.tsx y admin/layout.tsx simplificados a solo header + link volver.
- Se actualizó Next.js de 16.2.10 a 16.2.11 (parche), resolviendo 3 de 4
  vulnerabilidades high de npm audit. Queda pendiente 1 (brace-expansion vía
  ESLint), requiere salto de versión mayor, no urgente.

## 22-23/07/2026 — Open Graph + bug crítico de detalle de noticia

- Open Graph completo: metadataBase/openGraph/twitter, og-image.png,
  generateMetadata dinámico en noticias/[id]/page.tsx. Verificado en
  Facebook/WhatsApp.
- Bug encontrado: la sesión de paginación anterior había pegado por error
  el código del listado encima del detalle de noticia — se recuperó desde
  git y se restauró.
- Análisis de factibilidad de pasarela de pago en RD entregado (ver arriba,
  decisión final tomada el 25/07).

## 22/07/2026 — Sesión larga: correcciones + paginación

- Firma del diploma, rebranding a "Muvo RD Vial", fix de embeds de YouTube,
  correctas/incorrectas por pregunta en el examen, progreso automático entre
  sesiones con espera de 24h entre exámenes, logo circular + favicon nuevos,
  incidente de dominio en Vercel resuelto.
- Paginación implementada en Estudiantes, Noticias y Movimientos contables.

## Antes del 22/07/2026 — Construcción inicial (backend + frontend)

Ver ARQUITECTURA_BACKEND.md y ARQUITECTURA_FRONTEND.md para el resultado
final de esta etapa. Resumen: autenticación JWT sin cookies, 3 roles,
flujo completo de inscripción -> pago -> 3 sesiones con contenido y examen
-> diploma con descarga firmada desde Cloudinary, panel de coordinadora/
admin con CRUD de noticias/testimonios/FAQ/contenido de página/contabilidad.

---

## Pendientes abiertos (reemplaza cualquier lista anterior de esta sección)

### Pendiente de confirmación legal (no resuelto por Claude)

- **RESUELTO (10/09/2026):** tanto el test psicológico completo como el
  cuestionario `CuestionarioEscolar` (14 preguntas) ya pasaron revisión
  legal de Ley 172-13 — la fundadora confirmó que están bien. Puede
  usarse ambos con estudiantes reales.

### ALTA PRIORIDAD (28/08/2026) — recrear contenido real de las 4 sesiones

- **`ContenidoSesion` (PDFs) y `Examen` deben borrarse y recrearse desde
  cero.** Se cargaron en una sesión sin documentar, pero con dos bugs
  serios: PDFs con errores de codificación (texto con símbolos
  extraños) y exámenes con la respuesta correcta siempre en la opción A
  (patrón explotable). Ver entrada 28/08/2026 arriba para el detalle
  completo, y ARQUITECTURA_BACKEND.md / DATABASE.md para el plan (borrar
  vía `curl` contra los endpoints reales, no a mano en Atlas; recrear
  con PDFs corregidos y opciones de examen aleatorizadas).
- Aprovechar ese mismo momento para renombrar `Sesion.titulo` (sigue en
  "Sesión 1"..."Sesión 4" — solo los materiales individuales tienen
  títulos reales, no la sesión en sí).

### Prioridad #1 — pendiente, sin bloqueo (ya no depende de la fundadora)

- **Definir el destinatario real de las notificaciones internas** —
  `DestinatarioNotificacion` hoy solo tiene activo un correo personal de
  pruebas (`ramndiaz@gmail.com`); el envío ya funciona técnicamente
  (dominio propio verificado, ver entrada 27/08/2026), solo falta que la
  fundadora confirme o agregue el correo institucional real.
- Terminar Telegram para el celular de la fundadora (sacar su `chat_id` y
  agregarlo en el panel de Notificaciones). Manual paso a paso ya
  entregado. Sería el canal de respaldo, ya no el único camino
  disponible (el correo real ya funciona). **Único blocker de
  infraestructura que sigue abierto** al 10/09/2026 — Resend y la UI de
  admin para precios de plan ya se cerraron, ver entrada de esa fecha.

### Motorista y Pesados — pendientes de diseño (08-09/09/2026, Pesados agregado 10/09/2026)

- Escolar y Empresarial ya están construidos y en producción (ver
  entrada 08-09/09/2026). **Motorista** (anunciado 06/09/2026) y
  **Pesados** (conductores de camiones y trailers, agregado
  10/09/2026) siguen sin diseñar — ambos probablemente necesitan
  currículo y práctica distintos al de "estandar". Primer paso para
  cada uno: sostener con la fundadora la misma conversación de
  descubrimiento que ya se tuvo para Escolar/Empresarial, no asumir que
  aplica el mismo patrón entre ellos.
- **Decisiones de arquitectura ya cerradas el 10/09/2026** para cuando
  se construyan: `Sesion`/`Examen`/`ContenidoSesion` no se duplican por
  programa, se les agrega `programaContenido` (`estandar`/`motorista`/
  `pesados`); `Plan` sí necesitará fila propia para cada uno (se
  inscriben individualmente, a diferencia de Escolar/Empresarial). Ver
  entrada 10/09/2026.

### Pendiente de la sesión 08-09/09/2026 (Grupo), sin bloqueo

- Probar en vivo el cron de `/api/interno/reporte-grupos` — nunca se
  esperaron las 24h reales desde `fechaInicio` ni se disparó a mano
  desde la pestaña Actions de GitHub (`workflow_dispatch`).
- Revisar en Mongo Atlas si quedó una cuenta de estudiante huérfana (sin
  `Inscripcion`/`MovimientoContable`/`ProgresoEstudiante`) de la prueba
  donde salió el bug de `numeroReferencia` — no se limpió todavía.
- **RESUELTO (10/09/2026):** nombre de la colección `CuestionarioEscolar`
  confirmado con la fundadora — no se cambia.

### NUEVO (10/09/2026): permisos inconsistentes en soft delete de estudiantes

- `PATCH /api/usuarios/desactivar-lote` (nuevo) y
  `PATCH /api/usuarios/:id/estado` (ya existía) son ambos `admin`-only,
  pero la UI vive en `/panel`, al que también entra `coordinadora`. Sin
  resolver todavía si se le abre el permiso a ella en ambos endpoints.

### Corrección de documentación pendiente (sin bloqueo)

- Confirmar y corregir la estructura real de rutas del estudiante en
  ARQUITECTURA_FRONTEND.md — ver sección al inicio de este documento
  ("Corrección pendiente de documentación").

### Ideas a futuro sin diseñar (el disparador ya no es el obstáculo)

- Recordatorio por correo cuando el examen ya está disponible (pasaron las
  24h de espera).
- Recordatorio por correo de voucher pendiente_verificacion/rechazado sin
  seguimiento después de varios días.
- **Actualización (04/09/2026):** el problema de "sin cron real" que
  bloqueaba ambas ideas ya no aplica — el patrón GitHub Action
  programado → endpoint protegido por secreto, construido para el
  resumen diario (ver esa entrada arriba), se puede reutilizar tal cual.
  Sigue pendiente solo diseñar el contenido y la frecuencia exacta de
  cada recordatorio.

### Decisiones ya tomadas (cerradas, dejadas aquí solo como registro)

- Pasarela de pago automática (Azul): NO se hará. Cerrado.
- Limpieza de datos de prueba en Mongo: se pospone a propósito. La purga
  deberá incluir en cascada Inscripcion, IntentoExamen,
  ProgresoEstudiante, Diploma (ya hay 6 diplomas de prueba, confirmado en
  el backup del 31/07/2026) y MovimientoContable de pagos de prueba
  confirmados — no solo el User. Cómo identificar cuáles son de prueba:
  sin definir todavía, revisar juntos cuando llegue el momento.
- /verificar-diploma: se queda pública (ya no está en el navbar).
- Kit de Preparación y Contacto: se quedan como contenido estático por ahora.
- Seguridad/confiabilidad (rate limiting, CORS dinámico, Sentry, tests):
  al final, cuando la app esté más madura.
- Foro/preguntas por sesión: pospuesto a una segunda etapa, después de
  validar con las primeras estudiantes reales.
- Pedir testimonio automáticamente al graduarse: descartado — solo se
  necesitan algunos testimonios, no vale la pena automatizar la captura.
- Recordatorio/botón de backup automatizado desde el panel de admin:
  evaluado, descartado — el backup se mantiene 100% manual.
- **Asignación automática de instructor de práctica**: NO se hará por
  ahora — la estudiante contacta directo al chofer de una lista, sin
  match automático (decisión del 06/09/2026, ver esa entrada).

### Mejoras menores sin empezar

- "Me gusta" en comentarios individuales de noticias.
- Afinar el rol del usuario `backup_readonly` en Atlas de
  `readAnyDatabase@admin` a un rol Read específico sobre `mav_rd` (mínimo
  privilegio, no urgente ya que es de solo lectura de todas formas).
- Confirmar los valores hex reales de los tokens de color `brand-yellow`
  y `brand-mamey` (en uso en el home desde la sesión sin documentar) y
  actualizar la tabla de colores en ARQUITECTURA_FRONTEND.md.
- Monitorear si con más choferes hace falta algún filtro/paginación en
  `GET /instructores/activos` — hoy devuelve todos los activos sin
  distinción (06/09/2026).

### Ya resuelto (para no volver a preguntarlo)

- **Seguridad contra registro masivo de bots** (Cloudflare Turnstile,
  rate limiting por IP en 5 endpoints, honeypot) — construida en una
  sesión previa sin documentar, auditada y documentada formalmente el
  10/09/2026, ver esa entrada.
- **Bug del Home: el hero y la tarjeta "Así empezamos" no reflejaban
  los cambios guardados desde `/admin/contenido-pagina`** — `app/page.tsx`
  nunca leía `/api/contenido`. Corregido el 10/09/2026, ver esa entrada.
- **Contabilidad de `Grupo` sin prorrateo** (una sola entrada contable
  por grupo, por el monto total, en vez de una por estudiante) —
  cambiado el 10/09/2026 a pedido de la fundadora, ver esa entrada.
- **Cédula opcional para estudiantes sin cédula** (menores de un grupo
  tipo colegio) y **soft delete en lote** de estudiantes — resuelto el
  10/09/2026, ver esa entrada.
- **Login de admin/coordinadora caía siempre en Pagos** — corregido el
  10/09/2026, ahora cae en el dashboard del panel.
- **Nombre de la colección `CuestionarioEscolar` confirmado**, y
  **revisión legal (Ley 172-13)** tanto del test psicológico completo
  como del cuestionario Escolar — ambas cerradas el 10/09/2026.
- **Blockers de infraestructura:** dominio de Resend verificado y UI de
  admin para precios/nombre/descripción de plan — ambos confirmados
  cerrados el 10/09/2026 (el segundo ya existía desde el 07/09, solo
  faltaba marcarlo como tal). Solo queda abierto el `chat_id` de
  Telegram de la fundadora.

- **Programa Escolar/Empresarial completo** (colección `Grupo`, gates de
  práctica/cuestionario condicionales, formularios de grupo + roster,
  contabilidad simplificada el 10/09 (ver arriba), cron de reporte
  diario, badge/filtro de grupo en `/panel/estudiantes`) — construido,
  desplegado y probado en producción el 08-09/09/2026, ver esa entrada.
  Pendientes reales restantes: probar el cron en vivo, limpiar una
  posible cuenta huérfana, y diseñar Motorista/Pesados (ver arriba).

- **Seguimiento de práctica de manejo** (choferes creados desde el
  panel de admin, notificación al completar teoría, dashboard del
  conductor para aprobar práctica, gate del diploma) — construido y
  confirmado funcionando el 06/09/2026, ver esa entrada.
- **Bug de dominio en la tarjeta de diploma compartible** (el texto
  visible seguía en `muvo-rd.vercel.app` aunque el QR ya usaba el
  dominio propio) — corregido el 06/09/2026, ver esa entrada.
- Test psicológico de perfil conductual (54 preguntas + 5 reflexiones,
  gate obligatorio antes del contenido, sin puntaje calculado) —
  construido y confirmado funcionando el 05/09/2026, ver esa entrada.
  Pendiente real restante: confirmación legal (ver arriba).
- Chatbot interno para la fundadora (Gemini 3.6 Flash + function
  calling, 7 herramientas de solo lectura) y resumen diario automatizado
  por correo/Telegram (GitHub Action a las 9pm) — ambos construidos y
  verificados funcionando el 04/09/2026, ver esa entrada.
- El formulario de Empresas ya persiste los leads en Mongo
  (`SolicitudEmpresarial`) — resuelto el 04/09/2026 (antes solo se
  notificaba, sin guardar nada).
- SEO migrado al dominio propio (sitemap, robots.txt, metadata,
  propiedad nueva en Search Console verificada) — resuelto el
  28/08/2026, ver esa entrada. La propiedad vieja de Search Console
  (`muvo-rd.vercel.app`) no se perdió, solo dejó de ser la relevante.
- Bug de UI en aula virtual (botón "Marcar como visto" desalineado en
  contenido tipo pdf/enlace) — resuelto el 28/08/2026, ver esa entrada.
- Diploma compartible en redes sociales — construido de principio a fin,
  ver entrada 06-07/08/2026. Quedaba mal listado como pendiente en
  versiones viejas de este documento por un descuido de limpieza;
  corregido aquí, sin impacto en código.
- Dominio propio verificado en Resend (`muvordvial.com`) + `www.muvordvial.com`
  como dominio de producción en Vercel — resuelto el 27/08/2026 (ver esa
  entrada). El correo transaccional real ya funciona, incluida la
  notificación del formulario de Empresas.
- Badge de conteo de "pendientes por verificar" en la tarjeta "Pagos" del
  panel — implementado desde el 26/07/2026.
- Panel de estudiantes "que iba a crecer indefinidamente" — resuelto con
  las 3 pestañas (Activas/Graduadas/Inactivas) + archivar cuenta, ver
  entrada 04/08/2026.
- Examen/ContenidoSesion desactivables — confirmado que ya lo soportaban
  ambos desde antes, ver entrada 04/08/2026.