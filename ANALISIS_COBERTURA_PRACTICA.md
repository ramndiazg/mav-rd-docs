# Análisis — Cobertura geográfica de la práctica de manejo (cerrado)

> Reemplaza la sección 3 (preguntas abiertas) de la primera versión de
> este documento por las decisiones ya tomadas contigo, y agrega el
> diseño técnico concreto para pasar de análisis a construcción. Sigue
> siendo un documento de análisis — nada de esto está construido
> todavía.

## Decisiones cerradas

1. **Nuevo plan, dentro de Categoría 2 (`programa: "estandar"`), sin
   práctica.** No existe hoy — los 3 planes actuales (Fundación/Normal/
   VIP) todos incluyen práctica. Se agrega una cuarta opción con
   `codigo: "teorico"` (mismo código que ya usan Motorizados/Pesados,
   pero ahora también bajo `programa: "estandar"` — son combinaciones
   distintas, no chocan). **Precio: lo defines tú después, editable
   desde `/admin/planes` una vez creado** — no hace falta decidirlo en
   este documento.
2. **Cuenta para el diploma igual que Motorizados/Pesados** — no exige
   `practicaAprobada`, mismo criterio ya centralizado en
   `utils/elegibilidadPractica.js`.
3. **Municipio se marca en el registro, y se usa en la inscripción de
   Categoría 2.** Si el municipio de la estudiante está en la lista de
   cobertura, ve los planes con práctica (igual que hoy); si no, ve solo
   el plan teórico. La lista de municipios con cobertura vive en un
   panel de admin (selector de municipios), para que crecerla no
   dependa de tocar código ni Atlas a mano.
4. **Cuentas de prueba existentes: sin importancia**, se van a borrar.
   No hace falta un plan de migración para ellas.
5. **Sin recálculo automático si la cobertura crece después de que
   alguien ya se inscribió** — es un caso raro, se resuelve a mano fuera
   de la app si llega a pasar.
6. **Cobertura editable desde el panel de admin.**

## Diseño de datos

### 1. Referencia — provincias y municipios de RD

Lista completa de las 32 provincias (31 + Distrito Nacional) con sus
municipios reales — dato público, fuente única para poblar los
`<select>` encadenados en `/registro` y `/inscripcion`. Se construye
como archivo de datos en el backend (`src/data/municipiosRD.js`) y se
expone por un endpoint público de solo lectura:

```
GET /api/ubicaciones/provincias-municipios
```

Sin autenticación (es dato estático, no sensible), sin rate limit
especial. El Distrito Nacional se modela con un único "municipio" del
mismo nombre, ya que no se subdivide administrativamente.

Al construir esto voy a sacar la lista real de una fuente oficial
(Junta Central Electoral / ONE) en vez de escribirla de memoria — son
~158 municipios y un error ahí sería silencioso (nadie nota que falta
un municipio hasta que una estudiante de ese lugar no lo encuentra en
el `<select>`).

### 2. `User` — nuevo campo `municipio`

```js
municipio: { type: String, default: null }
```

Se pide junto a `provincia` en `/registro` (mismo `<select>` en cascada:
elegir provincia habilita y filtra el `<select>` de municipio). No se
migra nada para cuentas viejas (decisión 4).

### 3. `MunicipioPractica` — colección nueva (whitelist de cobertura)

```js
{
  _id: ObjectId,
  provincia: String,
  municipio: String,
  activo: Boolean,     // permite desactivar sin borrar, mismo patrón
                        // que Plan.activo / User.activo
  createdAt, updatedAt
}
```

Índice único compuesto `{ provincia, municipio }`. Solo contiene **las
filas que en algún momento tuvieron cobertura** — no se siembra con los
158 municipios en `false`, se agrega uno nuevo cuando de verdad se
habilita. Seed inicial (`scripts/sembrarCoberturaPractica.js`), los 8 ya
confirmados:

| provincia         | municipio           |
| ----------------- | ------------------- |
| Distrito Nacional | Distrito Nacional   |
| Santo Domingo     | Santo Domingo Este  |
| Santo Domingo     | Santo Domingo Oeste |
| Santo Domingo     | Santo Domingo Norte |
| Santiago          | Santiago            |
| San Cristóbal     | San Cristóbal       |
| Santiago          | Navarrete           |
| Monseñor Nouel    | Bonao               |

### 4. `Plan` — nueva fila

`{ programa: "estandar", codigo: "teorico", precio: <a definir>, ... }`
— mismos campos opcionales que ya tiene el `teorico` de Motorizados/
Pesados (`modalidadPractica`/`cantidadSesionesPractica`/
`duracionSesionMinutos`/`costoPorSesion` en `required: false`, no
aplican). Sembrado por script, editable después desde `/admin/planes`.

### 5. `ProgresoEstudiante` — nuevo campo `tipoPlan`

Hoy solo espeja `programa` desde `Inscripcion` al confirmar el pago. Se
agrega el mismo espejo para `tipoPlan`:

```js
tipoPlan: { type: String, default: null }
```

Necesario porque, a diferencia de Motorizados/Pesados (donde **todo** el
programa es teórico), en `estandar` conviven planes con y sin práctica
— el criterio ya no puede depender solo de `programa`.

## Cambios de lógica

### `utils/elegibilidadPractica.js`

```js
function requierePracticaDeManejo(usuario, programa, tipoPlan) {
  const tieneGrupo = Boolean(usuario?.grupoId);
  const esProgramaSinPractica = Boolean(
    programa && PROGRAMAS_SIN_PRACTICA.includes(programa),
  );
  const esPlanTeorico = tipoPlan === "teorico";
  return !tieneGrupo && !esProgramaSinPractica && !esPlanTeorico;
}
```

Se agrega un tercer parámetro opcional (no rompe llamadas viejas si
alguna quedara sin actualizar, aunque los 4 call sites reales sí se
actualizan para pasar `progreso.tipoPlan`). `PROGRAMAS_SIN_PRACTICA` se
queda igual, por claridad y porque no cuesta nada mantenerlo, aunque
técnicamente ahora `esPlanTeorico` ya cubriría también a Motorizados/
Pesados.

### `inscripcionController.js`

En `crearOReenviarInscripcionPropia`, cuando `programa === "estandar"`:

1. Buscar `MunicipioPractica.exists({ provincia: usuario.provincia, municipio: usuario.municipio, activo: true })`.
2. Si existe → `tipoPlan` válido: `["fundacion", "normal", "vip"]` (igual
   que hoy).
3. Si no existe → `tipoPlan` válido: `["teorico"]` únicamente — cualquier
   otro valor se rechaza con 400, aunque el frontend ya lo esté
   ocultando (la validación real vive en el backend, el frontend solo
   evita que alguien lo intente por accidente).
4. Si `usuario.municipio` no está seteado (cuenta vieja que nunca lo
   llenó): tratarlo igual que "no cubierto" — mejor pedirle el dato
   antes de dejarla avanzar que asumir cobertura sin saberlo.

Al confirmar el pago, donde ya se propaga `programa` a
`ProgresoEstudiante`, se propaga también `tipoPlan`.

### `GET /api/planes` (o el fetch que arma `/inscripcion`)

El frontend ya filtra planes por `programa`; se agrega un filtro
adicional por disponibilidad de práctica una vez que se sabe si el
municipio de la estudiante está cubierto — puede resolverse en el
propio frontend (trae los 4 planes de `estandar`, elige cuáles mostrar
según la respuesta de `GET /api/ubicaciones/.../cobertura` o un campo
que devuelva el propio perfil de la estudiante).

### Nuevo: `municipioPracticaController.js` + rutas (admin)

Mismo patrón que `planController.js`/`admin/planes`:

- `GET /api/municipios-practica` — admin, lista completa (activos e
  inactivos).
- `POST /api/municipios-practica` — admin, agrega `{ provincia, municipio }`.
- `PATCH /api/municipios-practica/:id` — admin, activar/desactivar.
- Endpoint público liviano para que `/inscripcion` (o el propio backend
  al validar) consulte si un `{ provincia, municipio }` puntual está
  cubierto, sin exponer la lista completa a cualquiera si no hace falta.

### Frontend

- **`/registro`**: el `<select>` de provincia ya existe; se agrega un
  segundo `<select>` de municipio, poblado desde
  `GET /api/ubicaciones/provincias-municipios` y filtrado por la
  provincia elegida (deshabilitado hasta que haya provincia).
- **`/inscripcion`**: cuando `programa === "estandar"`, antes de mostrar
  los planes, resolver si el municipio de la estudiante está cubierto.
  Si sí → los 3 planes de siempre. Si no → un mensaje explicando que la
  práctica todavía no está disponible en su municipio, y solo la opción
  teórica (mismo estilo visual que ya usa el plan `teorico` de
  Motorizados/Pesados).
- **`dashboard/page.tsx`**: `requierePractica` necesita el mismo tercer
  criterio (`progreso.tipoPlan === "teorico"`), para no mostrarle la
  pantalla de instructores a quien no la necesita.
- **Nuevo `/admin/cobertura-practica`**: lista de municipios habilitados
  (activo/inactivo, con botón para desactivar) + formulario para agregar
  uno nuevo (mismos `<select>` encadenados de provincia→municipio).
  Mismo patrón visual que `/admin/planes`.

## Orden de construcción sugerido

1. Datos de referencia (`municipiosRD.js` + endpoint) — todo lo demás
   depende de esto.
2. `User.municipio` + `<select>` en `/registro`.
3. `MunicipioPractica` (modelo + controller + rutas + seed de los 8) +
   pantalla `/admin/cobertura-practica`.
4. `Plan` nuevo (`estandar`/`teorico`) — script de siembra.
5. `elegibilidadPractica.js` + `ProgresoEstudiante.tipoPlan` +
   `inscripcionController.js` (validación real).
6. `/inscripcion` (frontend) — mostrar/ocultar planes según cobertura.
7. `dashboard/page.tsx` — tercer criterio de `requierePractica`.
8. Prueba de punta a punta: una cuenta con municipio cubierto ve los 3
   planes de siempre; una con municipio no cubierto solo ve el teórico,
   completa el curso y genera diploma sin pedir práctica.

## Fuera de alcance (sigue igual que la v1 de este análisis)

- Precio del plan teórico de `estandar` — lo defines tú en
  `/admin/planes`.
- Contenido de `Sesion`/`Examen` de `estandar` — sin cambios, la
  estudiante ve el mismo contenido teórico que ya existe.
- Escolar/Empresarial — sin cambios, no pasan por este flujo.

Anexo final:

Cuando el municipio SÍ tiene cobertura — un aviso corto, positivo, antes de las 3 tarjetas de siempre:

✅ Buenas noticias: en [Municipio] sí está disponible la práctica de manejo presencial. Puedes elegir cualquiera de estos planes.

Cuando el municipio NO tiene cobertura — reemplaza las 3 tarjetas por un aviso explicativo + la tarjeta teórica, no solo se ocultan las otras en silencio:

ℹ️ Por ahora, en [Municipio] no tenemos disponible la práctica de manejo presencial. Puedes inscribirte en la modalidad Solo Teórico: mismo curso, mismas 4 sesiones y examen, con diploma al terminar.
