# Fase 2 — Paso 1: Modelo canónico externo (`modelo.json`) — Plan de implementación

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Externalizar los datos del modelo a un archivo `modelo.json` que la app consume por `fetch` al iniciar, eliminando el array de instrumentos hardcodeado en `index.html`.

**Architecture:** Un único archivo `modelo.json` (fuente de verdad física) espeja la §10 del modelo maestro y agrega el catálogo completo de los 8 instrumentos. `index.html` lo trae con `fetch` en un `init()` asíncrono, lo guarda en un objeto global `MODELO`, asigna `data.instrumentos = MODELO.instrumentos` y luego renderiza. Si el `fetch` falla, muestra un mensaje de error claro.

**Tech Stack:** HTML/CSS/JS vanilla, single-page, Tailwind por CDN, íconos Lucide. Sin framework de tests (Standard Mode — verificación manual + validación de JSON por CLI). Se sirve siempre por HTTP (Vercel o servidor local), nunca por `file://`.

## Global Constraints

- El MVP debe simular estar completo: **sin sellos de validación** ni carteles de "dato a confirmar" en `modelo.json`. Las tasas se presentan como definitivas.
- Mantener el `id` string existente de cada instrumento (ej. `"uva-bna"`) como clave de matching. El id canónico 1–8 se agrega como campo aparte `idCanonico` (numérico).
- Los perfiles (`data.joven`, `data.perfilesJoven`), la matriz de elegibilidad y el fix de las contradicciones de Lucía quedan FUERA de alcance de este paso.
- Commits convencionales, sin atribución de IA.
- No usar `cat`/`grep`/`find`/`sed`/`ls` — usar `rg`/`fd`/`bat`.

---

### Task 1: Crear `modelo.json`

**Files:**
- Create: `modelo.json`

**Interfaces:**
- Produces: archivo `modelo.json` con claves de nivel superior `version`, `territorio`, `macro_jun2026`, `credito_uva_bna`, `caso_testigo`, `plusvalia_rivadavia`, `instrumentos`. `instrumentos` es un array de 8 objetos, cada uno con: `idCanonico` (number 1–8), `id` (string slug), `categoria`, `tipo`, `nombre`, `sub`, `tasa`, `tasaNum` (number), `plazo`, `plazoNum` (number), `monto`, `requisitos`, `custodia`, `regulador`, `icon`, `accent`, `matchOnAccept` (boolean).

- [ ] **Step 1: Crear el archivo `modelo.json` con el contenido completo**

```json
{
  "version": "fase0-2026-06-16",
  "territorio": {
    "partido": "Rivadavia",
    "cabecera": "América",
    "provincia": "Buenos Aires",
    "poblacion_partido_2022": 19849,
    "var_2010_2022": 0.158,
    "poblacion_america_2022": 13843,
    "var_america": 0.184,
    "intendente": "Juan Alberto Martínez"
  },
  "macro_jun2026": {
    "uva_ars": 1992.43,
    "dolar_mep_ars": 1455,
    "construccion_usd_m2": { "base": 1000, "estandar": 1375, "3dcp": 725 },
    "smvm_ars": 367800,
    "ripte_ars": 1775664,
    "ripte_usd": 1220
  },
  "credito_uva_bna": {
    "tasa_tna": 0.06,
    "plazo_max_anios": 30,
    "cuota_ingreso_max": 0.25,
    "ltv": 0.75,
    "tope_cuota_cvs": true
  },
  "caso_testigo": {
    "m2": 50,
    "costo_construccion_usd": 50000,
    "lote": "aporte_municipal",
    "prestamo_usd": 37500,
    "prestamo_uva": 27400,
    "cuota_20a_usd": 269,
    "cuota_30a_usd": 225,
    "cuota_ingreso_hogar_1700": 0.158,
    "cierra": true
  },
  "plusvalia_rivadavia": {
    "ordenanza": "3.803/2016",
    "captura_lotes": 0.14,
    "min_en_especie": 0.50
  },
  "instrumentos": [
    {
      "idCanonico": 1,
      "id": "lote-mi-tierra",
      "categoria": "suelo",
      "tipo": "Suelo",
      "nombre": "Lote · Programa \"Rivadavia, Mi Tierra\"",
      "sub": "Municipio de Rivadavia · 83 lotes (2025)",
      "tasa": "50% financiado",
      "tasaNum": 0,
      "plazo": "En cuotas",
      "plazoNum": 5,
      "monto": "Lote urbanizado",
      "requisitos": "Residente del partido · 1ª vivienda · cupos sociales, discapacidad y personas solas",
      "custodia": "Banco de Tierras municipal",
      "regulador": "Ord. 3.907/2016 · \"Mi Tierra\" 2025",
      "icon": "map-pin",
      "accent": "accent",
      "matchOnAccept": true
    },
    {
      "idCanonico": 2,
      "id": "banco-tierras",
      "categoria": "suelo",
      "tipo": "Suelo",
      "nombre": "Lote del Banco de Tierras (plusvalía)",
      "sub": "Captura del 14% · Ord. 3.803/2016",
      "tasa": "Aporte / equity",
      "tasaNum": 0,
      "plazo": "—",
      "plazoNum": 1,
      "monto": "Computa como anticipo",
      "requisitos": "Adjudicación por sorteo público entre quienes superan el umbral objetivo",
      "custodia": "Fondo de Desarrollo Urbano (Ord. 4.188/2019)",
      "regulador": "Municipio de Rivadavia · Ley 14.449",
      "icon": "recycle",
      "accent": "navy",
      "matchOnAccept": true
    },
    {
      "idCanonico": 3,
      "id": "uva-bna",
      "categoria": "credito",
      "tipo": "Crédito",
      "nombre": "Crédito UVA Banco Nación \"+Hogares\"",
      "sub": "Hipotecario 1ª vivienda",
      "tasa": "6% TNA fija",
      "tasaNum": 6,
      "plazo": "30 años",
      "plazoNum": 30,
      "monto": "LTV 75% · hasta 260.000 UVA",
      "requisitos": "Cuota ≤ 25% del ingreso · relación de dependencia · Opción Tope de Cuota por CVS",
      "custodia": "Banco de la Nación Argentina",
      "regulador": "BCRA",
      "icon": "landmark",
      "accent": "accent",
      "matchOnAccept": true
    },
    {
      "idCanonico": 4,
      "id": "uva-bapro",
      "categoria": "credito",
      "tipo": "Crédito",
      "nombre": "Crédito Hipotecario Banco Provincia",
      "sub": "Línea vivienda PBA · condiciones 2026 a confirmar",
      "tasa": "UVA",
      "tasaNum": 7,
      "plazo": "20 años",
      "plazoNum": 20,
      "monto": "Bonificación si la cuota supera el 35% del ingreso",
      "requisitos": "Residencia en PBA · ingresos formales (condiciones puntuales por confirmar)",
      "custodia": "Banco de la Provincia de Buenos Aires",
      "regulador": "BCRA",
      "icon": "building-2",
      "accent": "navy",
      "matchOnAccept": false
    },
    {
      "idCanonico": 5,
      "id": "circulo-cerrado",
      "categoria": "credito",
      "tipo": "Municipal",
      "nombre": "Círculo Cerrado Municipal",
      "sub": "Para hogares que no califican al banco",
      "tasa": "Sin spread",
      "tasaNum": 4,
      "plazo": "20 años",
      "plazoNum": 20,
      "monto": "El municipio financia la construcción",
      "requisitos": "Cuota indexada por CVS (modelo Trenque Lauquen, Ord. 891/94)",
      "custodia": "Municipio de Rivadavia",
      "regulador": "Ley 14.449 PBA",
      "icon": "users",
      "accent": "accent",
      "matchOnAccept": true
    },
    {
      "idCanonico": 6,
      "id": "construccion-3d",
      "categoria": "construccion",
      "tipo": "Construcción",
      "nombre": "Construcción 3D (3DCP)",
      "sub": "Envolvente en 48 hs · −30% de costo",
      "tasa": "USD 700–750/m²",
      "tasaNum": 0,
      "plazo": "≈ 3,5 meses",
      "plazoNum": 1,
      "monto": "Vivienda 50 m² ≈ USD 37.500",
      "requisitos": "⚠️ Requiere Certificado de Aptitud Técnica (CAT) para acceder a hipoteca",
      "custodia": "Operador privado / cooperativa",
      "regulador": "INTI · Secretaría de Hábitat",
      "icon": "printer",
      "accent": "navy",
      "matchOnAccept": false
    },
    {
      "idCanonico": 7,
      "id": "fideicomiso-rwa",
      "categoria": "fondeo",
      "tipo": "Fondeo",
      "nombre": "Fideicomiso / Tokenización RWA",
      "sub": "Capa opcional de escala · no es requisito del hogar",
      "tasa": "Inversión colectiva",
      "tasaNum": 0,
      "plazo": "Por proyecto",
      "plazoNum": 1,
      "monto": "Hasta 1.500.000 UVA por emisión",
      "requisitos": "Financiamiento colectivo con autorización automática · KYC/AML",
      "custodia": "Fiduciario habilitado · PSAV",
      "regulador": "CNV · RG 1069/2025, 1087/2025 y 1125/2026",
      "icon": "coins",
      "accent": "accent",
      "matchOnAccept": false
    },
    {
      "idCanonico": 8,
      "id": "coop-electrica",
      "categoria": "credito",
      "tipo": "Cooperativa",
      "nombre": "Coop. Eléctrica de América — Plan habitacional",
      "sub": "Socios consumidores · 28 viviendas entregadas",
      "tasa": "A confirmar",
      "tasaNum": 5,
      "plazo": "10 años",
      "plazoNum": 10,
      "monto": "Adjudicación por sorteo",
      "requisitos": "Socio activo (actividad en vivienda a confirmar con la entidad)",
      "custodia": "Cooperativa Eléctrica de América",
      "regulador": "INAES",
      "icon": "zap",
      "accent": "navy",
      "matchOnAccept": true
    }
  ]
}
```

- [ ] **Step 2: Validar que es JSON sintácticamente correcto**

Run: `python3 -m json.tool modelo.json > /dev/null && echo OK`
Expected: imprime `OK` (sin errores de parseo).

- [ ] **Step 3: Verificar que hay exactamente 8 instrumentos con id canónico**

Run: `rg -c '"idCanonico"' modelo.json`
Expected: `8`

- [ ] **Step 4: Commit**

```bash
git add modelo.json
git commit -m "feat: agregar modelo.json canónico con datos del modelo y catálogo de instrumentos"
```

---

### Task 2: Cablear `index.html` para consumir `modelo.json`

**Files:**
- Modify: `index.html` (objeto `data`, líneas ~1079-1224; función `init` y su listener, líneas ~2105-2115)

**Interfaces:**
- Consumes: `modelo.json` (Task 1) vía `fetch`. Usa `MODELO.instrumentos`.
- Produces: variable global `MODELO`; `data.instrumentos` poblado en tiempo de ejecución desde `MODELO.instrumentos`; función `init()` asíncrona; función `showFetchError()`.

- [ ] **Step 1: Reemplazar el array de instrumentos hardcodeado por uno vacío**

En `index.html`, dentro del objeto `const data = {`, reemplazar todo el bloque que empieza en `instrumentos: [` (línea ~1079) y termina en su `],` de cierre (línea ~1224, justo antes de `perfilesJoven: [`) por una sola línea:

```js
      instrumentos: [],
```

Dejar intactos `joven:` (arriba) y `perfilesJoven:` (abajo). El array de instrumentos ahora vive en `modelo.json`.

- [ ] **Step 2: Reemplazar el bloque `init` y su listener por la versión asíncrona con fetch**

En `index.html`, reemplazar este bloque (líneas ~2105-2115):

```js
    function init() {
      renderBottomNavs();

      if (state.role === 'joven') showScreen('screen-joven-perfil');
      else if (state.role === 'inversor') showScreen('screen-inv-perfil');
      else showScreen('screen-splash');

      refreshIcons();
    }

    document.addEventListener('DOMContentLoaded', init);
```

por:

```js
    let MODELO = null;

    async function loadModelo() {
      const res = await fetch('modelo.json');
      if (!res.ok) throw new Error('HTTP ' + res.status);
      return res.json();
    }

    function showFetchError() {
      const device = document.getElementById('device');
      if (!device) return;
      device.innerHTML =
        '<div style="padding:32px;text-align:center;font-family:sans-serif;color:#0a1d33">' +
        '<p style="font-weight:700;font-size:18px;margin-bottom:8px">No se pudo cargar el modelo de datos</p>' +
        '<p style="font-size:14px;color:#64748b">Revisá que <code>modelo.json</code> esté disponible y recargá la página.</p>' +
        '</div>';
    }

    async function init() {
      try {
        MODELO = await loadModelo();
        data.instrumentos = MODELO.instrumentos;
      } catch (err) {
        console.error('Error cargando modelo.json:', err);
        showFetchError();
        return;
      }

      renderBottomNavs();

      if (state.role === 'joven') showScreen('screen-joven-perfil');
      else if (state.role === 'inversor') showScreen('screen-inv-perfil');
      else showScreen('screen-splash');

      refreshIcons();
    }

    document.addEventListener('DOMContentLoaded', init);
```

- [ ] **Step 3: Servir la app y verificar el camino feliz**

Run: `python3 -m http.server 8000` (dejar corriendo) y abrir `http://localhost:8000` en el navegador.
Expected:
- La app carga (splash → elegir rol "joven" → perfil → feed).
- El feed de instrumentos muestra las 8 tarjetas, con los mismos nombres/tasas que antes (Lote Mi Tierra, Banco Nación 6% TNA, etc.).
- En el detalle de un instrumento de crédito (ej. Banco Nación), la cuota mensual se calcula y se muestra (no aparece `NaN` ni vacío).

- [ ] **Step 4: Verificar el camino de error**

Run: renombrar temporalmente el archivo — `mv modelo.json modelo.json.bak` — recargar `http://localhost:8000`.
Expected: aparece el mensaje "No se pudo cargar el modelo de datos" dentro del dispositivo, NO una pantalla en blanco. En consola del navegador se ve `Error cargando modelo.json:`.
Luego restaurar: `mv modelo.json.bak modelo.json` y recargar para confirmar que vuelve a funcionar. Detener el servidor (Ctrl-C).

- [ ] **Step 5: Confirmar que ya no quedan instrumentos hardcodeados en index.html**

Run: `rg -c "lote-mi-tierra|uva-bna|circulo-cerrado" index.html`
Expected: `0` (los slugs de instrumentos ya no están en el HTML; viven solo en `modelo.json`).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: cargar instrumentos desde modelo.json con init asíncrono y manejo de error"
```

---

## Verificación final (manual)

1. Servir la app (`python3 -m http.server 8000`) y recorrer ambas vistas (joven e inversor).
2. Confirmar que todas las pantallas dibujan: splash, perfil joven, feed, matches, detalle, y vistas de inversor.
3. Confirmar que los 8 instrumentos y sus parámetros (tasa, plazo) coinciden exactamente con los valores de `modelo.json`.
4. Confirmar que el camino de error funciona (sin `modelo.json`, mensaje claro en vez de pantalla en blanco).
5. (Opcional, deploy) Push a `main` y verificar que el deploy de Vercel carga `modelo.json` correctamente desde la URL de producción.

## Notas / riesgos conocidos

- Redundancia menor dentro de `modelo.json`: `credito_uva_bna.tasa_tna` (0.06) y el instrumento `uva-bna.tasaNum` (6) representan el mismo dato. Aceptable para el MVP porque ya viven en UN solo archivo físico; unificar deriva queda para un paso posterior.
- Las contradicciones del perfil de Lucía (monto/superficie/score) NO se tocan en este paso — son datos de perfil, fuera de alcance.
- Asegurar que Vercel sirva `modelo.json` en la raíz (ruta relativa `modelo.json`, misma carpeta que `index.html`). Como es un sitio estático, debería servirse sin configuración extra.
