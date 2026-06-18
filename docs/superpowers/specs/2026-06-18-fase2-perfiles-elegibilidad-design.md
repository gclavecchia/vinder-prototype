# Fase 2 — Paso 2: Perfiles, elegibilidad, datos territoriales y selector de usuario

**Fecha:** 2026-06-18
**Estado:** Diseño, pendiente de revisión del spec
**Contexto:** Continúa la Fase 2 (conectar Vinder a los datos del modelo). El paso 1 ([modelo.json canónico](2026-06-17-fase2-modelo-canonico-design.md)) ya externalizó los instrumentos. Este paso cubre los items 1–4 del roadmap de brechas + 20 perfiles + selector de usuario para demo.

## Objetivos

1. **Fix de contradicciones de Lucía** (item 1) — eliminar la duplicación entre `data.joven` y el perfil de la lista.
2. **Publicar datos territoriales y macro** (item 2) — mostrar en la app `MODELO.territorio` y `MODELO.macro_jun2026`, que hoy se cargan pero no se usan.
3. **Matriz de elegibilidad** (item 3) — cada perfil declara a qué instrumentos califica; el feed del joven se filtra por eso.
4. **Arquetipos reales** (item 4) — reemplazar los perfiles ficticios por 20 arquetipos anclados en la economía real de Rivadavia.
5. **Selector de usuario** — poder cambiar el joven activo en vivo durante la demo, mostrando perfiles/scores/matches distintos.

## No-objetivos (fuera de alcance)

- Inventar instrumentos financieros: se mantienen los **8 reales validados** de `modelo.json`. NO se agregan instrumentos.
- Motor de reglas de elegibilidad en runtime (la elegibilidad es dato pre-calculado).
- Validación de ingresos reales con el municipio (los 20 perfiles son arquetipos demo plausibles, no personas reales).
- Reincorporar sellos de validación (sigue pendiente post-MVP, ver memoria `fase2/sellos-validacion-pendiente`).

## Arquitectura de datos

- **`perfiles.json`** (archivo nuevo, raíz) — los 20 perfiles de jóvenes. Cargado por `fetch` igual que `modelo.json`. Se mantiene SEPARADO de `modelo.json` porque son datos demo, no la verdad validada del modelo.
- **Elegibilidad como dato** — cada perfil lleva `instrumentosElegibles: [id…]` con los `id` string de instrumentos de `modelo.json`. Determinista y crafteado para ser creíble. El ratio cuota/ingreso se calcula en vivo (ingreso del perfil + tasa del instrumento).
- **Joven activo = un perfil de la lista** — se elimina `data.joven` como objeto duplicado. El joven logueado es `PERFILES[state.jovenActivoIdx]`. Esto resuelve el bug de Lucía por diseño (una sola fuente, sin copias que diverjan).

## Schema del perfil (cada uno de los 20)

```
{
  "id": "lucia",                      // slug único, clave de matching
  "name": "Lucía Fernández",
  "initials": "LF",
  "age": 27,
  "sector": "salud",                  // agro | salud | municipal | educacion | oficios | comercio
  "profession": "Médica generalista",
  "education": "UNLP",
  "ciudad": "América (Rivadavia)",
  "ingreso": { "monto": 1300000, "tipo": "formal" },   // ARS/mes; formal | informal
  "hogar": "Pareja, sin hijos",
  "montoSolicitado": 54600000,        // ARS
  "plazo": 20,                        // años
  "superficie": 50,                   // m²
  "score": 8.3,                       // Vinder Score
  "subscores": [8.4, 9.0, 8.6, 7.0],  // [pago, suelo, anticipo, formalidad]
  "narrative": "Médica recién recibida...",
  "instrumentosElegibles": ["uva-bna", "lote-mi-tierra", "banco-tierras"],
  "calce": 92                         // % de calce para la vista inversor
}
```

Las etiquetas/detalles de los 4 subscores (Capacidad de pago, Acceso a suelo, Anticipo cubierto, Formalidad de ingresos) son **constantes en código** — el perfil solo aporta los 4 valores numéricos en el orden `[pago, suelo, anticipo, formalidad]`.

## Modelo de elegibilidad (criterio para craftear las listas)

La asignación de `instrumentosElegibles` por perfil sigue esta lógica (coherente con §5/§7 del modelo maestro), para que la demo sea creíble:

- `uva-bna` (Crédito UVA Banco Nación) → ingreso **formal** / relación de dependencia, con cuota proyectada ≤ 25%.
- `uva-bapro` (Banco Provincia) → ingreso formal, residencia PBA.
- `circulo-cerrado` (Círculo Cerrado Municipal) → para quienes **no** califican al banco (ingreso informal / monotributo).
- `lote-mi-tierra` y `banco-tierras` (suelo) → residentes del partido, 1ª vivienda — amplia elegibilidad.
- `construccion-3d` → opcional, para quien construye.
- `coop-electrica` → socios de la cooperativa.
- `fideicomiso-rwa` → capa de escala, opcional (no requisito del hogar).

Regla de oro de la demo: un perfil de ingreso formal y otro de ingreso informal deben mostrar sets de elegibilidad **claramente distintos**.

## Selector de usuario (demo)

- Botón "Cambiar usuario" en el header del perfil joven → abre una lista de los 20 (nombre, profesión, score).
- Al elegir, cambia `state.jovenActivoIdx` y se redibuja todo el lado joven (perfil, score, feed, matches).
- Es independiente del "Cambiar de rol" (joven ↔ inversor) ya existente.
- Al cambiar de usuario se resetea el estado de swipe/matches del joven (cada usuario arranca su feed desde cero).

## Feed del joven filtrado por elegibilidad

- El stack swipeable del joven muestra **solo** los instrumentos de `instrumentosElegibles` del joven activo (mapeados desde `MODELO.instrumentos` por `id`).
- El contador "Restantes" refleja la cantidad elegible de ese joven.
- Resultado: cambiar de usuario muestra oportunidades distintas — el núcleo de la demo.

## Datos territoriales y macro

- Card "Contexto territorial" en la pantalla de perfil joven, que lee de `MODELO.territorio` (población América, variación %, intendente) y un par de cifras de `MODELO.macro_jun2026` (UVA, costo construcción de referencia).
- Solo lectura, sin lógica. Refuerza la credibilidad del pitch.

## Vista inversor

- El feed del inversor muestra los 20 perfiles como oportunidades de inversión (mismas entidades, otra vista). Usa `calce` para ordenar/mostrar el % de match.
- Sin cambios de lógica más allá de leer la lista de 20 desde `perfiles.json`.

## Flujo de datos

1. `init()` (ya asíncrono) hace `fetch('modelo.json')` **y** `fetch('perfiles.json')` (en paralelo con `Promise.all`).
2. Guarda `MODELO` y `PERFILES` (array de 20).
3. `state.jovenActivoIdx = 0` por defecto.
4. Render del lado joven lee del perfil activo; el feed mapea `instrumentosElegibles` → objetos de `MODELO.instrumentos`.
5. Si cualquiera de los dos `fetch` falla → mensaje de error en `#device` (reusar `showFetchError`).

## Componentes / unidades

- **`perfiles.json`** — datos demo, sin lógica.
- **Carga** — extender `init()`/`loadModelo()` para traer ambos archivos.
- **`jovenActivo()`** — helper que devuelve `PERFILES[state.jovenActivoIdx]`. Todas las funciones de render del joven lo usan en vez de `data.joven`.
- **`instrumentosDelJoven()`** — helper que mapea `jovenActivo().instrumentosElegibles` a objetos de instrumento.
- **Render selector de usuario** — lista de 20 + handler de selección.
- **Render contexto territorial** — card que lee de `MODELO`.

## Manejo de errores

- `Promise.all` de los dos fetch dentro del `try`; si alguno rechaza, `showFetchError()` y `return` (sin pantalla en blanco).

## Verificación

Sin framework de tests (HTML vanilla). Verificación manual + smoke test en navegador (agent-browser, servido por HTTP):

1. La app carga ambos JSON; `PERFILES.length === 20`.
2. El perfil joven muestra al joven activo sin contradicciones (mismo monto/superficie/score en data y UI — fix de Lucía).
3. La card de contexto territorial muestra datos de `MODELO.territorio`.
4. El feed del joven muestra solo sus instrumentos elegibles; el contador coincide.
5. El selector de usuario cambia el joven activo y redibuja perfil/feed/matches; un perfil formal y uno informal muestran instrumentos elegibles distintos.
6. La vista inversor lista los 20 perfiles.
7. Camino de error: bloquear un fetch → mensaje de error, no pantalla en blanco.

## Riesgos

- Coherencia de los 20 perfiles: ingresos, scores y elegibilidad deben ser internamente consistentes (un score alto de "formalidad" no debería ir con ingreso informal). Se craftean con cuidado en la implementación.
- `index.html` ya es grande; agregar el selector y helpers debe seguir el patrón existente sin inflar de más.
- Cache del navegador sobre `perfiles.json` (mismo tema que `modelo.json`): considerar cache-busting en deploys futuros.
