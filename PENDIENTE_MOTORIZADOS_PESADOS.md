# Pendiente — Motorizados y Pesados

> Reemplaza a `ANALISIS_MOTORISTA_PESADOS.md` (11/09/2026), que era un
> documento de análisis previo a construir el programa. Ya se construyó
> (13/09/2026) — el detalle de arquitectura vive en
> ARQUITECTURA_BACKEND.md, ARQUITECTURA_FRONTEND.md y DATABASE.md. Este
> archivo solo guarda lo que **todavía** queda pendiente, para no perder
> esas preguntas entre la masa de detalle ya resuelto.

## Qué falta

1. **Cargar contenido de estudio y exámenes reales** para las 4
   sesiones de Motorizados y de Pesados — hoy están sembradas pero
   vacías, solo con título provisional. Cargar **al menos 2-3 versiones
   de examen activas por sesión desde el día uno** (no una sola), para
   no repetir el problema que tuvo `estandar` de quedar con una sola
   versión y la respuesta siempre en la opción A.
2. **Contenido del diploma:** ¿debe decir explícitamente "Motorizados"
   o "Pesados" en el PDF, o alcanza el mismo diseño genérico de hoy? Si
   tiene que decir el programa, hay que revisar la plantilla real del
   PDF (no está en los repos de código, habría que ubicarla).
3. **Pesados — camión vs. trailer:** el alcance inicial es "camiones y
   trailers". Falta confirmar con la fundadora si comparten exactamente
   las mismas 4 sesiones o si en algún punto el contenido diverge. Si
   diverge, probablemente conviene un campo `subtipo` en vez de un
   tercer `programaContenido`.
4. **Pestaña informativa "Educación Vial Escolar"** (`/escolar`, mismo
   patrón que `/empresas`) — decidida pero no construida todavía.
5. **Probar de punta a punta** una vez haya contenido real: inscripción
   → pago → cuestionario → 4 sesiones → exámenes con selección
   aleatoria real (repetir el mismo intento varias veces y confirmar
   que rota de versión) → diploma sin pedir práctica.
