# Cobertura de práctica — guía de instalación (13/09/2026)

Implementación completa del `ANALISIS_COBERTURA_PRACTICA.md`. 23 archivos:
17 en backend, 6 en frontend.

Las carpetas `backend/` y `frontend/` de esta descarga replican la
estructura exacta de cada repo — puedes copiar respetando las rutas.

---

## BACKEND — `mav-rd-backend`

### Archivos NUEVOS (7)

| Archivo                                          | Qué hace                                                     |
| ------------------------------------------------ | ------------------------------------------------------------ |
| `src/data/municipiosRD.js`                       | 32 provincias / 160 municipios. Fuente única.                |
| `src/controllers/ubicacionesController.js`       | Sirve el dato anterior.                                      |
| `src/routes/ubicacionesRoutes.js`                | `GET /api/ubicaciones/provincias-municipios` (pública).      |
| `src/models/MunicipioPractica.js`                | Whitelist de cobertura. Índice único provincia+municipio.    |
| `src/controllers/municipioPracticaController.js` | Listar / agregar / activar-desactivar / consultar cobertura. |
| `src/routes/municipioPracticaRoutes.js`          | `/api/municipios-practica` (admin) + `/cobertura` (pública). |
| `scripts/sembrarCoberturaPractica.js`            | Siembra los 8 municipios iniciales.                          |
| `scripts/sembrarPlanEstandarTeorico.js`          | Crea el plan `estandar`/`teorico`.                           |

### Archivos MODIFICADOS (9)

| Archivo                                      | Cambio                                                                              |
| -------------------------------------------- | ----------------------------------------------------------------------------------- |
| `src/models/User.js`                         | + campo `municipio` (default `null`).                                               |
| `src/models/ProgresoEstudiante.js`           | + campo `tipoPlan` (espejo de Inscripcion).                                         |
| `src/utils/elegibilidadPractica.js`          | Tercer parámetro `tipoPlan`; `"teorico"` ⇒ no requiere práctica.                    |
| `src/controllers/authController.js`          | `registro` acepta, valida y guarda `municipio`.                                     |
| `src/controllers/inscripcionController.js`   | `"teorico"` válido en `estandar`; valida cobertura; propaga `tipoPlan` al progreso. |
| `src/controllers/diplomaController.js`       | 2 call sites actualizados.                                                          |
| `src/controllers/practicaController.js`      | 1 call site actualizado.                                                            |
| `src/controllers/intentoExamenController.js` | 1 call site actualizado.                                                            |
| `src/app.js`                                 | Monta los 2 routers nuevos.                                                         |

---

## FRONTEND — `mav-rd-frontend`

### Archivo NUEVO (1)

| Archivo                                         | Qué hace                                                      |
| ----------------------------------------------- | ------------------------------------------------------------- |
| `app/(admin)/admin/cobertura-practica/page.tsx` | Panel para agregar municipios y activar/desactivar cobertura. |

### Archivos MODIFICADOS (5)

| Archivo                             | Cambio                                                                    |
| ----------------------------------- | ------------------------------------------------------------------------- |
| `contexts/AuthContext.tsx`          | `Usuario` gana `provincia`/`municipio`; `DatosRegistro` gana `municipio`. |
| `app/registro/page.tsx`             | Select provincia→municipio en cascada.                                    |
| `app/inscripcion/page.tsx`          | Consulta cobertura, filtra planes, muestra aviso.                         |
| `app/dashboard/page.tsx`            | Tercer criterio en `requierePractica`.                                    |
| `app/(coordinadora)/panel/page.tsx` | + tarjeta "Cobertura de práctica".                                        |

---

## Orden de despliegue

1. **Copia el backend** y despliega. Los endpoints nuevos son aditivos; nada se rompe.
2. **Siembra el plan teórico:**
   ```bash
   node scripts/sembrarPlanEstandarTeorico.js              # dry-run
   node scripts/sembrarPlanEstandarTeorico.js --confirmar  # real (precio RD$0)
   ```
   Luego ponle precio real desde `/admin/planes`, o siembra con `--precio=2500`.
3. **Siembra la cobertura:**
   ```bash
   node scripts/sembrarCoberturaPractica.js              # dry-run
   node scripts/sembrarCoberturaPractica.js --confirmar  # real
   ```
4. **Copia el frontend** y despliega.

Importante: haz el paso 2 **antes** de que el frontend esté vivo. Si una
estudiante de un municipio sin cobertura entra a `/inscripcion` y el plan
`teorico` de `estandar` no existe todavía, se queda sin ninguna opción
seleccionable.

---

## Qué probar

- **Registro:** al elegir provincia aparecen sus municipios; al cambiar de
  provincia el municipio se limpia.
- **Inscripción, municipio cubierto** (ej. Santiago): se ven los 4 planes de livianos.
- **Inscripción, municipio sin cobertura** (ej. Higüey): solo "Solo Teórico" + aviso azul.
- **Backend rechaza el bypass:** con un usuario de municipio sin cobertura,
  un `POST /api/inscripciones/mia` con `tipoPlan: "vip"` debe dar 400.
- **Cuenta vieja sin municipio:** se trata como sin cobertura (mensaje distinto, pide contactar).
- **Admin:** `/admin/cobertura-practica` agrega un municipio y lo desactiva;
  los ya agregados no reaparecen en el desplegable.
- **Diploma teórico:** una estudiante con plan `teorico` que termina la
  teoría debe recibir diploma sin pasar por práctica, y no debe aparecer en
  la lista de pendientes de práctica.

---

## Verificaciones ya hechas

- Los 17 archivos backend pasan `node --check`.
- `src/app.js` carga sin error con los routers nuevos montados.
- `npx tsc --noEmit` limpio en el frontend.
- Las 32 provincias del backend coinciden **exactamente** con el array
  `PROVINCIAS` que ya existía en `app/registro/page.tsx` (comparado uno a
  uno, cero diferencias).
- Los 8 municipios del seed existen en la lista de referencia.
- `requierePracticaDeManejo` probada en 7 casos, incluido el llamado viejo
  sin `tipoPlan` (sigue devolviendo `true`, no rompe nada).

## Pendiente que debes saber

`src/data/municipiosRD.js` se compiló de fuentes públicas generales
(Wikipedia, ONE, statoids), **no** de un archivo oficial verificado línea
por línea de la JCE/ONE — no logré descargar uno en esta sesión. La lista
da 32 provincias y 160 municipios, que es el orden correcto, pero antes de
producción conviene un repaso, sobre todo en San Cristóbal, Monte Plata y
San Pedro de Macorís. Un error aquí es silencioso: nadie lo nota hasta que
una estudiante no encuentra su municipio en el desplegable. La nota está
también dentro del propio archivo.

Aparte: `"Navarrete"` se incluyó bajo Santiago aunque formalmente es un
distrito municipal, no un municipio — porque el seed lo usa como fila
propia y tiene que ser seleccionable.
