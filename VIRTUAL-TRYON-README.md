# Product-Page Virtual Try-On — Implementation Guide

A camera-based lipstick try-on that runs **inside the product gallery** (no popup):
`try me on` → privacy consent → "getting ready" → live mirrored camera with the
selected shade rendered on the lips. Includes snapshot save, a draggable
before/after compare split, and live re-tinting when the shopper changes the
shade on the product page.

Built and proven on a Shopify Horizon theme (ST London), but the engine is
plain JavaScript — the Shopify parts are only "where the data comes from" and
can be swapped for any stack.

**Reference implementation in this repo**

| Piece | File |
|---|---|
| PDP section: stage markup, CSS, variant script, try-on engine | `sections/kj-main-product.liquid` |
| Shade name → hex map (Liquid snippet) | `snippets/kj-shade-hex.liquid` |
| Homepage "studio" variant (multi-category, add-all-to-cart) | `sections/kj-virtual-try-on.liquid` |

---

## 1. Architecture

```
[try me on button]                     (below the gallery image)
        │ click
        ▼
[consent card]  ── no thanks ──► close
        │ agree (checkbox-gated, remembered per page-load)
        ▼
[getting ready]  ── loads MediaPipe + model + getUserMedia (all lazy)
        │ first successful frame
        ▼
[live view]      video (hidden) ──► canvas (frame + lip tint), CSS-mirrored
  controls: ✕ close · snapshot save · compare split (draggable divider)
  shade source: product page's own swatches, synced via a CustomEvent
```

Three state layers stacked inside the gallery's image container
(`position:absolute; inset:0`); exactly one is visible at a time via the
`hidden` attribute.

**Engine: MediaPipe Tasks API** (`@mediapipe/tasks-vision`), *not* the classic
`face_mesh.js` solution. FaceLandmarker returns 478 face landmarks (irises
included) with a synchronous `detectForVideo(video, timestampMs)` call that we
drive from our own `requestAnimationFrame` loop.

Why Tasks and not the classics — learned the hard way:
- classic FaceMesh + Hands **deadlock the frame loop** when two solutions
  process the same frame (camera freezes);
- Holistic finds hands via body pose, which fails in a face close-up;
- the classic solutions are deprecated, 2021-era models; FaceLandmarker tracks
  visibly better and Tasks instances are isolated (no global collisions).

---

## 2. Dependencies (all lazy — nothing loads until consent)

```js
var MP_VER = 'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.21';   // pin the version!
// wasm:  MP_VER + '/wasm'
// model: https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task
```

~11 MB total on first run (wasm + model), cached by the browser afterwards.
**Pin the package version** — unpinned CDN URLs drift when Google publishes.

Camera: `getUserMedia({ video: { facingMode: 'user', width: { ideal: 1280 }, height: { ideal: 720 } } })`.
720p matters — 480p upscaled into a large gallery square looks soft.
HTTPS (or localhost) is required for camera access.

---

## 3. Markup (inside the gallery's main-image container)

The image container must be `position: relative; overflow: hidden` with a
fixed aspect. Add the stage as its last child, plus the button + shade-map
JSON right after the container:

```html
<div class="stage" data-vto-stage hidden>
  <div data-vto-consent hidden>            <!-- transparent layer, white card -->
    <div class="card">
      <button data-vto-close>×</button>
      <h4>privacy policy</h4>
      <p>by clicking "i agree", i consent to my device camera being used for the
         virtual try-on experience. camera frames are processed in real time on
         this device to render shades and are never stored or uploaded.</p>
      <label><input type="checkbox" data-vto-check> i agree</label>
      <button data-vto-agree disabled>agree</button>
      <button data-vto-close>no thanks</button>
    </div>
  </div>

  <div data-vto-ready hidden>              <!-- pale pink, spinner -->
    <span class="spinner"></span>
    <span data-vto-ready-label>getting ready</span>
  </div>

  <div data-vto-live hidden>               <!-- black, camera -->
    <video data-vto-video playsinline muted></video>   <!-- display:none -->
    <canvas data-vto-canvas></canvas>                  <!-- fills, mirrored -->
    <div data-vto-divider hidden><span data-vto-handle>‹ ›</span></div>
    <button data-vto-close>×</button>
    <button data-vto-snap>📷</button>
    <button data-vto-compare>◫</button>
    <div data-vto-flash></div>
  </div>
</div>

<button data-vto-open>try me on</button>

<script type="application/json" data-vto-map>
  { "shade name lowercase": "#RRGGBB", "another shade": "#A65E44" }
</script>
```

### Critical CSS details

```css
/* 1. hidden must ALWAYS win — display rules on the layers defeat the
      attribute otherwise and every layer renders stacked (real bug we hit) */
[data-vto-stage][hidden], [data-vto-stage] [hidden] { display: none !important; }

/* 2. selfie mirror: flip the CANVAS with CSS, never the bitmap */
[data-vto-live] canvas {
  position: absolute; inset: 0; width: 100%; height: 100%;
  object-fit: cover; transform: scaleX(-1);
}
[data-vto-live] video { display: none; }
```

---

## 4. The engine (complete, framework-free)

```js
(function () {
  if (window.__vto) return;   // idempotent — safe under DOM morphing / re-runs
  window.__vto = { stream: null, raf: null, face: null, color: null,
                   loading: false, agreed: false, compare: false, split: 0.5,
                   drag: false, map: null };
  var S = window.__vto;
  var MP_VER = 'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.21';

  function els() {
    var stage = document.querySelector('[data-vto-stage]');
    if (!stage) return null;
    return {
      stage: stage,
      consent: stage.querySelector('[data-vto-consent]'),
      ready:   stage.querySelector('[data-vto-ready]'),
      readyLabel: stage.querySelector('[data-vto-ready-label]'),
      live:    stage.querySelector('[data-vto-live]'),
      video:   stage.querySelector('[data-vto-video]'),
      canvas:  stage.querySelector('[data-vto-canvas]'),
      divider: stage.querySelector('[data-vto-divider]'),
      flash:   stage.querySelector('[data-vto-flash]')
    };
  }

  function shadeMap() {
    if (S.map) return S.map;
    try { S.map = JSON.parse(document.querySelector('[data-vto-map]').textContent); }
    catch (e) { S.map = {}; }
    return S.map;
  }

  function setState(name) {
    var e = els(); if (!e) return;
    e.consent.hidden = name !== 'consent';
    e.ready.hidden   = name !== 'ready';
    e.live.hidden    = name !== 'live';
  }

  /* ---- model + camera (lazy) ---- */
  function loadFace() {
    if (S.face) return Promise.resolve();
    return import(MP_VER + '/vision_bundle.mjs').then(function (m) {
      return m.FilesetResolver.forVisionTasks(MP_VER + '/wasm').then(function (fs) {
        var make = function (delegate) {
          return m.FaceLandmarker.createFromOptions(fs, {
            baseOptions: {
              modelAssetPath: 'https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task',
              delegate: delegate
            },
            runningMode: 'VIDEO',
            numFaces: 1
          });
        };
        return make('GPU').catch(function () { return make('CPU'); })
          .then(function (r) { S.face = r; });
      });
    });
  }

  function startAR() {
    var e = els(); if (!e || S.loading) return;
    S.loading = true;
    loadFace()
      .then(function () {
        return navigator.mediaDevices.getUserMedia({
          video: { facingMode: 'user', width: { ideal: 1280 }, height: { ideal: 720 } },
          audio: false
        });
      })
      .then(function (stream) {
        S.loading = false;
        S.stream = stream;
        e.video.srcObject = stream;
        var p = e.video.play(); if (p && p.catch) p.catch(function () {});
        startLoop();
      })
      .catch(function () {
        S.loading = false;
        var e2 = els();
        if (e2) e2.readyLabel.textContent = 'camera unavailable — please allow camera access';
      });
  }

  function startLoop() {
    if (S.raf) cancelAnimationFrame(S.raf);
    var tick = function () {
      var e = els();
      if (!e || e.stage.hidden || !S.stream) { S.raf = null; return; }
      if (e.video.readyState >= 2 && S.face) {
        var res = null;
        try { res = S.face.detectForVideo(e.video, performance.now()); } catch (err) {}
        drawFrame(e, res);
      }
      S.raf = requestAnimationFrame(tick);
    };
    S.raf = requestAnimationFrame(tick);
  }

  /* ---- render ---- */
  function drawFrame(e, res) {
    if (!e.ready.hidden) setState('live');           // first frame flips the state
    var c = e.canvas, ctx = c.getContext('2d');
    if (c.width !== e.video.videoWidth && e.video.videoWidth) {
      c.width = e.video.videoWidth; c.height = e.video.videoHeight;
    }
    if (!c.width) return;
    ctx.save();
    ctx.clearRect(0, 0, c.width, c.height);
    ctx.drawImage(e.video, 0, 0, c.width, c.height);
    if (res && res.faceLandmarks && res.faceLandmarks.length) {
      var lm = res.faceLandmarks[0];
      // FaceMesh lip topology: outer ring then inner ring, filled evenodd
      var outer = [61,146,91,181,84,17,314,405,321,375,291,409,270,269,267,0,37,39,40,185,61];
      var inner = [78,95,88,178,87,14,317,402,318,324,308,415,310,311,312,13,82,81,80,191,78];
      if (S.compare) {
        // canvas is CSS-mirrored: RIGHT of the on-screen divider = LEFT of bitmap
        ctx.beginPath();
        ctx.rect(0, 0, c.width * (1 - S.split), c.height);
        ctx.clip();
      }
      ctx.globalCompositeOperation = 'multiply';     // keeps lip texture
      ctx.fillStyle = (S.color || '#C98A78') + '80'; // 50% alpha
      ctx.beginPath();
      outer.forEach(function (i, k) {
        var p = lm[i];
        k ? ctx.lineTo(p.x * c.width, p.y * c.height) : ctx.moveTo(p.x * c.width, p.y * c.height);
      });
      inner.forEach(function (i, k) {
        var p = lm[i];
        k ? ctx.lineTo(p.x * c.width, p.y * c.height) : ctx.moveTo(p.x * c.width, p.y * c.height);
      });
      ctx.fill('evenodd');                           // inner ring punches the mouth out
      ctx.globalCompositeOperation = 'source-over';
    }
    ctx.restore();
  }

  function closeVto() {
    var e = els(); if (!e) return;
    e.stage.hidden = true;
    setState('none');
    S.compare = false;
    if (e.divider) e.divider.hidden = true;
    if (S.raf) { cancelAnimationFrame(S.raf); S.raf = null; }
    if (S.stream) { S.stream.getTracks().forEach(function (t) { t.stop(); }); S.stream = null; }
    if (e.video) e.video.srcObject = null;           // camera light OFF, always
  }

  /* ---- all interaction DELEGATED at document level ----
     (survives theme DOM morphing; nothing binds to elements directly) */
  document.addEventListener('click', function (ev) {
    if (ev.target.closest('[data-vto-open]')) {
      var e = els(); if (!e || !e.stage.hidden) return;
      S.color = currentShadeColor();
      e.readyLabel.textContent = 'getting ready';
      e.stage.hidden = false;
      if (S.agreed) { setState('ready'); startAR(); } else setState('consent');
      return;
    }
    if (ev.target.closest('[data-vto-close]')) { closeVto(); return; }
    var agree = ev.target.closest('[data-vto-agree]');
    if (agree && !agree.disabled) { S.agreed = true; setState('ready'); startAR(); return; }
    if (ev.target.closest('[data-vto-snap]')) {
      var e2 = els(); if (!e2 || !e2.canvas.width) return;
      var a = document.createElement('a');
      a.download = 'try-on.png';
      a.href = e2.canvas.toDataURL('image/png');
      a.click();
      e2.flash.classList.remove('on'); void e2.flash.offsetWidth; e2.flash.classList.add('on');
      return;
    }
    if (ev.target.closest('[data-vto-compare]')) {
      var e3 = els(); if (!e3) return;
      S.compare = !S.compare; S.split = 0.5;
      e3.divider.hidden = !S.compare;
      e3.divider.style.left = '50%';
    }
  });

  document.addEventListener('change', function (ev) {
    if (!ev.target.closest('[data-vto-check]')) return;
    var agree = document.querySelector('[data-vto-agree]');
    if (agree) agree.disabled = !ev.target.checked;
  });

  /* compare divider drag */
  document.addEventListener('pointerdown', function (ev) {
    if (!ev.target.closest('[data-vto-handle]')) return;
    ev.preventDefault(); S.drag = true;
  });
  document.addEventListener('pointermove', function (ev) {
    if (!S.drag) return;
    var e = els(); if (!e) return;
    var r = e.stage.getBoundingClientRect();
    S.split = Math.min(0.88, Math.max(0.12, (ev.clientX - r.left) / r.width));
    e.divider.style.left = (S.split * 100) + '%';
  });
  document.addEventListener('pointerup', function () { S.drag = false; });

  /* ---- shade sync with the product page ---- */
  function currentShadeColor() {
    var map = shadeMap();
    var label = document.querySelector('[data-shade-label]');   // your PDP's shade name element
    var key = label ? label.textContent.trim().toLowerCase() : '';
    if (map[key]) return map[key];
    for (var k in map) return map[k];
    return '#C98A78';
  }
  document.addEventListener('vto:shade', function (ev) {
    var map = shadeMap();
    var key = ((ev.detail && ev.detail.title) || '').toLowerCase().trim();
    if (map[key]) S.color = map[key];
  });
})();
```

Wiring the shade sync: in your product page's variant-change handler, add one line —

```js
document.dispatchEvent(new CustomEvent('vto:shade', { detail: { title: variantTitle } }));
```

Live re-tint then works from any swatch/dropdown for free.

---

## 5. The shade map

Rendering needs a solid hex per shade. Product images can't be sampled
reliably (white backgrounds, packaging), so maintain an explicit map.

**Shopify/Liquid** (this repo's `snippets/kj-shade-hex.liquid`): one long
`;name:#RRGGBB;name:#RRGGBB` string probed with `contains` — O(1)-ish, no
loops. The PDP emits per-product JSON from it at render:

```liquid
{%- for variant in product.variants -%}
  {%- capture hex -%}{% render 'kj-shade-hex', shade: variant.title %}{%- endcapture -%}
  {%- if hex != blank -%}"{{ variant.title | downcase }}":"{{ hex }}",{%- endif -%}
{%- endfor -%}
```

**Any other stack**: ship `{ "shade": "#hex" }` JSON from your product data.
Tips that carried over:
- key by lowercase trimmed shade name;
- single-variant products often carry the shade in the *title*
  ("… - ST008 - Ruby") — fall back to the title's last ` - ` segment;
- eyeball hexes against swatch photos; store typos need dual entries
  (we map both "natural fair" and "nataural fair").

Gate the whole feature on `map size > 0` — products with no mapped shades
should simply not show the button.

---

## 6. Behaviors & math worth knowing

- **Mirror + compare interaction.** The canvas is CSS-flipped, so screen-space
  and bitmap-space are horizontally reversed. With the divider at screen
  fraction `f` and "makeup on the right", the tint clip in bitmap space is
  `rect(0, 0, width * (1 - f), height)`. Get this wrong and the split feels
  inverted.
- **Snapshot** is just `canvas.toDataURL('image/png')` — the canvas already
  holds frame + tint (and the clip if compare is on). Note the saved PNG is
  unmirrored (like iPhone selfies).
- **`multiply` at 50% alpha** (`hex + '80'`) keeps lip texture through the
  color. Straight `source-over` looks like a sticker. (For *nail polish* we
  used source-over at ~70% because light shades vanish under multiply — but
  nails need a dedicated segmentation model to look right; landmark ellipses
  weren't good enough in practice.)
- **First frame flips getting-ready → live**, so the pink panel exactly covers
  model download + camera warm-up, however long that takes.
- **Consent is remembered per page load** (`S.agreed`), so re-opening skips
  straight to the camera. Persist to localStorage if you want it stickier.
- **Teardown matters**: cancel the rAF loop, `stop()` every track, and null
  `srcObject` — otherwise the camera light stays on after close.
- **Timestamps** for `detectForVideo` must be monotonically increasing —
  `performance.now()` in the rAF loop satisfies this.

## 7. Pitfalls checklist (each of these was a real bug)

1. `display: flex` on a state layer silently defeats the `hidden` attribute →
   keep the `[hidden] { display: none !important }` guard.
2. Don't run two classic MediaPipe solutions on one frame → camera freeze.
   Use the Tasks API.
3. Don't bind listeners to elements in themes that morph the DOM — delegate
   everything at `document` level, guarded by a `window.__x` idempotency flag.
4. Pin the CDN version of `tasks-vision` and keep the `/wasm` path on the
   same version.
5. Request 720p; the default 640×480 looks soft in a big gallery square.
6. The camera prompt only appears after a user gesture + consent — never
   auto-start on page load (browsers may hard-block, users definitely hate it).
7. Test on HTTPS; `getUserMedia` silently fails on plain HTTP (localhost is
   exempt).

## 8. Porting checklist

- [ ] Copy the stage markup into your gallery's image container
      (`relative` + `overflow:hidden` parent).
- [ ] Copy the CSS (states, mirror, controls, `[hidden]` guard) and the engine
      script; rename the `data-vto-*` attributes if they collide.
- [ ] Point `[data-shade-label]` / `vto:shade` dispatch at your PDP's variant UI.
- [ ] Generate the `data-vto-map` JSON from your product data.
- [ ] Gate the button on having ≥ 1 mapped shade.
- [ ] Verify: consent → ready → live; shade switch re-tints; snapshot
      downloads; compare divider drags; close kills the camera light;
      reopen skips consent.

For the fancier homepage version (category tabs, one applied product per
category rendered simultaneously — lips fill, cheek blush, lid shadow — an
applied-products tray and sequential add-all-to-cart), read
`sections/kj-virtual-try-on.liquid`; it's the same engine with more UI.
