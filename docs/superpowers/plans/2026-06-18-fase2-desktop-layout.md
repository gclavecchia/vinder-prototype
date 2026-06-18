# Layout desktop multi-panel (Fase 2, Tanda 2) — Plan de implementación

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development. Todas las tareas tocan `index.html` → ejecución SECUENCIAL estricta. Smoke test final via agent-browser. Steps usan checkbox `- [ ]`.

**Goal:** Agregar una vista desktop multi-panel (sidebar + feed + panel derecho con pestañas Detalle/Matches) accesible vía toggle manual, sin tocar la experiencia mobile. Ambos roles (joven e inversor). El modo se persiste en localStorage.

**Architecture:** Estrategia **reparenting + CSS `body.desktop`**. Las `<section id="screen-*">` existentes se MUEVEN (appendChild) a los slots del shell desktop al activar desktop, y se devuelven al `#device` al volver a mobile. Las funciones de render NO se modifican (siguen escribiendo a sus ids, que ahora viven en los slots). `showScreen` es no-op en desktop (guard); `renderLayout()` orquesta en desktop. Los handlers son inline `onclick` que se regeneran en cada render, así que reparentar no pierde listeners.

**Tech Stack:** HTML/CSS/JS vanilla, Lucide, Tailwind CDN. Sin tests; verificación con `python3 -m http.server` + agent-browser.

## Global Constraints

- **Mobile intacto:** cero regresiones en pantallas, navs, swipe, modales.
- **Cero lógica de negocio duplicada:** renders, swipe handlers, match logic sin reescribir (solo se extraen helpers verbatim donde se indique).
- **Ambos roles** tienen layout desktop.
- **Toggle manual + default por viewport (`window.innerWidth >= 1024`) + persistido** (`localStorage` clave `vinder_layout`).
- **`showScreen` en desktop = no-op** (guard al inicio); en mobile opera como hoy.
- Sin build. No usar cat/grep/find/sed/ls (rg/bat/fd). Commits convencionales sin atribución IA.
- Al extraer un bloque a un helper, MOVER el HTML existente verbatim (no reescribirlo).

---

### Task 1: Shell HTML + CSS desktop + state.layout + toggleLayout()

**Objetivo:** Andamiar todo el código nuevo sin activarlo. Al terminar: el archivo no tira errores de consola, `#desktop-shell` existe pero oculto, y `toggleLayout()` desde consola alterna `body.desktop`/`body.mobile`.

**Files:** Modify `index.html`

**Interfaces:**
- Produces: `#desktop-shell` en DOM; `state.layout`, `state.panelTab`; `LAYOUT_KEY`; `toggleLayout()`, `applyLayoutClass()`, `switchPanelTab()`, `renderLayout()` (placeholder); clases `body.desktop`/`body.mobile`.

- [ ] **Step 1:** Agregar `LAYOUT_KEY` y campos `layout`/`panelTab` al state. Localizar `const STORAGE_KEY = 'vinder_role';` y el objeto `const state = {`:
```js
const STORAGE_KEY = 'vinder_role';
const LAYOUT_KEY  = 'vinder_layout';
const state = {
  role: localStorage.getItem(STORAGE_KEY) || null,
  layout: null,          // 'mobile' | 'desktop' — se inicializa en init()
  panelTab: 'detail',    // 'detail' | 'matches'
  jovenActivoIdx: 0,
  jovenIdx: 0,
  invIdx: 0,
  jovenSwipes: {},
  invSwipes: {},
  jovenMatches: [],
  invMatches: [],
  isAnimating: false
};
```
> Verificá los campos reales del `state` actual con rg y preservalos; solo agregás `layout` y `panelTab` (y `LAYOUT_KEY`).

- [ ] **Step 2:** Guard en `showScreen`. Agregar como primera línea del cuerpo de `function showScreen(id) {`:
```js
      if (state.layout === 'desktop') return; // en desktop manda renderLayout()
```

- [ ] **Step 3:** Insertar el HTML del shell desktop DESPUÉS del cierre de `<div class="device-wrap">` y ANTES del `<script>`:
```html
  <!-- ============== DESKTOP SHELL ============== -->
  <div id="desktop-shell" class="desktop-shell" style="display:none">
    <div class="desk-topbar">
      <div class="desk-brand"><img src="logo.png" alt="Vinder" class="h-8 w-auto" /></div>
      <div class="desk-topbar-actions">
        <div id="desk-context" class="desk-context-chips"></div>
        <button id="btn-toggle-layout" class="desk-toggle-btn" onclick="toggleLayout()" title="Alternar vista">
          <i data-lucide="smartphone" class="w-4 h-4"></i><span>Vista mobile</span>
        </button>
      </div>
    </div>
    <div class="desk-grid">
      <aside id="desk-sidebar" class="desk-panel desk-panel--sidebar"></aside>
      <main  id="desk-feed"    class="desk-panel desk-panel--feed"></main>
      <div   id="desk-panel"   class="desk-panel desk-panel--right">
        <div class="desk-tabs">
          <button class="desk-tab active" data-tab="detail" onclick="switchPanelTab('detail')">
            <i data-lucide="file-text" class="w-4 h-4"></i> Detalle
          </button>
          <button class="desk-tab" data-tab="matches" onclick="switchPanelTab('matches')">
            <i data-lucide="heart-handshake" class="w-4 h-4"></i> Matches
          </button>
        </div>
        <div id="desk-panel-detail"  class="desk-tab-content desk-tab-content--active"></div>
        <div id="desk-panel-matches" class="desk-tab-content"></div>
      </div>
    </div>
  </div>
```
> Verificá con rg dónde cierra exactamente `device-wrap` (`rg -n 'device-wrap' index.html`) y el nombre real del logo (`rg -n 'logo' index.html` — usá el src real, ej. `logo.png` o el que exista).

- [ ] **Step 4:** Agregar el FAB toggle mobile dentro de `.device-wrap` pero fuera de `#device`. Localizar `<div class="device-wrap">` seguido de `<div class="device" id="device">` e insertar entre ambos:
```html
    <button id="btn-toggle-layout-mobile" class="desk-toggle-btn desk-toggle-btn--mobile" onclick="toggleLayout()" title="Cambiar a vista desktop" style="display:none">
      <i data-lucide="monitor" class="w-4 h-4"></i>
    </button>
```

- [ ] **Step 5:** Insertar el bloque CSS desktop antes del `</style>`:
```css
    body.desktop .device-wrap { display: none; }
    body.desktop #desktop-shell { display: flex !important; }
    body.mobile  #desktop-shell { display: none !important; }
    body.mobile  #btn-toggle-layout-mobile { display: flex !important; }

    .desktop-shell { width:100%; height:100vh; display:flex; flex-direction:column; background:#eef2f7; overflow:hidden; }
    .desk-topbar { height:56px; background:#fff; border-bottom:1px solid #e5ecf3; display:flex; align-items:center; justify-content:space-between; padding:0 24px; flex-shrink:0; gap:16px; }
    .desk-brand { display:flex; align-items:center; gap:10px; }
    .desk-brand img { height:32px; width:auto; }
    .desk-topbar-actions { display:flex; align-items:center; gap:12px; }
    .desk-toggle-btn { display:flex; align-items:center; gap:6px; padding:7px 14px; border-radius:10px; border:1px solid #e5ecf3; background:#f8fafc; color:#334155; font-size:13px; font-weight:500; cursor:pointer; transition:background 150ms,border-color 150ms; }
    .desk-toggle-btn:hover { background:#f1f5f9; border-color:#cbd5e1; }
    .desk-toggle-btn--mobile { position:fixed; bottom:24px; right:24px; z-index:200; width:44px; height:44px; padding:0; border-radius:50%; background:#0a1d33; color:#fff; border:none; box-shadow:0 4px 16px rgba(10,29,51,.25); display:none; align-items:center; justify-content:center; }
    .desk-toggle-btn--mobile:hover { background:#10b981; }
    .desk-grid { flex:1; display:grid; grid-template-columns:320px 1fr 360px; overflow:hidden; }
    .desk-panel { height:100%; overflow-y:auto; overflow-x:hidden; background:#fff; }
    .desk-panel--sidebar { border-right:1px solid #e5ecf3; }
    .desk-panel--feed { background:#f8fafc; border-right:1px solid #e5ecf3; }
    .desk-panel--right { display:flex; flex-direction:column; }
    .desk-tabs { display:flex; align-items:center; border-bottom:1px solid #e5ecf3; background:#fff; padding:0 16px; flex-shrink:0; gap:4px; }
    .desk-tab { display:flex; align-items:center; gap:6px; padding:12px 16px; border:none; background:transparent; color:#64748b; font-size:13px; font-weight:500; cursor:pointer; border-bottom:2px solid transparent; margin-bottom:-1px; transition:color 150ms,border-color 150ms; }
    .desk-tab.active { color:#0a1d33; border-bottom-color:#10b981; }
    .desk-tab-content { display:none; flex:1; overflow-y:auto; overflow-x:hidden; }
    .desk-tab-content--active { display:block; }
    body.desktop .screen { display:flex !important; flex:unset; height:100%; overflow-y:auto; }
    body.desktop .bottom-nav { display:none !important; }
    .desk-context-chips { display:flex; align-items:center; gap:8px; }
    .desk-chip { display:flex; align-items:center; gap:5px; padding:4px 10px; border-radius:20px; background:#f1f5f9; border:1px solid #e2e8f0; font-size:12px; font-weight:500; color:#334155; }
    .desk-chip--score { background:#ecfdf5; border-color:#a7f3d0; color:#065f46; font-weight:700; }
```

- [ ] **Step 6:** Agregar los helpers de layout DESPUÉS del bloque `// -------- ROUTING --------`:
```js
    // -------- LAYOUT (desktop / mobile) --------
    function applyLayoutClass() {
      document.body.classList.toggle('desktop', state.layout === 'desktop');
      document.body.classList.toggle('mobile',  state.layout === 'mobile');
    }
    function toggleLayout() {
      state.layout = state.layout === 'desktop' ? 'mobile' : 'desktop';
      localStorage.setItem(LAYOUT_KEY, state.layout);
      renderLayout();
    }
    function switchPanelTab(tab) {
      state.panelTab = tab;
      document.querySelectorAll('.desk-tab').forEach(btn => btn.classList.toggle('active', btn.dataset.tab === tab));
      document.querySelectorAll('.desk-tab-content').forEach(el => el.classList.toggle('desk-tab-content--active', el.id === `desk-panel-${tab}`));
    }
    function renderLayout() {   // implementación real en Tasks 2-4
      applyLayoutClass();
      refreshIcons();
    }
```

- [ ] **Step 7:** Verificar (servir + agent-browser): `agent-browser open` → eval `typeof toggleLayout` = "function"; eval `toggleLayout(); document.body.classList.contains('desktop')` → true; eval `document.getElementById('desktop-shell')!==null` → true. Sin errores de consola. Volver a mobile con `toggleLayout()`.

- [ ] **Step 8:** Commit: `feat: andamiar shell desktop + state.layout + toggle (sin activar render)`

---

### Task 2: renderLayout() joven — sidebar y feed

**Objetivo:** `renderLayout()` real para joven: mueve `screen-joven-perfil`→`#desk-sidebar`, `screen-joven-feed`→`#desk-feed`; en mobile devuelve secciones al `#device`.

**Files:** Modify `index.html`
**Interfaces:**
- Consumes: `state.layout`, `state.role`, slots desktop, `#device`, renders existentes.
- Produces: `renderLayout()`, `renderLayoutMobile()`, `renderLayoutDesktop()`, `renderLayoutDesktopJoven()`, `reparentToDesktop(screenId, slotId)`, `returnToMobile(anchorId)`.

- [ ] **Step 1:** Insertar anclas de reparenting dentro de `#device`. Antes de `<section id="screen-joven-perfil"`:
```html
      <div id="mobile-anchor-joven" data-screens="screen-joven-perfil,screen-joven-feed,screen-joven-matches,screen-joven-detail" style="display:none"></div>
```
Antes de `<section id="screen-inv-perfil"`:
```html
      <div id="mobile-anchor-inv" data-screens="screen-inv-perfil,screen-inv-dashboard,screen-inv-feed,screen-inv-matches,screen-inv-detail" style="display:none"></div>
```
> Verificá con `rg -n '<section id="screen-' index.html` los ids reales y ajustá las listas `data-screens` a los que EXISTAN.

- [ ] **Step 2:** Agregar funciones de reparenting en el bloque LAYOUT:
```js
    function reparentToDesktop(screenId, slotId) {
      const section = document.getElementById(screenId);
      const slot = document.getElementById(slotId);
      if (!section || !slot) return;
      slot.appendChild(section);
    }
    function returnToMobile(anchorId) {
      const anchor = document.getElementById(anchorId);
      if (!anchor) return;
      const ids = (anchor.dataset.screens || '').split(',').filter(Boolean);
      let ref = anchor;
      ids.forEach(id => {
        const section = document.getElementById(id);
        if (section) { ref.insertAdjacentElement('afterend', section); ref = section; }
      });
    }
```

- [ ] **Step 3:** Reemplazar el `renderLayout()` placeholder por la versión real:
```js
    function renderLayout() {
      applyLayoutClass();
      if (state.layout === 'desktop') renderLayoutDesktop();
      else renderLayoutMobile();
      refreshIcons();
    }
    function renderLayoutMobile() {
      returnToMobile('mobile-anchor-joven');
      returnToMobile('mobile-anchor-inv');
      ['desk-sidebar','desk-feed','desk-panel-detail','desk-panel-matches'].forEach(id => {
        const el = document.getElementById(id); if (el) el.innerHTML = '';
      });
      if (state.role === 'joven') showScreen('screen-joven-perfil');
      else if (state.role === 'inversor') showScreen('screen-inv-perfil');
      else showScreen('screen-splash');
    }
    function renderLayoutDesktop() {
      if (state.role === 'joven') renderLayoutDesktopJoven();
      else if (state.role === 'inversor') renderLayoutDesktopInversor();
    }
    function renderLayoutDesktopJoven() {
      reparentToDesktop('screen-joven-perfil', 'desk-sidebar');
      reparentToDesktop('screen-joven-feed', 'desk-feed');
      renderRightPanelJoven();
      renderJovenPerfil();
      renderJovenStack();
    }
```
> `renderRightPanelJoven` y `renderLayoutDesktopInversor` se definen en Tasks 3 y 4. Para que Task 2 no rompa, agregá stubs temporales: `function renderRightPanelJoven(){}` y `function renderLayoutDesktopInversor(){}` (Tasks 3/4 los reemplazan).

- [ ] **Step 4:** Verificar (agent-browser, viewport 1440): setear rol joven, `toggleLayout()` → eval `document.getElementById('desk-sidebar').querySelector('#screen-joven-perfil')!==null` → true; eval `document.getElementById('desk-feed').querySelector('#screen-joven-feed')!==null` → true. Volver a mobile → eval `document.getElementById('device').querySelector('#screen-joven-perfil')!==null` → true. Screenshot desktop.

- [ ] **Step 5:** Commit: `feat: renderLayout joven — reparenting de perfil y feed a slots desktop`

---

### Task 3: Panel derecho joven — tabs Detalle y Matches

**Objetivo:** `renderRightPanelJoven()` con tabs. Detalle muestra el instrumento top del feed en vivo; Matches lista aceptados; clickear un match abre su detalle. Swipe actualiza el detalle.

**Files:** Modify `index.html`
**Interfaces:**
- Consumes: slots `#desk-panel-detail`/`#desk-panel-matches`, `screen-joven-detail`, `screen-joven-matches`, `openJovenDetail`, `renderJovenMatches`, `instrumentosDelJoven`, `cuotaMensual`, `MODELO`.
- Produces: `renderRightPanelJoven()`, `renderRightPanelTopCardJoven()`, `openJovenDetailDesktop(id)`, helpers `buildBloqueFinancieroJoven(...)` y `buildJovenDetailBodyHTML(...)`; `openJovenDetail` adaptado.

- [ ] **Step 1:** Refactor de `openJovenDetail`: extraer el cuerpo HTML a dos helpers PUROS, moviendo el HTML existente VERBATIM (no reescribir). Leer la función actual `openJovenDetail` y separar:
  - `buildBloqueFinancieroJoven(inst, j, cuota, costoTotal, ratioPct, okRatio)` → devuelve el string del bloque financiero (el template literal `bloqueFinanciero` actual).
  - `buildJovenDetailBodyHTML(inst, bloqueFinanciero)` → devuelve el string completo que hoy se asigna a `$('#joven-detail-body').innerHTML`.
  Luego `openJovenDetail` queda:
```js
    function openJovenDetail(id) {
      const inst = MODELO.instrumentos.find(i => i.id === id);
      if (!inst) return;
      if (state.layout === 'desktop') { openJovenDetailDesktop(id); switchPanelTab('detail'); return; }
      const j = jovenActivo();
      const esCredito = inst.categoria === 'credito';
      const ingresoHogar = j.ingreso.monto;
      const cuota = esCredito ? cuotaMensual(j.montoSolicitado, inst.tasaNum, inst.plazoNum) : 0;
      const costoTotal = cuota * inst.plazoNum * 12;
      const ratioPct = esCredito ? Math.round((cuota / ingresoHogar) * 100) : 0;
      const okRatio = ratioPct <= 25;
      $('#joven-detail-title').textContent = inst.nombre;
      $('#joven-detail-body').innerHTML = buildJovenDetailBodyHTML(inst, buildBloqueFinancieroJoven(inst, j, cuota, costoTotal, ratioPct, okRatio));
      showScreen('screen-joven-detail');
      refreshIcons();
    }
```
> CRÍTICO: el contenido de los helpers debe ser el HTML existente movido tal cual. Verificá los nombres reales de variables/títulos en la `openJovenDetail` actual (`rg -n 'openJovenDetail' index.html` y leé la función) y conservalos.

- [ ] **Step 2:** Agregar la versión desktop y el render del panel:
```js
    function openJovenDetailDesktop(id) {
      const inst = MODELO.instrumentos.find(i => i.id === id);
      if (!inst) return;
      const j = jovenActivo();
      const esCredito = inst.categoria === 'credito';
      const ingresoHogar = j.ingreso.monto;
      const cuota = esCredito ? cuotaMensual(j.montoSolicitado, inst.tasaNum, inst.plazoNum) : 0;
      const costoTotal = cuota * inst.plazoNum * 12;
      const ratioPct = esCredito ? Math.round((cuota / ingresoHogar) * 100) : 0;
      const okRatio = ratioPct <= 25;
      $('#joven-detail-title').textContent = inst.nombre;
      $('#joven-detail-body').innerHTML = buildJovenDetailBodyHTML(inst, buildBloqueFinancieroJoven(inst, j, cuota, costoTotal, ratioPct, okRatio));
      refreshIcons();
    }
    function renderRightPanelTopCardJoven() {
      if (state.layout !== 'desktop') return;
      const instrumentos = instrumentosDelJoven();
      if (instrumentos.length - state.jovenIdx <= 0) return;
      openJovenDetailDesktop(instrumentos[state.jovenIdx].id);
    }
    function renderRightPanelJoven() {
      if (state.layout !== 'desktop') return;
      reparentToDesktop('screen-joven-detail', 'desk-panel-detail');
      reparentToDesktop('screen-joven-matches', 'desk-panel-matches');
      switchPanelTab(state.panelTab);
      renderRightPanelTopCardJoven();
      renderJovenMatches();
    }
```
> Eliminá el stub `function renderRightPanelJoven(){}` de Task 2.

- [ ] **Step 3:** Hook en `swipeJoven()`. Localizar dentro del `setTimeout` la línea `renderJovenStack();` y dejar:
```js
        renderJovenStack();
        if (state.layout === 'desktop') {
          renderRightPanelTopCardJoven();
          if (state.panelTab === 'matches') renderJovenMatches();
        }
```

- [ ] **Step 4:** Ocultar el botón Volver del detalle en desktop (CSS, antes de `</style>` o en el bloque desktop). Verificá con rg el selector real del botón volver en `screen-joven-detail`/`screen-inv-detail` y agregá una regla `body.desktop ... { display:none }` para ese botón.

- [ ] **Step 5:** Verificar (agent-browser desktop joven): eval `document.getElementById('joven-detail-title').textContent` → nombre del instrumento top; `swipeJoven('reject')` luego eval del título → cambió al siguiente; `switchPanelTab('matches')` → `document.getElementById('desk-panel-matches').classList.contains('desk-tab-content--active')` true. Screenshot.

- [ ] **Step 6:** Commit: `feat: panel derecho joven con tabs Detalle/Matches en desktop`

---

### Task 4: Layout desktop inversor

**Objetivo:** Replicar Tasks 2-3 para inversor.

**Files:** Modify `index.html`
**Interfaces:**
- Consumes: `screen-inv-perfil`, `screen-inv-feed`, `screen-inv-detail`, `screen-inv-matches`, `openInvDetail`, `renderInvMatches`, `renderInvStack`, `PERFILES`.
- Produces: `renderLayoutDesktopInversor()`, `renderRightPanelInversor()`, `renderRightPanelTopCardInversor()`, `openInvDetailDesktop(id)`, helper `buildInvDetailBodyHTML(p)`; `openInvDetail` adaptado.

- [ ] **Step 1:** Refactor `openInvDetail`: extraer `buildInvDetailBodyHTML(p)` (HTML del `#inv-detail-body` actual, VERBATIM). `openInvDetail` queda:
```js
    function openInvDetail(id) {
      const p = PERFILES.find(x => x.id === id);
      if (!p) return;
      if (state.layout === 'desktop') { openInvDetailDesktop(id); switchPanelTab('detail'); return; }
      $('#inv-detail-title').textContent = p.name;
      $('#inv-detail-body').innerHTML = buildInvDetailBodyHTML(p);
      showScreen('screen-inv-detail');
      refreshIcons();
    }
```
> Verificá los ids reales del detalle inversor (`#inv-detail-title`/`#inv-detail-body` u otros) con rg y usá los que existan.

- [ ] **Step 2:** Agregar funciones desktop inversor (reemplaza el stub `renderLayoutDesktopInversor` de Task 2):
```js
    function renderLayoutDesktopInversor() {
      reparentToDesktop('screen-inv-perfil', 'desk-sidebar');
      reparentToDesktop('screen-inv-feed', 'desk-feed');
      renderRightPanelInversor();
      renderInvStack();
    }
    function renderRightPanelInversor() {
      if (state.layout !== 'desktop') return;
      reparentToDesktop('screen-inv-detail', 'desk-panel-detail');
      reparentToDesktop('screen-inv-matches', 'desk-panel-matches');
      switchPanelTab(state.panelTab);
      renderRightPanelTopCardInversor();
      renderInvMatches();
    }
    function renderRightPanelTopCardInversor() {
      if (state.layout !== 'desktop') return;
      if (PERFILES.length - state.invIdx <= 0) return;
      openInvDetailDesktop(PERFILES[state.invIdx].id);
    }
    function openInvDetailDesktop(id) {
      const p = PERFILES.find(x => x.id === id);
      if (!p) return;
      $('#inv-detail-title').textContent = p.name;
      $('#inv-detail-body').innerHTML = buildInvDetailBodyHTML(p);
      refreshIcons();
    }
```

- [ ] **Step 3:** Hook en `swipeInversor()`. Localizar `renderInvStack();` dentro del setTimeout:
```js
        renderInvStack();
        if (state.layout === 'desktop') {
          renderRightPanelTopCardInversor();
          if (state.panelTab === 'matches') renderInvMatches();
        }
```

- [ ] **Step 4:** Decisión de scope: el `screen-inv-dashboard` NO se incluye en el desktop de esta tanda (la spec no lo pide como panel). En desktop, ocultar el botón que navega al dashboard desde el perfil inversor con CSS `body.desktop`. Verificá con rg el selector real del botón (`showScreen('screen-inv-dashboard')`) y agregá la regla.

- [ ] **Step 5:** Verificar (agent-browser desktop, `setRole('inversor')`): eval `document.getElementById('desk-sidebar').querySelector('#screen-inv-perfil')!==null` true; feed con perfiles; `document.getElementById('inv-detail-title').textContent` → nombre del perfil top; swipe actualiza. Screenshot.

- [ ] **Step 6:** Commit: `feat: layout desktop inversor (sidebar, feed, panel con tabs)`

---

### Task 5: Default por viewport + persistencia + wiring en init()/setRole()/selectUser()

**Objetivo:** Conectar al ciclo de vida real.

**Files:** Modify `index.html`
**Interfaces:**
- Consumes: `LAYOUT_KEY`, `window.innerWidth`, `init`, función de selección de rol, `selectUser`, `backToSplash`/equivalente.
- Produces: init wiring; selección de rol y `selectUser` usan `renderLayout()`.

- [ ] **Step 1:** En `init()`, reemplazar el bloque que hoy hace `showScreen(...)` final por:
```js
      renderBottomNavs();
      const savedLayout = localStorage.getItem(LAYOUT_KEY);
      if (savedLayout === 'desktop' || savedLayout === 'mobile') state.layout = savedLayout;
      else state.layout = window.innerWidth >= 1024 ? 'desktop' : 'mobile';
      if (!state.role) state.layout = 'mobile';  // splash siempre en mobile
      applyLayoutClass();
      renderLayout();
      refreshIcons();
```
> Verificá el bloque final real de `init()` (Task previa lo dejó cargando MODELO/PERFILES y luego showScreen) y reemplazá solo la parte de routing.

- [ ] **Step 2:** La función que setea el rol (buscá con rg: puede llamarse `setRole`, `chooseRole`, o ser inline en los botones del splash). Hacé que, tras setear y persistir el rol, llame a `renderLayout()` en vez de `showScreen(...)`, y resetee `state.panelTab='detail'`. Ejemplo si es `setRole`:
```js
    function setRole(role) {
      state.role = role;
      localStorage.setItem(STORAGE_KEY, role);
      renderBottomNavs();
      state.panelTab = 'detail';
      renderLayout();
    }
```

- [ ] **Step 3:** `selectUser(idx)` → reemplazar el `showScreen('screen-joven-perfil')` final por `renderLayout();` (respeta el modo actual). Conservar el reset de `jovenIdx/jovenSwipes/jovenMatches` y `closeUserSelector()`.

- [ ] **Step 4:** La función de volver al splash (buscá `backToSplash` o equivalente / botón cambiar rol que vuelve a splash): forzar mobile:
```js
      state.role = null;
      localStorage.removeItem(STORAGE_KEY);
      state.layout = 'mobile';
      applyLayoutClass();
      renderLayoutMobile();
```
> Si "cambiar de rol" alterna joven↔inversor directamente (no vuelve a splash), entonces solo asegurate de que llame `renderLayout()` tras cambiar el rol.

- [ ] **Step 5:** Verificar (agent-browser): viewport 1440 + recargar con rol guardado → `state.layout` 'desktop'; viewport 390 + recargar (limpiando `vinder_layout`) → 'mobile'; toggle persiste (`localStorage.getItem('vinder_layout')`); cambiar de usuario en desktop mantiene desktop.

- [ ] **Step 6:** Commit: `feat: wiring de layout en init/setRole/selectUser + default por viewport y persistencia`

---

### Task 6: Polish CSS — paneles, overlays, topbar context

**Objetivo:** Que las secciones se vean como paneles y los overlays funcionen en desktop.

**Files:** Modify `index.html`
**Interfaces:** Consumes CSS existente + `body.desktop`. Produces CSS adicional + `renderDeskContext()`.

- [ ] **Step 1:** CSS de paneles (agregar al bloque desktop). Ajustá selectores a los reales (verificá headers/clases con rg):
```css
    body.desktop #screen-joven-feed .stack,
    body.desktop #screen-inv-feed .stack { max-width:420px; margin:0 auto; }
    body.desktop #user-selector-panel { position:fixed; inset:0; z-index:500; }
    body.desktop #match-modal { position:fixed; inset:0; z-index:600; }
```
> Verificá el id real del modal de match (`rg -n 'match-modal|openMatchModal' index.html`) y usá el correcto.

- [ ] **Step 2:** Agregar `renderDeskContext()` y llamarlo al final de `renderLayoutDesktopJoven()`, `renderLayoutDesktopInversor()` y `selectUser()`:
```js
    function renderDeskContext() {
      const el = document.getElementById('desk-context');
      if (!el || state.layout !== 'desktop') return;
      if (state.role === 'joven') {
        const j = jovenActivo();
        el.innerHTML = `<span class="desk-chip"><i data-lucide="user-round" class="w-3.5 h-3.5"></i>${j.name}</span><span class="desk-chip desk-chip--score">${j.score.toFixed(1)}</span>`;
      } else if (state.role === 'inversor') {
        el.innerHTML = `<span class="desk-chip"><i data-lucide="building-2" class="w-3.5 h-3.5"></i>Mutual Bonaerense</span>`;
      } else { el.innerHTML = ''; }
      refreshIcons();
    }
```
> En `selectUser`, llamá `renderDeskContext()` después de `renderLayout()` (solo tiene efecto en desktop por el guard).

- [ ] **Step 3:** Verificar (agent-browser desktop): el selector de usuario abre como overlay full-screen; el modal de match (hacer un accept) se ve centrado y no detrás del shell; la topbar muestra el chip con nombre+score del joven. Screenshots.

- [ ] **Step 4:** Commit: `feat: polish CSS de paneles desktop + overlays + chips de contexto en topbar`

---

### Task 7: Smoke test integral (agent-browser)

**Objetivo:** Verificar todo el comportamiento en navegador. No se toca código.

**Files:** ninguno.

- [ ] **Step 1:** Servir `python3 -m http.server 8765` (background).

- [ ] **Step 2:** Ejecutar escenarios (agent-browser), reportando real vs esperado + screenshots:
  - **A. Desktop joven:** viewport 1440x900, abrir, rol joven. `document.body.classList.contains('desktop')`→true; `#desk-sidebar` con `#screen-joven-perfil`; `#desk-feed` con `#screen-joven-feed`; `#joven-detail-title` con nombre del top. Screenshot.
  - **B. Toggle→mobile:** `toggleLayout()` → `body.mobile` true; `.device-wrap` visible; `#screen-joven-perfil` de vuelta en `#device`. Screenshot.
  - **C. Persistencia:** `localStorage.getItem('vinder_layout')` coincide con el modo actual.
  - **D. Mobile angosto:** limpiar `vinder_layout`, viewport 390x800, recargar → `body.mobile` true por default. Screenshot.
  - **E. Inversor desktop:** viewport 1440, `setRole('inversor')` → `#desk-sidebar` con `#screen-inv-perfil`; feed de perfiles; `#inv-detail-title` con nombre. Screenshot.
  - **F. Swipe en vivo:** en desktop joven, `swipeJoven('reject')` → `#joven-detail-title` cambia al siguiente.
  - **G. Tabs:** `switchPanelTab('matches')` → `#desk-panel-matches` activo; clickear/llamar `openJovenDetail(<id de un match>)` → vuelve a Detalle con ese instrumento.
  - **H. Mobile sin regresiones:** en mobile, recorrer splash→rol→perfil→feed→swipe→matches→detalle; navs y modales OK.

- [ ] **Step 3:** Reportar fallos con valor real vs esperado + screenshot. El controlador decide fixes.

---

## Riesgos

| Riesgo | Mitigación |
|--------|-----------|
| Reparenting rompe overlays (`#user-selector-panel`, match modal) posicionados sobre `#device` (oculto en desktop) | CSS `body.desktop ... { position:fixed; inset:0; z-index alto }` — Task 6 |
| `showScreen` llamado desde código no actualizado | Guard `if (state.layout==='desktop') return` (Task 1) cubre todos los casos |
| Helpers extraídos no idénticos al HTML original | Extraer VERBATIM; el reviewer compara contra el original |
| Stack de swipe necesita altura en el panel feed | CSS de `.stack`/contenedor en Task 6; verificar en smoke test |
| Nombres de función reales difieren (setRole, ids de detalle, match modal) | Cada task indica verificar con rg y usar los reales |
| Cache del navegador | Hard-reload en cada verificación |

**Ejecución: secuencial estricta** — todas tocan `index.html`.
