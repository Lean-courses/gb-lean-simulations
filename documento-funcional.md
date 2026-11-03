# Documento funcional — Lean Six Sigma Simulations Hub (GreenBelt)

**Versión:** 1.0 · **Fecha:** 2026-11-03 · **Repositorio:** `drotondi/GreenBelt` · **Rama de referencia:** `main` (commit `9d61b69`)

**Propósito de este documento:** permitir que un programador que nunca vio el proyecto lo replique desde cero, o lo extienda, sin necesidad de leer el código fuente. Describe qué hace el producto, por qué está construido así, cómo está organizado, y la especificación funcional detallada de cada simulación (modelo matemático, controles, métricas, estados y textos).

---

## Índice

1. [Resumen ejecutivo](#1-resumen-ejecutivo)
2. [Restricciones de diseño (no negociables)](#2-restricciones-de-diseño-no-negociables)
3. [Arquitectura general](#3-arquitectura-general)
4. [Estructura de carpetas](#4-estructura-de-carpetas)
5. [Capa compartida (`shared/`)](#5-capa-compartida-shared)
6. [Registry y página Home](#6-registry-y-página-home)
7. [Anatomía de una página de simulación](#7-anatomía-de-una-página-de-simulación)
8. [Simulación 1 — Galton Board](#8-simulación-1--galton-board)
9. [Simulación 2 — Little's Law](#9-simulación-2--littles-law)
10. [Simulación 3 — Push vs. Pull](#10-simulación-3--push-vs-pull)
11. [Simulación 4 — Process capability (Cp/Cpk)](#11-simulación-4--process-capability-cpcpk)
12. [Accesibilidad](#12-accesibilidad)
13. [Testing](#13-testing)
14. [Despliegue y distribución](#14-despliegue-y-distribución)
15. [Guía de replicación paso a paso](#15-guía-de-replicación-paso-a-paso)
16. [Glosario](#16-glosario)

---

## 1. Resumen ejecutivo

### 1.1 Qué es

Un sitio web estático con **simulaciones interactivas de conceptos Lean Six Sigma**. Cada simulación es una página autónoma que anima un modelo matemático en un `<canvas>` y deja que el usuario mueva parámetros para ver el efecto en vivo.

No es un curso ni un LMS: es material de apoyo para que un instructor (o un participante por su cuenta) **vea** un concepto que normalmente se explica con fórmulas.

### 1.2 Para quién

| Audiencia | Uso |
|---|---|
| Instructor Green/Black Belt | Proyectar en clase, mover parámetros en vivo, correr el juego Push vs. Pull con un grupo |
| Participante de la formación | Explorar solo, repetir escenarios, verificar que entendió |
| Facilitador de taller | Modo Juego con semilla compartida: todos los equipos viven los mismos eventos ocultos |

### 1.3 Alcance funcional actual

Cuatro simulaciones publicadas más una plantilla oculta:

| # | Simulación | Concepto que enseña | Estado |
|---|---|---|---|
| 1 | Galton Board | Variación por causas comunes y origen de la distribución normal | `live` |
| 2 | Little's Law: 3-process line | WIP = Throughput × Lead time; cuello de botella; inestabilidad | `live` |
| 3 | Push vs. Pull: planning game | MRP por pronóstico vs. kanban; inventario vs. nivel de servicio | `live` |
| 4 | Process capability (Cp/Cpk) | Voz del proceso vs. voz del cliente; Cp/Cpk; causas especiales | `live` |
| — | `_template` | Esqueleto copiable para crear simulaciones nuevas | `draft` (oculto del Home) |

### 1.4 Idiomas

Inglés, español y portugués. El idioma se resuelve en este orden:

1. Parámetro de URL `?lang=en|es|pt`
2. Idioma del navegador (`navigator.languages`)
3. Inglés (default)

El idioma elegido **persiste entre páginas**: todos los enlaces internos reescriben su `href` agregando `?lang=` del idioma activo.

---

## 2. Restricciones de diseño (no negociables)

Estas restricciones condicionan todas las decisiones técnicas. Un reimplementador que las ignore produce otro producto.

| # | Restricción | Razón |
|---|---|---|
| R1 | **Sitio 100% estático**: solo HTML, CSS y JavaScript vanilla. Sin frameworks, sin build step, sin npm, sin bundler | Debe poder distribuirse como un ZIP y abrirse sin instalar nada |
| R2 | **Debe funcionar abriendo `index.html` con doble clic** (protocolo `file://`) | El firewall corporativo de Amcor bloquea `github.io`. La distribución real es por ZIP (Teams/SharePoint/correo) |
| R3 | **Cero peticiones externas**: sin CDNs, sin fuentes remotas, sin analytics, sin telemetría | Funciona offline y sin fugas de información hacia terceros |
| R4 | **Nunca versionar los activos de marca de Amcor** (`assets/fonts/`, `assets/logo-*`). Están en `.gitignore` | Son propiedad de Amcor; el repositorio es público |
| R5 | **Todo texto visible pasa por i18n** en los tres idiomas. Cero strings hardcodeados en la UI | Coherencia y mantenibilidad del material en LatAm |
| R6 | **Todo color, radio, sombra y duración sale de `shared/tokens.css`** | Una sola fuente de verdad del sistema de diseño Amcor |
| R7 | Los motores de simulación son **puros y testeables en Node**, sin acceso al DOM | Permite tests unitarios sin navegador |
| R8 | Los costos y umbrales mostrados son **ilustrativos, nunca cifras de Amcor ni requisitos de un cliente** | Integridad de la información |

### 2.1 Consecuencia crítica de R2: nada de ES modules

Los navegadores **bloquean los ES modules bajo `file://`** (política CORS: un `import` desde `file://` falla con error de origen opaco). Por eso:

- Todo el JavaScript usa `<script defer src="...">` **clásicos**, no `type="module"`.
- No existe ningún `import` / `export` en el código de la aplicación.
- Todo lo compartido vive bajo **un único namespace global: `window.LSS`**.
- **El orden de los `<script>` importa** y está fijado en cada `index.html`.
- Los enlaces apuntan siempre a un **`index.html` explícito** (`simulations/galton-board/index.html`), nunca a la carpeta, porque un enlace a carpeta no resuelve a `index.html` desde disco.

Cada módulo compartido sigue este patrón de IIFE:

```js
(function (LSS) {
  'use strict';
  // ...
  LSS.miModulo = { /* API pública */ };
})(window.LSS = window.LSS || {});
```

Los motores que además deben correr en Node usan una variante de doble exportación:

```js
(function (root) {
  'use strict';
  const api = { /* ... */ };
  if (typeof module !== 'undefined' && module.exports) module.exports = api;   // Node (tests)
  if (root && root.LSS) root.LSS.MiMotor = api;                                 // navegador
})(typeof window !== 'undefined' ? window : globalThis);
```

### 2.2 Namespace `window.LSS`

| Clave | Archivo | Responsabilidad |
|---|---|---|
| `LSS.config` | `shared/config.js` | Flag de activos de marca y URL raíz del sitio |
| `LSS.i18n` | `shared/i18n.js` | Diccionarios, traducción, formato de números, enlaces con idioma |
| `LSS.icons` | `shared/icons.js` | Iconos SVG inline |
| `LSS.motion` | `shared/motion.js` | Curva de easing compartida para animaciones en canvas |
| `LSS.header` | `shared/header.js` | Header, selector de idioma, footer |
| `LSS.registry` | `simulations/registry.js` | Catálogo de simulaciones que alimenta el Home |
| `LSS.sim` | `shared/sim/random.js` | RNG con semilla y distribuciones |
| `LSS.chart` | `shared/sim/chart.js` | Helpers de dibujo de gráficos en canvas |
| `LSS.LittleEngine`, `LSS.PushPull`, … | `simulations/*/engine.js` | Motores de cada simulación |

---

## 3. Arquitectura general

### 3.1 Modelo mental

```
                    ┌──────────────────────────────┐
                    │  index.html  (Home)          │
                    │  lee simulations/registry.js │
                    │  y pinta una tarjeta por     │
                    │  simulación con status=live  │
                    └───────────────┬──────────────┘
                                    │  enlace con ?lang=
                                    ▼
        ┌───────────────────────────────────────────────────┐
        │  simulations/<id>/index.html                      │
        │                                                   │
        │   shared/  →  tokens.css · base.css               │
        │               config · i18n · icons · motion ·    │
        │               header · sim/random · sim/chart     │
        │                                                   │
        │   propios →  <id>.css                             │
        │               i18n.js   (diccionario EN/ES/PT)    │
        │               engine.js (modelo puro, testeable)  │
        │               <id>.js   (UI + render + frame loop)│
        └───────────────────────────────────────────────────┘
```

### 3.2 Separación de responsabilidades dentro de una simulación

Esta separación es la decisión de arquitectura más importante a nivel de simulación:

| Archivo | Qué contiene | Qué NO contiene |
|---|---|---|
| `engine.js` | El modelo matemático puro: estado, eventos, métricas. Determinista dada una semilla | Ningún acceso al DOM, ningún color, ningún texto |
| `<id>.js` | Lectura de controles, bucle de animación (`requestAnimationFrame`), dibujo en canvas, formato de números, textos vía i18n | Lógica del modelo |
| `i18n.js` | Todos los textos visibles en EN/ES/PT bajo un prefijo propio | Lógica |
| `<id>.css` | Estilos específicos de esa simulación, usando siempre variables de `tokens.css` | Valores de color/espaciado hardcodeados |

Las simulaciones simples (Galton Board) fusionan motor y UI en un solo archivo porque el modelo es trivial (una secuencia de Bernoulli). Las complejas (Little's Law, Push vs. Pull, Process capability) separan `engine.js`.

### 3.3 Flujo de arranque de una página

1. El navegador carga el HTML y los `<script defer>` **en orden declarado**.
2. Cada script registra su API en `window.LSS`.
3. El `i18n.js` de la simulación llama a `LSS.i18n.registerStrings({en, es, pt})`, fusionando su diccionario con el compartido.
4. El `<id>.js` de la simulación:
   - llama a `LSS.header.renderHeader({ variant: 'sim' })` y `renderFooter()`
   - construye su estado inicial y sus listeners
   - se suscribe con `LSS.i18n.onLangChange(cb)` para repintar textos y etiquetas de canvas al cambiar idioma
   - llama a `LSS.i18n.initI18n()` **al final**, que aplica traducciones y dispara los callbacks
5. Arranca el `requestAnimationFrame` loop.

> **Regla:** `initI18n()` se llama una sola vez por página y siempre al final del arranque, después de registrar los strings y de renderizar el chrome compartido.

---

## 4. Estructura de carpetas

```
GreenBelt/
├── index.html                      Home: pinta tarjetas desde el registry
├── README.md                       Documentación corta para el repositorio
├── LICENSE                         Todos los derechos reservados
├── .gitignore                      Ignora assets/fonts/ y assets/logo-*
├── .nojekyll                       Evita que GitHub Pages procese con Jekyll
│
├── shared/
│   ├── tokens.css                  Design tokens (única fuente de verdad visual)
│   ├── base.css                    Estilos base y componentes compartidos
│   ├── config.js                   LSS.config  (flag de marca, ROOT_URL)
│   ├── i18n.js                     LSS.i18n
│   ├── icons.js                    LSS.icons
│   ├── motion.js                   LSS.motion
│   ├── header.js                   LSS.header
│   └── sim/
│       ├── random.js               LSS.sim    (RNG con semilla, gamma, normal)
│       └── chart.js                LSS.chart  (helpers de gráficos en canvas)
│
└── simulations/
    ├── registry.js                 Catálogo que alimenta el Home
    ├── _template/                  Esqueleto copiable (status draft)
    │   ├── index.html
    │   ├── sim.js
    │   ├── sim.css
    │   ├── i18n.js
    │   └── thumbnail.svg
    ├── galton-board/
    │   ├── index.html · galton.js · galton.css · i18n.js · thumbnail.svg
    ├── littles-law/
    │   ├── index.html · engine.js · engine.test.js
    │   ├── littles-law.js · littles-law.css · i18n.js · thumbnail.svg
    ├── push-pull/
    │   ├── index.html · engine.js · engine.test.js
    │   ├── push-pull.js · game.js · push-pull.css · i18n.js · thumbnail.svg
    └── process-capability/
        ├── index.html · engine.js · engine.test.js
        ├── capability.js · spc.js · challenge.js
        ├── capability.css · i18n.js · thumbnail.svg
```

**Tamaño total:** ~10.000 líneas, sin dependencias de terceros.

---

## 5. Capa compartida (`shared/`)

### 5.1 `shared/config.js` → `LSS.config`

```js
LSS.config = {
  USE_BRAND_ASSETS: false,
  ROOT_URL: new URL('../', document.currentScript.src),
  BRAND_ASSETS: {
    font: 'assets/fonts/AmcorPro-Book.ttf',
    logo: 'assets/logo-amcor-mark.png',
  },
};
```

- `ROOT_URL` se deriva de la ubicación del propio script (`document.currentScript.src`), no de `location`. Así cada página de simulación resuelve correctamente la raíz del sitio, esté donde esté, también bajo `file://`.
- `USE_BRAND_ASSETS`:
  - `false` (default, seguro para repo público): tipografía Arial/Helvetica, sin logo.
  - `true`: inyecta una regla `@font-face` para `AmcorPro` con `font-display: swap`, cambia `--font-family` a `--font-family-brand`, y muestra el logo de 28×28 px en el header.
- **Nota operativa:** algunos navegadores (Chrome, Edge) bloquean archivos de fuente cargados desde `file://`. En ese caso el logo sí se ve y el texto cae a Arial. Es un degradado aceptado.

### 5.2 `shared/tokens.css` — sistema de diseño

Única fuente de verdad visual. El código de canvas lee estos tokens con `getComputedStyle(document.documentElement).getPropertyValue('--nombre')`, de modo que **los gráficos y el HTML nunca se desincronizan**.

**Marca:**

| Token | Valor |
|---|---|
| `--amcor-navy` | `#00395A` |
| `--amcor-cyan` | `#00A1DE` |
| `--amcor-teal` | `#1BCDAC` |
| `--amcor-green` | `#00A551` |
| `--gradient-signature` | `linear-gradient(90deg, #1BCDAC 0%, #24BBEE 50%, #00A1DE 100%)` |

Rampas completas: `--navy-900…--navy-050`, `--cyan-700…--cyan-050`, y neutrales con tinte azul `--neutral-0…--neutral-700`.

**Roles semánticos:** `--color-bg`, `--color-surface`, `--color-text`, `--color-text-muted` (7,9:1 sobre blanco), `--color-text-subtle` (5,2:1), `--color-border`, `--color-interactive`, `--color-interactive-strong` (≥3:1, para UI no textual), `--color-link` (4,9:1), `--color-focus`, `--color-primary`.

**Colores de estado — significado reservado, nunca decorativo:**

| Token | Valor | Significado |
|---|---|---|
| `--color-bottleneck` | `#E98300` (naranja) | Cuello de botella |
| `--color-success` | `#00A551` (verde) | Verificado / dentro de control |
| `--color-danger` | `#C90062` (magenta) | Inestable / fuera de especificación |
| `--color-danger-tint` | `#fbe6ef` | Fondo tenue del estado de peligro |

> **Regla de contraste:** naranja y verde **no** cumplen AA como texto pequeño sobre blanco. Se usan solo en bordes, marcadores e iconos; la etiqueta de texto va en navy. El magenta sí cumple (5,7:1) y puede usarse como texto.

**Paleta de simulación:** `--sim-peg`, `--sim-ball`, `--sim-ball-highlight`, `--sim-ball-edge`, `--sim-curve`, `--sim-grid`, `--sim-divider`, `--sim-floor`, `--sim-label`, y las tres paradas del degradado `--sim-gradient-0/50/100`.

**Tipografía:** una sola familia. Escala `--font-size-xs` (0,75rem) → `--font-size-2xl` (2rem). `--line-height-tight: 1.2`, `--line-height-base: 1.5`.

**Espaciado:** escala `--space-1: 4px` → `--space-8: 64px`.

**Forma:** `--radius-control: 6px`, `--radius-card: 8px`, `--radius-chip: 4px`.

**Movimiento:** `--ease-standard: cubic-bezier(.2, 0, .2, 1)`, `--duration-fast: 120ms`, `--duration-base: 200ms`, `--duration-slow: 320ms`.

**Layout:** `--container-max: 1200px`, `--header-height: 64px`, `--accent-bar-height: 3px`.

### 5.3 `shared/base.css` — componentes compartidos

Inventario de clases reutilizables:

| Clase | Qué es |
|---|---|
| `.skip-link` | Enlace "saltar al contenido", visible solo con foco de teclado |
| `.container` | Ancho máximo 1200 px con padding responsive |
| `.site-header`, `.site-header--home`, `.site-header--sim` | Header en sus dos variantes |
| `.accent-bar` | Barra de 3 px con el degradado de marca bajo el header |
| `.lang-switch`, `.lang-switch__btn` | Selector EN·ES·PT; el activo usa `aria-pressed="true"` |
| `.back-link` | "← Inicio" en páginas de simulación |
| `.card`, `.card-grid` | Tarjeta y grilla responsive (1 / 2 / 3 columnas) |
| `.sim-card` y modificadores `__media __body __title __desc __action` | Tarjeta de simulación del Home |
| `.chips`, `.chip` | Etiquetas de concepto |
| `.btn`, `.btn--primary`, `.btn--block`, `.btn-row` | Botones |
| `.control-panel`, `.control-group`, `.field`, `.field__head/__label/__value/__range` | Panel de controles |
| `input[type=range]` | Slider con relleno progresivo vía variable CSS `--fill` |
| `.switch` | Checkbox con apariencia de interruptor (`role="switch"`) |
| `.sim-layout`, `.sim-stage`, `.sim-controls`, `.sim-about` | Grilla de la página de simulación |
| `.stats`, `.stat`, `.stat--theory` | Fila de lecturas numéricas (`font-variant-numeric: tabular-nums`) |
| `.disclosure` | `<details>` estilizado para la sección "Acerca de" |
| `.hint`, `kbd` | Pistas y teclas |
| `.sr-only` | Solo para lectores de pantalla |

**Breakpoints:**

| Ancho | Efecto |
|---|---|
| ≥ 640 px | Grilla del Home a 2 columnas |
| ≥ 720 px | Padding mayor; `.stats` a 5 columnas |
| ≥ 960 px | `.sim-layout` pasa a dos columnas: escenario + panel de controles **sticky** |
| ≥ 1024 px | Grilla del Home a 3 columnas |

**Orden en móvil (una columna):** `stage` → `controls` → `about`. El usuario ve primero la animación.

**`prefers-reduced-motion: reduce`:** desactiva transiciones y el "lift" de las tarjetas al pasar el mouse. Cada simulación además salta las animaciones y aplica el resultado de inmediato (ver cada sección).

### 5.4 `shared/i18n.js` → `LSS.i18n`

Idiomas soportados: `['en', 'es', 'pt']`. Locales de formato: `en-US`, `es-AR`, `pt-BR`. **Esto implica que los números se formatean con coma decimal en español y portugués** (`0,50`) y con punto en inglés (`0.50`).

**API pública:**

| Función | Qué hace |
|---|---|
| `registerStrings({en, es, pt})` | Fusiona un diccionario con el global. Cada simulación llama a esto con su prefijo |
| `t(key, vars)` | Traduce. Interpola `{placeholders}`. Cae a inglés, y luego a la propia clave |
| `has(key)` | ¿Existe la clave? |
| `localize(obj)` | Toma `{en, es, pt}` y devuelve el valor del idioma activo (lo usa el registry) |
| `getLang()` / `getLocale()` | `'es'` / `'es-AR'` |
| `formatNumber(v, opts)` | `Intl.NumberFormat` con caché de formateadores por idioma+opciones |
| `withLang(href, lang)` | Agrega `?lang=` preservando otros parámetros y el hash |
| `applyTranslations(root)` | Recorre el DOM y aplica traducciones |
| `updateLinks(root)` | Reescribe los `href` de los enlaces marcados |
| `onLangChange(cb)` | Suscripción; devuelve función para desuscribir |
| `setLang(lang)` | Cambia idioma, actualiza URL y DOM, notifica suscriptores |
| `initI18n()` | Arranque: sincroniza el documento y dispara los callbacks |

**Enlaces declarativos en el HTML:**

| Atributo | Efecto |
|---|---|
| `data-i18n="clave"` | Asigna `textContent` |
| `data-i18n-attr="aria-label:clave,title:clave2"` | Asigna atributos |
| `data-keep-lang` (+ `data-href` opcional) | El `href` se reescribe con `?lang=` del idioma activo |

**Detalle de robustez:** `setLang()` envuelve `history.replaceState()` en `try/catch`, porque algunos contextos `file://` restringidos lanzan excepción. Si falla, los enlaces igual llevan `?lang=`, así que la navegación conserva el idioma.

**Claves compartidas** (prefijos `hub.`, `nav.`, `a11y.`, `lang.`, `home.`, `footer.`, `tags.`). El pie de página dice: *"Amcor Digital Transformation Office · Material de formación en Lean Six Sigma"*.

### 5.5 `shared/header.js` → `LSS.header`

- `renderHeader({ variant: 'home' })`: título del hub + subtítulo.
- `renderHeader({ variant: 'sim' })`: enlace "← Inicio" + separador + nombre del hub.
- Ambas variantes incluyen el selector de idioma y la `accent-bar` con el degradado de marca.
- `renderFooter()`: crédito del DTO.
- El `HOME_HREF` se construye como `new URL('index.html', ROOT_URL).href` — **`index.html` explícito** por la restricción R2.

### 5.6 `shared/icons.js` → `LSS.icons`

Iconos SVG inline estilo Lucide: grilla de 24 px, trazo de 2 px, `currentColor`, `aria-hidden="true"`, `focusable="false"`.

Set disponible: `arrowLeft`, `arrowRight`, `play`, `pause`, `rotateCcw`, `arrowDown`, `arrowDownToLine`, `chevronDown`, `info`, `shuffle`, `check`, `alert`, `loader`, `keyboard`.

Uso: `element.innerHTML = LSS.icons.icon('play', { size: 16 })`.

### 5.7 `shared/motion.js` → `LSS.motion`

Expone la misma curva de easing que usa el CSS (`cubic-bezier(.2, 0, .2, 1)`) como **tabla de consulta precalculada de 256 entradas** con interpolación lineal, para que las animaciones de canvas tengan exactamente la misma sensación que las transiciones CSS sin costo de cálculo por frame.

- `LSS.motion.ease(t)` → progreso suavizado para `t ∈ [0, 1]`
- `LSS.motion.makeCubicBezier(x1, y1, x2, y2, n)` → constructor genérico

### 5.8 `shared/sim/random.js` → `LSS.sim`

Núcleo de la reproducibilidad. Pura lógica, sin DOM; exporta también para Node.

| Función | Especificación |
|---|---|
| `mulberry32(seed)` | PRNG de 32 bits, rápido y determinista. Devuelve uniformes en `[0, 1)`. **Misma semilla ⇒ misma secuencia**, que es lo que hace reproducible el modo Juego |
| `makeSampler(rng)` | Devuelve `sample(mean, cv)`: valor **positivo** con media y coeficiente de variación dados. `cv = 0` → devuelve exactamente `mean` (determinista); `cv = 1` → exponencial |
| `sample.normal()` | Normal estándar por Box–Muller **con caché del valor sobrante** (`spare`), del mismo flujo |
| `sample.uniform()` | Uniforme `[0, 1)` del mismo flujo |
| `percentile(xs, q)` | Percentil por rango más cercano |

**Implementación de `sample(mean, cv)`:** gamma de Marsaglia–Tsang con `k = 1 / cv²`, escalada como `gamma(k) · mean / k`. Para `k < 1` usa el truco de aumento `gamma(k+1) · U^(1/k)`.

> **Por qué gamma y no normal:** los tiempos de proceso y los intervalos entre llegadas **no pueden ser negativos**. Una normal truncada distorsiona la media; la gamma parametrizada por media y CV da siempre positivos y permite barrer desde determinista (CV 0) hasta muy variable con un solo mando.

**Nota sobre el flujo compartido:** `normal`, `uniform` y `gamma` consumen del mismo `rng`. Por eso cualquier cambio en el **orden** en que el modelo pide números altera toda la secuencia posterior. Es un contrato implícito que hay que respetar al modificar un motor, o los tests de reproducibilidad fallan.

### 5.9 `shared/sim/chart.js` → `LSS.chart`

Helpers mínimos de gráficos en canvas, sin tocar el DOM más allá del propio canvas.

| Función | Especificación |
|---|---|
| `setup(canvas)` | Escala por DPR (**tope de 2**), redimensiona el backing store solo si cambió, limpia y devuelve `{ctx, w, h, dpr}` |
| `scale(d0, d1, r0, r1)` | Escala lineal; la función devuelta expone `.invert(px)`, `.domain` y `.range` |
| `niceTicks(min, max, count=5)` | Marcas "redondas": paso de 1, 2, 2.5, 5 o 10 × potencia de 10. Normaliza `−0` a `0` |
| `line(ctx, x0,y0,x1,y1, {color,width,dash})` | Línea simple |
| `gridY(ctx, y, ticks, x0, x1, color)` | Líneas de grilla horizontales **alineadas a medio píxel** (`Math.round(y)+0.5`) para que se vean nítidas |
| `polyline(ctx, pts, {color,width,dash})` | Polilínea; los puntos `NaN`/no finitos **cortan** la línea (útil para series con huecos) |
| `dots(ctx, pts, {color, r})` | Puntos; omite los no finitos |
| `vmarkers(ctx, xs, x, y0, y1, {...})` | Marcadores verticales (eventos), recortados al rango |
| `text(ctx, str, px, py, {...})` | Texto a 11 px con familia, alineación, baseline y peso configurables |

> **Regla de oro del canvas en este proyecto:** `devicePixelRatio` se limita a 2. Subir de ahí cuadruplica el área a pintar en pantallas 3× sin ganancia visual perceptible, y hunde el frame rate en móviles.

---

## 6. Registry y página Home

### 6.1 El contrato del registry

`simulations/registry.js` es **la única fuente de verdad del catálogo**. El Home no conoce ninguna simulación de otra manera.

```js
{
  id: 'galton-board',                                   // identificador único, kebab-case
  path: 'simulations/galton-board/index.html',          // index.html EXPLÍCITO (R2)
  thumbnail: 'simulations/galton-board/thumbnail.svg',  // preview 320×180
  status: 'live',                                       // 'live' = visible | 'draft' = oculto
  tags: ['variation', 'normal-distribution'],           // ids; etiqueta desde i18n 'tags.<id>'
  title:       { en: '…', es: '…', pt: '…' },
  description: { en: '…', es: '…', pt: '…' },
}
```

- `liveSimulations()` filtra por `status === 'live'`.
- Una entrada `draft` **sigue siendo accesible por URL directa**: sirve para probar una simulación antes de publicarla.
- Si un `tag` no tiene clave `tags.<id>` en el i18n compartido, el Home genera una etiqueta legible a partir del id (reemplaza guiones por espacios y capitaliza).

**Orden de las entradas = orden de las tarjetas.** Las entradas existentes no se reordenan ni se editan al agregar una nueva: se **agrega al final**.

### 6.2 Comportamiento del Home

1. Renderiza header (`variant: 'home'`) y footer.
2. Pinta una tarjeta `<li><a class="card sim-card">` por cada simulación `live`, con miniatura, título, descripción, chips de concepto y la llamada a la acción "Abrir simulación".
3. Si no hay ninguna `live`, muestra el texto de `home.empty`.
4. **Todo el contenido dinámico pasa por una función `esc()`** que escapa `& < > " '` antes de inyectarse como HTML.
5. `updateLinks(grid)` agrega `?lang=` a cada tarjeta.
6. Se re-renderiza en cada cambio de idioma (`onLangChange(renderCards)`).

**Orden de scripts del Home** (importa):
`config.js` → `i18n.js` → `icons.js` → `header.js` → `registry.js` → script inline.

---

## 7. Anatomía de una página de simulación

Toda simulación sigue la misma estructura. El esqueleto vive en `simulations/_template/`.

### 7.1 Estructura HTML canónica

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title data-i18n="tpl.pageTitle"></title>
  <link rel="icon" href="data:,">                          <!-- evita el 404 del favicon -->
  <link rel="stylesheet" href="../../shared/tokens.css">
  <link rel="stylesheet" href="../../shared/base.css">
  <link rel="stylesheet" href="sim.css">
  <!-- Scripts clásicos. EL ORDEN IMPORTA. -->
  <script defer src="../../shared/config.js"></script>
  <script defer src="../../shared/i18n.js"></script>
  <script defer src="../../shared/icons.js"></script>
  <script defer src="../../shared/motion.js"></script>
  <script defer src="../../shared/header.js"></script>
  <script defer src="i18n.js"></script>
  <script defer src="sim.js"></script>
</head>
<body>
  <a class="skip-link" href="#main" data-i18n="a11y.skip"></a>
  <header id="site-header"></header>

  <main id="main" class="page container">
    <div class="page-intro">
      <p class="eyebrow" data-i18n="tpl.eyebrow"></p>
      <h1 data-i18n="tpl.title"></h1>
      <p class="lead" data-i18n="tpl.lead"></p>
    </div>

    <div class="sim-layout">
      <section class="sim-stage card">
        <div class="stage" id="stage">
          <canvas id="sim-canvas" role="img" data-i18n-attr="aria-label:tpl.canvasLabel"></canvas>
        </div>
        <dl class="stats">…</dl>
        <p class="sr-only" id="live-region" aria-live="polite" aria-atomic="true"></p>
      </section>

      <aside class="sim-controls card control-panel" data-i18n-attr="aria-label:tpl.controlsLabel">
        <div class="control-group">…</div>
      </aside>

      <details class="sim-about card disclosure">
        <summary><span data-i18n="tpl.about.title"></span><span id="about-icon"></span></summary>
        <div class="disclosure__body"><p data-i18n="tpl.about.p1"></p></div>
      </details>
    </div>
  </main>

  <footer id="site-footer"></footer>
</body>
</html>
```

Las simulaciones con motor propio agregan `<script defer src="../../shared/sim/random.js">`, `chart.js` y `engine.js` **antes** de su `<id>.js`.

### 7.2 Patrón del bucle de animación

```js
function frame(now) {
  const dt = Math.min(0.1, Math.max(0, (now - last) / 1000));  // delta acotado
  last = now;
  if (needsLayout) { layout(); needsLayout = false; }
  if (playing && sim) sim.advance(dt * SPEEDS[speedIndex]);
  draw();
  if (now - lastUi > UI_INTERVAL) { renderUi(); lastUi = now; }      // lecturas numéricas
  if (now - lastLive > LIVE_INTERVAL) { renderLive(); lastLive = now; } // aria-live
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

Tres decisiones deliberadas:

1. **`dt` acotado a 0,1 s.** Si la pestaña queda en segundo plano, el primer frame al volver traería un `dt` enorme que haría saltar la simulación. Acotarlo congela el tiempo simulado en vez de romperlo.
2. **El UI numérico se refresca más lento que el canvas.** Repintar texto a 60 fps es caro y además ilegible.
3. **La región `aria-live` se refresca aún más lento**, porque un lector de pantalla no puede seguir actualizaciones rápidas.

### 7.3 Lectura de colores desde los tokens

```js
const css = getComputedStyle(document.documentElement);
const color = (name) => css.getPropertyValue(name).trim();
ctx.fillStyle = color('--sim-ball');
```

Nunca se escribe un literal de color en el JavaScript de dibujo.

---

## 8. Simulación 1 — Galton Board

**Carpeta:** `simulations/galton-board/` · **Archivos:** `index.html` (109 líneas), `galton.js` (743), `galton.css` (15), `i18n.js` (117), `thumbnail.svg`
**Particularidad:** es la única sin `engine.js` separado — el modelo es una secuencia de Bernoulli, demasiado simple para justificar la separación.

### 8.1 Propósito didáctico

Enseña **variación por causas comunes** y el **Teorema del Límite Central**. El destino de cada bola es la suma de *n* eventos binarios independientes (un rebote izquierda/derecha por fila) ⇒ **Binomial(n, p)**. Al acumular bolas, los compartimentos dibujan una campana.

Conclusión que debe sacar el usuario (texto `galton.about.p2`): *la salida de un proceso varía por la misma razón — muchas causas pequeñas e independientes se suman y generan una distribución en forma de campana, que en Six Sigma llamamos variación por causas comunes.*

Refuerzos en la UI: media y desvío **observados** junto a los **teóricos** (marcados con `.stat--theory`), y la curva normal teórica superpuesta. Mover `p` fuera de 0,5 muestra la asimetría; subir las filas muestra cómo mejora la aproximación normal.

### 8.2 Modelo matemático

- `n = rows` (4…20) filas de clavos ⇒ **n+1 compartimentos** (`k = 0…n`).
- En cada fila la bola va a la derecha con probabilidad `p` (`Math.random() < p` ⇒ `k++`). Compartimento final = número de rebotes a la derecha.
- **RNG: `Math.random()` nativo, sin semilla.** Esta simulación **no es reproducible** — y no necesita serlo, porque no hay decisiones del usuario que comparar.

**Teoría (`computeTheory()`)** — recurrencia de la PMF binomial, sin factoriales (evita desbordamiento con n grande):

```
q = 1 − p
pmf = q^n ; max = pmf
for k = 1..n:  pmf = pmf·(n−k+1)·p / (k·q) ;  max = Math.max(max, pmf)
theory = { mean: n·p, sd: Math.sqrt(n·p·q), pmfMax: max }
```

- **Media teórica = n·p** · **Desvío teórico = √(n·p·(1−p))**
- `pmfMax` solo sirve para escalar el eje vertical.
- Curva normal superpuesta: `normalPdf(x) = exp(−0.5·z²) / (sd·√(2π))`, con `z = (x − mean)/sd`.

**Estadísticos observados** (acumuladores de una sola pasada: `sum`, `sumSq`):

```
mean = n ? sum / n : NaN
sd   = n > 1 ? sqrt( max(0, (sumSq − n·mean²) / (n − 1)) ) : NaN     // desvío MUESTRAL (n−1)
```

El `max(0, …)` protege de negativos por redondeo. Con 0 bolas la media muestra `–`; con 1 bola, el desvío muestra `–`.

**Histograma:** `bins = Uint32Array(rows+1)` (conteos) y `disp = Float32Array(rows+1)` (alturas animadas en px).
**Escala vertical compartida** (evita saltos): `scaleMax() = max(total · pmfMax · 1.1, maxBin · 1.04, 1)`.

### 8.3 Controles

| id | Tipo | min | max | step | Default | Salida |
|---|---|---|---|---|---|---|
| `in-rows` | range | 4 | 20 | 1 | **12** | `out-rows` |
| `in-total` | range | 100 | 5000 | 100 | **1000** | `out-total` |
| `in-speed` | range | 0 | 7 | 1 | **3** (= 1×) | `out-speed`, `"{n}×"` |
| `in-p` | range | 0,1 | 0,9 | 0,05 | **0,5** | `out-p`, 2 decimales |
| `in-curve` | checkbox `role="switch"` | — | — | — | **activado** | curva teórica |
| `in-trails` | checkbox `role="switch"` | — | — | — | **desactivado** | estelas |

Botones: `btn-play` (Play/Pause), `btn-drop1` (Drop 1), `btn-drop100` (Drop 100), `btn-reset`.

Lecturas (`<dl class="stats">`): `stat-dropped` (`aterrizadas / total`), `stat-mean`, `stat-tmean` (teórica), `stat-sd`, `stat-tsd` (teórica). Las teóricas llevan `.stat--theory`.

**Atajos:** `Espacio` play/pausa · `R` reset · `D` soltar una (única tecla con auto-repetición).

### 8.4 Animación y rendimiento

**Tres capas de canvas superpuestas** (`position: absolute; inset: 0`):

| Capa | Resolución | Contenido | Cuándo se redibuja |
|---|---|---|---|
| `board__static` | W·dpr × H·dpr | Embudo, clavos, paredes, piso, etiquetas | Una vez por layout |
| `board__trails` | **1× (sin dpr)** | Estelas | Cada frame, solo si están activas |
| `#galton-canvas` | W·dpr × H·dpr | Barras, curva teórica, bolas | Solo si `needsDraw` |

Las estelas van a 1× deliberadamente: el barrido de desvanecimiento cubre toda la pantalla cada frame y a 2× costaría cuatro veces más sin ganancia visible.

**Constantes de temporización (a 1×):**

| Constante | Valor |
|---|---|
| `SPEEDS` | `[0.25, 0.5, 0.75, 1, 1.5, 2, 3, 4]` |
| `PLAY_RATE` | 30 bolas/s (reproducción continua) |
| `BURST_RATE` | 200 bolas/s (Drop 100) |
| `HOP_TIME` | 0,16 s por fila |
| `FUNNEL_TIME` | 0,14 s |
| `FALL_TIME` | 0,26 s |
| `FADE_MS` | 180 ms |
| `STATS_INTERVAL` | 100 ms |
| `LIVE_INTERVAL` | 1000 ms |
| `MAX_DPR` | 2 |
| `MAX_TRAIL_BALLS` | 800 |

**Física de cada bola:** objeto reutilizado de un `pool`, con `kind` = 0 (caída por el embudo), 1 (salto entre clavos) o 2 (caída final).
- Salto: `x = x0 + (x1−x0)·ease(t)`; `y = y0 + (y1−y0)·t² − lift·sin(π·t)` → arco parabólico con elevación.
- Caída: `y = y0 + (y1−y0)·t²`; la caída final termina **sobre la cima actual de la barra**.
- El tiempo sobrante se arrastra al tramo siguiente, de modo que a 4× una bola puede atravesar varias filas en un frame sin perder tiempo.

**Técnicas de rendimiento (cero asignaciones en el bucle caliente):**
- Sprite de bola pre-renderizado con gradiente radial, dibujado con `drawImage` en **coordenadas de píxel de dispositivo enteras** (sin remuestreo).
- `pool` de objetos bola; globales `posX`/`posY` en vez de devolver objetos.
- Arrays tipados para los conteos; eliminación O(1) por intercambio con el último elemento.
- Todos los clavos en un único `Path2D`.
- Redibujo **bajo demanda** (`needsDraw`): en reposo el bucle corre pero no pinta.
- `dt` acotado a **0,05 s** por frame.

**Techo de carga:** ritmo máximo `BURST_RATE × 4 = 800 bolas/s`; vida de una bola a 4× con 20 filas ≈ 0,9 s ⇒ pico de ~720 bolas simultáneas, justo por debajo de `MAX_TRAIL_BALLS`. Total acumulable: **5000 bolas**.

### 8.5 Máquina de estados

| Acción | Efecto |
|---|---|
| `togglePlay()` | Si está reproduciendo → pausa. Si terminó (`remaining() === 0` y sin bolas en vuelo) → **reinicia y vuelve a empezar**. Si no → reproduce |
| `drop(n)` | Encola `min(n, remaining())` en la ráfaga. **No cambia** el estado play/pausa |
| `reset()` | Devuelve todas las bolas al pool, recrea los arrays, pone a cero los acumuladores, recalcula teoría y layout, limpia estelas. El botón siempre **pausa** |
| `fadeReset()` | Funde el tablero a opacidad 0, reinicia a los **180 ms** y reaparece, **conservando** el estado de reproducción |
| Cambiar `in-rows` o `in-p` | **`fadeReset()`** — cambia el modelo, los datos acumulados ya no corresponden |
| Cambiar `in-total` o `in-speed` | **No reinicia**: solo afecta `remaining()` / la escala / el ritmo, en caliente |
| Conmutar curva o estelas | Solo marca redibujo |
| Cambio de idioma | Reetiqueta todo y **reconstruye la capa estática** (las etiquetas de compartimento usan dígitos del locale). **No** reinicia |

### 8.6 Movimiento reducido

- Las bolas **no se animan**: se resuelven al instante con `land(sampleBin())`.
- Las barras se mueven sin suavizado.
- `fadeReset()` omite el fundido.
- Si el usuario **activa** la preferencia en caliente, se vuelca la cola de ráfaga y se resuelven de inmediato las bolas en vuelo.

### 8.7 Accesibilidad específica

- `#live-region` actualizada cada 1000 ms, con *early return* si el conteo no cambió (evita anuncios redundantes).
- `aria-label` dinámico del canvas con filas, compartimentos y bolas aterrizadas.
- `aria-valuetext` en cada slider con el valor formateado y legible ("1,5×", "0,50").
- Las capas estática y de estelas llevan `aria-hidden="true"`.
- Tras un clic **de puntero** (`e.detail > 0`) los botones hacen `blur()`, para que la barra espaciadora siga significando play/pausa y no re-dispare el último botón. La activación por teclado conserva el foco.

### 8.8 Claves i18n (prefijo `galton.`)

32 claves por idioma. Con interpolación:

| Clave | Texto EN | Placeholders |
|---|---|---|
| `galton.canvasLabel` | Galton Board: balls fall through {rows} rows of pegs into {bins} bins. {dropped} balls in the bins. | `{rows} {bins} {dropped}` |
| `galton.shortcuts` | Keyboard: {space} play or pause · {r} reset · {d} drop one | `{space} {r} {d}` (inyectados como `<kbd>`) |
| `galton.live` | {dropped} of {total} balls dropped. Observed mean {mean}, standard deviation {sd}. Theoretical mean {tmean}, standard deviation {tsd}. | 6 placeholders |

Resto (sin placeholders): `pageTitle`, `eyebrow` ("Variation"), `title`, `lead`, `group.run/params/display`, `controlsLabel`, `play`, `pause`, `drop1`, `drop100`, `reset`, `rows` ("Peg rows"), `total` ("Total balls"), `speed`, `p` ("Probability of bouncing right (p)"), `curve`, `trails`, `key.space`, `statsLabel`, `stat.dropped/mean/sd/tmean/tsd`, `about.title/p1/p2`.

---

## 9. Simulación 2 — Little's Law: 3-process line

**Carpeta:** `simulations/littles-law/` · **Archivos:** `index.html` (202), `engine.js` (348), `engine.test.js` (137), `littles-law.js` (821), `littles-law.css` (248), `i18n.js` (300)

### 9.1 Propósito didáctico

Enseña la **Ley de Little**: `WIP = Throughput × Lead time`, verificada **por medición**, nunca por fórmula. Los tres términos se miden por separado y el usuario ve que **coinciden**.

Mensajes clave:
1. En cualquier sistema **estable**, el WIP promedio es igual al throughput promedio por el lead time promedio.
2. Se cumple sin importar las distribuciones, la cantidad de pasos o la regla de secuenciación. Es una **identidad de flujo, no un modelo estadístico**: la variabilidad cambia los números pero nunca rompe la ecuación.
3. Lección operativa: **el cuello de botella fija el throughput**, así que la palanca directa para bajar el lead time es **bajar el WIP**.

El escenario 3 enseña el contraejemplo: si λ ≥ capacidad del cuello de botella, no existe promedio de largo plazo y la ley **no se puede verificar**.

### 9.2 Topología del motor

```
Llegadas → [backlog] → Q1 → P1 → Q2 → P2 → Q3 → P3 → Terminadas
```

| Constante | Valor | Significado |
|---|---|---|
| `STATIONS` | 3 | estaciones en serie, **capacidad 1 unidad cada una** (servidor único, sin preempción) |
| `HISTORY_MINUTES` | 200 | ventana de historial retenida |
| `HISTORY_STEP` | 0,25 | minutos entre muestras de WIP |
| `ROLLING_WINDOW` | 30 | minutos para el throughput rodante |
| `RECENT_DONE` | 24 | unidades terminadas que se conservan para mostrar |

Colas FIFO de **capacidad infinita**. Unidad: `{ id, arrive, enter, exit, leadTime }`. Unidad de tiempo: **minutos simulados**.

`DEFAULTS` del motor: `lambda 1.5`, `mu [2,2,2]`, `cv 0.2`, `policy 'push'`, `wipCap 10`, `warmup 20`, `seed 20240601`.

**La UI fija `CV = 0.3`, `warmup = 0` y `policy = 'push'`.** El usuario no puede cambiarlos (decisión de simplificación tomada deliberadamente; la política `pull`/CONWIP sigue existiendo y testeada en el motor, pero sin exponerse).

### 9.3 Eventos y su orden

Solo dos tipos: **llegada** (`kind = −1`) y **fin de servicio de la estación i** (`kind = 0..2`).

```js
let next = this.nextArrival;
let kind = -1;
for (let i = 0; i < STATIONS; i++) {
  if (this.stations[i].end < next) { next = this.stations[i].end; kind = i; }
}
if (next > target) break;
```

**Reglas de desempate** (consecuencia del `<` estricto y del orden de evaluación): en empate exacto gana **la llegada**; entre estaciones empatadas gana el **índice menor**. Antes de ejecutar un evento se registra historial y se integran las áreas hasta `next`.

> `nextArrival` se inicializa en **0**: la primera unidad llega en `t = 0`, antes de consumir ningún número aleatorio para llegadas. El primer `sample()` consumido es el del servicio de la estación 1. **Este orden de consumo del stream es un contrato: alterarlo rompe la reproducibilidad.**

### 9.4 Distribuciones

Llegadas y servicios usan `sample(mean, cv)` de `shared/sim/random.js` (gamma de Marsaglia–Tsang):
- Tiempo entre llegadas: media `1/lambda`, CV = `cv`.
- Servicio en la estación *i*: media `1/mu[i]`, CV = `cv`.

### 9.5 Integración temporal

```js
const dt = to - from;
for (cada estación ocupada) busySince[i] += dt;          // desde measureStart, incluye warm-up
const a = Math.max(from, this.warmupEnd);
if (to > a) {
  const w = to - a;
  wipArea += wip * w;
  backlogArea += backlog.length * w;
  // por estación: busyArea[i] += w si ocupada;  queueArea[i] += queues[i].length * w
}
```

Las áreas se acumulan **solo después de `warmupEnd`**; `Math.max(from, warmupEnd)` maneja correctamente el intervalo que cruza el fin del calentamiento, acumulando solo la fracción posterior. `busySince` acumula siempre, lo que permite detectar el cuello de botella antes de que la ventana sea válida.

### 9.6 Estabilidad

```js
get stable() {
  if (policy === 'pull') return true;        // lazo cerrado: WIP acotado por el cap
  return lambda < Math.min(...mu);           // umbral ESTRICTO
}
```

`λ = min(μ)` se declara **inestable** (verificado por el test 6).

### 9.7 Salida de `metrics()`

Cálculos previos: `span = t − warmupEnd`, `warm = span > 0`, `n = completedInWindow`, `nLT = leadTimes.length`, `sumLT = Σ leadTimes`.

| Campo | Fórmula |
|---|---|
| `wipAvg` | `wipArea / span` |
| `throughput` | `completedInWindow / span` |
| `throughputRolling` | `exitTimes.length / min(30, t)` |
| `leadTime` | `sumLT / nLT` |
| `leadTimeP90` | `percentile(leadTimes, 0.9)` |
| `littleProduct` | `throughput × leadTime` |
| `littleDiff` | `abs(wipAvg − product) / wipAvg` |
| `elapsed`, `wipArea`, `leadTimeSum`, `completedInWindow`, `leadTimeCount` | **Insumos crudos**, para que la UI muestre la aritmética viva |
| `stations[i].util` | `busyArea[i] / span` |
| `stations[i].avgQueue` | `queueArea[i] / span` |
| `bottleneck` | índice 0..2 (ver abajo) |

También: `t`, `warm`, `warmupRemaining`, `stable`, `wip`, `backlog`, `avgBacklog`, `completedTotal`, `stations[i].{queue, busy, utilSinceStart}`.

`metrics({ percentiles: false })` existe solo para evitar el `sort()` del P90 en el camino caliente.

**Detección del cuello de botella en el motor:** mayor utilización medida; mientras no hay datos (`sinceStart <= 1`) se usa `1/mu[i]` (menor capacidad = mayor tiempo de ciclo). Margen de desempate `1e-9`.

### 9.8 La regla del "régimen actual" (detalle crítico)

**Problema:** al cambiar un parámetro pueden quedar unidades dentro de la línea provenientes del régimen anterior — por ejemplo una cola enorme creada por una sobrecarga. Al salir traen un lead time gigantesco que **no corresponde al régimen actual** e inflaría el promedio, rompiendo visualmente la igualdad de Little.

**Implementación** (dos líneas en `complete()`):

```js
if (this.t >= this.warmupEnd) this.completedInWindow++;             // throughput: TODA salida de la ventana
if (unit.enter >= this.warmupEnd) this.leadTimes.push(unit.leadTime); // lead time: SOLO régimen actual
```

El criterio es el **instante de entrada a la línea**, no el de salida. Las unidades arrastradas **sí** cuentan en WIP y throughput mientras drenan. Esta asimetría es deliberada y está explicada al usuario en el texto de la regla de lead time.

### 9.9 Ventana de medición y `setParams()`

`resetMeasurement()` pone a cero: `wipArea`, `backlogArea`, `busyArea`, `queueArea`, `busySince`, `leadTimes`, `completedInWindow`; y fija `measureStart = t`, `warmupEnd = t + warmup`, `lastT = t`.

`setParams(partial)`:
1. Aplica los parámetros (copiando `mu` si viene).
2. Si cambió `lambda`, **re-sortea la llegada pendiente** para que el cambio aplique de inmediato.
3. Si pasó de `pull` a `push`, vuelca el backlog a la línea.
4. `resetMeasurement()`.

| Conserva | Reinicia |
|---|---|
| Unidades en colas y estaciones, `wip`, el servicio en curso (no se re-sortea), estado del RNG, `completedTotal`, `recent`, `history`, el reloj `t` | Toda la ventana de medición: áreas, conteos, lista de lead times; y arranca de nuevo el warm-up |

A nivel UI, `applyParams()` además **inserta un punto `NaN`** en las series de promedio acumulado, lo que **corta el trazo** de la línea punteada en el gráfico — señal visual de que empezó una medición nueva.

### 9.10 Controles y escenarios

| id | Tipo | min | max | step | Default |
|---|---|---|---|---|---|
| `in-lambda` | range | 0,2 | 5 | 0,1 | 1,5 |
| `in-mu0` / `in-mu1` / `in-mu2` | range | 0,5 | 6 | 0,1 | 2 |
| `in-speed` | range | 0 | 5 | 1 | **2** (= 2×) |

`SPEEDS = [0.5, 1, 2, 3, 5, 10]`; **1× = 1 minuto simulado por segundo real**.

Botones: `btn-play`, `btn-reset` (mismo seed y params), `btn-seed` (nueva semilla aleatoria 1…999999 y reinicio).

**Los tres escenarios (valores exactos):**

```js
const SCENARIOS = [
  { key: 's1', params: { lambda: 1.5, mu: [2, 2, 2] } },
  { key: 's2', params: { lambda: 1.5, mu: [2.5, 1.7, 2.5] } },
  { key: 's3', params: { lambda: 2.0, mu: [2.5, 1.7, 2.5] } },
];
```

| # | Tecla | Nombre | Qué ocurre |
|---|---|---|---|
| 1 | `1` | Balanced line | ρ = 75 % en los tres; colas casi vacías; la ley se verifica en pocos minutos |
| 2 | `2` | Bottleneck at process 2 | P2 ≈ 88 % (1,5/1,7), marcado como cuello de botella; la cola crece **solo delante de él**; el throughput sigue siendo λ |
| 3 | `3` | Overloaded | λ (2,0) > cuello (1,7) ⇒ inestable; el throughput satura en 1,7 mientras WIP y lead time crecen sin límite |

Mover cualquier slider llama a `setScenario(-1)` y el texto pasa a "ajustes personalizados".

**Atajos:** `Espacio` play/pausa · `R` reset · `1`–`3` escenarios.

### 9.11 Chip de estado

```js
function chipState(m) {
  if (!m.stable) return 'unstable';
  if (m.leadTimeCount < 10 || !Number.isFinite(m.littleDiff)) return 'waiting';
  return m.littleDiff <= 0.05 ? 'ok' : 'converging';
}
```

| Estado | Icono | Color | Texto |
|---|---|---|---|
| `unstable` | `alert` | `--color-danger` + fondo `--color-danger-tint` | "Inestable: las llegadas alcanzan o superan la capacidad del cuello de botella…" |
| `waiting` | `loader` | base (navy) | "Midiendo: esperando las primeras unidades terminadas" |
| `ok` | `check` | `--color-success` | "Verificado: WIP y Throughput × Lead time difieren en {diff}" |
| `converging` | `info` | base | "Convergiendo: {diff} de diferencia (objetivo 5 % o menos)" |

**Umbral de verificación: 5 %. Umbral de datos suficientes: 10 unidades terminadas del régimen actual.**

### 9.12 Cuello de botella en la UI (con histéresis)

Distinto del cálculo del motor, para evitar que la etiqueta parpadee entre estaciones:

```js
const BOTTLENECK_MARGIN = 0.03;   // margen para DECLARAR un cuello de botella
const BOTTLENECK_KEEP   = 0.01;   // tolerancia para MANTENER el ya declarado
```

- Si el sistema es inestable → directamente la estación de menor capacidad.
- En los primeros 5 minutos → ρ teórico (`λ/μᵢ`), porque aún no hay medición confiable.
- `-1` significa **ninguno marcado** (línea balanceada): no se señala un cuello de botella que no existe.

El nodo marcado recibe sombra interior naranja, muestra el badge "Bottleneck" y pinta su barra de utilización en `--color-bottleneck`.

### 9.13 Bloque "Cómo se calculan los números"

Es el corazón didáctico añadido en la simplificación. Tres columnas (una en móvil), cada una con la **regla** fija y la **aritmética viva**:

| Término | Regla | Cadena viva |
|---|---|---|
| WIP | Promedio de unidades dentro de la línea a lo largo del tiempo: área bajo la curva de WIP ÷ tiempo transcurrido | `Ahora: {area} unidad·min ÷ {t} min = {v}` |
| Throughput | Unidades que salieron de la línea ÷ tiempo transcurrido | `Ahora: {n} unidades ÷ {t} min = {v} u/min` |
| Lead time | Para cada unidad terminada, salida − entrada; luego el promedio **de las que entraron después del último reinicio o cambio de parámetro** (las que ya estaban adentro cuentan en el throughput, no acá) | `Ahora: {sum} min ÷ {n} unidades = {v} min` |

La cadena de WIP y la de throughput se muestran si `elapsed > 0`; la de lead time, solo si `leadTimeCount > 0`.

### 9.14 Renderizado

**Diagrama de flujo:** grid DOM de 5 columnas (origen + 3 procesos + sink) con un `<canvas>` superpuesto (`pointer-events: none`). Las cajas DOM **definen la geometría**, medida con `getBoundingClientRect()`. Por debajo de **719 px** el layout pasa a vertical y el cálculo cambia de rama.

Geometría: `UNIT = 10` px por unidad, `GAP = 3`, sink de **6 × 4 = 24** huecos (= `RECENT_DONE`). Si una cola excede la capacidad dibujable se imprime una etiqueta `+N`.

Tween de unidades: `TWEEN_MS = 220`, `TWEEN_MIN_MS = 120`, acortado con la velocidad. Con movimiento reducido, salto directo.

Anillo de progreso por estación: arco de `−π/2` a `−π/2 + frac·2π` con `frac = (t − start)/(end − start)`.

**Gráficos:** dos canvas de 150 px.

| Aspecto | Detalle |
|---|---|
| Eje X | Ventana deslizante de **200 minutos simulados**; etiquetas cada 50 min |
| Eje Y | `niceMax(max × 1.1)` con pasos 1 / 2 / 2,5 / 5 / 10 × potencia de 10 |
| Grilla | 5 líneas horizontales, etiqueta solo en las pares, alineadas a medio píxel |
| Serie medida | Cian. WIP → polilínea (muestra cada 0,25 min). Lead time → **puntos** de radio 1,8 (uno por unidad terminada) |
| Promedio acumulado | Navy, **discontinuo `[5, 4]`**; se **interrumpe** en cada `NaN` (el que inserta un cambio de parámetro) |

`sampleAverages()` empuja un punto cada **0,5 minutos simulados**.

### 9.15 Tests del motor (9 bloques, 28 aserciones)

| # | Qué protege |
|---|---|
| 1 | **Caso determinista calculable a mano** (`cv = 0`): LT = 1,5 exacto, TH = 1,0, WIP = 1,5, utilizaciones 0,5, P90 = 1,5 |
| 2 | Un proceso más lento suma al lead time; el cuello de botella detectado es P2 |
| 3 | **Reproducibilidad**: misma semilla ⇒ run idéntico; semilla distinta ⇒ run distinto |
| 4 (×4) | **La ley se cumple en ≤ 5 %** con CV 0,3 / 0,6 / 1,0 y distintas semillas, tras 3000 min |
| 5 | **Validez del modelo contra teoría de colas**: con CV = 1 la línea es un tándem M/M/1; ρ = 0,75 ⇒ L = 3 por estación ⇒ WIP = 9, LT = 6 min, TH = 1,5 (tolerancias 10 % / 10 % / 3 %) |
| 6 | **Detección de inestabilidad**: el flag se enciende con λ ≥ cuello (umbral estricto) y el WIP crece sin límite |
| 7 | **CONWIP**: el WIP nunca excede el cap; `wip` cuenta exactamente las unidades de la línea (el backlog queda fuera); el throughput satura en la tasa del cuello; la ley se cumple **para la línea** |
| 8 | **Regla del régimen actual**: el lead time excluye las unidades arrastradas mientras el throughput las cuenta |
| 9 | **`setParams` reinicia la medición y conserva las unidades**: nuevo warm-up y el WIP físico intacto |

### 9.16 Claves i18n (prefijo `ll.`)

84 claves por idioma. Las que llevan interpolación: `eq.wipNow`, `eq.thRolling`, `eq.ltP90`, `chip.ok`, `chip.converging`, `chip.product`, `node.process`, `node.completed`, `node.tooltip`, `capacity`, `rateValue`, `cycle`, `interval`, `seed`, `clock`, `charts.wipLabel`, `charts.ltLabel`, `shortcuts`, `live`, `live.ok`, `live.converging`, `flowCanvasLabel`, `calc.wipLive`, `calc.thLive`, `calc.ltLive`.

> Nota de mantenimiento: `ll.group.run` y `ll.group.model` existen en el diccionario pero **ya no se usan** en el HTML ni el JS (quedaron de la versión anterior con más controles). Son candidatas a limpieza.

---

## 10. Simulación 3 — Push vs. Pull: planning game

**Carpeta:** `simulations/push-pull/` · **Archivos:** `index.html`, `engine.js`, `engine.test.js`, `push-pull.js`, `game.js`, `push-pull.css`, `i18n.js`
**Es la simulación más compleja del hub:** dos líneas en paralelo, dos modos de uso (Explorar y Juego) y un panel de facilitador.

### 10.1 Propósito didáctico

Enfrenta **dos líneas de producción idénticas** — mismos procesos, misma capacidad, **el mismo flujo de pedidos y el mismo pronóstico** — que difieren solo en la **regla de liberación de trabajo**:

- **PUSH (MRP):** libera según el pronóstico cada período de planificación.
- **PULL (kanban):** libera una unidad por cada tarjeta que vuelve cuando un cliente consume.

Conclusiones esperadas:
1. Push decide **antes** de que el cliente pida; todo error de pronóstico se convierte en **exceso del ítem equivocado y faltante del correcto**, y el plan solo reacciona en el ciclo siguiente.
2. Pull libera solo ante consumo real: el WIP queda **topado por la cantidad de tarjetas** y la mezcla **sigue automáticamente** a la demanda real.
3. **Dimensionar el kanban es el punto crítico**: `tarjetas ≈ demanda × lead time de reposición + seguridad`. Pocas tarjetas → línea hambreada (preset 4); demasiadas → pull se comporta como push (preset 5).
4. El KPI **"inventario de mezcla errónea"** aísla el modo de falla de push: stock terminado de un SKU **mientras** clientes del otro esperan.

### 10.2 Topología de cada línea

```
Materia prima → A → B → Producto terminado (FG) → Cliente
```

- **2 SKUs** (SKU 1 cuadrado, SKU 2 círculo).
- **Capacidades:** `capA = 1.4` u/min (≈0,714 min/u), `capB = 1.3` u/min (≈0,769 min/u). **B es el cuello de botella.**
- Tiempos de proceso **determinísticos** (`1/capacidad`): toda la variabilidad viene de la demanda y del pronóstico, no del proceso. Decisión deliberada, para que la comparación aísle la política de liberación.
- Inventario inicial de FG = `cards` por SKU: ambas líneas arrancan con 4 unidades de SKU 1 y 3 de SKU 2.
- Unidad: `{ id, sku, release, fgTime }` (`makeUnit(sku, t, release)` → `{ release, fgTime: t }`; al liberar se pasa `fgTime = NaN` y se completa al salir de B).
- **Backorders, no ventas perdidas**: la demanda no atendida queda pendiente.

### 10.3 Proceso de demanda (compartido por ambas líneas)

- `rate(0) = demandRate · rateFactor · mixNow`; `rate(1) = demandRate · rateFactor · (1 − mixNow)`.
- Tiempos entre llegadas gamma con media `1/rate(s)` y CV `cv`.
- **Tres generadores `mulberry32` independientes:** demanda SKU 0 (`seed ^ 0x9E3779B9`), demanda SKU 1 (`seed ^ 0x85EBCA6B`), ruido de pronóstico (`seed ^ 0xC2B2AE35`).

> **Esta separación de streams es la decisión de modelado más importante de la simulación:** la demanda **no depende** de la configuración de las líneas, de modo que la comparación push vs. pull es justa y la misma semilla da la misma secuencia de pedidos sea cual sea la configuración. Está protegida por el test 2b.

**Pronóstico** (se fija en cada tick de planificación):

```
forecast[s] = max(0, rate(s) · (1 + bias) · (1 + noise · N(0,1)))
```

Los eventos cambian la demanda real **al instante**, pero el pronóstico solo los ve en el **siguiente tick**. Ese desfase es justamente lo que el ejercicio quiere mostrar.

### 10.4 Política PUSH (MRP)

**Tick de planificación:** en `t = 0` y luego cada `period` minutos. En cada tick: cierra el período en ambas líneas → recalcula el pronóstico → abre el período nuevo → si `autoRelease`, ejecuta la liberación MRP.

**Regla de posición de inventario**, por SKU:

```
need   = forecast[s]·period + safety·forecast[s] − (fg[s].length + wip(s)) + backlog[s]
qty[s] = max(0, round(need))
```

`safety` se expresa en **minutos de demanda pronosticada** (no en unidades).

**Secuenciación por fecha de necesidad** (`manualRelease`), usada tanto por el MRP como por la liberación manual del juego:

```
have = fg[s].length + wip(s) − backlog[s]
r    = max(forecast[s], 1e-6)
// la k-ésima unidad de un SKU se necesita en (have + k) / r
order.sort((a, b) => a[0] - b[0] || a[1] - b[1]);   // por fecha; empate → SKU menor
```

Así **un SKU nunca espera detrás del lote completo del otro**. Si `changeover` está activo, se reordena en **campañas** (todas las unidades del SKU más urgente primero), para no pagar un setup por unidad.

### 10.5 Política PULL (kanban)

**Invariante del sistema** (protegida por el test 1c):

```
WIP(s) + FG(s) + tarjetas_en_tránsito(s) = cards[s] + retire[s]
```

- Al consumirse una unidad de FG (venta inmediata **o** unidad que sale de B directo a un backorder), la tarjeta viaja de vuelta con `cardDelay = 1` min.
- Al llegar, libera **exactamente una** unidad. No hay pronóstico, ni período, ni safety stock.
- `setCards()` en caliente:
  - **Subir:** cancela retiros pendientes y libera de inmediato las unidades que las tarjetas nuevas autorizan.
  - **Bajar:** quita primero tarjetas libres en tránsito (de la más reciente a la más vieja); las que falten se anotan en `retire[s]` y se descartan cuando vuelvan. **Nunca se destruyen unidades físicas**: el excedente drena.

**Regla práctica mostrada en la UI:** `rep = cardDelay + 1/capA + 1/capB + 1` (≈ 3,48 min con los valores por defecto), y `tarjetas ≈ ceil(demanda_SKU × rep + 1)`.

### 10.6 KPIs y modelo de costos

| KPI | Fórmula | Mejor |
|---|---|---|
| Fill rate | `served / demand` (atendidos **de inmediato** desde FG) | alto |
| Inventario promedio | `(invArea[0] + invArea[1]) / t`, con `invArea[s] += (wip(s) + fg[s].length)·dt` | bajo |
| Lead time | `ltSum / ltN`, midiendo `fin_de_B − release` | bajo |
| Backorders | `ahora / acumulado` | bajo |
| Inv. de mezcla errónea | `wrongArea / t`, con `wrong = (backlog[1]>0 ? fg[0].length : 0) + (backlog[0]>0 ? fg[1].length : 0)` | bajo |
| **Índice de costo** | `holdCost · avgInventory + backorderCost · (backordersTotal / t) · 10` | bajo |

> ⚠️ **`holdCost = 1` y `backorderCost = 5` son ilustrativos y NO son cifras de Amcor.** El código lo marca en comentarios y la UI lo dice en dos lugares: *"Illustrative, not Amcor figures"* y *"Round numbers for discussion only; they are not Amcor figures."* Cualquier reimplementación debe conservar esa advertencia.

**Marcado del ganador:** tolerancia de 0,005 absoluto para el fill rate y **2 %** de `max(|a|,|b|)` para el resto. Dentro de la tolerancia **no se marca ninguna línea** — evita declarar un ganador por ruido.

### 10.7 Modo Explorar

**Parámetros base:** `demandRate 1.0 · cv 0.5 · mix 0.6 · bias 0 · noise 0.1 · capA 1.4 · capB 1.3 · changeover false · period 10 · safety 5 · cards [4,3] · holdCost 1 · backorderCost 5 · autoRelease true`.

**Los 5 presets (overrides exactos sobre la base):**

| # | Tecla | Nombre | Overrides | Qué muestra |
|---|---|---|---|---|
| 1 | `1` | Perfect forecast | `noise: 0, safety: 6` | Ambas sirven bien, pero pull con **mucho menos inventario** |
| 2 | `2` | Forecast bias +30% | `bias: 0.3, cards: [5,4]` | Push acumula inventario sin mejorar el servicio; pull ignora el pronóstico |
| 3 | `3` | Mix shift mid-run | `cards: [5,4], schedule: [{ t: 30.5, type: 'mix' }]` | Tras el minuto 30 la mezcla pasa de 60/40 a 30/70: push apila SKU 1 y se queda sin SKU 2 |
| 4 | `4` | Too few kanban cards | `cards: [1,1]` | Pull se hambrea: inventario mínimo **y servicio mínimo** |
| 5 | `5` | Too many kanban cards | `noise: 0, safety: 6, cards: [15,12]` | Pull degenera en push: inventario alto sin mejora de servicio |

Los presets 4 y 5 son el contrapeso honesto del ejercicio: **pull mal dimensionado no es mejor que push.**

**Controles:**

| Grupo | id (`in-…`) | min | max | step | Default |
|---|---|---|---|---|---|
| Demanda | `demandRate` | 0,5 | 1,5 | 0,05 | 1,0 |
| | `cv` | 0 | 1 | 0,05 | 0,5 |
| | `mix` | 0,1 | 0,9 | 0,05 | 0,6 |
| Pronóstico | `bias` | −0,5 | 0,5 | 0,05 | 0 |
| | `noise` | 0 | 0,5 | 0,05 | 0,1 |
| Capacidad | `capA` | 0,8 | 3 | 0,05 | 1,4 |
| | `capB` | 0,8 | 3 | 0,05 | 1,3 |
| | `changeover` | switch | — | — | false (0,3 min) |
| Push | `period` | 5 | 30 | 5 | 10 |
| | `safety` | 0 | 15 | 1 | 5 |
| Pull | `cards0` | 1 | 20 | 1 | 4 |
| | `cards1` | 1 | 20 | 1 | 3 |
| Costos | `holdCost` | 0 | 5 | 0,5 | 1 |
| | `backorderCost` | 0 | 20 | 1 | 5 |

Los cambios son **en caliente** y las métricas **siguen acumulando** desde el último reinicio.

**Botones de evento** (aplican a **ambas** líneas, para que la comparación siga siendo justa):

| Botón | Efecto |
|---|---|
| Demand spike +50% | `rateFactor = 1.5` durante `EVENT_MINUTES = 10` min |
| Demand drop −50% | `rateFactor = 0.5` durante 10 min |
| Mix shift | `mixNow = mix / 2` (60/40 → 30/70), **persistente** |
| Back to baseline | Restaura `rateFactor = 1` y `mixNow = mix` |

Cada evento queda registrado y se dibuja como marca vertical en las sparklines.

**Velocidad:** `SPEEDS = [0.5, 1, 2, 3, 5, 10]`, default **2×**. **Atajos:** `Espacio`, `R`, `1`–`5`. Modo inicial por URL: `?mode=game`.

### 10.8 Modo Juego

**6 rondas de 10 minutos simulados** (60 min). Flujo: **predecir → (decidir → correr) × 6 → debrief**.

Parámetros del juego: iguales a Explorar salvo **`noise: 0.2`** y **`autoRelease: false`** (push libera **solo** lo que decide el jugador).

**1. Predecir.** Dos preguntas obligatorias: ¿qué línea tendrá mejor fill rate? ¿cuál tendrá menos inventario? (push / pull / igual). La predicción queda guardada y se confronta en el debrief. *Comprometerse antes es lo que hace memorable el ejercicio.*

**2. Decidir (cada ronda).** Tabla con, por SKU: pronóstico a 10 min, FG, WIP, backorders, y un campo de liberación **precargado con `round(forecast × 10)`** (el jugador puede aceptarlo o corregirlo). **Las tarjetas kanban solo se pueden cambiar en las rondas 1 y 4** (`CARD_ROUNDS = [1, 4]`); el resto del tiempo pull corre solo — justamente el punto que se quiere transmitir.

**3. Eventos ocultos derivados de la semilla:**

```js
rng = mulberry32((seed ^ 0x5BD1E995) >>> 0)
rounds = [2,3,4,5,6]            // la ronda 1 NUNCA tiene evento
// Fisher–Yates con ese rng, se toman los dos primeros
plan[r-1] = EVENT_TYPES[floor(rng() * 3)]   // 'spike' | 'drop' | 'mix'
```

**Exactamente 2 eventos**, en dos rondas distintas entre la 2 y la 6. Se programan en `t = ronda·10 + 0.01`, es decir **0,01 min después del inicio de la ronda**: así el pronóstico que vio el jugador al decidir **nunca** los incluye.

**4. Panel del facilitador:**

| Control | Función |
|---|---|
| Semilla (1–999999) | Compartirla hace que toda la clase viva el mismo escenario. Cambiarla **regenera** los eventos |
| Hide KPIs until the end | Oculta scorecards, sparklines y tabla comparativa durante las rondas (en el debrief siempre se muestran) |
| Evento por ronda (6 selects) | Fuerza manualmente `none / spike / drop / mix` en cada ronda |
| Apply and restart | Reinicia con la misma semilla |

**5. Debrief:**
- **Predicción vs. resultado**, con tolerancias: fill rate `|d| ≤ 0.02` ⇒ "igual"; inventario `|a−b| ≤ 0.1·max(a,b)` ⇒ "igual".
- **Score:** tabla de KPIs; gana el **menor índice de costo**.
- **Gráfico** de inventario promedio por ronda: push en navy con marcadores cuadrados, pull en cian con círculos, y **líneas verticales naranjas en las rondas con evento**.
- **Leyenda que revela los eventos ocultos** — el momento de "ajá" del ejercicio.
- **Tres preguntas de discusión**, incluida: *"¿dónde en tu propia operación podría una señal kanban reemplazar un pronóstico, y cómo dimensionarías las tarjetas?"*
- Botones: "jugar de nuevo con la misma semilla" y "escenario nuevo".

### 10.9 Determinismo y el bug del reloj

`advance(dt)` delega en `advanceTo(target)`, que sale si `!(target > this.t)` y **termina fijando `this.t = target` exactamente**.

```js
let step = dt * SPEEDS[speedIndex];
if (sim.t + step >= runTarget - 1e-9) sim.advanceTo(runTarget);   // aterriza exacto
else sim.advance(step);
if (sim.t >= runTarget) { /* fin de ronda */ }
```

> **El bug que esto corrige (commit `be1c152`).** El código anterior recortaba el paso y daba la ronda por terminada con `sim.t >= runTarget - 1e-9`. Por acumulación de error en coma flotante, una ronda podía cerrarse en t = 29,999999… **dentro de la tolerancia pero sin ajustar el reloj**. A partir de ahí, las decisiones de la ronda siguiente, los cierres de período y los eventos programados quedaban desfasados: *"misma semilla + mismas decisiones"* daba resultados distintos, y a veces el juego terminaba en el minuto 59. El test 11 es la regresión: 1799 pasos de 1/60 seguidos de `advanceTo(30)` deben dar `t === 30` y métricas idénticas a un único `advance(30)`.

### 10.10 Tests del motor (11 bloques, 25 aserciones)

| # | Qué protege |
|---|---|
| 1a | **El kanban topa el inventario**: `wip(k) + fg(k) ≤ cards[k]` siempre |
| 1b | Bajar tarjetas no rompe el tope |
| 1c | **Conservación de tarjetas**: `held + transit === cards + retire`; el excedente drena sin destruir unidades |
| 2a-c | **Ambas líneas ven los mismos pedidos**; el stream **no depende** de la configuración; semillas distintas ⇒ escenarios distintos |
| 3 | Con CV 0 y pronóstico perfecto, **ambas** políticas sirven el 100 % (el modelo no penaliza injustamente a ninguna) |
| 4 (×4 semillas) | El **sesgo del pronóstico solo infla el inventario de push** |
| 5a-b | **Determinismo completo** por semilla |
| 6a (×2) | Pocas tarjetas ⇒ la línea se hambrea, fill rate < 50 % (sustenta el preset 4) |
| 6b (×2) | Demasiadas tarjetas ⇒ **pull degenera en push** (sustenta el preset 5) |
| 7 (×3) | Tras un cambio de mezcla, push acumula inventario de mezcla errónea y genera más backorders del SKU 2 |
| 8 | Los eventos programados disparan **en el tiempo exacto** y terminan tras 10 min |
| 9 | El tiempo de cambio de formato sí penaliza el lead time |
| 10 | Con `autoRelease: false` **no hay ninguna liberación MRP automática** |
| 11 | **Regresión del bug del reloj** (ver 10.9) |

### 10.11 Claves i18n (prefijo `pp.`)

**161 claves por idioma** — la simulación con más texto del hub. Grupos: estructura de página, etiquetas de línea y estación, leyenda, KPIs (con sus versiones largas y la nota de costos), tabla comparativa, controles de Explorar, eventos, presets y sus textos de observación, bloque "Learn", atajos, región live, y todo el modo Juego (`pp.game.*`, ~60 claves) incluido el panel del facilitador.

Claves con interpolación destacadas: `pp.clock {t}`, `pp.boBadge {n}`, `pp.pulse {t}{a}{b}`, `pp.seed {seed}`, `pp.note.sizing {rep}{a}{b}`, `pp.live` (6 placeholders), `pp.canvasLabel` (7 placeholders), `pp.game.round {r}{n}`, `pp.game.pullAuto {a}{b}`, `pp.game.chartAriaValues {push}{pull}`, `pp.game.eventAt {r}{e}`.

---

## 11. Simulación 4 — Process capability (Cp/Cpk)

**Carpeta:** `simulations/process-capability/` · **Archivos:** `index.html`, `engine.js`, `engine.test.js`, `capability.js`, `spc.js`, `challenge.js`, `capability.css`, `i18n.js`
**Es la simulación con más contenido estadístico** y la única con **tres modos** y un módulo SPC separado.

### 11.1 Propósito didáctico

| Concepto | Cómo se enseña |
|---|---|
| **Cp** (capacidad potencial) | Compara tolerancia contra dispersión; no depende del centrado. Muestra `N/A` si la especificación es unilateral |
| **Cpk** (capacidad real) | Una "regla de Cpk" dibujada bajo la curva: la distancia de μ a cada límite medida en pasos de 3σ; **el lado más corto se resalta** y se etiqueta "el mínimo es el Cpk" |
| **Cp/Cpk vs. Pp/Ppk** | Cp/Cpk usan σ **dentro** (corto plazo); Pp/Ppk usan σ **total** (largo plazo). Bajo deriva, Ppk cae por debajo de Cpk |
| **Voz del proceso vs. voz del cliente** | Los límites de control salen de los datos propios y se **congelan**; los de especificación los pone el cliente. Un toggle los superpone, **con la advertencia de que las especificaciones aplican a valores individuales, no a promedios de subgrupo** |
| **Causa común vs. especial** | Eventos inyectables sobre un proceso base estable; las reglas de Nelson detectan las especiales |
| **La capacidad solo vale en control** | Si el proceso está fuera de control, los KPI de Cp y Cpk aparecen **tachados** con un tooltip explicativo |
| **Supuesto de normalidad** | Texto explícito advirtiendo sobre datos sesgados o acotados antes de confiar en un PPM |

### 11.2 Modelo del proceso

La característica medida es genérica; solo cambian **etiqueta y escala** según el contexto elegido:

| Contexto | Centro | Semi-rango | Decimales | Etiqueta |
|---|---|---|---|---|
| `generic` | 100 | 10 | 1 | Generic (units) |
| `film` | 50 | 2,5 | 2 | Film thickness (µm) |
| `seal` | 30 | 5 | 1 | Seal strength (N/15 mm) |
| `fill` | 500 | 5 | 1 | Fill weight (g) |

Al cambiar de contexto se **remapea linealmente** el estado: `map(v) = nuevo.centro + ((v − viejo.centro)/viejo.semi) · nuevo.semi`.

> Las etiquetas y rangos son **ilustrativos, no datos reales** — declarado en la UI (`pc.param.contextNote`).

**Muestreo:** no hay modelo temporal de llegadas. Cada paso produce **un subgrupo completo** de tamaño *n*. La frecuencia es de reproducción: `SPEEDS = [0.5, 1, 2, 5, 10, 20]` subgrupos/s (default 2/s), con tope de 40 pasos por frame.

**Distribución:** cada valor individual es `mu + sd · sample.normal()` (Box–Muller sobre `mulberry32`). Determinista por semilla.

**Parámetros por defecto:**

```
mu 100 · sigma 2 · lsl 90 · usl 110 · target 100
n 5 · baseline 25 · seed 20240601
shiftSize 1.5 (σ) · driftPerSubgroup 0.1 (σ) · varFactor 1.5
outlierProb 0.12 · outlierSize 4.5 (σ) · rules {1,2,3,4 activas}
```

**Mecanismos de causa especial** (todos se aplican desde el subgrupo siguiente):

| Evento | Efecto exacto |
|---|---|
| `shift` | `offset += 1.5σ` — **acumulativo** |
| `drift` | `offset += 0.1σ` en cada subgrupo (desgaste de herramienta) |
| `variation` | `sigmaFactor *= 1.5` — **multiplicativo** |
| `outliers` | Con probabilidad 0,12 por subgrupo, un valor se reemplaza por `mu ± 4.5σ` (σ **base**, no la inflada) |
| `baseline` | Reinicia offset, drift, sigmaFactor y outliers |

`setParams()` cambia parámetros **sin reiniciar** — como un proceso real.

### 11.3 Estadística exacta

**Índices:**

```
Cp  = (USL − LSL) / (6σ)              // NaN si la especificación es unilateral
Cpu = (USL − μ) / (3σ)
Cpl = (μ − LSL) / (3σ)
Cpk = min(Cpu, Cpl)                    // o el único lado activo
ppmBelow = Φ((LSL − μ)/σ) · 1e6
ppmAbove = Φ(−(USL − μ)/σ) · 1e6
```

Pp/Ppk usan **las mismas fórmulas** con σ total.

**Estimación de σ (sobre la ventana de 25 subgrupos):**

| Estimador | Método |
|---|---|
| σ **dentro** (within) | **`R̄ / d2`** — promedio de rangos sobre la constante `d2`. **No** se usa `s/c4` |
| σ **total** (overall) | Desviación estándar muestral de **todos los individuales** de la ventana, por el algoritmo de **Welford** (denominador n−1) |

**Constantes SPC** (fuente: ASTM MNL 7 / Montgomery, apéndice VI), para n = 2…10:

| n | d2 | d3 | A2 | D3 | D4 |
|---|---|---|---|---|---|
| 2 | 1,128 | 0,853 | 1,880 | 0 | 3,267 |
| 3 | 1,693 | 0,888 | 1,023 | 0 | 2,574 |
| 4 | 2,059 | 0,880 | 0,729 | 0 | 2,282 |
| 5 | 2,326 | 0,864 | 0,577 | 0 | 2,114 |
| 6 | 2,534 | 0,848 | 0,483 | 0 | 2,004 |
| 7 | 2,704 | 0,833 | 0,419 | 0,076 | 1,924 |
| 8 | 2,847 | 0,820 | 0,373 | 0,136 | 1,864 |
| 9 | 2,970 | 0,808 | 0,337 | 0,184 | 1,816 |
| 10 | 3,078 | 0,797 | 0,308 | 0,223 | 1,777 |

Relaciones verificadas por test: `A2 = 3/(d2·√n)`, `D4 = 1 + 3·d3/d2`, `D3 = max(0, 1 − 3·d3/d2)`.

**Carta de control: X̄-R** (no X̄-S).

```
X̿    = Σ mean / m          R̄    = Σ range / m
UCLx = X̿ + A2·R̄            LCLx = X̿ − A2·R̄
UCLr = D4·R̄                 LCLr = D3·R̄
sigmaWithin = R̄ / d2
```

Se calculan con los primeros **25** subgrupos y luego quedan **congelados**. "Recalcular límites" toma los **últimos 25** como nueva línea base.

**Reglas de control: Nelson 1 a 4** (exactamente cuatro, cada una marca el punto que **completa** el patrón):

| Regla | Condición |
|---|---|
| 1 | Un punto fuera de los límites 3σ |
| 2 | **Nueve** puntos consecutivos del mismo lado de la línea central |
| 3 | **Seis** puntos consecutivos subiendo o bajando monótonamente |
| 4 | **Catorce** puntos consecutivos alternando arriba/abajo |

Cada una se puede desactivar por separado. En la carta R solo se aplica la regla 1.

**CDF normal: algoritmo de Hart (1968)** en la forma de West (2005), con error absoluto ~1e-15 y buena precisión **en las colas**.

> **Decisión técnica importante:** se eligió Hart explícitamente **en lugar de** Abramowitz–Stegun 7.1.26, cuyo error de 1,5e-7 arruinaría el PPM con Cpk alto. Un test verifica Φ(−6) con error relativo < 1e-9. Quien reimplemente esto y use la aproximación fácil obtendrá PPM sin sentido en la zona que más importa.

**PPM esperado** (del modelo normal, con σ within) vs. **PPM observado** (conteo real de individuales fuera de especificación sobre los últimos 100 / 500 / todos). En modo Explorar el observado es `N/A` ("necesita muestreo").

### 11.4 Las dos ventanas

| Constante | Valor | Uso | Justificación |
|---|---|---|---|
| `WINDOW` | **25** subgrupos | Ventana móvil de capacidad: Cp, Cpk, Pp, Ppk y PPM esperado | Coincide con el tamaño clásico de línea base |
| `STATE_WINDOW` | **10** subgrupos | El proceso se declara fuera de control si **alguna** regla disparó en los últimos 10 | Con límites de 25 subgrupos y reglas 1–4, un proceso **estable** muestra ~19 % de tiempo en falsa alarma con ventana 10, contra ~40 % con ventana 25, **y la detección de un corrimiento de 1,5σ es igual de rápida**. El test 8 lo verifica: 20,3 % medido, umbral < 25 % |
| `baseline` | 25 | Subgrupos para fijar los límites | — |
| `CHART_SUBGROUPS` | 50 | Carta X̄-R visible | — |
| `ROLL_SUBGROUPS` | 100 | Gráfico de Cpk móvil visible | — |
| `RESULT_AFTER` / `RUN` | 25 | Espera antes de resolver una predicción / corrida tras cada acción del desafío | — |

Estados: `baseline` (aún sin límites) → `in` → `out`. La UI agrega `theory` cuando no hay muestreo.

### 11.5 Los tres modos

| Modo | Qué muestra | Para qué sirve |
|---|---|---|
| **Explore** | Solo la curva teórica; sin muestreo. Pp/Ppk coinciden con Cp/Cpk | Entender las fórmulas moviendo μ, σ y los límites |
| **Process** | Muestreo en el tiempo + carta X̄-R + Cpk móvil + histograma + eventos + predicción | Ver control estadístico y causas especiales |
| **Challenge** | Caso sembrado con costos; oculta el panel de parámetros | Decidir con restricción de presupuesto |

Modo inicial por URL: `?mode=process|challenge`.

### 11.6 Controles

Los rangos **dependen del contexto** (c = centro, h = semi-rango). Para `generic` (c=100, h=10):

| id | Tipo | Rango | Paso | Default |
|---|---|---|---|---|
| `in-mu` | range | c±2h → 80…120 | h/100 = 0,1 | 100 |
| `in-sigma` | range | 0,05h…0,8h → 0,5…8 | h/200 | 2 |
| `in-lsl` | range | 75…110 | 0,1 | 90 |
| `in-usl` | range | 90…125 | 0,1 | 110 |
| `in-target` | range | 80…120 | 0,1 | 100 |
| `in-n` | range | 2…10 | 1 | **5** (cambiarlo **reinicia** el proceso) |
| `in-speed` | range | 0…5 | 1 | **2** (= 2 subgrupos/s) |
| `in-shift` | number | 0,25…4 | 0,25 | 1,5 σ |
| `in-drift` | number | 0,01…1 | 0,01 | 0,1 σ |
| `in-threshold` | number | 0,5…3 | 0,01 | **1,33** |
| `in-context` | select | 4 opciones | — | `generic` |
| `hist-window` | select | 100 / 500 / todos | — | 100 |
| `show-specs`, `predict-on`, `on-lsl`, `on-usl` | switch | — | — | off / on / on / on |
| `[data-rule="1..4"]` | 4 checkbox | — | — | todas activas |

**Restricciones:** `lsl ≤ usl − h/10` y `usl ≥ lsl + h/10`. **No se pueden desactivar ambos límites a la vez** (el código revierte el último).

**Handles arrastrables sobre el gráfico** (`role="slider"`) para LSL, USL y μ, con soporte completo de teclado: flechas = ±1 paso, **Shift ×10**, `Home`/`End` a los extremos, y `aria-valuenow`/`aria-valuetext`.

**Atajos:** `Espacio` play/pausa (solo Process) · `S` un paso · `R` reset (Process) o recargar preset (Explore) · `1`–`5` escenarios (solo Explore).

### 11.7 Los 5 escenarios (modo Explorar)

Definidos relativos al contexto. Valores para `generic` (LSL 90, USL 110, T 100):

| # | Nombre | μ, σ | Resultado | Qué enseña |
|---|---|---|---|---|
| 1 | Capable and centered | 100, 2 | Cp = Cpk ≈ **1,67**; PPM ≪ 1 | El caso bueno de referencia |
| 2 | Off-center | 105, 1,5 | Cp ≈ 2,22, Cpk ≈ **1,11** | Dispersión excelente arruinada por el centrado. **Recentrar suele ser la mejora más barata** |
| 3 | Too much variation, centered | 100, 4 | Cp = Cpk ≈ **0,83**; ≈ 12.400 PPM | Centrar no alcanza: hay que reducir variación |
| 4 | Just capable | 100, 3,333 | Cp = Cpk = **1,00**; ≈ 2.700 PPM (1.350 por lado) | El ancla numérica clásica de Six Sigma |
| 5 | One-sided spec | 100, 2,5 (**sin USL**) | Cp = **N/A**, Cpk = Cpl ≈ 1,33; ≈ 32 PPM | Cuando solo importa un mínimo (p. ej. fuerza de sellado) |

### 11.8 Indicadores y chips

**Seis KPI:** `Cp`, `Cpk`, `Pp`, `Ppk` (2 decimales), `Expected PPM` y `Observed PPM`. Valores no finitos muestran `–`. **Cp y Cpk se tachan** (`text-decoration: line-through`) cuando el proceso está fuera de control.

**Chip de estado:**

| Estado | Icono | Color | Texto |
|---|---|---|---|
| theory | `info` | base | "Teórico: sin muestreo" |
| baseline | `loader` | base | "Estableciendo límites de control: {k} de 25 subgrupos" |
| in | `check` | `--color-success` | "En control" |
| out | `alert` | `--color-danger` | "Fuera de control: la capacidad no es válida" |

**Chip de banda** (Cpk contra el umbral): `met` (≥ umbral), `mid` (entre 1,00 y el umbral), `poor` (< 1,00, "no capaz").

### 11.9 Los tres gráficos

| Vista | Canvas | Contenido |
|---|---|---|
| **A. Distribución** | 330 px (280 en móvil) | Curva normal (240 segmentos), **histograma** de valores medidos (ancho de bin h/6, normalizado como densidad), área dentro de especificación con el degradado de marca, **colas fuera de especificación en magenta**, marcadores ±3σ, líneas de LSL/USL/target/μ, y la **regla de Cpk** con el lado mínimo resaltado |
| **B. Carta X̄-R** | 190 px + 130 px | Últimos 50 subgrupos; banda gris de línea base; CL sólida y UCL/LCL discontinuos con etiquetas; marcadores verticales naranjas en los eventos; puntos señalados con círculo magenta y **el número de regla encima**; overlay opcional de especificaciones con patrón distinto |
| **C. Capacidad en el tiempo** | 150 px | Cpk móvil de los últimos 100 subgrupos; **banda verde** desde el umbral hacia arriba; línea de referencia en 1,00 |

La distribución usa **suavizado exponencial** hacia los valores verdaderos (`D.x += (target − D.x) · k`, con `k = min(1, dt·14)`), desactivado con movimiento reducido. La distribución se redibuja en todos los frames mientras se está asentando; las cartas solo cuando hay datos nuevos.

Lista de **señales recientes** (hasta 4, de la más nueva hacia atrás) y tooltip al acercar el puntero a menos de 14 px de un punto marcado.

### 11.10 Modo Challenge

**No es un cuestionario**: es un juego de decisiones con costos. El objetivo es llegar a **Cpk ≥ umbral y en control, gastando lo mínimo**.

**Caso inicial sembrado:**

```
sigma  = 2,9 … 3,9  (1 decimal)
offset = 3 … 6      (pasos de 0,5), signo aleatorio
mu     = 100 ± offset
```

Resultado: Cp ≈ 0,85–1,15 y Cpk muy por debajo de 1. Siempre un desafío real (verificado por test).

**Las cuatro acciones:**

| Acción | Efecto | Costo |
|---|---|---|
| Recentrar ↓ / ↑ | `mu ∓ 1` unidad | **1** |
| Reducir variación | `σ × 0,9` | **5** |
| Pedir especificación más amplia (±1) | Máximo **2 usos**; se acepta con probabilidad **0,5**. **Si el cliente rechaza, igual se cobra** | **8** |

Tras cada acción corren **25 subgrupos** y se **recalculan los límites de control** — detalle pedagógico deliberado: tras mejorar un proceso, los límites viejos ya no representan la voz del proceso.

**Camino óptimo de comparación:** búsqueda exhaustiva sobre `m` reducciones y `k` recentrados que alcanza el umbral al menor costo. **Excluye deliberadamente ampliar la especificación**, porque el cliente puede rechazarla. Un test garantiza que el óptimo **siempre incluye recentrar primero**, reforzando el mensaje didáctico.

**Panel de facilitador:** semilla (mismo caso para toda la clase), umbral de Cpk y los tres costos, todos editables.

**Predicción (modo Process):** antes de cada evento, dos preguntas obligatorias — "¿qué le pasará al Cpk?" (sube / baja / igual) y "¿lo detectará la carta en 25 subgrupos?" (sí / no). Tras 25 subgrupos se muestra "lo que dijiste" vs. "el resultado", **incluido el retardo de detección en subgrupos**. No hay puntaje acumulado ni persistencia.

### 11.11 Tests del motor (28 aserciones, todas pasan)

| Grupo | Qué protege |
|---|---|
| 1 (×4) | Fórmulas de Cp/Cpk/Cpu/Cpl, incluido el caso unilateral |
| 2.1 | **Exactitud de Φ(x)** contra valores publicados en 0, −1, −2, −3, −4, **−6** y 1,96 (error relativo real 5,35e-10) |
| 2.3 | Ancla didáctica: Cpk = 1,00 centrado ⇒ **2.699,8 PPM** con colas simétricas |
| 3 (×2) | Constantes SPC **idénticas a la tabla publicada** y **consistentes entre sí** (protege contra erratas) |
| 4 (×3) | **`R̄/d2` recupera σ = 2 dentro del 5 %** para n = 2, 5 y 10 sobre 1000 subgrupos: valida a la vez el estimador y el generador normal |
| 5 (×6) | Cada regla de Nelson dispara **en el punto exacto** y no uno antes; sin falsos positivos en una serie limpia; las reglas se pueden desactivar |
| 6 (×4) | Límites congelados tras 25 subgrupos; **un corrimiento de 1,5σ se detecta dentro de 25 subgrupos**; el Cpk cae ≈ 0,5 (= 1,5σ/3σ) |
| 7 | **Bajo deriva, σ total supera a σ within ⇒ Ppk < Cpk** (Cpk 1,07 vs. Ppk 0,71) |
| 8 | **Un proceso estable da 20,3 % de falsa alarma** (umbral < 25 %): justifica `STATE_WINDOW = 10` |
| 9 (×4) | Determinismo por semilla; caso del desafío reproducible, siempre descentrado con Cpk < 1; el camino óptimo alcanza el umbral y **siempre recentra** |

### 11.12 Claves i18n (prefijo `pc.`)

**221 claves por idioma** — la simulación con más texto del hub. Las que llevan interpolación incluyen: `kpi.noteProcess {w}{s}`, `state.baseline {k}{n}`, `band.met {thr}`, `live` (6 placeholders), `dist.aria` (5), `spc.hint {n}{b}`, `spc.xAria {n}{s}`, `roll.aria {v}{thr}`, `predict.title {e}`, `predict.delay {n}`, `ch.now` (5), `ch.optimalText {k}{m}{c}{cpk}`, `ch.final {cpk}{start}{state}`.

### 11.13 Advertencia obligatoria sobre el 1,33

> **1,33 se presenta siempre como una referencia de uso común en la industria, NUNCA como un requisito de Amcor ni de un cliente.** Está escrito literalmente en el código, en los tres idiomas, en seis lugares distintos, y además **el umbral es configurable** (0,5–3) tanto en el panel de parámetros como en el del facilitador. Lo mismo vale para los costos del desafío: *"valores ilustrativos únicamente; no son cifras de Amcor ni de un cliente"*. **Cualquier reimplementación debe conservar estas advertencias.**

---

## 12. Accesibilidad

La accesibilidad no es un añadido: condiciona el diseño visual (los colores de estado se eligieron por contraste) y la arquitectura (toda animación tiene un camino alternativo sin movimiento).

### 12.1 Requisitos implementados

| Área | Implementación |
|---|---|
| **Salto de navegación** | `.skip-link` al inicio del `<body>`, oculto hasta recibir foco de teclado |
| **Contraste** | Todos los textos cumplen WCAG AA. Los tokens documentan la ratio: `--color-text-muted` 7,9:1; `--color-text-subtle` 5,2:1; `--color-link` 4,9:1 |
| **Color como único canal** | Prohibido. Naranja (cuello de botella) y verde (verificado) solo en bordes, marcadores e iconos; la etiqueta siempre lleva texto. El estado también se comunica con palabras en el chip de estado |
| **Canvas** | Cada `<canvas>` lleva `role="img"` y un `aria-label` traducido que describe qué se está mostrando |
| **Región en vivo** | `<p class="sr-only" id="live-region" aria-live="polite" aria-atomic="true">` con un resumen textual de los números clave, actualizado a intervalos **largos** (no en cada frame) para que un lector de pantalla pueda seguirlo |
| **Foco** | `:focus-visible` con anillo de 2 px en `--color-focus` y offset. Nunca se elimina el outline sin reemplazarlo |
| **Controles** | Todo `input` tiene `<label for>`; los sliders tienen un `<output for>` que muestra el valor actual; los interruptores usan `role="switch"` |
| **Selector de idioma** | `role="group"` con `aria-label`; el botón activo lleva `aria-pressed="true"` |
| **Números** | `font-variant-numeric: tabular-nums` en todas las lecturas, para que no "bailen" al actualizarse |
| **Movimiento reducido** | `@media (prefers-reduced-motion: reduce)` desactiva transiciones y el lift de tarjetas; cada simulación además **aplica el resultado de inmediato** en lugar de animarlo |
| **Teclado** | Cada simulación expone atajos documentados en pantalla (bloque `.hint` con elementos `<kbd>`). Los atajos se ignoran mientras el foco está en un campo numérico, select o textarea |
| **Orden en móvil** | Escenario → controles → acerca de |

### 12.2 Regla de atajos de teclado

Todas las simulaciones comparten el mismo guardia:

```js
document.addEventListener('keydown', (e) => {
  if (e.altKey || e.ctrlKey || e.metaKey) return;         // no secuestrar combinaciones del sistema
  const typing = e.target.closest('input[type="number"], select, textarea');
  if (e.key === ' ' || e.code === 'Space') {
    if (typing || e.target.closest('button, a, summary, input[type="checkbox"], input[type="radio"]')) return;
    e.preventDefault();
    setPlaying(!playing);
  }
  // … otros atajos, también protegidos por `typing`
});
```

La barra espaciadora **no** se intercepta cuando el foco está en un botón o enlace, porque ahí su significado nativo es "activar".

---

## 13. Testing

### 13.1 Tests unitarios de motores (en el repositorio)

Sin dependencias, sin framework, sin npm. Se ejecutan con Node a secas:

```sh
node simulations/littles-law/engine.test.js
node simulations/push-pull/engine.test.js
node simulations/process-capability/engine.test.js
```

Cada archivo imprime una línea `PASS` o `FAIL` por caso y termina con `All engine tests passed` o `N test(s) failed`, y fija `process.exitCode` para poder encadenarlo en CI.

El patrón del helper es mínimo:

```js
let failed = 0;
const check = (name, ok, extra = '') => {
  console.log(`${ok ? 'PASS' : 'FAIL'}  ${name}${extra ? '  — ' + extra : ''}`);
  if (!ok) failed++;
};
```

Esto es posible **solo porque los motores son puros**: no tocan el DOM, así que `require()` funciona directamente en Node.

> **Verificado el 2026-11-03:** los tres archivos pasan en `main`.

### 13.2 Qué debe cubrir un test de motor

Tres familias de invariantes, que conviene replicar en cualquier simulación nueva:

1. **Casos calculados a mano.** Un escenario simple cuyo resultado se puede obtener con lápiz y papel, comparado con tolerancia explícita.
2. **Leyes teóricas.** Por ejemplo, que la Ley de Little se cumpla dentro del 5 % en un escenario estable, o que un sistema M/M/1 reproduzca su cola teórica.
3. **Determinismo.** Dos instancias con la misma semilla y las mismas entradas deben producir métricas idénticas. **Este es el test más valioso del proyecto**: es el que detectó el error del reloj del juego Push vs. Pull.

> **Lección aprendida (importante para quien replique):** al comparar métricas de punto flotante entre dos corridas que avanzaron con pasos distintos, hay que **redondear antes de comparar** (en el proyecto, a 9 decimales). Las integrales acumuladas difieren en el orden de 1e-14 por el orden de las sumas, aunque el modelo sea idéntico. Comparar strings JSON crudos produce falsos negativos.

### 13.3 Tests de navegador (no versionados)

Durante el desarrollo se usaron suites ad-hoc de **Playwright** (`chromium`), ejecutadas **en dos modos** contra la misma build:

- `file:///ruta/GreenBelt/` — el modo real de distribución
- `http://localhost:8765/GreenBelt/` — equivalente a GitHub Pages

> Estas suites **no están en el repositorio** (vivían en un directorio temporal de la sesión de desarrollo). Esta sección documenta qué cubrían para que puedan reconstruirse.

Cobertura que tenía cada suite:

| Suite | Casos | Qué verificaba |
|---|---|---|
| Hub | 35 | Tarjetas desde el registry (y que el `draft` no aparece), los tres idiomas, persistencia de `?lang=` al navegar, detección por idioma del navegador, formato numérico localizado, foco de teclado en las tarjetas, sin desbordamiento horizontal a 375 px, movimiento reducido, plantilla accesible por URL |
| Little's Law | 27 | Los 3 escenarios, verificación de la ley dentro de tolerancia, detección de cuello de botella, estado inestable, aritmética del bloque de cálculo, ausencia de calentamiento, nota de escenario personalizado |
| Push vs. Pull | 27 | Presets, eventos, juego completo de 6 rondas, reproducibilidad por semilla, rondas con tarjetas editables, panel de facilitador, debrief |
| Process capability | 30 | Cálculo de Cp/Cpk, límites de especificación, cartas de control, reglas de señal, modo desafío |

**Dos invariantes transversales que toda suite comprobaba, y que son la red de seguridad de las restricciones R2 y R3:**

```js
check('No external network requests', external.length === 0);
check('No console errors / 4xx', errors.length === 0);
```

Se instrumentan así:

```js
page.on('request', (r) => {
  const u = new URL(r.url());
  if (u.protocol.startsWith('http') && u.hostname !== 'localhost') external.push(r.url());
});
page.on('pageerror', (e) => errors.push(e.message));
page.on('console', (m) => { if (m.type() === 'error') errors.push(m.text()); });
page.on('response', (r) => { if (r.status() >= 400) errors.push(`${r.status()} ${r.url()}`); });
```

### 13.4 Checklist de verificación manual antes de publicar

1. Abrir `index.html` con **doble clic** (no por servidor) y navegar a cada simulación.
2. Cambiar idioma en una simulación, volver al Home y confirmar que el idioma persiste.
3. Reducir la ventana a 375 px de ancho y confirmar que no hay scroll horizontal.
4. Activar "reducir movimiento" en el sistema operativo y confirmar que todo sigue siendo usable.
5. Navegar una página completa solo con `Tab` y confirmar que el foco siempre es visible.
6. Abrir la consola del navegador: debe estar limpia, sin errores ni peticiones externas.

---

## 14. Despliegue y distribución

### 14.1 Dos canales, por una razón concreta

| Canal | Cómo | Cuándo |
|---|---|---|
| **GitHub Pages** | Push a `main`; el archivo `.nojekyll` evita que Jekyll procese el sitio. URL: `https://drotondi.github.io/GreenBelt/` | Acceso público, fuera de la red corporativa |
| **ZIP** | Comprimir la carpeta y distribuir por Teams, SharePoint o correo. El destinatario descomprime y hace doble clic en `index.html` | **Canal principal**: el firewall corporativo de Amcor bloquea `github.io` |

El segundo canal es el motivo de la restricción R2 y de todas sus consecuencias (scripts clásicos, `index.html` explícito, cero peticiones externas). **Si se rompe el funcionamiento desde `file://`, el producto deja de servir para su uso principal.**

Para generar el ZIP desde el repositorio, sin incluir `.git`:

```sh
git archive --format=zip --prefix=GreenBelt/ -o greenbelt.zip HEAD
```

### 14.2 Servidor local opcional

Mismo comportamiento, útil para desarrollo:

```sh
python3 -m http.server
# abrir http://localhost:8000/
```

### 14.3 Qué nunca se publica

- `assets/fonts/` y `assets/logo-*` (activos de marca de Amcor, en `.gitignore`).
- Cualquier cifra real de Amcor: costos, volúmenes, nombres de proveedores o sistemas.

### 14.4 Licencia

`Copyright (c) 2026 drotondi. All rights reserved.` Publicado solo para visualización; no se otorga licencia de copia, modificación o distribución. Los nombres, logos y tipografías de Amcor son marcas o activos propietarios de Amcor plc y **no forman parte del repositorio**.

---

## 15. Guía de replicación paso a paso

Orden recomendado para construir el proyecto desde cero.

### Fase 0 — Esqueleto (medio día)

1. Crear el repositorio con `.gitignore` (ignorando `assets/fonts/` y `assets/logo-*`), `LICENSE` y `.nojekyll`.
2. Escribir `shared/tokens.css` **primero**. Es la base de todo lo visual; si se deja para el final, aparecen colores hardcodeados por todo el código.
3. Escribir `shared/base.css` con los componentes compartidos y los cuatro breakpoints.

### Fase 1 — Capa compartida (1–2 días)

4. `shared/config.js` con el flag de marca y `ROOT_URL` derivado de `document.currentScript.src`.
5. `shared/i18n.js` completo, incluidos los enlaces declarativos `data-i18n`, `data-i18n-attr` y `data-keep-lang`.
6. `shared/icons.js` y `shared/motion.js`.
7. `shared/header.js` con las dos variantes de header y el footer.

> **Prueba de fuego de esta fase:** una página en blanco que solo renderice header y footer debe poder abrirse con doble clic, cambiar de idioma y conservarlo al navegar.

### Fase 2 — Home y registry (medio día)

8. `simulations/registry.js` vacío (array sin entradas).
9. `index.html` con el renderizado de tarjetas, el escapado HTML y la re-renderización por cambio de idioma.
10. Verificar que con el registry vacío se muestre el mensaje de `home.empty`.

### Fase 3 — Plantilla (medio día)

11. `simulations/_template/` completo: HTML canónico, `sim.js` con canvas DPR-escalado, bucle de frames, play/reset, un slider, región `aria-live` y soporte de movimiento reducido.
12. Registrarla con `status: 'draft'` y comprobar que **no** aparece en el Home pero **sí** abre por URL directa.

### Fase 4 — Primera simulación (1–2 días)

13. Galton Board. Es la más simple (sin `engine.js` separado) y valida toda la infraestructura: canvas, animación, i18n, tokens, atajos de teclado, rendimiento.

### Fase 5 — Helpers de simulación (medio día)

14. `shared/sim/random.js` con el patrón de doble exportación (navegador + Node).
15. `shared/sim/chart.js`.
16. Escribir el primer `engine.test.js` y verificar que corre con `node` sin instalar nada.

### Fase 6 — Simulaciones con motor (2–4 días cada una)

17. Para cada una: `engine.js` puro primero, con sus tests; recién después la UI.

> **Orden recomendado dentro de cada simulación:** motor → tests del motor → UI → i18n (los tres idiomas de una vez) → CSS → verificación de accesibilidad. Dejar el i18n para el final siempre produce textos hardcodeados olvidados.

### Fase 7 — Verificación y publicación

18. Pasar la checklist manual de la sección 13.4.
19. Cambiar `status` a `'live'` en el registry.
20. Publicar y generar el ZIP.

### 15.1 Cómo agregar una simulación nueva (proceso estable)

1. Copiar `simulations/_template/` a `simulations/<nuevo-id>/`.
2. Renombrar el prefijo de claves `tpl.` por el propio en `i18n.js` y en el HTML.
3. Implementar el motor y la UI. Todo texto visible al `i18n.js` de la carpeta, en EN/ES/PT.
4. Reemplazar `thumbnail.svg` por una vista previa de 320×180.
5. **Agregar una entrada al final** de `simulations/registry.js` con `status: 'draft'`. No reordenar, editar ni borrar entradas existentes.
6. Probar abriendo el `index.html` de la simulación directamente desde disco.
7. Cuando esté lista, cambiar a `status: 'live'`. No hay ningún otro cambio que hacer.

### 15.2 Errores conocidos y cómo evitarlos

Problemas reales encontrados durante el desarrollo. Vale la pena leerlos antes de replicar:

| Problema | Causa | Solución aplicada |
|---|---|---|
| La página no carga desde disco | ES modules bloqueados bajo `file://` | Scripts clásicos + namespace global `LSS` |
| Un enlace a carpeta no abre nada desde disco | `file://` no resuelve `index.html` implícito | Enlaces siempre con `index.html` explícito |
| El juego daba resultados distintos con la misma semilla | El reloj cerraba la ronda en 29,999999… en lugar de 30, corriendo el cierre de período respecto de las decisiones | `advanceTo(target)` en el motor, que aterriza exactamente en el objetivo, y el bucle de frames lo usa siempre al llegar al final de ronda |
| Tests de determinismo con falsos negativos | Comparación de floats acumulados con orden de suma distinto | Redondear a 9 decimales antes de comparar |
| Caída de frame rate con muchos elementos | Repintado completo del canvas por frame | Capas separadas y *blitting* de la parte estática |
| Texto de escenario desactualizado tras tocar un control | La UI no distinguía "preset activo" de "parámetros personalizados" | Estado de preset `-1` y texto de "personalizado" |
| Contraste insuficiente en etiquetas de estado | Naranja y verde fallan AA como texto pequeño | Color solo en bordes/iconos; la etiqueta en navy |

---

## 16. Glosario

| Término | Definición en el contexto del proyecto |
|---|---|
| **WIP** | *Work in process*: unidades dentro de la línea, en cola o en proceso, aún no terminadas |
| **Throughput** | Unidades que salen de la línea por unidad de tiempo |
| **Lead time** | Tiempo que una unidad pasa dentro del sistema, desde que entra hasta que sale |
| **Ley de Little** | WIP = Throughput × Lead time, válida en régimen estable y de largo plazo |
| **Cuello de botella** | La estación con menor capacidad efectiva; determina el throughput de toda la línea |
| **CV** | Coeficiente de variación: desvío estándar dividido por la media. CV 0 = determinista |
| **Push (MRP)** | Liberar trabajo según un plan basado en pronóstico, sin mirar el estado aguas abajo |
| **Pull (kanban)** | Liberar trabajo solo cuando el consumo aguas abajo libera una tarjeta |
| **Kanban card** | Autorización para producir una unidad; el número de tarjetas acota el inventario |
| **Fill rate** | Proporción de la demanda atendida desde stock al momento de pedirla |
| **Backorder** | Demanda no atendida que queda pendiente |
| **Cp** | Capacidad potencial: ancho de la especificación contra la dispersión del proceso, ignorando el centrado |
| **Cpk** | Capacidad real: considera además cuán descentrado está el proceso respecto de la especificación |
| **Voz del proceso** | Lo que el proceso efectivamente produce (su dispersión natural) |
| **Voz del cliente** | Los límites de especificación |
| **Causa común** | Variación inherente al proceso, presente siempre |
| **Causa especial** | Variación por un factor identificable y no habitual; es lo que una carta de control detecta |
| **Semilla (seed)** | Entero que inicializa el generador de números aleatorios. Misma semilla ⇒ misma corrida |
| **Modo Explorar** | Modo libre: el usuario mueve parámetros y observa |
| **Modo Juego** | Modo guiado por rondas, con decisiones del usuario y un debrief final |
| **Facilitador** | Panel oculto que permite fijar la semilla, forzar eventos y ocultar KPIs durante la clase |
| **DTO** | Digital Transformation Office, el área de Amcor propietaria del material |

---

## Anexo A — Resumen de decisiones de arquitectura

| Decisión | Alternativa descartada | Por qué |
|---|---|---|
| Vanilla JS, sin framework | React / Vue | Un framework exige build step; el build step rompe el doble clic y agrega dependencias que caducan |
| Scripts clásicos + `window.LSS` | ES modules | Los módulos están bloqueados bajo `file://` |
| Canvas 2D | SVG / DOM | Cientos de elementos animados a 60 fps; el DOM no escala a eso |
| Tokens CSS leídos por el canvas | Constantes de color en JS | Evita que el gráfico y el HTML se desincronicen |
| Motores puros + tests en Node | Tests solo de navegador | Los tests corren en milisegundos y sin instalar nada |
| RNG propio con semilla | `Math.random()` | `Math.random()` no es reproducible; la reproducibilidad es un requisito pedagógico |
| Gamma por media y CV | Normal truncada | Los tiempos deben ser positivos; la gamma permite barrer de determinista a muy variable con un solo mando |
| Registry como única fuente del catálogo | Tarjetas escritas a mano en el HTML | Agregar una simulación toca un solo archivo |
| i18n con prefijo por simulación | Un diccionario global plano | Evita colisiones de claves al crecer |
