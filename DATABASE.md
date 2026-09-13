# Esquema de Base de Datos — MongoDB Atlas

> Base de datos: mav_rd, dentro del cluster compartido
> mujeresalvolante.rd4sofa.mongodb.net (versión real confirmada: 8.0.29).
> Mongoose como ODM. Todas las colecciones usan \_id (ObjectId) automático
> y createdAt/updatedAt (timestamps automáticos de Mongoose), salvo que se
> indique lo contrario. Refleja el estado real al 10/09/2026.

---

## Segunda purga: usuarios de prueba, todos los roles (07/09/2026)

Se corrió `scripts/purgarUsuariosPrueba.js` (nuevo, no confundir con
`purgarDatosPrueba.js` de la purga anterior) en modo real. A diferencia de
la purga del 06/08/2026, esta vez el criterio fue "todos los usuarios
excepto `maria@test.com`, sin importar rol" (confirmado explícitamente con
el usuario, incluyendo coordinadora/admin/conductor) — y a diferencia del
script viejo, **no tocó `Sesion`, `Examen` ni `ContenidoSesion`** (esas ya
tenían contenido real, aunque con bugs pendientes de corregir aparte).

Se borraron 4 usuarios (3 `estudiante` + 1 `conductor`, el mismo que se
había probado de punta a punta el 06/09) y su cascada: 2 `inscripciones`,
5 `intentosExamen`, 2 `progresoEstudiante`, 1 `diploma`, 1
`testPsicologico`, 1 `instructor`. **Sobrevivió únicamente** la cuenta
`maria@test.com` (rol `admin`). No se tocó `movimientosContables`
(confirmado que no había pagos reales confirmados todavía).

**Efecto colateral esperado, no un bug:** `GET /instructores/activos`
devuelve vacío hasta que se cree un chofer real de nuevo.

---

## Purga completa de datos de prueba (06/08/2026) — para referencia histórica

Se corrió `scripts/purgarDatosPrueba.js` (backend) en modo real. Conteos
purgados:

| Colección             | Documentos borrados |
| --------------------- | ------------------- |
| users (excepto admin) | 17                  |
| sesiones              | 3                   |
| examenes              | 15                  |
| contenidoSesion       | 22                  |
| intentosExamen        | 30                  |
| progresoEstudiante    | 11                  |
| inscripciones         | 13                  |
| diplomas              | 6                   |

**Sobrevivió únicamente** la cuenta `maria@test.com` (rol `admin`).

**No se tocaron**: `configuracion`, `destinatariosNotificacion`,
`noticias`, `testimonios`, `faqs`, `contenidoPagina`,
`movimientosContables`, `balancesMensuales` — ninguna depende de
estudiantes ni de sesiones.

**Actualización (13/08/2026): `sesiones` ya no está vacía.**
`scripts/crearSesionesIniciales.js --confirmar` se ejecutó — las 4
sesiones existen con títulos provisionales ("Sesión 1"..."Sesión 4").
`examenes` y `contenidoSesion` **siguen vacías** — ese script solo crea
las sesiones, no contenido ni exámenes; esos se cargan aparte una vez
definidos los temas reales.

---

## 1. users — cédula ahora opcional (10/09/2026), + valor de rol "conductor" (05/09/2026)

```js
{
  _id: ObjectId,
  nombre: String, apellido: String,
  cedula: String,          // NUEVO (10/09/2026): ya no es `required`,
                            // pasó a `unique + sparse` (mismo patrón
                            // que Inscripcion.numeroReferencia). Los
                            // menores de un Grupo tipo colegio no
                            // tienen cédula; con `required + unique`
                            // normal, dos estudiantes con "N/A" a mano
                            // chocaban como duplicado. Sigue siendo
                            // obligatoria en autoregistro
                            // (authController.js la exige él mismo);
                            // opcional solo en el roster de Grupo
                            // (grupoController.js), donde además
                            // cualquier variante de "n/a"/vacío se
                            // guarda como undefined real, no como texto.
  telefono: String,
  email: String,           // único
  passwordHash: String,    // bcrypt
  provincia: String,
  fechaNacimiento: Date,
  rol: String,             // 'estudiante' | 'coordinadora' | 'admin' | 'conductor'
                            // 'conductor' agregado 05/09/2026 — instructor
                            // que aprueba la práctica de manejo. Ninguno de
                            // los 4 roles tiene registro público excepto
                            // 'estudiante' — los otros 3 los crea un admin.
  activo: Boolean,          // soft delete. NUEVO (10/09/2026):
                            // PATCH /api/usuarios/desactivar-lote
                            // (admin) desactiva varios de una vez
                            // (`{ ids: [...] }` → `updateMany`), para
                            // cuando el roster de un Grupo cambia.

  emailVerificado: Boolean,
  tokenVerificacionEmail: String,
  tokenVerificacionExpira: Date,

  tokenRecuperacion: String,
  tokenRecuperacionExpira: Date,

  // NUEVO (08/09/2026): ref a Grupo (ver sección 22). null para
  // autoregistro individual (flujo de siempre, sin cambios). Con valor
  // solo para estudiantes creadas en bloque por un Grupo — determina si
  // se le exige práctica de manejo para el diploma y cuál cuestionario
  // previo aplica (TestPsicologico vs. CuestionarioEscolar,
  // ver sección 23).
  grupoId: { type: ObjectId, ref: "Grupo", default: null },

  createdAt: Date, updatedAt: Date
}
```

## 2. inscripciones — reestructuración de planes + campo `programa` nuevo (07/09/2026)

**`tipoPlan` pasó de `["normal","vip"]` a `["fundacion","normal","vip"]`**
— ver sección nueva "21. Plan" más abajo para el detalle completo de cada
uno (precio, sesiones de práctica, características). La inscripción
`"normal"` vieja (RD$1,500) se reetiquetó a `"fundacion"` vía
`scripts/migrarPlanes.js` — 0 documentos migrados en la práctica porque ya
no quedaba ninguna inscripción real al momento de correrlo (ver purga de
arriba).

**NUEVO campo `programa`** (`String, default: "estandar"`, sin enum
cerrado a propósito): decisión tomada en esta sesión al discutir que se
va a necesitar contenido diferente para Escolar, Empresarial y — anunciado
por la fundadora el 06/09/2026 — un curso para motoristas. `programa`
(qué currículo cursa) queda deliberadamente separado de `tipoPlan` (qué
nivel de práctica/precio dentro de ese currículo) — son dos dimensiones
distintas. Hoy solo existe el programa `"estandar"`; el campo se agregó
ahora, sin nada que migrar, para no pagar una migración más cara después.
Ver `ESPECIFICACION_PROGRAMAS_NUEVOS.md` para el diseño completo de los
programas nuevos.

`tipoPlan` (dentro de cada programa) sigue siendo la única diferencia
estructurada entre planes — la teoría es la misma para todos los planes
de un mismo programa, la diferencia real está en la práctica de manejo.
Ver ARQUITECTURA_BACKEND.md para el detalle de los dos flujos de pago.

**NUEVO valor de `tipoPlan` (09/09/2026): `"grupo"`.** Enum completo
ahora `["fundacion","normal","vip","grupo"]`. Exclusivo de estudiantes
creadas por un `Grupo` (Escolar/Empresarial, ver sección 22) — no tienen
nivel individual de plan, el precio vive solo en `Grupo.precioAcordado`,
nunca pasa por la colección `Plan` (**decisión reconfirmada 10/09/2026**).

**CAMBIO (10/09/2026): `monto` en inscripciones de `Grupo` ya es solo
referencia interna, no contabilidad real.** Sigue calculándose igual
que antes (prorrateo de `precioAcordado` entre el roster, residuo en la
primera estudiante del lote) porque el campo `monto` de `Inscripcion`
sigue siendo obligatorio, pero **ya no alimenta ningún
`MovimientoContable`** — ver sección 22 (`Grupo`) para el detalle de la
única entrada contable por grupo que la reemplaza.

**Bug corregido (09/09/2026): `numeroReferencia` con `default: null` +
índice `unique, sparse`.** Un índice `sparse` en Mongo solo excluye
documentos donde el campo está _ausente_, no donde vale `null` explícito
— con el `default: null` que tenía el schema, cualquier `Inscripcion`
creada sin voucher (flujo "efectivo" del admin, y cada estudiante de un
`Grupo`) quedaba con `numeroReferencia: null` _guardado de verdad_, así
que la segunda de esas chocaba como "duplicado" contra la primera. Se
quitó el `default` — ahora el campo queda genuinamente `undefined`
cuando no se manda, que el índice `sparse` sí excluye. No se tocó el
índice en Mongo, solo lo que la app escribe. Puede quedar como máximo un
documento viejo con `numeroReferencia: null` explícito de antes del fix
(nunca pudo haber dos, la colisión ya lo habría impedido) — inofensivo,
no hace falta limpiarlo a mano.

## 3. configuracion (key-value) — precios de plan DEPRECADOS (07/09/2026)

`precio_plan_normal` y `precio_plan_vip` **ya no los lee ningún endpoint**
— los precios y el resto de atributos de cada plan viven ahora en la
colección `Plan` nueva (ver sección 21). Los registros viejos con esas
claves quedan huérfanos en Atlas; no se borraron, pero no se debe seguir
escribiendo ahí para precios. El resto de `configuracion` (lo que no sea
`precio_plan_*`) sigue funcionando igual, sin cambios.

## 4. sesiones — ya recreada tras la purga (13/08/2026), índice corregido (11/09/2026)

```js
{
  _id: ObjectId,
  numero: Number,            // 1 a 4 — único junto con programaContenido, no solo
  programaContenido: String, // default "estandar" — NUEVO 11/09/2026
  titulo: String,
  teoria: String,   // HTML/Markdown
  videos: [{ titulo: String, url: String }],
  activo: Boolean,
  createdAt: Date, updatedAt: Date
}
```

Índice: `{ programaContenido: 1, numero: 1 }`, único. **CORREGIDO
(11/09/2026):** antes `numero` tenía `unique: true` a nivel de campo
(único global) — con eso, una `Sesion { numero: 1 }` para Motorizados
habría chocado como duplicado contra la de `estandar`. **Paso de
despliegue pendiente:** el índice viejo `numero_1` sigue existiendo en
Atlas hasta que se dropee a mano o se corra `syncIndexes()` — hacerlo
antes de sembrar sesiones de un programa nuevo.

4 documentos existentes (todos `programaContenido: "estandar"` por el
`default`) con títulos **todavía provisionales** ("Sesión 1"..."Sesión
4") — confirmado el 28/08/2026 revisando la pantalla real del aula
virtual (el `<h1>` sigue mostrando "Sesión 1", no un tema real).
Renombrarlos a los temas reales es una simple actualización de `titulo`
vía `PATCH /sesiones/:numero`, no requiere cambio de esquema ni de
código. **Aclaración importante:** esto es distinto de los títulos de
`ContenidoSesion` (ver más abajo) — esos sí tienen nombres reales
("1.1 Bienvenida a Muvo RD Vial", etc.), pero son los títulos de cada
material individual dentro de la sesión, no el título de la sesión
misma. Sigue pendiente renombrar el nivel de `Sesion.titulo`.

## 5. examenes — YA NO está vacía, pero tiene un bug grave (28/08/2026)

```js
{
  _id: ObjectId,
  sesionId: ObjectId,   // ref: sesiones
  nombreVersion: String,
  preguntas: Array,     // exactamente 10 elementos
  activo: Boolean,
  createdAt: Date, updatedAt: Date
}
```

Se crearon las 4 versiones (una por sesión), pero con un **bug serio
detectado por el usuario**: la respuesta correcta cae siempre en la
opción A, en todas las preguntas — patrón predecible y fácilmente
explotable por cualquier estudiante que se dé cuenta (probablemente el
proceso que generó las preguntas no aleatorizó el orden de las
opciones). **No usable en este estado.** Plan acordado: borrar todo
(vía `curl` contra los endpoints reales, no a mano en Atlas) y volver a
crear las 4 versiones desde cero, esta vez con las opciones en orden
aleatorio por pregunta. Ver Pendiente (base de datos) al final de este
documento.

## 6. intentosExamen — sin cambios de esquema, colección vacía

## 7. progresoEstudiante — NUEVOS campos de práctica (05/09/2026)

```js
{
  _id: ObjectId,
  userId: ObjectId,       // ref: users, único
  sesionActualDesbloqueada: Number,
  sesionesAprobadas: [Number],
  cursoCompletado: Boolean,       // 4 sesiones + 4 exámenes aprobados
  contenidosVistos: [ObjectId],   // ref: contenidoSesion
  fechasAprobacionSesion: [{ sesion: Number, fecha: Date }],

  // NUEVO — seguimiento de práctica de manejo. Requisito adicional y
  // separado de cursoCompletado; ambos son necesarios para generar el
  // diploma (ver diplomaController.js en ARQUITECTURA_BACKEND.md).
  practicaAprobada: Boolean,       // default false
  fechaAprobacionPractica: Date,   // default null
  practicaAprobadaPor: ObjectId,   // ref: users (el conductor que aprobó)

  createdAt: Date, updatedAt: Date
}
```

## 8. contenidoSesion — NUEVO campo `publicIdCloudinary` (13/08/2026)

```js
{
  _id: ObjectId,
  sesionId: ObjectId,      // ref: sesiones
  titulo: String,
  tipo: String,            // 'video' | 'pdf' | 'enlace' | 'texto'
  url: String,              // video/pdf/enlace
  publicIdCloudinary: String, // NUEVO — solo si el pdf se subió como archivo
                              // (permite generar una URL de descarga
                              // firmada al momento; si está vacío, se usa
                              // `url` directo como fallback)
  contenidoTexto: String,   // texto
  imagenUrl: String,        // portada opcional, cualquier tipo
  orden: Number,
  activo: Boolean,
  createdAt: Date, updatedAt: Date
}
```

Se cargó contenido real para las 4 sesiones (títulos reales tipo "1.1
Bienvenida a Muvo RD Vial", "1.2 Cultura Vial y el Valor de la Vida",
etc. — confirmados en captura real del aula virtual), pero **con un
problema serio detectado el 28/08/2026**: los PDFs subidos tienen
errores de codificación — texto con marcas y símbolos extraños,
ilegible o poco profesional para una estudiante real. **No usable en
este estado.** Mismo plan que `examenes`: borrar todo vía `curl` y
volver a subir los PDFs corregidos desde cero. El flujo técnico de
subida en sí (Cloudinary + URL firmada) sigue funcionando bien — el
problema es la calidad/codificación de los archivos PDF originales, no
el código del sistema.

## 9. diplomas — sin cambios de esquema, colección vacía tras la purga

## 10. noticias, 11. testimonios, 12. faqs, 13. contenidoPagina — sin cambios, no purgadas

## 14. movimientosContables — sin cambios, no purgada

## 15. balancesMensuales — sin cambios, no purgada

## 16. destinatariosNotificacion — sin cambios, no purgada

Es el mismo mecanismo que usan la solicitud del formulario de Empresas,
el chatbot no (es consulta directa, no notificación push) y el nuevo
resumen diario automatizado (ver ARQUITECTURA_BACKEND.md) — no se
agregó ninguna colección nueva para el mecanismo de envío en sí.
**Sigue separada** de la nueva `destinatariosPractica` (ver más abajo)
— nunca se mezclan.

## 17. solicitudesEmpresariales — NUEVA (04/09/2026)

```js
{
  _id: ObjectId,
  nombreEmpresa: String,
  contacto: String,
  cargo: String,        // opcional
  telefono: String,
  email: String,
  cantidadEstudiantes: Number,  // opcional
  mensaje: String,       // opcional
  contactado: Boolean,   // default false — para que la fundadora (o el
                          // chatbot) sepan si ya se le dio seguimiento
  createdAt: Date, updatedAt: Date
}
```

Antes, el formulario de `/empresas` solo enviaba una notificación por
correo/Telegram sin guardar nada — si Resend fallaba, no quedaba
registro. Ahora se guarda primero y luego se notifica, resolviendo ese
pendiente. También es la colección que consulta la herramienta
`solicitudesEmpresariales` del chatbot nuevo (ver ARQUITECTURA_BACKEND.md).

## 18. testsPsicologicos — NUEVA (05/09/2026)

```js
{
  _id: ObjectId,
  userId: ObjectId,       // ref: users, único — una sola vez por estudiante
  respuestas: [Number],   // exactamente 54 elementos, cada uno 1-5
  reflexiones: [String],  // exactamente 5 elementos, pueden venir vacíos
  createdAt: Date, updatedAt: Date
}
```

Digitaliza solo las secciones A-H de un test en papel más grande — las
secciones I/J/K (indicadores/recomendación del evaluador) se dejaron
fuera a propósito, requieren criterio profesional humano. **No se
guarda ningún promedio ni puntaje calculado** — el instrumento mezcla
preguntas en sentido positivo y negativo a propósito, así que un
promedio simple sería engañoso. Gate real: no se puede acceder a
ninguna sesión del aula virtual sin tener un documento en esta
colección (ver `sesionController.js` en ARQUITECTURA_BACKEND.md).
Acceso de lectura restringido a coordinadora/admin — ninguna estudiante
puede ver las respuestas de otra, ni las propias después de enviarlas.

**Nota de cumplimiento, sin resolver:** esta colección probablemente
contiene "datos sensibles" bajo la Ley 172-13 de Protección de Datos de
RD — pendiente que la fundadora lo confirme con asesoría legal antes de
usarlo con estudiantes reales (ver ARQUITECTURA_BACKEND.md).

## 19. instructores — NUEVA (05/09/2026)

```js
{
  _id: ObjectId,
  userId: ObjectId,        // ref: users, único — el User debe tener rol "conductor"
  diasDisponibles: [{ dia: String, horario: String }],
                            // dia: enum de días de la semana en minúscula
                            // horario: texto libre, ej "2:00 PM - 5:00 PM"
  activo: Boolean,          // default true
  createdAt: Date, updatedAt: Date
}
```

Se crea junto con el `User` (`rol: "conductor"`) en un solo paso, vía
`POST /api/usuarios/conductor` (admin, panel "Solo fundadora" — no hay
registro público para este rol, igual que coordinadora/admin). Nombre,
teléfono y correo viven en `User`, no se duplican aquí. Deliberadamente
**sin ningún campo de asignación a estudiante** — el flujo es que la
estudiante ve la lista de instructores activos y los contacta ella
misma, no hay match automático (ver ARQUITECTURA_BACKEND.md).

## 20. destinatariosPractica — NUEVA (05/09/2026)

```js
{
  _id: ObjectId,
  tipo: String,      // 'email' | 'telegram'
  valor: String,
  etiqueta: String,
  activo: Boolean,
  creadoPor: ObjectId, // ref: users
  createdAt: Date, updatedAt: Date
}
```

Esquema idéntico a `destinatariosNotificacion`, pero **colección
separada a propósito** — solo recibe avisos de "estudiante lista para
práctica", nunca se mezcla con vouchers/balance/empresas. Gestionada
desde un panel de admin aparte (`/admin/notificaciones-practica`).

---

## 21. Plan — NUEVA (07/09/2026)

```js
{
  _id: ObjectId,
  programa: String,        // default "estandar" — sin enum cerrado, a
                            // propósito, por los programas futuros
                            // (escolar/empresarial/motorizados)
  codigo: String,           // enum: 'fundacion' | 'normal' | 'vip'
  nombre: String,
  precio: Number,
  fraseDestacada: String,   // copy corto, tarjetas del Home
  modalidadPractica: String, // 'grupal' | 'individual'
  cantidadSesionesPractica: Number, // null si modalidadPractica es grupal
  duracionSesionMinutos: Number,
  costoPorSesion: Number,   // combustible — informativo, se paga en el
                            // lugar de la práctica, NO se cobra en la app
  caracteristicas: [String], // detalle largo, se muestra en /inscripcion
  activo: Boolean,
  orden: Number,
  createdAt: Date, updatedAt: Date
}
```

Reemplaza `precio_plan_normal`/`precio_plan_vip` de `configuracion` (ver
sección 3). Sembrada por `scripts/migrarPlanes.js` con los 3 planes
reales:

| codigo    | nombre                                  | precio   | modalidad  | sesiones práctica           | combustible/sesión |
| --------- | --------------------------------------- | -------- | ---------- | --------------------------- | ------------------ |
| fundacion | Plan de la Fundación Mujeres al Volante | RD$1,000 | grupal     | 15 min c/u, sin número fijo | RD$300             |
| normal    | Plan Estándar                           | RD$4,500 | individual | 8 sesiones de 60 min        | RD$500             |
| vip       | Plan VIP                                | RD$7,500 | individual | 10 sesiones de 60 min       | RD$500             |

VIP incluye además en `caracteristicas`: acompañamiento al INTRANT,
preparación para su examen teórico, instrucciones para el examen del
permiso de aprendizaje, e instrucciones para el examen práctico de la
licencia.

**Historial de precios:** el plan de entrada se llamó "Normal" a
RD$1,500 y el más completo "VIP" a RD$7,000 antes del 07/09/2026. Se
restructuró a 3 niveles (Fundación/Estándar/VIP) el mismo día; el precio
de Fundación bajó a RD$1,000 tras una corrección pedida por la fundadora
horas después del cambio inicial (que había quedado en RD$1,500).

**Editable desde el panel:** `admin/planes/page.tsx` (nuevo,
07/09/2026) — no hace falta tocar código ni Atlas para cambiar precio,
nombre, frase destacada, características, o activar/desactivar un plan.
Ver ARQUITECTURA_FRONTEND.md. **Nota (11/09/2026):** el esquema actual
asume práctica de manejo en 4 campos `required` — no calza limpio con
un programa 100% teórico como Motorizados/Pesados. Ver decisión
pendiente en `ANALISIS_MOTORISTA_PESADOS.md`, sección 3.

---

## 22. Grupo — NUEVA (08/09/2026)

```js
{
  _id: ObjectId,
  tipo: String,             // enum: 'colegio' | 'empresa'
  nombreInstitucion: String,
  contactoNombre: String,
  contactoEmail: String,
  contactoTelefono: String,
  precioAcordado: Number,   // total negociado, se rellena a mano
  cantidadEstudiantesEstimada: Number, // del Formulario 1, solo referencia
  fechaInicio: Date,        // null hasta confirmar el roster — de ahí
                             // cuentan las 24h del primer reporte diario
  pendienteRoster: Boolean, // true hasta cargar el roster real
  activo: Boolean,          // false cuando todas completan el curso
                             // (automático) o a mano como respaldo
  notas: String,
  creadoPor: ObjectId,      // ref: users
  createdAt: Date, updatedAt: Date
}
```

Colegios/empresas que inscriben en bloque (programas Escolar/Empresarial
— mismo currículo que `estandar`, sin práctica de manejo, precio único
negociado en vez de por plan). Las instituciones nunca entran a la app;
`User.grupoId` referencia esta colección. `cantidadEstudiantesReal` NO se
guarda como campo — se calcula al vuelo con `User.aggregate` por
`grupoId` cada vez que se necesita (`GET /api/grupos`), para no tener un
número desincronizado si se agregan estudiantes tarde.

**CAMBIO (10/09/2026): contabilidad de `Grupo` ya no se prorratea.**
Antes: un `MovimientoContable` por estudiante, cada uno con su parte
del prorrateo (`precioAcordado / cantidadTotal`). Ahora: una sola
`MovimientoContable` por grupo, por el monto TOTAL, creada solo en la
primera confirmación del roster (el primer pago real) — las adiciones
tardías ya no generan ningún movimiento nuevo, porque no hay cobro
adicional real que registrar. `Inscripcion.monto` de cada estudiante
sigue calculándose prorrateado (ver sección 2) pero es solo referencia
interna. Ver ARQUITECTURA_BACKEND.md (`grupoController.js`) para el
detalle y el cron de reporte diario.

**NUEVO (10/09/2026): cédula opcional para el roster.** `User.cedula`
ahora acepta quedar sin valor (ver sección 1) — pensado para grupos
tipo colegio con estudiantes menores sin cédula.

## 23. CuestionarioEscolar — NUEVA (08/09/2026), nombre confirmado (10/09/2026), código sincronizado (11/09/2026)

```js
{
  _id: ObjectId,
  userId: ObjectId,     // ref: users, único — una por estudiante
  respuestas: [Number],  // 12 respuestas escala 1-5
  reflexiones: [String], // 2 respuestas de texto libre
  createdAt: Date, updatedAt: Date
}
```

Cuestionario informativo de 14 preguntas (12 + 2 abiertas) para
estudiantes de Escolar (`Grupo.tipo === "colegio"`) — reemplaza a
`TestPsicologico` solo para ellas; Empresarial sigue usando el test
completo igual que `estandar`, y lo mismo va a aplicar a Motorizados y
Pesados (confirmado 11/09/2026, ver `ANALISIS_MOTORISTA_PESADOS.md`).
Colección deliberadamente separada, no reusa `TestPsicologico` —
preguntas sobre conocimiento vial y logística, sin ningún eje de
autocontrol/estrés/percepción de riesgo. **Revisión legal (Ley 172-13)
ya hecha y aprobada (confirmado 10/09/2026)** — puede usarse con
estudiantes reales. Nombre confirmado con la fundadora el 10/09/2026;
**el código (modelo/controller/rutas/mount) recién se puso al día el
11/09/2026** — hasta entonces seguía llamándose
`InformacionComplementariaEscolar` en disco pese a que este documento
ya decía "renombrado". Endpoint: `/api/cuestionario-escolar`.

## Índices recomendados — sin cambios excepto 2 nuevos

- users: único en email; único (sparse) en cedula (NUEVO, 10/09/2026 —
  ver sección 1).
- inscripciones: { userId }, único (sparse) en numeroReferencia.
- intentosExamen: compuesto { userId, sesionId }.
- diplomas: único en codigoVerificacion.
- movimientosContables: { fecha }.
- contenidoPagina: único en clave.
- contenidoSesion: { sesionId, activo }.
- examenes: recomendado { sesionId, activo } (no confirmado si ya existe
  físico en Atlas).
- balancesMensuales: compuesto único { mes, anio }.
- solicitudesEmpresariales: { createdAt: -1 } (ya definido en el
  esquema con `.index()`).
- testsPsicologicos: único en userId (ya definido en el esquema).
- **instructores: único en userId (NUEVO, 05/09/2026).**
- **Plan: único compuesto { programa, codigo } (NUEVO, 07/09/2026) — no
  global, para que un mismo código ("vip", por ejemplo) pueda repetirse
  en programas distintos el día que existan.**
- **sesiones: único compuesto { programaContenido, numero } (NUEVO,
  11/09/2026, reemplaza el único global que tenía antes solo `numero`
  — ver sección 4). El índice viejo `numero_1` sigue en Atlas hasta que
  se dropee a mano o se corra `syncIndexes()`.**

## Notas de diseño

- `activo` como patrón de soft delete sigue siendo consistente en las 3
  colecciones donde importa preservar historial (`users`, `examenes`,
  `contenidoSesion`).
- La purga de datos de prueba tiene un script formal y repetible
  (`scripts/purgarUsuariosPrueba.js`, con dry-run por defecto — único
  script de purga desde el 11/09/2026, ver ARQUITECTURA_BACKEND.md).

## Pendiente (base de datos)

- **ALTA PRIORIDAD (28/08/2026): borrar y recrear `examenes` +
  `contenidoSesion` desde cero.** Ambas colecciones tienen contenido
  cargado pero defectuoso — ver el detalle en las secciones 5 y 8 de
  arriba (respuesta siempre en A / PDFs con codificación rota). Plan:
  borrar todo vía `curl` contra los endpoints reales (no a mano en
  Atlas, para no romper referencias con `Sesion`/`ProgresoEstudiante`),
  luego volver a crear los 4 exámenes con opciones aleatorizadas y
  volver a subir los PDFs corregidos.
- Renombrar `Sesion.titulo` de las 4 sesiones (siguen en "Sesión
  1"..."Sesión 4") a los temas reales — separado del pendiente de
  arriba, pero buen momento para hacerlo junto ya que se va a tocar
  contenido de las mismas sesiones de todas formas.
- Construir una UI de admin para editar `configuracion` (lo que NO sean
  precios de plan, que ya se resolvió — ver `Plan` arriba) en vez de
  cambiarlos a mano en Atlas.
- Afinar el rol `backup_readonly` en Atlas de `readAnyDatabase@admin` a
  un rol Read específico sobre `mav_rd` (no urgente).
- **NUEVO:** monitorear si con más choferes hace falta algún
  filtro/paginación en `GET /instructores/activos` — hoy devuelve todos
  los activos sin distinción, suficiente para la cantidad actual. Nota:
  con la purga del 07/09, hoy no hay ningún instructor activo — hace
  falta crear uno de nuevo cuando se retome la prueba de práctica.
- **CONSTRUIDO (08-09/09/2026): programas Escolar y Empresarial.**
  Colecciones `Grupo` y `CuestionarioEscolar` nuevas (ver
  secciones 22 y 23), `User.grupoId`, `Inscripcion.tipoPlan` con el valor
  nuevo `"grupo"`.
- **Motorizados y Pesados — en diseño (11/09/2026), base de esquema ya
  lista.** `Sesion.programaContenido` + índice compuesto ya
  implementados (sección 4). Falta: sembrar sus `Sesion` reales,
  decidir cantidad de sesiones, y resolver `Plan` para un programa sin
  práctica de manejo (nota en la sección 21). Nombre confirmado:
  **"Motorizados"**, no "Motoristas". Detalle completo en
  `ANALISIS_MOTORISTA_PESADOS.md`.
