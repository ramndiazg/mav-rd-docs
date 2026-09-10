# Especificación: Programas Escolar, Empresarial y Motorista

> Creado el 07/09/2026. **Actualizado el 09/09/2026: Escolar y
> Empresarial ya están construidos y desplegados** (ver
> ARQUITECTURA_BACKEND.md sección "Programa Escolar/Empresarial" y
> HISTORIAL_MODIFICACIONES.md entrada 08-09/09/2026 para el detalle
> completo de qué se construyó y qué bugs salieron). Este documento
> queda vivo solo por **Motorista**, que sigue sin diseñarse — todo lo
> de las secciones 1-4 de abajo describe lo que ya se construyó (útil
> como referencia de las decisiones tomadas), no trabajo pendiente. Ve
> directo a la sección 5 para lo que realmente falta.
>
> Requisito previo, ya construido: el campo `programa` (`String, default:
"estandar"`) existe en `Plan` e `Inscripcion` desde el 07/09/2026 — ver
> ARQUITECTURA_BACKEND.md y DATABASE.md. Todo lo de este documento se
> apoya en ese campo.

## 0. Resumen de una línea

Escolar y Empresarial **no son un currículo nuevo** — son la misma teoría,
los mismos exámenes y el mismo test psicológico (con una excepción, ver
más abajo) que el programa `estandar`, pero **inscritos en bloque por una
institución** en vez de individualmente, **sin práctica de manejo**, y con
un **reporte de avance periódico** para el contacto de la institución en
vez de que el colegio/empresa entre a la app.

Motorista es una idea mencionada por la fundadora el 06/09/2026 (un curso
para motoristas/motorizados) — **no se ha diseñado a este nivel de
detalle todavía**. Es probable que sí necesite currículo y hasta práctica
distintos (manejar un carro y una motora no es lo mismo). Primer paso de
la próxima sesión: sostener con la fundadora la misma conversación de
descubrimiento que ya se tuvo para Escolar/Empresarial, antes de asumir
que aplica el mismo patrón de "sin currículo distinto".

## 1. Decisiones ya tomadas (no volver a preguntar)

- **Sin práctica de manejo** para Escolar y Empresarial — solo teórico,
  el gate del diploma salta directo al completar la teoría (4 sesiones +
  4 exámenes aprobados), sin pasar por `practicaAprobada`.
- **Test psicológico:** para Escolar, se reemplaza por un cuestionario
  informativo corto y distinto (ver sección 3) — no es una versión corta
  del mismo test, es una colección y un formulario aparte, sin el
  framing de "test psicológico". Empresarial **sí** usa el
  `TestPsicologico` completo, igual que `estandar` (no se excluyó
  explícitamente, así que aplica el default).
- **Precio por grupo:** un solo total negociado (`precioAcordado`), no
  por estudiante. Para contabilidad, se prorratea entre la cantidad real
  de estudiantes del roster (no la cantidad estimada al crear el grupo).
- **Las instituciones nunca entran a la app.** No hay rol nuevo, no hay
  RBAC con visibilidad restringida, no hay dashboard para colegio/empresa.
  Muvo (admin/coordinadora) crea las cuentas de los estudiantes.
- **Reporte de avance:** por correo, diario, a las 10:00 AM hora RD,
  empezando 24 horas después de creado el grupo, y **terminando
  automáticamente cuando todos los estudiantes del grupo tienen
  `cursoCompletado: true`** (no cuando tienen diploma generado, que
  puede ser un paso administrativo posterior).
- **Formulario de inscripción de grupo: en dos pasos, no uno.** Un
  formulario para los datos del grupo (institución, contacto, precio
  pactado, cantidad estimada de estudiantes) y otro, separado, para el
  roster real de estudiantes (idealmente con carga CSV/pegado, no solo
  fila por fila, por volumen). El pago se da por hecho al llenar el
  primer formulario, pero el prorrateo contable espera al roster real.
- **Discrepancia estimado vs. roster real:** se muestra como aviso, no
  bloqueante. Si dijeron "45 estudiantes" y el roster trae 42, se crea
  igual con 42.

## 2. Modelo de datos a construir

### `Grupo` (colección nueva)

```js
{
  _id: ObjectId,
  tipo: "colegio" | "empresa",
  nombreInstitucion: String,
  contactoNombre: String,
  contactoEmail: String,
  contactoTelefono: String,
  precioAcordado: Number,        // total negociado, se rellena a mano
  cantidadEstudiantesEstimada: Number, // del primer formulario, solo referencia
  fechaInicio: Date,              // se fija cuando se confirma el roster,
                                   // no cuando se crea el grupo — de ahí
                                   // cuentan las 24h para el primer reporte
  pendienteRoster: Boolean,       // true hasta que se cargue el roster real
  activo: Boolean,                // false cuando todos completan el curso
                                   // (apagado automático) o a mano como
                                   // respaldo
  notas: String,
  creadoPor: ObjectId,            // ref: users
  createdAt: Date, updatedAt: Date
}
```

### `User` — un campo nuevo, sin tocar nada existente

```js
grupoId: { type: mongoose.Schema.Types.ObjectId, ref: "Grupo", default: null }
```

`null` para todos los estudiantes que se autoregistran (el flujo actual,
sin cambios). Con valor solo para estudiantes de un `Grupo`.

### `ProgresoEstudiante` — gate del diploma condicional

En `diplomaController.js` (o donde esté la lógica de elegibilidad hoy),
el chequeo pasa de:

```js
const elegible = progreso.cursoCompletado && progreso.practicaAprobada;
```

a:

```js
const requierePractica = !usuario.grupoId;
const elegible =
  progreso.cursoCompletado && (!requierePractica || progreso.practicaAprobada);
```

Efectos en cascada a revisar al implementar:

- `entregarIntento()` no debe notificar "lista para práctica"
  (`notificarEstudianteListaParaPractica`) si `usuario.grupoId` existe.
- El paso visual de "Práctica" en `ProgresoCarretera.tsx` (dashboard)
  debe ocultarse para estudiantes con `grupoId`, no mostrarse vacío.

### Cuestionario informativo de Escolar (colección nueva, NO reusar `TestPsicologico`)

Nombre sugerido: `InformacionComplementariaEscolar` (ajustar si la
fundadora prefiere otro). Gate en `obtenerSesionParaEstudiante`: si el
`Grupo` del estudiante es `tipo: "escolar"`, exige esta colección en vez
de `TestPsicologico`; para todo lo demás (incluido Empresarial), sigue
exigiendo `TestPsicologico` igual que hoy.

**14 preguntas ya redactadas y acordadas** (escala 1-5, mismo formato
que el test original, más 2 abiertas) — deliberadamente sin ningún eje
que se parezca a los del test psicológico completo (nada de
autocontrol/estrés/emociones/percepción de riesgo):

_Conocimiento previo de educación vial (4):_ identificación de señales,
comprensión de marcas viales, si ya recibió alguna charla de seguridad
vial antes, si sabe los pasos para cruzar como peatón.

_Experiencia práctica como peatón/pasajero/ciclista (3):_ frecuencia de
caminar/transporte público al colegio, si ha andado en bici/motor en
calle real, si un adulto le ha explicado el cinturón de seguridad.

_Logística de aprendizaje (3):_ preferencia por video vs. texto, acceso
a internet/dispositivo en casa, mejor momento del día para concentrarse.

_Contexto de manejo en el hogar (2):_ si hay carro/motor de uso regular
en casa, si planea aprender a manejar al ser mayor de edad.

_Abiertas (2):_ qué le gustaría aprender del curso, cómo prefiere que le
avisen de un examen disponible.

**Nota de cautela, no resuelta:** que las preguntas sean "menores" no
las excluye automáticamente de la Ley 172-13 si alguna toca salud,
comportamiento o algo conductual — pasar el set final por la misma
revisión legal pendiente para el test completo antes de usarlo con
estudiantes reales (ver el pendiente legal ya existente en
ARQUITECTURA_BACKEND.md).

### Prorrateo contable

Al confirmar el roster (no al crear el grupo), generar N registros en
`MovimientosContables`, uno por estudiante, `monto = precioAcordado /
cantidadEstudiantesReal`. Si no divide exacto, el residuo va en el
primer registro (redondeo hacia abajo en los demás), para que la suma
cuadre con el total pactado. Si después se agrega más estudiantes al
mismo grupo (roster tardío), **no re-prorratear retroactivamente** —
tratar como movimiento aparte, para no editar movimientos ya cerrados.

## 3. Flujo de inscripción de grupo (2 formularios)

**Formulario 1 — Datos del grupo:** crea el `Grupo` con
`pendienteRoster: true`. No dispara nada más (ni cuentas, ni movimientos
contables).

**Formulario 2 — Roster** (aparte, mismo día o después, referencia al
`grupoId`): carga masiva (CSV/pegado) + fallback fila por fila para
grupos chicos o adiciones tardías. Al confirmar:

1. Crea las N cuentas de `User` (rol `estudiante`, `grupoId` seteado),
   credenciales enviadas por correo (reutilizar Resend, mismo patrón que
   verificación de cuenta) — requiere que el roster traiga el correo de
   cada estudiante, no hay forma de evitarlo si nunca entran ellos mismos
   a configurar nada.
2. Crea las `Inscripcion` correspondientes ya confirmadas (sin pasar por
   la cola de verificación de voucher — el pago ya se dio por hecho en
   el Formulario 1).
3. Genera los movimientos contables prorrateados (ver sección 2).
4. Fija `Grupo.fechaInicio = ahora`, `pendienteRoster: false`.
5. Si el conteo real difiere del estimado, mostrar aviso (no bloqueante)
   en la UI de confirmación.

## 4. Cron de reporte diario (10:00 AM, termina solo)

Mismo patrón que ya existe para `resumenDiario.js` (GitHub Action →
`POST` a endpoint protegido por `CRON_SECRET`), pero:

- Cron: `"0 14 * * *"` (10:00 AM AST = 14:00 UTC, RD no tiene DST).
- El endpoint recorre todos los `Grupo` con `activo: true` **y**
  `fechaInicio` de hace más de 24 horas.
- Por cada uno, calcula el progreso de _sus_ estudiantes
  (`User.grupoId`) — sesiones completadas, exámenes aprobados/pendientes,
  diplomas generados — y envía un correo aparte a
  `Grupo.contactoEmail` (fan-out, no un solo correo para todos).
- Si **todos** los estudiantes del grupo tienen
  `progreso.cursoCompletado: true`, ese envío se marca como reporte
  final y se pone `Grupo.activo = false` en la misma pasada (para que no
  vuelva a entrar en el cron del día siguiente). El toggle manual de
  `activo` sigue disponible como respaldo si quieren cortarlo antes.

## 5. Estado real al 09/09/2026 — qué falta de verdad

1. **Motorista: sigue sin diseñar.** Currículo distinto? Práctica en
   moto, con instructor propio? ¿Mismos 4 planes de precio o una
   estructura distinta? Empezar por una conversación de descubrimiento
   con la fundadora antes de asumir que aplica el mismo patrón de
   Escolar/Empresarial (que resultó ser "misma teoría, sin práctica,
   precio negociado en bloque" — Motorista probablemente NO calza en
   ese mismo patrón, ver sección 0).
2. Nombre final de la colección `InformacionComplementariaEscolar` —
   sigue sin confirmar con la fundadora si le gusta o prefiere otro
   (se construyó y desplegó con este nombre mientras tanto).
3. Confirmar con asesoría legal el set final de 14 preguntas de Escolar
   antes de usarlo con estudiantes reales — mismo pendiente que el test
   psicológico completo (Ley 172-13).
4. Probar en vivo el cron de reporte diario
   (`POST /api/interno/reporte-grupos`) — nunca se esperaron las 24h
   reales ni se disparó a mano desde GitHub Actions.
5. Revisar en Mongo Atlas si quedó una cuenta de estudiante huérfana de
   la prueba real donde salió el bug de `numeroReferencia` (ver
   ARQUITECTURA_BACKEND.md) — no se limpió todavía.

**Ya resuelto, quedaba abierto en la versión anterior de este
documento:**

- `Sesion`/`Examen`/`ContenidoSesion` de Escolar/Empresarial reusan las
  mismas colecciones de `estandar` con el campo `programa` (no hizo
  falta tocar el índice de `Sesion` — ese cambio queda pendiente para el
  día que algún programa sí necesite su propio currículo distinto).
- El precio de Escolar/Empresarial vive solo en `Grupo.precioAcordado`,
  nunca pasa por `Plan` — confirmado al construir (`Inscripcion.tipoPlan`
  usa el valor nuevo `"grupo"` en vez de una entrada en `Plan`).
