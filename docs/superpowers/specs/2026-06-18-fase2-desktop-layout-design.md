# Fase 2 — Tanda 2: Layout desktop multi-panel + toggle mobile↔desktop

**Fecha:** 2026-06-18
**Estado:** Diseño, pendiente de revisión del spec
**Contexto:** Segunda tanda de Fase 2 (ver memoria `fase2/secuencia-perfiles-y-desktop`). El prototipo Vinder hoy es un mockup de teléfono (`.device`) con modelo de "una pantalla activa por vez" (`showScreen`). Esta tanda agrega una vista desktop multi-panel opcional, sin tocar la experiencia mobile.

## Objetivo

Permitir alternar manualmente entre la vista **mobile** (actual, intacta) y una vista **desktop** multi-panel (layout A: sidebar + feed + panel derecho), para presentaciones en pantalla grande. Ambos roles (joven e inversor) tienen layout desktop.

## No-objetivos

- No se rediseña ni modifica la experiencia mobile.
- No se cambia ninguna lógica de negocio (datos, elegibilidad, matching, scoring) — solo presentación.
- No hay layout responsive fluido intermedio: son dos modos discretos (mobile / desktop) con un toggle.

## Layout desktop (A): sidebar + feed + panel derecho

Grid de 3 columnas:

- **Sidebar (izquierda):** logo/marca, toggle de rol (joven↔inversor), selector de usuario (en rol joven), navegación (Perfil/Feed/Matches), y el resumen de perfil + Vinder Score (joven) o perfil de la entidad (inversor). Reusa el contenido de la pantalla de perfil.
- **Feed (centro):** el stack de tarjetas swipeable (instrumentos para joven; perfiles para inversor) con sus botones aceptar/descartar. Reusa el feed actual.
- **Panel derecho:** dos pestañas:
  - **Detalle** — el detalle de la tarjeta que está arriba del feed, actualizado en vivo a medida que se swipea. Reusa el render de detalle.
  - **Matches** — la lista de vínculos aceptados; al clickear uno, su detalle se muestra en la pestaña Detalle.

## Toggle mobile ↔ desktop

- Botón manual (ícono monitor/teléfono) siempre visible, alterna el modo.
- Default en primera carga: `window.innerWidth >= 1024` → desktop; si no → mobile.
- La elección se persiste en `localStorage` (clave `vinder_layout`), igual que el rol (`vinder_role`).
- Cambiar de modo re-renderiza la vista actual en el shell correspondiente; el estado de la app (rol, joven activo, swipes, matches) se conserva.

## Arquitectura

**Principio rector: reusar las funciones de render existentes, cero lógica duplicada, mobile intacto.**

- **Shell desktop** (`#desktop-shell`): contenedor CSS Grid con tres slots — `#desk-sidebar`, `#desk-feed`, `#desk-panel` — más una topbar mínima con el toggle. Oculto en modo mobile; visible en modo desktop.
- **Mounting target-aware:** las funciones de render que hoy escriben a ids fijos (`#joven-stack`, `#joven-score-bars`, `#joven-perfil-body`, `#joven-detail-body`, `#joven-matches-list`, y sus equivalentes inversor) pasan a resolver su contenedor destino mediante un helper `mountTarget(slot)` que devuelve el elemento correcto según `state.layout`:
  - en `mobile` → el contenedor de pantalla actual (comportamiento de hoy).
  - en `desktop` → el slot del panel correspondiente (`#desk-sidebar`/`#desk-feed`/`#desk-panel`).
- **`state.layout`** (`'mobile' | 'desktop'`) nuevo en el estado, persistido.
- **`renderLayout()`**: función orquestadora que, según `state.layout` y `state.role`, monta perfil→sidebar, feed→feed, detalle/matches→panel (desktop), o delega a `showScreen` (mobile). El toggle y los cambios de rol/usuario llaman a `renderLayout()`.
- **CSS:** un `body.desktop` activa el grid y reestila los bloques reusados para que encajen como paneles (sin el cromo de pantalla full: headers/navs/padding adaptados por CSS, no por duplicación de HTML). El `.device` (marco de teléfono) se muestra solo en `body.mobile`.

Esto mantiene una sola fuente de render por cada pieza de UI; el modo solo cambia **dónde** se monta y **cómo** se estila.

## Componentes / unidades

- **`#desktop-shell` + slots** — HTML del contenedor desktop (sidebar/feed/panel/topbar).
- **`state.layout` + persistencia** — lectura/escritura en localStorage, default por viewport.
- **`mountTarget(slot)`** — helper que resuelve el contenedor destino por modo.
- **`renderLayout()`** — orquestador de montaje según modo+rol.
- **`toggleLayout()`** — alterna modo, persiste, re-renderiza.
- **Panel derecho** — `renderRightPanel()` con pestañas Detalle/Matches y `state.panelTab`.
- **CSS `body.desktop`** — grid y reestilado de los bloques reusados.

## Flujo de datos / interacción

1. `init()` (tras cargar MODELO+PERFILES) lee `state.layout` de localStorage o lo deduce del ancho, setea `body.classList` y llama `renderLayout()`.
2. En desktop: `renderLayout()` monta perfil→sidebar, feed→feed, panel derecho (Detalle de la tarjeta top por defecto).
3. Swipe: al cambiar la tarjeta top, se re-renderiza el panel Detalle en vivo (si la pestaña activa es Detalle).
4. Aceptar: agrega a matches; si la pestaña activa es Matches, se actualiza la lista.
5. Toggle: `toggleLayout()` cambia modo, persiste y re-renderiza conservando estado.
6. Cambio de rol / usuario: re-render del shell activo.

## Manejo de errores

- Sin nuevas fuentes de error de datos (no hay fetch nuevo). Si algún slot no existe en el DOM, los helpers de render hacen guard (no-op) como ya hacen hoy.

## Verificación

Sin framework de tests. Smoke test en navegador (agent-browser, servido), en viewport ancho (ej. 1440px) y angosto (ej. 390px):

1. Carga en viewport ancho → arranca en desktop (grid de 3 paneles visible, sin marco de teléfono).
2. Carga en viewport angosto → arranca en mobile (marco de teléfono, una pantalla).
3. Toggle desktop→mobile y viceversa: el estado (rol, joven activo, matches) se conserva; `localStorage.vinder_layout` se actualiza.
4. Desktop joven: sidebar con perfil+score+selector, feed swipeable al centro, panel derecho con Detalle del top que cambia al swipear; pestaña Matches lista los aceptados.
5. Desktop inversor: mismo patrón con perfiles.
6. El selector de usuario y el cambio de rol funcionan en ambos modos.
7. Mobile sin regresiones: las pantallas y navs siguen igual que antes de esta tanda.

## Riesgos

- `index.html` ya es grande; el shell + CSS desktop suma volumen. Mantener el patrón existente, sin duplicar lógica.
- Reestilar bloques pensados para pantalla full como paneles puede requerir ajustes finos de CSS (headers/navs que sobran en desktop). Resolver por CSS condicional (`body.desktop`), no duplicando HTML.
- El modelo "una pantalla activa" (`showScreen`) coexiste con el shell desktop: definir con claridad que en desktop manda `renderLayout()` y `showScreen` solo opera en mobile (o se adapta para enrutar al panel correcto).
- Cache del navegador (sin cambios de datos, pero el index.html cambia): hard-reload al verificar.
