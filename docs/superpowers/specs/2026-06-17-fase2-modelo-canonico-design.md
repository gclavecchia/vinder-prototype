# Fase 2 — Paso 1: Modelo canónico externo (`modelo.json`)

**Fecha:** 2026-06-17
**Estado:** Diseño aprobado, pendiente de revisión del spec
**Contexto:** Fase 2 de la arquitectura de tres fases (Fase 0 = modelo maestro `docs/nucleo-modelo-america.md`; Fase 1 = documento de política; Fase 2 = conectar la app Vinder a los datos sustantivos del modelo).

## Problema

Los datos de la app (`index.html`) están hardcodeados y duplicados en múltiples lugares, lo que produce **drift** (deriva): copias del mismo dato que se separan con el tiempo. Síntomas confirmados en el análisis de brechas:

- Contradicciones internas (perfil Lucía: monto 54,6M en `data` vs $80M en UI; superficie 50m² vs 60m²; score 8.3 vs 7.8).
- Drift entre el documento maestro (`nucleo-modelo-america.md` §10) y la app: dos copias de los mismos números.
- Tasas y parámetros sin única fuente.

## Objetivo de este paso

Establecer una **única fuente de verdad física** para los datos del modelo: un archivo `modelo.json` que la app consume al iniciar. Eliminar los números hardcodeados que existan en el modelo, haciendo que la UI y la lógica lean de un solo lugar.

**No-objetivos (fuera de alcance de este paso):**
- Reemplazar los 10 perfiles ficticios por arquetipos validados con el municipio.
- Implementar la matriz de elegibilidad (quién califica a qué instrumento).
- Resolver las contradicciones del perfil de Lucía (bug de perfiles, no del modelo).
- Sellos de validación en los datos (decisión: el MVP simula estar completo; ver memoria `fase2/sellos-validacion-pendiente`).

## Decisión de arquitectura

**Opción elegida: A — `modelo.json` externo + `fetch` al iniciar.**

La app siempre se ejecuta servida (Vercel o servidor local), nunca por `file://`, por lo que `fetch` de un archivo local funciona sin problemas de CORS. Esta opción es la única que logra que el documento y la app compartan una misma fuente física, eliminando el drift doc↔app en la raíz (las opciones B y C solo resuelven duplicados internos de la app).

## Contenido de `modelo.json`

Datos validados del modelo (espejo de §10 del modelo maestro + catálogo de instrumentos):

- `version` — string de versión del modelo (ej. `"fase0-2026-06-16"`).
- `territorio` — partido, cabecera, provincia, población, variaciones, intendente.
- `macro_jun2026` — UVA, dólar MEP, costo construcción por tipo, SMVM, RIPTE.
- `credito_uva_bna` — tasa_tna, plazo_max_anios, cuota_ingreso_max, ltv, tope_cuota_cvs.
- `caso_testigo` — m², costo construcción, préstamo, cuotas 20a/30a, ratio cuota/ingreso, cierra.
- `plusvalia_rivadavia` — ordenanza, captura_lotes, min_en_especie.
- `instrumentos` — **catálogo completo de los 8 instrumentos** (id canónico 1–8, nombre, tasa, plazo, LTV y demás parámetros que la app necesita para mostrar y calcular). Este es el agregado clave: hoy §10 solo tiene los IDs; acá sube el catálogo entero al dato canónico. Los valores se presentan como definitivos (sin sellos de validación, por decisión de MVP).

## Contenido que queda FUERA de `modelo.json` (a propósito)

- Los 10 perfiles de jóvenes (ficción demostrativa): siguen como datos demo en la app.
- La matriz de elegibilidad: paso posterior.
- Lógica del Vinder Score y sus subscores: vive en el código; solo lee umbrales del modelo (ej. cuota máx 25%).
- Datos del inversor demo (Mutual Bonaerense): dato demo de la app.

## Flujo de consumo en la app

1. Al cargar, la app hace `fetch('modelo.json')`.
2. La pantalla de splash existente cubre el instante de carga (no se percibe demora).
3. Al resolver, el JSON se guarda en un objeto único `MODELO`.
4. Toda la UI y la lógica que hoy hardcodean valores presentes en el modelo (tasas, plazos, cuota máx 25%, LTV 75%, caso testigo, instrumentos) pasan a leer de `MODELO`.
5. Si el `fetch` falla, se muestra un mensaje de error claro (no pantalla en blanco).

## Componentes / unidades

- **`modelo.json`** — artefacto de datos. Una responsabilidad: contener el modelo canónico. Sin lógica.
- **Carga del modelo** (en `index.html`) — función que hace `fetch`, valida que llegó, expone `MODELO`, y dispara el render. Maneja el caso de error.
- **Render de instrumentos** — lee `MODELO.instrumentos` en vez del array hardcodeado actual.
- **Cálculo financiero** — lee tasas/plazos/LTV/cuota-máx de `MODELO` en vez de constantes hardcodeadas.

## Manejo de errores

- `fetch` falla o JSON inválido → mensaje de error visible y claro en pantalla, en lugar de fallar en silencio o quedar en blanco.

## Verificación

No hay framework de tests (HTML vanilla single-page). Verificación manual:

1. Servir la app (servidor local) y abrirla.
2. Confirmar que todas las pantallas dibujan (splash, perfil joven, feed, matches, detalle, vistas inversor).
3. Confirmar que los números mostrados (tasas, plazos, instrumentos, parámetros financieros) coinciden **exactamente** con los valores de `modelo.json`.
4. Forzar un error de `fetch` (renombrar temporalmente `modelo.json`) y confirmar que aparece el mensaje de error, no una pantalla en blanco.

## Riesgos

- Algún valor hardcodeado en la UI que NO exista en el modelo (ej. score de Lucía) queda fuera de alcance y puede seguir mostrando inconsistencias de perfiles. Documentado como paso siguiente.
- Asegurar que `modelo.json` se sirva correctamente en Vercel (ruta relativa correcta).
