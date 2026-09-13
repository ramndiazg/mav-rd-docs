# Análisis — Programas Motorizados y Pesados (flujo completo)

> **Actualización 11/09/2026, después de aprobado este análisis:** se
> cerraron con la fundadora las preguntas 1 ("¿usan el `TestPsicologico`
> completo?" → sí) y parte de la 5 (se suma una pestaña informativa
> **"Educación Vial Escolar"**, `/escolar`, mismo patrón que
> `/empresas` — fuera del alcance original de este documento, que solo
> cubría Motorizados/Pesados/`estandar` dentro de `/inscripcion`). Ver
> `HISTORIAL_MODIFICACIONES.md` (11/09/2026) para el detalle de las 3
> correcciones preparatorias ya aplicadas al código (rename de
> `CuestionarioEscolar`, gate de práctica centralizado, índice de
> `Sesion`) y el bug de `CuestionarioEscolar` que se encontró y corrigió
> de paso. Lo que sigue de este documento es la versión original,
> ya con el nombre corregido a "Motorizados" en todo el texto.
>
> Documento de análisis previo a implementación. Responde al pedido del
> 11/09/2026: agregar los programas **Motorizados** y **Pesados**
> (camiones/trailers), cada uno con su propio material de
> estudio (solo teoría, sin práctica) organizado en sesiones, y con
> varios exámenes por sesión de los que se elige uno al azar en cada
> intento — el mismo patrón que ya existe hoy para `estandar`.
>
> Este documento parte de `ARQUITECTURA_BACKEND.md`,
> `ARQUITECTURA_FRONTEND.md`, `DATABASE.md` y una revisión directa del
> código real en los dos zips subidos (no solo de lo que dicen los
> `.md`, que en algún punto quedaron desincronizados del código — ver
> sección 6). No incluye código todavía, tal como se pidió.

---

## 1. Lo que ya existe y en lo que nos apoyamos

Antes de diseñar algo nuevo, esto es clave: **la decisión de diseño de
fondo ya está tomada y documentada** (`ARQUITECTURA_BACKEND.md`,
"Decisión cerrada 10/09/2026"): `Sesion`, `ContenidoSesion` y `Examen`
**no se duplican por programa**. Se les agrega un campo
`programaContenido` (`estandar` | `motorizados` | `pesados`) y siguen
siendo las mismas tres colecciones de siempre. Este análisis desarrolla
esa decisión hasta el nivel de "qué archivo toca, qué endpoint cambia,
qué pantalla se crea" — que es lo que faltaba.

También confirmé en el código (no solo en los `.md`) que el mecanismo
de "varios exámenes por sesión, uno al azar por intento" que pediste
**ya existe y funciona**, así que Motorizados/Pesados lo heredan gratis,
sin tocar lógica:

```js
// examenController.js — intentarDesbloquear()
const versionesActivas = await Examen.find({ sesionId, activo: true });
...
const examenElegido =
  versionesActivas[Math.floor(Math.random() * versionesActivas.length)];
```

El campo `Examen.nombreVersion` ya permite tener "Versión A", "Versión
B", etc. para una misma sesión — el problema real hoy no es de diseño,
es que en la práctica solo se cargó **una** versión por sesión (y con
el bug de la opción A, ver sección 6). Para Motorizados/Pesados hay que
cargar de entrada **al menos 2-3 versiones activas por sesión**, no una,
o el "al azar" no tiene nada entre qué elegir.

---

## 2. Flujo funcional propuesto, de punta a punta

1. **`/inscripcion`** — la estudiante primero elige **programa**
   (Escolares — el `estandar` actual, mantiene su nombre público
   actual —, Motorizados, Pesados), y recién después ve los planes de
   _ese_ programa y el formulario de voucher.
2. **Pago** — mismo flujo de voucher/verificación que hoy. No cambia
   nada del mecanismo, solo qué `programa` queda registrado en la
   `Inscripcion`.
3. **Confirmación de pago** — igual que hoy, coordinadora aprueba el
   voucher (o efectivo). Se crea `ProgresoEstudiante` igual que ahora.
4. **Cuestionario previo — RESUELTO (11/09/2026):** Motorizados y
   Pesados usan el mismo `TestPsicologico` de 54+5 preguntas que hoy es
   obligatorio antes de la sesión 1, igual que el resto de los
   choferes. No necesitan un cuestionario corto propio al estilo
   `CuestionarioEscolar` (eso queda solo para Escolar).
5. **Aula virtual** — la estudiante ve sus sesiones (Sesión 1, 2, 3...)
   con contenido tipo video/pdf/enlace/texto, **filtradas por su
   programa**. Mismo componente de UI que hoy (unificado desde el
   28/08), solo cambia qué `Sesion` se le muestra.
6. **Examen por sesión** — al terminar de ver el contenido de una
   sesión se desbloquea el examen, igual que hoy: 24h de espera entre
   sesión y sesión (excepto override de coordinadora), 3 intentos,
   versión aleatoria entre las activas de esa sesión.
7. **Fin de curso** — al aprobar la última sesión, `cursoCompletado =
true`. **Aquí hay una diferencia real con `estandar`:** dijiste que
   por ahora Motorizados y Pesados son **solo teoría, sin parte
   práctica** — "va a ser parecido al curso para escolares". Eso
   significa que el gate de práctica de manejo (instructor, aprobación,
   `practicaAprobada`) **no debe aplicar** a estos dos programas, igual
   que ya no aplica hoy a Escolar/Empresarial. Ver sección 4 para el
   cambio concreto que esto exige en `diplomaController.js` y
   `practicaController.js` — es el hallazgo más importante de este
   análisis, porque hoy el gate de práctica está atado a `grupoId`
   nada más, no a `programa`, y Motorizados/Pesados individuales
   **no tienen `grupoId`** (se inscriben solas, como `estandar`).
8. **Diploma/certificado** — se genera igual que hoy. Queda pendiente
   de diseño si el PDF debe decir "Motorizados"/"Pesados" en vez de un
   texto genérico de "Educación Vial" (ver sección 7, pregunta 4).

---

## 3. Modelo de datos — qué cambia en cada colección

### `Sesion` — cambio de esquema real, no solo agregar un campo

```js
// models/Sesion.js — actual
numero: { type: Number, required: true, unique: true, min: 1, max: 4 },
```

El índice `unique: true` es **global sobre `numero`**, no por programa.
Si hoy existe `Sesion { numero: 1 }` para `estandar`, **no se puede
crear** `Sesion { numero: 1 }` para `motorizados` sin tocar el índice —
Mongo lo rechazaría por duplicado. Este es el primer cambio de esquema
real que exige el pedido, no es opcional:

```js
numero: { type: Number, required: true, min: 1, max: 4 }, // quitar unique acá
programaContenido: {
  type: String,
  enum: ["estandar", "motorizados", "pesados"],
  default: "estandar",
},
```

```js
sesionSchema.index({ programaContenido: 1, numero: 1 }, { unique: true });
```

El límite `max: 4` también hay que revisarlo — hoy asume 4 sesiones
para todo el curso. Motorizados/Pesados podrían necesitar un número
distinto de sesiones (ver pregunta 2 en sección 7); si el número final
también es 4, no hace falta tocar el límite, pero si difiere hay que
decidir si el límite se vuelve configurable o específico por programa.

### `Examen` y `ContenidoSesion` — no necesitan campo propio

Ambos referencian `sesionId`, y `Sesion` ya va a cargar
`programaContenido`. Con eso alcanza para filtrar correctamente sin
duplicar el campo — coincide con lo ya documentado. Sin cambios de
esquema aquí, solo de datos (cargar contenido y exámenes nuevos
apuntando a las `Sesion` de cada programa nuevo).

### `ProgresoEstudiante` — recomendación: denormalizar `programa`

Hoy `obtenerSesionParaEstudiante` (`sesionController.js`) hace
`Sesion.findOne({ numero })` **sin filtrar programa en absoluto** —
hay que arreglar eso sí o sí para que una estudiante de Motorizados no
termine viendo, por accidente, la Sesión 1 de `estandar` (mismo
`numero`, `Sesion` distinta). Para resolverlo sin una consulta extra en
cada request, propongo agregar:

```js
programa: { type: String, default: "estandar" }, // espejo de Inscripcion.programa
```

Se setea una sola vez, al confirmar el pago (`confirmarPago`) o al
crear el roster de un `Grupo`, copiando `inscripcion.programa`. Así
`obtenerSesionParaEstudiante` puede hacer
`Sesion.findOne({ numero, programaContenido: progreso.programa })`
directo, sin un `populate` ni una segunda consulta a `Inscripcion`.
Alternativa sin tocar el esquema: buscar la `Inscripcion` activa de la
estudiante en cada request y leer `programa` de ahí — funciona, pero es
una consulta adicional en el path más caliente del sistema (cada carga
de sesión). Prefiero la denormalización por eso.

### `Plan` — el esquema asume práctica, y Motorizados/Pesados no tienen

Este es el segundo hallazgo importante. `Plan.codigo` es un enum
cerrado (`["fundacion", "normal", "vip"]`) pensado para **niveles de
práctica de manejo** — cada plan de hoy se diferencia por
`modalidadPractica`, `cantidadSesionesPractica`,
`duracionSesionMinutos`, `costoPorSesion`, y esos 4 campos son
`required` en el esquema actual. Motorizados/Pesados **no tienen
práctica** (por ahora, según dijiste), así que un `Plan` de Motorizados
no tiene con qué llenar esos campos de forma honesta.

Dos caminos, a decidir con la fundadora (pregunta 3, sección 7):

- **(a) Un solo plan por programa**, con un `codigo` nuevo (p. ej.
  `"teorico"`) que represente "curso completo, sin niveles" — más
  simple, evita forzar datos de práctica inventados. Requiere ampliar
  el enum de `codigo` y volver esos 4 campos de práctica opcionales
  (`required: false`) a nivel de esquema, condicionados a que el
  programa tenga práctica.
- **(b) Varios niveles igual que `estandar`** (p. ej. distintos precios
  por algún otro criterio: urgencia, incluye o no acompañamiento al
  examen del INTRANT, etc.) — mantiene el enum `fundacion/normal/vip`
  pero solo tiene sentido si la fundadora efectivamente quiere
  diferenciar niveles dentro de Motorizados/Pesados.

Este análisis recomienda (a) porque es lo que mejor calza con "va a ser
parecido al curso para escolares... por ahora solo la teoría" — pero es
una decisión de negocio, no la tomo por mi cuenta.

### `Inscripcion` — ya soporta `programa`, pero el controller no lo usa

El esquema ya tiene `programa: { type: String, default: "estandar" }`
desde el 07/09. El problema es que **ningún controller lo lee del
`req.body` todavía** — ver sección 4, es 100% un cambio de backend, no
de esquema.

---

## 4. Backend — endpoints a modificar y a crear

### A modificar

- **`inscripcionController.js#crearOReenviarInscripcionPropia`** (y
  `crearInscripcion`, la versión que usa la coordinadora): hoy
  hardcodea `programa: "estandar"` al buscar el `Plan`
  (`Plan.findOne({ codigo: tipoPlan, programa: "estandar", activo: true })`)
  y ni siquiera acepta `programa` en el body. Hay que:
  - aceptar `programa` del body (`"estandar" | "motorizados" | "pesados"`,
    con default `"estandar"` para no romper el flujo actual),
  - usarlo en el `Plan.findOne(...)`,
  - guardarlo en la `Inscripcion` creada,
  - validar `tipoPlan` contra los códigos válidos **de ese programa**
    (si se adopta la opción (a) de la sección 3, el enum de `tipoPlan`
    en `Inscripcion` también necesita el código nuevo, p. ej.
    `"teorico"`).
- **`sesionController.js#obtenerSesionParaEstudiante`**: filtrar por
  `programaContenido` (ver sección 3, `ProgresoEstudiante.programa`).
  Sin este cambio, el bug de "ve la sesión equivocada" es inmediato en
  cuanto exista más de un programa con `numero` repetidos.
- **`sesionController.js#listarSesiones`**: hoy trae _todas_ las
  sesiones sin filtrar programa — para el panel de coordinadora hay que
  agregar `?programaContenido=` como filtro opcional (si no, la lista
  de gestión mezcla las sesiones de los 3 programas sin poder
  distinguirlas fácilmente en la UI).
- **`diplomaController.js`** (`listarElegibles` y `generarDiploma`):
  el gate `const requierePractica = !usuario.grupoId;` necesita
  ampliarse. Como Motorizados/Pesados no tienen `grupoId` (se inscriben
  individualmente, igual que `estandar`), con el código actual
  **quedarían atrapadas para siempre esperando una aprobación de
  práctica que nunca va a llegar porque no hay instructor asignado a
  su programa.** Cambio necesario:
  ```js
  const SIN_PRACTICA = ["motorizados", "pesados"]; // + grupo, ya cubierto
  const requierePractica =
    !usuario.grupoId && !SIN_PRACTICA.includes(inscripcion.programa);
  ```
  Esto obliga a traer también `inscripcion.programa` (o
  `progreso.programa`, ver sección 3) en esta consulta, cosa que hoy
  no se hace — solo se trae `grupoId` del `User`.
- **`practicaController.js#listarPendientes`**: mismo filtro que ya
  existe para `grupoId` (`.filter((p) => !usuariosPorId.get(...)?.grupoId)`)
  necesita extenderse con el mismo criterio de `SIN_PRACTICA`, o las
  estudiantes de Motorizados/Pesados van a aparecer para siempre en la
  lista de "esperando práctica" del panel del chofer — el mismo bug que
  ya se encontró y corrigió una vez para `Grupo` (documentado en
  `ARQUITECTURA_BACKEND.md`, 09/09/2026). Vale la pena aprovechar y
  centralizar este criterio en una sola función helper
  (`estudianteRequierePractica(usuario, inscripcion)`) en vez de
  repetir la condición en 3 archivos distintos — ya se repitió 2 veces
  (`diplomaController` y `practicaController`) antes de este pedido, y
  ahora sería la tercera.
- **`intentoExamenController.js`**: la notificación
  `notificarEstudianteListaParaPractica` se dispara al completar el
  curso. Con Motorizados/Pesados sin práctica, esa notificación no debería
  salir — mismo criterio `SIN_PRACTICA` aplicado aquí también (ya hoy
  se salta para estudiantes de `Grupo`, ver
  `ARQUITECTURA_BACKEND.md` sección Escolar/Empresarial).
- **`planController.js`**: sin cambios de lógica si se adopta la
  opción (a) de la sección 3 (sigue siendo genérico por `programa` +
  `codigo`), pero si se adopta (b) con campos de práctica opcionales,
  hay que ajustar la validación de `PATCH /api/planes/:codigo`.

### A crear

- **`scripts/migrarProgramasNuevos.js`** (o extender
  `crearSesionesIniciales.js`): siembra las `Sesion` de Motorizados y
  Pesados (con `programaContenido` correspondiente) y, si se adopta la
  opción (a), los `Plan` nuevos (`programa: "motorizados"` /
  `"pesados"`, `codigo: "teorico"`).
- Nada nuevo en `Examen`/`ContenidoSesion` a nivel de rutas — los
  endpoints existentes (`crearExamen`, `crearContenido`, etc.) ya
  reciben `sesionId`, así que cargar contenido/exámenes de Motorizados y
  Pesados es simplemente apuntar al `sesionId` correcto desde el panel
  de coordinadora, sin tocar código.

---

## 5. Frontend — páginas a modificar y a crear

### `/inscripcion` (modificar, cambio de estructura más que de detalle)

Hoy la página asume un solo programa: llama a `GET /api/planes` sin
`?programa=` (default `estandar`) y no hay ningún selector de curso.
Para cumplir "empieza en la página de inscripción mostrando estas
nuevas dos opciones con detalles" hace falta:

- Un paso/selector nuevo **antes** de "Elige tu plan": tres tarjetas
  (Escolares/`estandar`, Motorizados, Pesados) con imagen, descripción
  corta y foco (p. ej. "Para conductores de motocicleta" /
  "Para conductores de camiones y trailers, dos tipos de vehículo").
  Puede vivir en la misma página como un estado (`programaSeleccionado`)
  que decide qué llamar en `GET /api/planes?programa=...`, sin
  necesidad de una ruta nueva — más simple que crear
  `/inscripcion/motorizados` y `/inscripcion/pesados` como páginas
  separadas, y evita duplicar el formulario de voucher 3 veces.
- El formulario de envío (`enviarFormulario`) necesita mandar
  `programa` en el body de `POST /api/inscripciones/mia`, no solo
  `tipoPlan`.
- El bloque "Cómo inscribirte" y el copy del hero pueden quedar
  genéricos (ya lo son en gran parte) o adaptarse por programa —
  decisión de copy, no técnica.
- Imágenes: hoy usa `/inscripcion/teoria-1.jpg` etc., genéricas — para
  Motorizados/Pesados probablemente convenga fotografía propia
  (motocicleta/camión), a definir con la fundadora igual que se hizo
  con las fotos actuales.

### "Selección de curso" — no existe hoy como paso separado

Repaso importante: hoy **no hay una pantalla de "selección de curso"**
independiente — el `dashboard/page.tsx` asume directamente
`SESIONES = [1, 2, 3, 4]` fijo, sin ningún concepto de programa. Lo que
pediste ("en la selección del curso los agregamos") lo estoy
interpretando como el selector nuevo que se agrega dentro de
`/inscripcion` (arriba) — si en realidad te referís a un paso
_distinto_, posterior al login, antes de `/inscripcion` (p. ej. una
estudiante ya logueada eligiendo qué curso empezar), avísame y lo
separamos en una ruta propia; tal como está el resto del flujo, no creo
que haga falta porque cada estudiante ya queda "atada" a un programa
desde su `Inscripcion` — no hay caso de uso de "cambiar de programa"
después de inscribirse.

### `dashboard/page.tsx` (modificar)

- `SESIONES = [1, 2, 3, 4]` hardcodeado deja de ser válido en cuanto el
  número de sesiones pueda variar por programa (ver pregunta 2, sección
  7). Aunque termine siendo igual (4 sesiones para los 3 programas), lo
  correcto es dejar de hardcodear y traer la lista de sesiones activas
  del programa de la estudiante desde el backend
  (`GET /api/sesiones?programaContenido=...`, ya filtrable con el
  cambio de la sección 4), no un array fijo en el frontend.
- El texto del dashboard ("Sesión 1 de 4", copy del curso) probablemente
  tiene referencias genéricas a "el curso" que están bien así, pero
  vale una revisión rápida de copy si en algún punto dice explícitamente
  "Educación Vial" o algo específico de `estandar`.

### `(coordinadora)/panel/aula-virtual/page.tsx` y `panel/examenes/page.tsx` (modificar)

Necesitan un selector de programa (pestañas o dropdown) para poder
gestionar contenido/exámenes de los 3 programas por separado — hoy
asumen un solo conjunto de 4 sesiones. Mismo patrón que
`?programaContenido=` del backend.

### `test-psicologico/page.tsx` — RESUELTO (11/09/2026): sin cambios de código

Confirmado con la fundadora: Motorizados/Pesados usan el
`TestPsicologico` existente sin ninguna adaptación — el gate ya es
genérico por usuario, no por programa. No hace falta una pantalla de
cuestionario corto propia (eso quedó solo para Escolar).

### Página informativa "Educación Vial Escolar" — NUEVO (11/09/2026, fuera del alcance original de este análisis)

Se suma `/escolar` como pestaña propia (mismo patrón que `/empresas`,
ver `ARQUITECTURA_FRONTEND.md`), con suficiente detalle de la oferta
para colegios — distinto de Motorizados/Pesados/`estandar`, que se
explican dentro de `/inscripcion` como se describe arriba, no como
páginas propias. Requiere: página nueva `app/escolar/page.tsx`, link en
`components/layout/Navbar.tsx` y `app/page.tsx` (donde ya vive el de
"Empresas"), y definir con la fundadora si el botón de esa página lleva
a `/inscripcion` o a un flujo de contacto como el que ya usa Empresas.

### Nuevo, opcional: página informativa por programa (Motorizados/Pesados)

No estrictamente necesaria si el selector de `/inscripcion` ya trae el
detalle suficiente, pero si el contenido "con detalles" que pediste es
más largo que lo que cabe en una tarjeta (como hoy pasa con
`/empresas`, que es su propia página informativa), podría convenir
`/motorizados` y `/pesados` como páginas propias de marketing, con
`/inscripcion?programa=motorizados` como destino del botón "Inscríbete" —
mismo patrón que ya existe para Empresas. Vale decidirlo junto con la
fundadora en la conversación de descubrimiento (pregunta 5).

---

## 6. Correcciones detectadas de paso (no pedidas, pero relevantes antes de tocar este código)

Revisando el código real (no solo los `.md`) para este análisis
aparecieron 3 cosas que conviene resolver **antes o junto con** este
trabajo, porque Motorizados/Pesados va a tocar justo los archivos donde
están:

1. **Desincronización real entre `ARQUITECTURA_BACKEND.md` y el
   código — CORREGIDO (11/09/2026).** El documento decía que
   `InformacionComplementariaEscolar` ya se había renombrado a
   `CuestionarioEscolar` (10/09/2026), pero el código en disco seguía
   con el nombre viejo. Ya está renombrado en ambos lados (modelo,
   controller, rutas, mount, y las referencias en el frontend).
2. **El gate de práctica (`requierePractica = !usuario.grupoId`) — CORREGIDO (11/09/2026).**
   Estaba repetido en 2 archivos (`diplomaController.js` y
   `practicaController.js`) antes de este pedido. Se extrajo a
   `utils/elegibilidadPractica.js`, usado ahora también en
   `intentoExamenController.js` (que tenía una cuarta copia del mismo
   criterio que no se había detectado hasta revisar ese archivo).
3. **`Sesion.numero` con índice único global — CORREGIDO (11/09/2026).**
   Ya tiene `programaContenido` + índice compuesto
   `{ programaContenido, numero }`. Queda un paso de despliegue
   pendiente: dropear el índice viejo `numero_1` en Atlas antes de
   sembrar sesiones de un programa nuevo.

Las tres correcciones ya se aplicaron al código (ver
`HISTORIAL_MODIFICACIONES.md`, 11/09/2026) — quedan documentadas acá
como contexto de por qué se hicieron antes del trabajo mayor.

---

## 7. Preguntas abiertas para la fundadora (antes de escribir código)

> Pregunta original 1 (test psicológico) resuelta el 11/09/2026 — ver
> el aviso al inicio del documento. Preguntas 1 y 2 de abajo, resueltas
> el 13/09/2026. Queda un error de edición corregido en esta misma
> fecha: la pregunta de "Estructura de precio" se había borrado por
> accidente de una versión anterior de este documento al renumerar —
> restaurada acá con su respuesta ya incluida.

1. **RESUELTO (13/09/2026): ¿Cuántas sesiones tiene la teoría de
   Motorizados y de Pesados?** Confirmado: **4 sesiones**, igual que
   `estandar` — mismo límite `max: 4` en `Sesion`, no hace falta
   cambiarlo.
2. **RESUELTO (13/09/2026): estructura de precio.** Confirmado: **un
   solo plan por programa**, sin niveles (no hay práctica de manejo de
   por medio). Esto cierra la opción (a) de la sección 3 para `Plan` —
   un `codigo` nuevo (ej. `"teorico"`) en vez de niveles
   Fundación/Estándar/VIP, y los 4 campos de práctica (`modalidadPractica`,
   `cantidadSesionesPractica`, `duracionSesionMinutos`,
   `costoPorSesion`), hoy `required`, necesitan volverse opcionales a
   nivel de esquema para este tipo de plan.
3. **Contenido del diploma/certificado:** ¿debe decir explícitamente
   "Motorizados" o "Pesados" en el PDF, o alcanza con el mismo diseño
   genérico de hoy? Si tiene que decir el programa, hay que revisar la
   plantilla real del PDF (no incluida en los archivos que me pasaste,
   habría que ubicarla en el proyecto).
4. **Marketing/contenido de cada opción en `/inscripcion`** — parte
   resuelta el 11/09/2026: alcanza con tarjetas dentro de `/inscripcion`
   (nombre + descripción + precio), sin página propia por programa —
   Escolar es la excepción, ver más arriba. Queda por definir el nivel
   de detalle real de cada tarjeta con la fundadora.
5. Para Pesados: confirmaste "alcance inicial limitado a" camiones y
   trailers — ¿eso afecta solo al copy/marketing, o hay contenido
   _distinto_ dentro del programa según tipo de vehículo (p. ej.
   ¿camión y trailer comparten exactamente las mismas 4 sesiones, o en
   algún punto se separan)? Si comparten todo, no hace falta nada
   especial en el modelo; si en algún momento necesitan diverger, capaz
   conviene pensarlo como un campo más (`subtipo`) en vez de un tercer
   programa.

---

## 8. Orden de trabajo sugerido (cuando se pase a código)

1. ~~Corregir el índice de `Sesion` + renombrar
   `InformacionComplementariaEscolar` → `CuestionarioEscolar`~~ — **ya
   hecho (11/09/2026)**, junto con el gate de práctica centralizado y la
   pantalla `/panel/cuestionario-escolar` que faltaba (ver
   `HISTORIAL_MODIFICACIONES.md`).
2. Backend: filtro por `programaContenido` en
   `obtenerSesionParaEstudiante`/`listarSesiones` (el campo ya existe en
   el esquema, falta usarlo en las consultas), `programa` en
   `Inscripcion`/`ProgresoEstudiante` de punta a punta
   (`crearOReenviarInscripcionPropia`, `confirmarPago`).
3. Definir y crear los `Plan` de Motorizados/Pesados según lo que se
   decida en la pregunta 3.
4. Script de siembra de `Sesion` (sin contenido real todavía, como se
   hizo originalmente con `estandar`).
5. Frontend: selector de programa en `/inscripcion`, `dashboard` sin
   `SESIONES` hardcodeado, filtro de programa en el panel de
   coordinadora (aula virtual y exámenes).
6. Cargar contenido real y exámenes (con **varias versiones activas por
   sesión desde el día uno**, aprendiendo del problema que tuvo
   `estandar` de quedar con una sola versión y la opción A siempre
   correcta) para Motorizados y Pesados.
7. Probar de punta a punta: inscripción → pago → cuestionario → 4
   sesiones → exámenes con selección aleatoria real (repetir el mismo
   intento varias veces y confirmar que rota de versión) → diploma sin
   pedir práctica.
