# Fase 2 — Paso 2: Perfiles, elegibilidad, selector y territorio — Plan de implementación

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reemplazar los perfiles ficticios y el objeto duplicado `data.joven` por 20 arquetipos reales de Rivadavia en `perfiles.json`; el joven activo pasa a ser `PERFILES[state.jovenActivoIdx]`, el feed se filtra por elegibilidad, y se agregan un selector de usuario para demo y una card de contexto territorial.

**Architecture:** `index.html` (single-file vanilla JS) carga `modelo.json` (ya existe, 8 instrumentos validados) y `perfiles.json` (nuevo, 20 perfiles) por `fetch` en paralelo dentro de `init()`. Globals `MODELO` y `PERFILES`. El joven logueado es `PERFILES[state.jovenActivoIdx]` (sin `data.joven` duplicado). El feed del joven mapea `instrumentosElegibles` → objetos de `MODELO.instrumentos`.

**Tech Stack:** HTML/CSS/JS vanilla, Tailwind por CDN, íconos Lucide. Sin framework de tests (Standard Mode — validación JSON por CLI + smoke test con agent-browser). Se sirve siempre por HTTP.

## Global Constraints

- **NO inventar instrumentos.** Los `id` de `instrumentosElegibles` deben ser exactamente uno o más de: `uva-bna`, `uva-bapro`, `circulo-cerrado`, `lote-mi-tierra`, `banco-tierras`, `construccion-3d`, `coop-electrica`, `fideicomiso-rwa`.
- **Elegibilidad como dato** pre-calculado en cada perfil (`instrumentosElegibles: [id…]`). No hay motor de reglas en runtime.
- **Joven activo = `PERFILES[state.jovenActivoIdx]`.** El objeto `data.joven` se elimina.
- **Subscores = array de 4 números** `[pago, suelo, anticipo, formalidad]`. Las etiquetas son constantes en código (`SUBSCORE_META`).
- **MVP simula estar completo** (sin sellos de validación / sin "a confirmar"). Ver memoria `fase2/sellos-validacion-pendiente`.
- Commits convencionales, sin atribución de IA. No usar cat/grep/find/sed/ls — usar rg/bat/fd. No build.

---

## Task 1: Crear `perfiles.json` — 20 arquetipos de Rivadavia

**Files:**
- Create: `perfiles.json`

**Interfaces:**
- Produces: `perfiles.json` = array de 20 objetos con schema:
```
{
  id: string,            // slug único, clave de matching (ej "lucia")
  name, initials: string,
  age: number,           // 24–35
  sector: "salud"|"agro"|"municipal"|"educacion"|"oficios"|"comercio",
  profession, education: string,
  ciudad: "América (Rivadavia)",
  ingreso: { monto: number, tipo: "formal"|"informal" },   // ARS/mes
  hogar: string,
  montoSolicitado: number,   // ARS
  plazo: number,             // años
  superficie: number,        // m²
  score: number,             // 1 decimal
  subscores: [number, number, number, number],  // [pago, suelo, anticipo, formalidad]
  narrative: string,
  instrumentosElegibles: string[],  // subconjunto de los 8 ids válidos
  calce: number              // entero 65–95 (vista inversor)
}
```

**Distribución requerida (20 perfiles):**

| Sector | Cant. | Tipo ingreso |
|--------|-------|--------------|
| salud | 4 | 3 formal, 1 informal |
| agro | 4 | 2 formal, 2 informal |
| municipal | 3 | 3 formal |
| educacion | 3 | 2 formal, 1 informal |
| oficios | 3 | 3 informal |
| comercio | 3 | 1 formal, 2 informal |

Resultado: ~11 formal / ~9 informal.

**Rangos:** Formal → score 7.4–9.0, subscore formalidad ≥ 7.5. Informal → score 6.5–7.6, subscore formalidad ≤ 6.5. Montos 45M–130M ARS. Superficie 40–80 m². Plazo 15–25 años.

**Ingresos de referencia (ARS/mes, jun 2026):** médico 1.800.000–2.500.000; enfermero/municipal 950.000–1.300.000; docente 1.100.000–1.500.000; agrónomo/veterinario formal 1.400.000–1.900.000; productor informal 800.000–1.200.000; oficio informal 700.000–1.100.000; comercio informal 600.000–900.000.

**Reglas de elegibilidad (aplicar mecánicamente por perfil):**

| id | Criterio |
|----|----------|
| `uva-bna` | `ingreso.tipo === "formal"` Y cuotaProxy ≤ 25% del ingreso |
| `uva-bapro` | igual que `uva-bna` |
| `circulo-cerrado` | `ingreso.tipo === "informal"` O cuotaProxy > 25% |
| `lote-mi-tierra` | TODOS |
| `banco-tierras` | TODOS |
| `construccion-3d` | ~6 perfiles (agro + oficios) |
| `coop-electrica` | ~8 perfiles (complemento, no excluyente) |
| `fideicomiso-rwa` | ~5 perfiles con score ≥ 8.0 |

Screening de cuota:
```js
function cuotaProxy(monto, plazoAnos) {
  const i = 0.08 / 12, n = plazoAnos * 12;
  return monto * i / (1 - Math.pow(1 + i, -n));
}
// banco elegible si cuotaProxy(montoSolicitado, plazo) / ingreso.monto <= 0.25
```

**Perfil default obligatorio (idx 0):**
```json
{
  "id": "lucia", "name": "Lucía Fernández", "initials": "LF", "age": 27,
  "sector": "salud", "profession": "Médica generalista", "education": "UNLP",
  "ciudad": "América (Rivadavia)",
  "ingreso": { "monto": 1800000, "tipo": "formal" },
  "hogar": "Soltera, sin hijos",
  "montoSolicitado": 54600000, "plazo": 20, "superficie": 50,
  "score": 8.3, "subscores": [8.4, 9.0, 8.6, 8.1],
  "narrative": "Médica recién recibida en UNLP, vuelve a su ciudad natal América para integrarse al hospital municipal Dr. Saturnino Unzué.",
  "instrumentosElegibles": ["uva-bna", "uva-bapro", "lote-mi-tierra", "banco-tierras", "coop-electrica"],
  "calce": 92
}
```

Ejemplo informal (agro):
```json
{
  "id": "roberto", "name": "Roberto Ibáñez", "initials": "RI", "age": 34,
  "sector": "agro", "profession": "Productor agropecuario familiar",
  "education": "Escuela Agrotécnica Rivadavia", "ciudad": "América (Rivadavia)",
  "ingreso": { "monto": 950000, "tipo": "informal" },
  "hogar": "Casado, 2 hijos",
  "montoSolicitado": 62000000, "plazo": 25, "superficie": 55,
  "score": 7.1, "subscores": [5.8, 9.5, 7.2, 5.9],
  "narrative": "Productor familiar de tercera generación, trabaja campo arrendado en Rivadavia, busca vivienda urbana estable para su familia.",
  "instrumentosElegibles": ["circulo-cerrado", "lote-mi-tierra", "banco-tierras", "construccion-3d"],
  "calce": 71
}
```

Ejemplo municipal (formal, OJO con el screening de cuota):
```json
{
  "id": "marcela", "name": "Marcela Gutiérrez", "initials": "MG", "age": 31,
  "sector": "municipal", "profession": "Administrativa municipal",
  "education": "Terciario ISFD América", "ciudad": "América (Rivadavia)",
  "ingreso": { "monto": 1050000, "tipo": "formal" },
  "hogar": "Casada, 1 hijo",
  "montoSolicitado": 40000000, "plazo": 20, "superficie": 45,
  "score": 7.6, "subscores": [7.3, 8.8, 7.6, 8.0],
  "narrative": "Empleada del municipio de Rivadavia hace 6 años, busca vivienda propia en el casco urbano de América.",
  "instrumentosElegibles": ["uva-bna", "uva-bapro", "lote-mi-tierra", "banco-tierras", "coop-electrica"],
  "calce": 80
}
```
> Nota: con montoSolicitado 40M y plazo 20, cuotaProxy ≈ 335k; 335k/1.050k = 31.9% → todavía supera 25%. Para que califique a banco hay que bajar el monto (~30M) o subir plazo/ingreso. El implementador DEBE aplicar `cuotaProxy` perfil por perfil y ajustar `montoSolicitado` o la lista de elegibles para que sean coherentes. Este ejemplo ilustra el chequeo, no es un valor final intocable.

- [ ] **Step 1.1:** Crear `perfiles.json` con los 20 perfiles según la distribución y reglas. El perfil `lucia` debe ser idx 0 con los datos de arriba. Aplicar `cuotaProxy` a cada perfil formal antes de asignar `uva-bna`/`uva-bapro`; si la cuota supera 25%, ajustar montoSolicitado o mover a `circulo-cerrado`.

- [ ] **Step 1.2:** Validar JSON:
  Run: `python3 -m json.tool perfiles.json > /dev/null && echo OK`
  Expected: `OK`

- [ ] **Step 1.3:** Verificar invariantes:
```bash
python3 -c "
import json
VALID={'uva-bna','uva-bapro','circulo-cerrado','lote-mi-tierra','banco-tierras','construccion-3d','coop-electrica','fideicomiso-rwa'}
d=json.load(open('perfiles.json'))
assert len(d)==20, f'esperados 20, hay {len(d)}'
assert d[0]['id']=='lucia', 'lucia debe ser idx 0'
for p in d:
    bad=set(p['instrumentosElegibles'])-VALID
    assert not bad, f'{p[\"id\"]}: ids invalidos {bad}'
    assert 'lote-mi-tierra' in p['instrumentosElegibles'] and 'banco-tierras' in p['instrumentosElegibles'], f'{p[\"id\"]}: todos deben tener suelo'
    assert p['ingreso']['tipo'] in ('formal','informal')
    if p['ingreso']['tipo']=='informal':
        assert 'uva-bna' not in p['instrumentosElegibles'] and 'uva-bapro' not in p['instrumentosElegibles'], f'{p[\"id\"]}: informal con bancario'
        assert p['subscores'][3]<=6.5, f'{p[\"id\"]}: informal formalidad>6.5'
    else:
        assert p['subscores'][3]>=7.5, f'{p[\"id\"]}: formal formalidad<7.5'
print('invariantes OK')
"
```
  Expected: `invariantes OK`

- [ ] **Step 1.4:** Commit: `git commit -m "feat: agregar perfiles.json con 20 arquetipos reales de Rivadavia"`

---

## Task 2: Extender `init()` — cargar ambos JSON en paralelo

**Files:**
- Modify: `index.html` (declaración de `MODELO`, función `loadModelo`, `init`)

**Interfaces:**
- Consumes: `perfiles.json`, `modelo.json`
- Produces: globals `MODELO` y `PERFILES` poblados antes de cualquier render; función `loadPerfiles()`; `showFetchError(archivo)` ahora toma el nombre del archivo.

- [ ] **Step 2.1:** Localizar `let MODELO = null;` y reemplazar por:
```js
    let MODELO = null;
    let PERFILES = null;
```

- [ ] **Step 2.2:** Reemplazar el bloque `loadModelo` + `showFetchError` + `init` por:
```js
    async function loadModelo() {
      const res = await fetch('modelo.json');
      if (!res.ok) throw new Error('HTTP ' + res.status);
      return res.json();
    }

    async function loadPerfiles() {
      const res = await fetch('perfiles.json');
      if (!res.ok) throw new Error('HTTP ' + res.status);
      return res.json();
    }

    function showFetchError(archivo) {
      const device = document.getElementById('device');
      if (!device) return;
      device.innerHTML =
        '<div style="padding:32px;text-align:center;font-family:sans-serif;color:#0a1d33">' +
        '<p style="font-weight:700;font-size:18px;margin-bottom:8px">No se pudo cargar el modelo de datos</p>' +
        '<p style="font-size:14px;color:#64748b">Revisá que <code>' + archivo + '</code> esté disponible y recargá la página.</p>' +
        '</div>';
    }

    async function init() {
      let modelo, perfiles;
      try {
        modelo = await loadModelo();
      } catch (err) {
        console.error('Error cargando modelo.json:', err);
        showFetchError('modelo.json');
        return;
      }
      try {
        perfiles = await loadPerfiles();
      } catch (err) {
        console.error('Error cargando perfiles.json:', err);
        showFetchError('perfiles.json');
        return;
      }
      MODELO = modelo;
      PERFILES = perfiles;
      data.instrumentos = MODELO.instrumentos;

      renderBottomNavs();

      if (state.role === 'joven') showScreen('screen-joven-perfil');
      else if (state.role === 'inversor') showScreen('screen-inv-perfil');
      else showScreen('screen-splash');

      refreshIcons();
    }
```
> Las dos cargas son secuenciales con catch individual para identificar con precisión qué archivo falló (más claro para la demo que `Promise.all` con error genérico). La diferencia de latencia es irrelevante (dos archivos chicos servidos localmente/CDN).

- [ ] **Step 2.3:** Servir `python3 -m http.server 8080` (background) y verificar con agent-browser:
  Run: `agent-browser open http://localhost:8080 && agent-browser wait --load networkidle && agent-browser eval 'JSON.stringify({p: PERFILES.length, i: MODELO.instrumentos.length})'`
  Expected: `{"p":20,"i":8}`

- [ ] **Step 2.4:** Camino de error (controlador/humano): bloquear la request con `agent-browser network route "**/perfiles.json*" --abort`, recargar, y `agent-browser eval 'document.getElementById("device").innerText.slice(0,60)'` → debe contener "No se pudo cargar el modelo de datos". Quitar la ruta después.

- [ ] **Step 2.5:** Commit: `git commit -m "feat: cargar perfiles.json en paralelo con modelo.json en init()"`

---

## Task 3: Eliminar `data.joven`, introducir `jovenActivo()` y render dinámico del perfil

**Files:**
- Modify: `index.html` (`const data`, `const state`, helpers, `renderJovenScoreBars`, `openJovenDetail`, HTML de `#screen-joven-perfil`, `showScreen`)

**Interfaces:**
- Consumes: `PERFILES`, `state.jovenActivoIdx`
- Produces: `state.jovenActivoIdx` (number, default 0); `const SUBSCORE_META`; `jovenActivo()` → perfil; `instrumentosDelJoven()` → Instrumento[]; `renderJovenPerfil()`.

- [ ] **Step 3.1:** Agregar `jovenActivoIdx: 0,` al objeto `const state = {` (junto a los demás campos).

- [ ] **Step 3.2:** Insertar antes del comentario `// -------- ROUTING --------`:
```js
    // -------- SUBSCORE METADATA --------
    const SUBSCORE_META = [
      { key: 'pago',       label: 'Capacidad de pago',      detail: 'Cuota proyectada / ingreso del hogar (meta < 25%)' },
      { key: 'suelo',      label: 'Acceso a suelo',          detail: 'Lote en "Mi Tierra" / Banco de Tierras' },
      { key: 'anticipo',   label: 'Anticipo cubierto',       detail: 'El lote computa como equity (LTV)' },
      { key: 'formalidad', label: 'Formalidad de ingresos',  detail: 'Registración y estabilidad laboral' }
    ];

    // -------- JOVEN ACTIVO --------
    function jovenActivo() {
      return PERFILES[state.jovenActivoIdx];
    }
    function instrumentosDelJoven() {
      const j = jovenActivo();
      return j.instrumentosElegibles
        .map(id => MODELO.instrumentos.find(i => i.id === id))
        .filter(Boolean);
    }
```

- [ ] **Step 3.3:** Reemplazar `renderJovenScoreBars` por (lee de `jovenActivo()` y `SUBSCORE_META`):
```js
    function renderJovenScoreBars() {
      const wrap = $('#joven-score-bars');
      if (!wrap) return;
      const j = jovenActivo();
      wrap.innerHTML = SUBSCORE_META.map((meta, i) => {
        const value = j.subscores[i];
        return `
        <div>
          <div class="flex items-center justify-between mb-1.5">
            <div>
              <span class="text-sm font-semibold text-navy-900">${meta.label}</span>
              <span class="text-[11px] text-slate-500 block">${meta.detail}</span>
            </div>
            <span class="text-sm font-bold text-navy-900">${value.toFixed(1)}</span>
          </div>
          <div class="bar-track"><div class="bar-fill" style="width: ${value * 10}%"></div></div>
        </div>`;
      }).join('');
    }
```

- [ ] **Step 3.4:** En `openJovenDetail`, reemplazar las referencias a `data.joven`. Localizar `const cuota = esCredito ? cuotaMensual(data.joven.monto, ...)` y la línea de `ingresoHogar` previa, y dejar:
```js
      const j = jovenActivo();
      const ingresoHogar = j.ingreso.monto;
      const cuota = esCredito ? cuotaMensual(j.montoSolicitado, inst.tasaNum, inst.plazoNum) : 0;
```
  Y en el texto del bloque financiero reemplazar `fmtARS(data.joven.monto)` por `fmtARS(j.montoSolicitado)`. También cambiar el lookup `data.instrumentos.find` por `MODELO.instrumentos.find` (ver Task 4.4 — si Task 4 ya corre antes, ya está hecho).

- [ ] **Step 3.5:** Hacer el perfil 100% dinámico. Localizar el `<div class="px-5 pb-6">` interior de `#screen-joven-perfil` (el bloque con el avatar/score/métricas hardcodeados de Lucía) y reemplazar TODO su interior por:
```html
          <div class="px-5 pb-6" id="joven-perfil-body"></div>
```
  Agregar la función `renderJovenPerfil()` junto a `renderJovenScoreBars`:
```js
    function renderJovenPerfil() {
      const j = jovenActivo();
      const body = $('#joven-perfil-body');
      if (!body) return;
      body.innerHTML = `
        <div class="rounded-2xl bg-navy-900 text-white p-5 shadow-card relative overflow-hidden">
          <div class="absolute -top-12 -right-10 w-44 h-44 rounded-full bg-accent-500/20 blur-2xl"></div>
          <div class="relative flex items-center gap-4">
            <div class="avatar lg ring-soft">${j.initials}</div>
            <div class="flex-1">
              <div class="text-lg font-bold">${j.name}</div>
              <div class="text-xs text-white/70 mt-0.5">${j.age} años · ${j.profession}</div>
              <div class="flex items-center gap-1.5 mt-2 text-xs text-white/80">
                <i data-lucide="map-pin" class="w-3.5 h-3.5 text-accent-300"></i><span>${j.ciudad}</span>
              </div>
            </div>
          </div>
          <p class="relative text-[13px] text-white/75 leading-relaxed mt-4">${j.narrative}</p>
        </div>

        <div class="mt-5 rounded-2xl bg-white border border-navy-100 shadow-card p-5">
          <div class="flex items-center justify-between">
            <div>
              <p class="section-title">Vinder Score</p>
              <div class="flex items-baseline gap-1 mt-1">
                <span class="text-4xl font-extrabold text-navy-900">${j.score.toFixed(1)}</span>
                <span class="text-sm font-medium text-slate-500">/10</span>
              </div>
            </div>
            <span class="chip accent"><i data-lucide="trending-up" class="w-3 h-3"></i> Apto para financiar</span>
          </div>
          <div id="joven-score-bars" class="mt-5 space-y-3.5"></div>
        </div>

        <div class="mt-5 grid grid-cols-2 gap-3">
          <div class="metric-card">
            <p class="text-[11px] text-slate-500 font-medium">Monto solicitado</p>
            <p class="text-base font-bold text-navy-900 mt-1">${fmtCompact(j.montoSolicitado)}</p>
          </div>
          <div class="metric-card">
            <p class="text-[11px] text-slate-500 font-medium">Vivienda objetivo</p>
            <p class="text-base font-bold text-navy-900 mt-1">${j.superficie} m²</p>
          </div>
        </div>

        ${renderContextoTerritorial()}

        <button class="mt-6 w-full gradient-cta text-white font-semibold py-3.5 rounded-2xl flex items-center justify-center gap-2"
          onclick="showScreen('screen-joven-feed')">
          Ver oportunidades <i data-lucide="arrow-right" class="w-4 h-4"></i>
        </button>
      `;
      renderJovenScoreBars();
      refreshIcons();
    }
```
> `renderContextoTerritorial()` se define en Task 6; hasta entonces devolvé `''` (definila como stub `function renderContextoTerritorial(){return '';}` en este task si Task 6 corre después, o asegurate de que Task 6 esté hecho antes de probar). Para evitar romper, en este task agregá el stub y Task 6 lo reemplaza.

- [ ] **Step 3.6:** Agregar stub temporal (será reemplazado por Task 6) junto a los helpers:
```js
    function renderContextoTerritorial() { return ''; }
```

- [ ] **Step 3.7:** Actualizar `showScreen`: localizar `if (id === 'screen-joven-perfil') renderJovenScoreBars();` y reemplazar por `if (id === 'screen-joven-perfil') renderJovenPerfil();`

- [ ] **Step 3.8:** Eliminar el objeto `joven: { ... }` de `const data = {`. Verificar que no queden referencias:
  Run: `rg "data\.joven" index.html`
  Expected: ninguna línea.

- [ ] **Step 3.9:** Verificar con agent-browser (servido):
  Run: `agent-browser open http://localhost:8080 && agent-browser eval '(function(){var b=[...document.querySelectorAll("button")].find(x=>x.innerText.includes("Soy joven"));b&&b.click();})()' && agent-browser wait 400 && agent-browser eval 'JSON.stringify({n:jovenActivo().name,s:jovenActivo().score,m:jovenActivo().montoSolicitado})'`
  Expected: `{"n":"Lucía Fernández","s":8.3,"m":54600000}`
  Screenshot de `screen-joven-perfil` → score 8.3, monto ~$54,6M, superficie 50 m² (sin contradicción).

- [ ] **Step 3.10:** Commit: `git commit -m "refactor: reemplazar data.joven por jovenActivo() leyendo de PERFILES"`

---

## Task 4: Feed del joven filtrado por elegibilidad

**Files:**
- Modify: `index.html` (`renderJovenStack`, `swipeJoven`, `renderJovenMatches`, `openJovenDetail`)

**Interfaces:**
- Consumes: `instrumentosDelJoven()`. `state.jovenIdx` es índice dentro de ese subarray.

- [ ] **Step 4.1:** Reemplazar `renderJovenStack` para usar `instrumentosDelJoven()`:
```js
    function renderJovenStack() {
      const stack = $('#joven-stack');
      const empty = $('#joven-empty');
      if (!stack) return;

      const instrumentos = instrumentosDelJoven();
      const remaining = instrumentos.length - state.jovenIdx;
      $('#joven-counter').textContent = remaining;

      if (remaining <= 0) {
        stack.innerHTML = '';
        empty.classList.remove('hidden');
        refreshIcons();
        return;
      }
      empty.classList.add('hidden');

      const visible = instrumentos.slice(state.jovenIdx, state.jovenIdx + 3);
      stack.innerHTML = visible.map((inst, idx) => instrumentCardHTML(inst, idx)).reverse().join('');
      refreshIcons();
    }
```
> Verificar el nombre real de la función que arma la tarjeta (puede ser `instrumentCardHTML` u otra); usar el nombre que exista en el archivo.

- [ ] **Step 4.2:** En `swipeJoven`, reemplazar el origen de datos por `instrumentosDelJoven()` y simplificar la lógica de match (todo accept genera match — los objetos de `MODELO.instrumentos` no tienen `matchOnAccept`):
```js
    function swipeJoven(direction) {
      if (state.isAnimating) return;
      const instrumentos = instrumentosDelJoven();
      const remaining = instrumentos.length - state.jovenIdx;
      if (remaining <= 0) return;

      const inst = instrumentos[state.jovenIdx];
      const topCard = $('#joven-stack .swipe-card[data-stack="0"]');
      if (!topCard) return;

      state.isAnimating = true;
      topCard.classList.add(direction === 'accept' ? 'is-out-right' : 'is-out-left');

      setTimeout(() => {
        state.jovenSwipes[inst.id] = direction;
        state.jovenIdx += 1;

        if (direction === 'accept') {
          if (!state.jovenMatches.includes(inst.id)) {
            state.jovenMatches.push(inst.id);
            openMatchModal({
              title: `¡Match con ${inst.nombre}!`,
              subtitle: 'Ambos lados aceptaron el vínculo. Ya podés iniciar el contacto.',
              cta: () => openJovenDetail(inst.id)
            });
          }
        } else {
          showToast('Pasaste. Buscamos otro calce.');
        }

        renderJovenStack();
        state.isAnimating = false;
      }, 320);
    }
```
> Verificar los nombres reales de selectores/clases (`.swipe-card[data-stack="0"]`, `is-out-right`) y de helpers (`openMatchModal`, `showToast`) contra el archivo; usar los que existan. La estructura (origen = `instrumentosDelJoven()`, match en accept) es lo que importa.

- [ ] **Step 4.3:** En `renderJovenMatches`, reemplazar `data.instrumentos.find` por `MODELO.instrumentos.find`:
```js
      const items = state.jovenMatches.map(id => MODELO.instrumentos.find(i => i.id === id)).filter(Boolean);
```

- [ ] **Step 4.4:** En `openJovenDetail`, reemplazar el lookup `data.instrumentos.find(i => i.id === id)` por `MODELO.instrumentos.find(i => i.id === id)`.

- [ ] **Step 4.5:** Verificar con agent-browser (rol joven, perfil lucia):
  Run: `agent-browser eval 'JSON.stringify({elig:jovenActivo().instrumentosElegibles,feed:instrumentosDelJoven().map(i=>i.id)})'`
  Expected: `feed` = mismos ids que `elig` (mismo orden), todos presentes.
  Screenshot del feed → counter = `instrumentosElegibles.length`.

- [ ] **Step 4.6:** Commit: `git commit -m "feat: filtrar feed joven por instrumentosElegibles del perfil activo"`

---

## Task 5: Selector de usuario (demo)

**Files:**
- Modify: `index.html` (header de `#screen-joven-perfil` — ahora dentro de `renderJovenPerfil` NO, el header es estático fuera del body; ver nota —, HTML del panel, JS del selector)

> **Nota importante:** en Task 3 se vació el `<div class="px-5 pb-6">` (el body), pero el `<header>` de `#screen-joven-perfil` sigue siendo HTML estático. El botón va en ese header estático.

**Interfaces:**
- Consumes: `PERFILES`, `state.jovenActivoIdx`
- Produces: panel `#user-selector-panel`; `openUserSelector()`, `closeUserSelector()`, `selectUser(idx)`, `renderUserSelectorList()`.

- [ ] **Step 5.1:** Localizar el `<header>` de `#screen-joven-perfil` con el botón `onclick="switchRole()"`. Agregar, ANTES de ese botón, un botón nuevo, dejando ambos dentro de un contenedor flex:
```html
        <div class="flex items-center gap-2">
          <button class="icon-btn" onclick="openUserSelector()" title="Cambiar usuario">
            <i data-lucide="users" class="w-4 h-4"></i>
          </button>
          <button class="icon-btn" onclick="switchRole()" title="Cambiar de rol">
            <i data-lucide="repeat" class="w-4 h-4"></i>
          </button>
        </div>
```
> Reemplaza el botón `switchRole` suelto por este contenedor con los dos. Mantené la clase/ícono originales de `switchRole` si difieren.

- [ ] **Step 5.2:** Agregar el panel del selector en el HTML, junto a los otros overlays (buscar el modal de match o el toast y ponerlo cerca):
```html
    <!-- ============== USER SELECTOR PANEL ============== -->
    <div id="user-selector-panel" class="absolute inset-0 z-50 flex-col bg-white" style="display:none">
      <header class="px-5 pt-6 pb-4 flex items-center justify-between border-b border-navy-100">
        <div>
          <p class="section-title">Demo · Perfiles disponibles</p>
          <h2 class="text-xl font-bold text-navy-900 mt-1">Elegí un joven</h2>
        </div>
        <button class="icon-btn" onclick="closeUserSelector()"><i data-lucide="x" class="w-4 h-4"></i></button>
      </header>
      <div id="user-selector-list" class="flex-1 overflow-y-auto px-5 py-4 space-y-2"></div>
    </div>
```
> Usar `absolute inset-0` (no `fixed`) para que quede dentro del marco del dispositivo. El contenedor `#device` debe tener `position: relative` (verificar; si no, el panel se posiciona respecto al viewport — aceptable para demo pero idealmente relativo al device).

- [ ] **Step 5.3:** Agregar las funciones JS (junto a los otros render/handlers):
```js
    // -------- USER SELECTOR --------
    function openUserSelector() {
      renderUserSelectorList();
      const panel = $('#user-selector-panel');
      if (panel) panel.style.display = 'flex';
      refreshIcons();
    }
    function closeUserSelector() {
      const panel = $('#user-selector-panel');
      if (panel) panel.style.display = 'none';
    }
    function selectUser(idx) {
      state.jovenActivoIdx = idx;
      state.jovenIdx = 0;
      state.jovenSwipes = {};
      state.jovenMatches = [];
      closeUserSelector();
      showScreen('screen-joven-perfil');
    }
    function renderUserSelectorList() {
      const list = $('#user-selector-list');
      if (!list || !PERFILES) return;
      list.innerHTML = PERFILES.map((p, idx) => {
        const active = idx === state.jovenActivoIdx;
        return `
        <button class="w-full text-left rounded-2xl border p-4 flex items-center gap-3 ${active ? 'bg-navy-900 border-navy-900' : 'bg-white border-navy-100'}"
          onclick="selectUser(${idx})">
          <div class="avatar sm" style="${active ? 'background:rgba(255,255,255,0.15);color:#fff' : ''}">${p.initials}</div>
          <div class="flex-1 min-w-0">
            <p class="text-sm font-bold truncate ${active ? 'text-white' : 'text-navy-900'}">${p.name}</p>
            <p class="text-[11px] mt-0.5 truncate ${active ? 'text-white/70' : 'text-slate-500'}">${p.profession} · ${p.sector}</p>
            <p class="text-[11px] mt-0.5 ${active ? 'text-white/60' : 'text-slate-400'}">${p.ingreso.tipo} · ${p.instrumentosElegibles.length} instrumentos</p>
          </div>
          <div class="text-right shrink-0">
            <p class="text-lg font-extrabold ${active ? 'text-white' : 'text-navy-900'}">${p.score.toFixed(1)}</p>
            ${active ? '<p class="text-[10px] text-white/60 uppercase tracking-wider">activo</p>' : ''}
          </div>
        </button>`;
      }).join('');
    }
```

- [ ] **Step 5.4:** Verificar con agent-browser:
  - Screenshot del perfil con botón "Cambiar usuario".
  - Click → screenshot del panel con 20 perfiles.
  - Seleccionar un perfil informal; `agent-browser eval 'JSON.stringify({t:jovenActivo().ingreso.tipo,ids:instrumentosDelJoven().map(i=>i.id)})'` → `tipo` `"informal"`, ids SIN `uva-bna`/`uva-bapro`.

- [ ] **Step 5.5:** Commit: `git commit -m "feat: agregar selector de usuario para demo (20 perfiles)"`

---

## Task 6: Card de contexto territorial (datos reales de MODELO)

**Files:**
- Modify: `index.html` (reemplazar el stub `renderContextoTerritorial`)

**Interfaces:**
- Consumes: `MODELO.territorio` (campos: `partido`, `cabecera`, `provincia`, `poblacion_partido_2022`, `var_2010_2022`, `poblacion_america_2022`, `var_america`, `intendente`) y `MODELO.macro_jun2026` (campos: `uva_ars`, `dolar_mep_ars`, `construccion_usd_m2`{`base`,`estandar`,`3dcp`}, `smvm_ars`, `ripte_ars`, `ripte_usd`).

> Los nombres de campo de arriba son los REALES de `modelo.json` (sección 10 del modelo maestro). No usar otros.

- [ ] **Step 6.1:** Reemplazar el stub `function renderContextoTerritorial() { return ''; }` por:
```js
    function renderContextoTerritorial() {
      if (!MODELO || !MODELO.territorio || !MODELO.macro_jun2026) return '';
      const t = MODELO.territorio;
      const m = MODELO.macro_jun2026;
      const pct = (x) => (x * 100).toFixed(1).replace('.', ',') + '%';
      return `
        <div class="mt-5 rounded-2xl bg-white border border-navy-100 shadow-card p-5">
          <p class="section-title">Contexto territorial</p>
          <div class="mt-4 grid grid-cols-2 gap-3">
            <div>
              <p class="text-[10px] uppercase tracking-wider text-slate-500 font-semibold">Localidad</p>
              <p class="text-sm font-bold text-navy-900 mt-0.5">${t.cabecera} · ${t.partido}</p>
            </div>
            <div>
              <p class="text-[10px] uppercase tracking-wider text-slate-500 font-semibold">Población (2022)</p>
              <p class="text-sm font-bold text-navy-900 mt-0.5">${t.poblacion_america_2022.toLocaleString('es-AR')} hab.</p>
            </div>
            <div>
              <p class="text-[10px] uppercase tracking-wider text-slate-500 font-semibold">Crecimiento 2010–2022</p>
              <p class="text-sm font-bold text-navy-900 mt-0.5">+${pct(t.var_america)}</p>
            </div>
            <div>
              <p class="text-[10px] uppercase tracking-wider text-slate-500 font-semibold">Intendente</p>
              <p class="text-sm font-bold text-navy-900 mt-0.5 truncate">${t.intendente}</p>
            </div>
          </div>
          <div class="mt-3 pt-3 border-t border-navy-100 grid grid-cols-2 gap-3">
            <div>
              <p class="text-[10px] uppercase tracking-wider text-slate-500 font-semibold">Valor UVA (jun 2026)</p>
              <p class="text-sm font-bold text-navy-900 mt-0.5">$${m.uva_ars.toLocaleString('es-AR')}</p>
            </div>
            <div>
              <p class="text-[10px] uppercase tracking-wider text-slate-500 font-semibold">Construcción estándar</p>
              <p class="text-sm font-bold text-navy-900 mt-0.5">USD ${m.construccion_usd_m2.estandar.toLocaleString('es-AR')}/m²</p>
            </div>
          </div>
        </div>`;
    }
```

- [ ] **Step 6.2:** Confirmar los campos reales en modelo.json (sanity check):
  Run: `python3 -c "import json;m=json.load(open('modelo.json'));print(m['territorio']['poblacion_america_2022'],m['territorio']['var_america'],m['territorio']['intendente'],m['macro_jun2026']['uva_ars'],m['macro_jun2026']['construccion_usd_m2']['estandar'])"`
  Expected: `13843 0.184 Juan Alberto Martínez 1992.43 1375`

- [ ] **Step 6.3:** Verificar con agent-browser:
  Screenshot de `screen-joven-perfil` → card "Contexto territorial" con: América · Rivadavia, 13.843 hab., +18,4%, Juan Alberto Martínez, $1.992,43, USD 1.375/m². Sin `undefined`/`NaN`.

- [ ] **Step 6.4:** Commit: `git commit -m "feat: mostrar card de contexto territorial desde MODELO en perfil joven"`

---

## Task 7: Vista inversor — leer de `PERFILES`

**Files:**
- Modify: `index.html` (`renderInvStack`, `swipeInversor`, `renderInvMatches`, `openInvDetail`, posible `youthCardHTML`, `const data`, `const state`)

**Interfaces:**
- Consumes: `PERFILES`. Reemplaza todos los usos de `data.perfilesJoven`.

- [ ] **Step 7.1:** Listar las referencias:
  Run: `rg -n "data\.perfilesJoven" index.html`
  Reemplazar cada una: `data.perfilesJoven` → `PERFILES` (mismo método/índice; ej. `data.perfilesJoven.length`→`PERFILES.length`, `data.perfilesJoven[state.invIdx]`→`PERFILES[state.invIdx]`, `data.perfilesJoven.find(...)`→`PERFILES.find(...)`).

- [ ] **Step 7.2:** Campo renombrado `monto` → `montoSolicitado` en perfiles. Buscar usos en tarjetas/detalle del inversor:
  Run: `rg -n "p\.monto\b|profile\.monto\b|\.monto\b" index.html`
  En las funciones de la vista inversor (`youthCardHTML`/`renderInvStack`/`openInvDetail`), reemplazar `<perfil>.monto` por `<perfil>.montoSolicitado`. (No tocar `inst.monto` de instrumentos, que es un string distinto.)

- [ ] **Step 7.3:** Revisar `state.invMatches` default. Si tiene `['mateo']` y `mateo` existe en `PERFILES` con ese `id`, dejar igual; si no existe, cambiar a `[]`:
  Run: `python3 -c "import json;print('mateo' in [p['id'] for p in json.load(open('perfiles.json'))])"`
  Si imprime `False`, cambiar el default a `invMatches: [],`.

- [ ] **Step 7.4:** Verificar con agent-browser (rol inversor):
  Run: `agent-browser eval 'PERFILES.length'` → `20`. Screenshot del feed → counter 20. Abrir un detalle → muestra nombre, profesión, score, monto correctos.

- [ ] **Step 7.5:** Eliminar el array `perfilesJoven: [...]` de `const data = {`. Verificar:
  Run: `rg -n "perfilesJoven" index.html`
  Expected: ninguna línea.

- [ ] **Step 7.6:** Commit: `git commit -m "refactor: reemplazar data.perfilesJoven por PERFILES en vista inversor"`

---

## Task 8: Smoke test integral y cleanup

**Files:**
- Modify: `index.html` (limpieza final de `const data`)

- [ ] **Step 8.1:** Verificar si `data.instrumentos` sigue usándose:
  Run: `rg -n "data\.instrumentos" index.html`
  Si quedan usos, reemplazar por `MODELO.instrumentos`. Si `const data` quedó con solo `instrumentos: []`, dejarlo (init lo asigna) o eliminar `data` por completo si ninguna referencia `data.` queda:
  Run: `rg -n "\bdata\." index.html`

- [ ] **Step 8.2:** Smoke test completo con agent-browser (servir en `http://localhost:8080`). Checklist:

  | # | eval / acción | esperado |
  |---|---|---|
  | 1 | `PERFILES.length` | 20 |
  | 2 | `MODELO.instrumentos.length` | 8 |
  | 3 | `jovenActivo().name` | "Lucía Fernández" |
  | 4 | `jovenActivo().montoSolicitado` | 54600000 |
  | 5 | screenshot perfil joven | score 8.3, $54,6M, 50 m², card territorial OK |
  | 6 | `instrumentosDelJoven().length` | = `jovenActivo().instrumentosElegibles.length` |
  | 7 | selector → perfil informal → `instrumentosDelJoven().map(i=>i.id)` | sin `uva-bna`/`uva-bapro` |
  | 8 | rol inversor → counter | 20 |
  | 9 | `rg "data\.joven|data\.perfilesJoven" index.html` | sin resultados |

- [ ] **Step 8.3:** Commit (si hubo cambios): `git commit -m "refactor: limpiar objeto data residual"`

---

## Orden de dependencias

```
Task 1 → Task 2 → ┬→ Task 3 → ┬→ Task 4 → Task 5
                  │           └→ Task 6
                  └→ Task 7
Task 8 (final) ← todas
```
Tasks 6 y 7 pueden correr en paralelo tras 3 y 2 respectivamente. Task 5 requiere 3+4. Ejecutar secuencialmente con subagent-driven-development es lo más seguro dada la edición concurrente del mismo archivo `index.html`.

## Notas / riesgos

- Edición concurrente de `index.html`: Tasks 3–8 tocan el mismo archivo. Ejecutar de a una (no en paralelo) para evitar conflictos.
- `renderJovenPerfil` llama a `renderContextoTerritorial`: el stub de Task 3.6 evita romper antes de Task 6.
- Verificar nombres reales de helpers/clases (`instrumentCardHTML`, `youthCardHTML`, `openMatchModal`, `showToast`, `.swipe-card`) contra el archivo al implementar; el plan usa los nombres observados pero el implementador confirma.
- Cache del navegador sobre `perfiles.json` (igual que modelo.json): hard-reload al verificar.
