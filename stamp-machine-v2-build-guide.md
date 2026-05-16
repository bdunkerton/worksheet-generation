# The Stamp Machine v2 — Build Guide

---

## Engineering Assistant Prompt

> You are a senior web engineer with 20+ years of experience building production software. You have deep expertise in vanilla JavaScript, browser APIs, PDF manipulation, and canvas rendering. You have strong opinions, you know the modern web platform well, and you prefer using native browser APIs over libraries wherever they're a better fit. You know the tricks — the ones that only come from having debugged the same class of problem a dozen times. Before you make suggestions you always search the web to understand what's most current!
>
> Your role is **teacher and guide, not implementer.** When I get stuck, help me understand *why* something works the way it does, not just what to type. Use concrete examples to illustrate concepts. Point me toward the right API or pattern, explain the gotcha I'm about to hit, and let me write the code myself. If I'm about to make a structural mistake, tell me before I've written 200 lines I'll need to delete.
>
> The project is **The Stamp Machine** — a single HTML file (no build tools, no framework, no backend) that lets a preschool teacher stamp student names in a dotted tracing font onto scanned worksheet PDFs. The architecture is already decided; your job is to help me execute it cleanly.
>
> Key constraints to keep in mind at all times:
> - Single `.html` file. No bundler. No npm. CDN dependencies only.
> - Vanilla JS with a minimal hand-rolled `el()` DOM helper (see Phase 0).
> - State is a plain object. One `render()` function redraws the UI from state.
> - `localStorage` only — no IndexedDB, no server.
> - The rendering pipeline must be DPI-aware and retina-correct.
> - Box coordinates are always stored as percentages of PDF page dimensions.
> - Names are rendered as SVG with `preserveAspectRatio="none"` so they stretch to fill the box regardless of length.

---

## Architecture Overview

### The one function everything depends on

```
renderNameToCanvas(canvas, name, fontFamily) → void
```

This draws an SVG-stretched name onto any canvas element. The preview and the PDF pipeline both call this exact function. It is written first, tested first, and never duplicated.

### State shape

```js
const state = {
  // Persisted to localStorage
  box: null,          // { xPct, yPct, wPct, hPct } — % of PDF page dimensions
  names: [],          // string[] — normalized, uppercase

  // Session only
  templatePdf: null,  // ArrayBuffer
  batchPdfs: [],      // ArrayBuffer[]
  fontBytes: null,    // ArrayBuffer

  // UI
  phase: "setup",     // "setup" | "ready" | "generating"
  progress: null,     // { current, total, label } | null
};
```

No per-name state. No locked flags. No x/y per name. If you find yourself adding per-name properties, stop and reconsider.

### Library dependencies (CDN)

```html
<!-- PDF manipulation -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf-lib/1.17.1/pdf-lib.min.js"></script>
<!-- Font embedding support for pdf-lib -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/fontkit/1.8.1/fontkit.min.js"></script>
<!-- PDF preview rendering -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
```

pdf.js also requires its worker:
```js
pdfjsLib.GlobalWorkerOptions.workerSrc =
  "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js";
```

### Rendering pipeline sequence (locked — do not reorder)

```
1. PDFDocument.load(existingPdfBytes)          // load source worksheet page
2. pdfDoc.registerFontkit(fontkit)             // MUST happen before embedFont
3. pdfDoc.embedFont(fontBytes)                 // returns PDFFont handle
4. renderNameToCanvas(offscreenCanvas, name, fontFamily)
5. offscreenCanvas.toBlob() → PNG bytes
6. pdfDoc.embedPng(pngBytes)
7. page.drawImage(image, { x, y, width, height })  // coordinates in PDF points
8. pdfDoc.save() → Uint8Array
```

Step 2 before 3 is mandatory. fontkit must be registered before any font is embedded or you get silent failure.

---

## Build Phases

- [ ] Phase 0 — Scaffold
- [ ] Phase 1 — Name Renderer
- [ ] Phase 2 — Font Loading
- [ ] Phase 3 — Template PDF Preview + Box Drawing
- [ ] Phase 4 — Name Input
- [ ] Phase 5 — PDF Generation Pipeline
- [ ] Phase 6 — Phase Gating & UI Flow
- [ ] Phase 7 — Design

---

### Phase 0 — Scaffold

**Goal:** A working HTML file with state, the `el()` helper, and a visible render loop. No PDF logic yet.

**What to build:**

Create `stamp-machine.html` with:

1. HTML boilerplate with a `<div id="app"></div>` mount point
2. Three CDN `<script>` tags (pdf-lib, fontkit, pdf.js) in `<head>`
3. A `<style>` block with CSS custom properties (design tokens). Start simple — you'll finalize design in the last phase. Define at minimum: `--bg`, `--ink`, `--muted`, `--line`, `--accent`
4. The `el(tag, props, ...kids)` helper — this is your entire "framework." It creates DOM elements, attaches event listeners, sets attributes. V1's implementation is a solid reference.
5. A `state` object matching the shape above
6. A `render()` function that reads `state.phase` and renders different content into `#app`
7. A `boot()` async IIFE that loads persisted state from localStorage, then calls `render()`

**localStorage keys to use:**
- `sm_box` — JSON string of `{ xPct, yPct, wPct, hPct }` or absent
- `sm_names` — JSON array of name strings or absent
- `sm_font` — base64 string of font bytes or absent

**Checkpoint:** 
- [ ] Open the file in a browser — see placeholder UI
- [ ] Verify localStorage persistence (set `sm_names`, reload, verify data)
- [ ] Clear and reload to confirm empty state works

---

### Phase 1 — Name Renderer (`renderNameToCanvas`)

**Goal:** A single function that draws a stretched, font-rendered name onto a canvas. This is the heart of the whole tool.

**What to build:**

```js
function renderNameToCanvas(canvas, name, fontFamilyName) {
  const ctx = canvas.getContext("2d");
  const { width, height } = canvas;

  // 1. Clear
  ctx.clearRect(0, 0, width, height);

  // 2. Build an SVG string with the name as a <text> element
  //    viewBox matches canvas dimensions exactly
  //    preserveAspectRatio="none" is the key — it stretches independently in X and Y
  const svg = `<svg xmlns="http://www.w3.org/2000/svg"
    viewBox="0 0 ${width} ${height}"
    preserveAspectRatio="none"
    width="${width}" height="${height}">
    <text
      x="50%" y="50%"
      dominant-baseline="middle"
      text-anchor="middle"
      font-family="${fontFamilyName}"
      font-size="${height * 0.85}"
      fill="#222">
      ${name}
    </text>
  </svg>`;

  // 3. Blob → object URL → Image → drawImage onto canvas
  const blob = new Blob([svg], { type: "image/svg+xml" });
  const url = URL.createObjectURL(blob);
  const img = new Image();
  img.onload = () => {
    ctx.drawImage(img, 0, 0, width, height);
    URL.revokeObjectURL(url);
  };
  img.src = url;
}
```

**Important things to understand before writing this:**
- The SVG `viewBox` and the canvas `width`/`height` must match or the stretch math is wrong
- `preserveAspectRatio="none"` on the SVG element is what makes "AJ" and "BARTHOLOMEW" fill the same box — ask your assistant to explain what this attribute actually does if it's not clear
- `font-size` here doesn't need to be precise — the SVG viewBox + stretch handles fitting. Set it to ~85% of height as a starting point.
- The font must be loaded as a web font (via `@font-face` in CSS) for the SVG to render it correctly. Ask your assistant about the `FontFace` API for loading fonts dynamically from an ArrayBuffer.

**Checkpoint:**
- [ ] Add test canvas and verify both "AJ" and "BARTHOLOMEW" fill the same width
- [ ] Confirm `preserveAspectRatio="none"` is on the `<svg>` element
- [ ] Remove the test canvas

---

### Phase 2 — Font Loading

**Goal:** Load the font from localStorage on boot, or prompt the user to upload it. Register it as a web font so Phase 1's canvas renderer can use it.

**What to build:**

1. On `boot()`: check `localStorage.getItem("sm_font")`. If present, decode base64 → ArrayBuffer, register as web font, set `state.fontBytes`.
2. A `loadFont(arrayBuffer)` function that:
   - Registers the font using the `FontFace` API: `new FontFace("KGPrimaryDots", arrayBuffer)` → `font.load()` → `document.fonts.add(font)`
   - Converts the ArrayBuffer to base64 and saves to `localStorage.setItem("sm_font", base64)`
   - Sets `state.fontBytes = arrayBuffer`
   - Calls `render()`
3. A file picker trigger (a plain `<input type="file" accept=".ttf">` hidden off-screen, `.click()`'d programmatically) for the upload button
4. In `render()`: if `!state.fontBytes`, show a "Load font" prompt panel. If `state.fontBytes`, don't show it.

**Things to understand:**
- `ArrayBuffer` → base64: use `btoa(String.fromCharCode(...new Uint8Array(buffer)))`. Ask your assistant about the chunking trick for large buffers — naive spread will stack overflow on large fonts.
- base64 → `ArrayBuffer`: reverse of the above. Ask if you need the pattern.
- The `FontFace` API is the modern way to load fonts dynamically. Ask your assistant to show you how it differs from `@font-face` in CSS.

**Checkpoint:**
- [ ] Load fresh — see font upload prompt
- [ ] Upload `.ttf` file and verify it loads
- [ ] Reload — prompt gone, font loaded from localStorage
- [ ] Verify `sm_font` in localStorage contains base64 string

---

### Phase 3 — Template PDF Preview + Box Drawing

**Goal:** Upload a template PDF, render it as a preview using pdf.js, and let the user drag-draw a rectangle to define the name box. Save box as percentages. Show the saved box as an overlay on reload.

**What to build:**

This is the most complex phase. Build it in three sub-steps.

#### 3a — PDF Upload and Preview Render

1. A file picker for PDF upload. On select: read as ArrayBuffer, store in `state.templatePdf` (session only — do not persist the PDF itself).
2. After loading, use pdf.js to render page 1 to a canvas:
   ```js
   const pdfDoc = await pdfjsLib.getDocument({ data: arrayBuffer }).promise;
   const page = await pdfDoc.getPage(1);
   const PREVIEW_SCALE = 1.5;
   const viewport = page.getViewport({ scale: PREVIEW_SCALE });
   canvas.width = viewport.width;
   canvas.height = viewport.height;
   await page.render({ canvasContext: ctx, viewport }).promise;
   ```
3. Store `state.previewScale = PREVIEW_SCALE` and `state.pdfPageWidth/Height` (from `page.getViewport({ scale: 1 })`).

#### 3b — Box Drawing Interaction

Add mouse event listeners to the preview canvas. On `mousedown`: record start. On `mousemove`: redraw the PDF render + a semi-transparent rectangle. On `mouseup`: finalize.

**The critical coordinate math — get this right:**
```js
function canvasToPdfPoints(canvasX, canvasY) {
  const dpr = window.devicePixelRatio || 1;
  return {
    x: canvasX / (state.previewScale * dpr),
    y: canvasY / (state.previewScale * dpr),
  };
}
```

On mouseup, convert both corners to PDF points, then to percentages:
```js
const x1 = Math.min(startPdf.x, endPdf.x);
const y1 = Math.min(startPdf.y, endPdf.y);
const w  = Math.abs(endPdf.x - startPdf.x);
const h  = Math.abs(endPdf.y - startPdf.y);

state.box = {
  xPct: x1 / state.pdfPageWidth,
  yPct: y1 / state.pdfPageHeight,
  wPct: w  / state.pdfPageWidth,
  hPct: h  / state.pdfPageHeight,
};
localStorage.setItem("sm_box", JSON.stringify(state.box));
```

#### 3c — Box Overlay on Reload

In `boot()`: if `sm_box` is in localStorage and `state.templatePdf` is loaded, draw the saved box overlay on top of the rendered PDF preview. This is just:
```js
const x = state.box.xPct * viewportWidth;
const y = state.box.yPct * viewportHeight;
const w = state.box.wPct * viewportWidth;
const h = state.box.hPct * viewportHeight;
ctx.strokeStyle = "rgba(255, 80, 0, 0.9)";
ctx.lineWidth = 2;
ctx.strokeRect(x, y, w, h);
```

**Checkpoint:**
- [ ] Upload template PDF and see it rendered as preview
- [ ] Draw a box over the name area
- [ ] Reload — box persists in same position
- [ ] Resize browser window — box remains accurate
- [ ] Draw a new box and verify old one is replaced

---

### Phase 4 — Name Input

**Goal:** A textarea where the teacher pastes a list of names. Names are parsed, normalized, deduplicated, and displayed as a list. Saved to localStorage.

**What to build:**

1. A `<textarea>` for paste input — one name per line. On input/paste event, call `parseNames(text)`.
2. `parseNames(rawText)` — returns a sorted, deduplicated array of normalized names:
   ```js
   function parseNames(raw) {
     return [...new Set(
       raw.split(/[\n,]+/)
          .map(s => s.trim().toUpperCase())
          .filter(s => /^[A-Z][A-Z\-\']*[A-Z]$|^[A-Z]$/.test(s))
     )].sort();
   }
   ```
   Note the regex allows hyphens (Mary-Kate) and apostrophes (O'Brien). Reject anything else silently.
3. After parsing, set `state.names` and save to `localStorage.setItem("sm_names", JSON.stringify(state.names))`.
4. Display the parsed names as a simple list below the textarea. Each entry shows the name and a remove button.
5. A "Clear all" option.

**Checkpoint:** Paste this into the textarea:
```
Ava
bartholomew
AJ
Mary-Kate
o'brien
123invalid
Ava
```
Expected result: `["AJ", "AVA", "BARTHOLOMEW", "MARY-KATE", "O'BRIEN"]` — sorted, deduplicated, normalized, invalid entry dropped.

**Checkpoint:**
- [ ] Verify test input parses correctly (sorted, deduplicated, normalized, invalid dropped)
- [ ] Reload — names persist from localStorage
- [ ] Remove a name, reload — it stays removed

---

### Phase 5 — PDF Generation Pipeline

**Goal:** Upload N batch PDFs, click Generate, download one merged PDF with all worksheets stamped for all students (student-first ordering).

**What to build:**

#### 5a — Batch PDF Upload

A multi-file picker (`<input type="file" accept=".pdf" multiple>`). On select, read all files as ArrayBuffers into `state.batchPdfs`. Display file names and count. Session only — do not persist batch PDFs.

#### 5b — The stamp function

This is the core pipeline. Write it as a standalone async function:

```js
async function stampName(sourcePdfBytes, name, box, fontBytes) {
  const { PDFDocument } = PDFLib;

  // Load the source worksheet
  const pdfDoc = await PDFDocument.load(sourcePdfBytes);

  // Register fontkit BEFORE embedding font
  pdfDoc.registerFontkit(fontkit);
  const font = await pdfDoc.embedFont(fontBytes);  // custom font

  const pages = pdfDoc.getPages();
  for (const page of pages) {
    const { width, height } = page.getSize();

    // Convert percentage box to PDF points
    const x = box.xPct * width;
    const y = box.yPct * height;  // NOTE: pdf-lib Y is bottom-up — see below
    const w = box.wPct * width;
    const h = box.hPct * height;

    // Rasterize name to offscreen canvas
    const canvas = document.createElement("canvas");
    canvas.width  = Math.round(w * 3);   // 3x for ~216 DPI
    canvas.height = Math.round(h * 3);
    await renderNameToCanvas(canvas, name, "KGPrimaryDots");

    // Canvas → PNG bytes
    const pngBytes = await new Promise(resolve => {
      canvas.toBlob(blob => blob.arrayBuffer().then(resolve), "image/png");
    });

    // Embed and draw
    const img = await pdfDoc.embedPng(pngBytes);

    // IMPORTANT: pdf-lib uses bottom-left origin. Convert Y:
    const pdfY = height - y - h;

    page.drawImage(img, { x, y: pdfY, width: w, height: h });
  }

  return pdfDoc.save();
}
```

**Critical thing to understand — pdf-lib's coordinate system:** pdf-lib uses PDF's native coordinate system where Y=0 is the *bottom* of the page. Your box coordinates (from pdf.js, which uses top-left origin) need to be flipped: `pdfY = pageHeight - boxY - boxHeight`. Ask your assistant to draw you a diagram of this if it's not clicking.

#### 5c — The generation loop

```js
async function generate() {
  const merged = await PDFLib.PDFDocument.create();
  const total = state.names.length * state.batchPdfs.length;
  let current = 0;

  for (const name of state.names) {
    for (const pdfBytes of state.batchPdfs) {
      state.progress = { current, total, label: `${name}` };
      render();
      // yield to browser so progress UI updates
      await new Promise(r => setTimeout(r, 0));

      const stamped = await stampName(pdfBytes, name, state.box, state.fontBytes);
      const donor = await PDFLib.PDFDocument.load(stamped);
      const copiedPages = await merged.copyPages(donor, donor.getPageIndices());
      copiedPages.forEach(p => merged.addPage(p));
      current++;
    }
  }

  const bytes = await merged.save();
  downloadPdf(bytes, "worksheets.pdf");
  state.progress = null;
  render();
}
```

#### 5d — Download helper

```js
function downloadPdf(bytes, filename) {
  const blob = new Blob([bytes], { type: "application/pdf" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}
```

**Checkpoint:**
- [ ] Upload 2 batch PDFs and set 3 names ("AJ", "BARTHOLOMEW", and one other)
- [ ] Hit Generate and verify download completes
- [ ] Downloaded PDF has 6 pages in correct order (name-first ordering)
- [ ] Names are different widths but fill same box area
- [ ] Zoom in — dot tracing font is clean at print scale

---

### Phase 6 — Phase Gating & UI Flow

**Goal:** The UI shows the right thing at each stage. The Generate button is only available when all prerequisites are met.

**What to build:**

Derive a readiness state from `state`:

```js
function getReadiness() {
  return {
    hasFont:    !!state.fontBytes,
    hasBox:     !!state.box,
    hasNames:   state.names.length > 0,
    hasBatch:   state.batchPdfs.length > 0,
    canGenerate: !!state.fontBytes && !!state.box &&
                 state.names.length > 0 && state.batchPdfs.length > 0,
  };
}
```

Use this in `render()` to:
- Show a checklist of what's ready / not ready
- Enable/disable the Generate button
- Show progress UI during generation

**Checkpoint:**
- [ ] Missing font — Generate button disabled, UI shows what's missing
- [ ] Missing box — Generate button disabled, UI shows what's missing
- [ ] No names — Generate button disabled, UI shows what's missing
- [ ] No batch files — Generate button disabled, UI shows what's missing
- [ ] Fill everything in — Generate button enables

---

### Phase 7 — Design

**Goal:** Apply the editorial / print-inspired aesthetic. Black and white. Typography-forward. The tool should feel like the thing it makes.

**Do this last.** The pipeline must be proven before you spend time on visual polish.

**Design direction:**
- Monochrome palette: off-white paper background, near-black ink, no color except a single functional accent (used only on the active/enabled Generate button)
- Typography: find a single typeface that feels like it belongs on a printed form or worksheet header. Not a sans-serif. Consider something with a slightly mechanical or stencil quality.
- Layout: generous whitespace, clear vertical rhythm, no cards or shadows — borders only, like printed form fields
- The preview canvas area should feel like a lightbox or document viewer — dark surround, white page
- Progress state: minimal, text-only, no animated spinners

**Checkpoint:**
- [ ] Apply monochrome + typography-focused design
- [ ] Preview canvas has dark surround (lightbox effect)
- [ ] Show to someone unfamiliar — do they say "printing/school supply" first?
- [ ] Polish and refine based on feedback

---

## Cross-Cutting Notes

### The `renderNameToCanvas` async timing issue

The SVG-to-canvas path (Blob → object URL → Image → drawImage) is asynchronous. Your Phase 1 implementation uses `img.onload`. In the pipeline (Phase 5), you need to `await` this — wrap it in a Promise:

```js
function renderNameToCanvas(canvas, name, fontFamilyName) {
  return new Promise((resolve) => {
    // ... build svg, blob, url ...
    img.onload = () => {
      ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
      URL.revokeObjectURL(url);
      resolve();
    };
    img.src = url;
  });
}
```

Return the Promise from the start. Both callers (preview and pipeline) can then `await` it.

### Page dimension mismatch handling

If a batch PDF has different dimensions than the template, the percentage-based coordinates will still land in the right *proportional* location. This is the main benefit of percentage storage — no special handling required. Log a console warning if dimensions differ by more than 5%, but proceed.

### Error handling philosophy

Wrap the entire `generate()` call in try/catch. On error: show the error message (including which name/file failed), offer a Retry button. Don't silently swallow errors — the teacher needs to know if something went wrong.

---

## File Structure (single file, sectioned with comments)

```
stamp-machine.html
├── <head>
│   ├── CDN scripts (pdf-lib, fontkit, pdf.js)
│   └── <style> (CSS custom properties + all styles)
└── <body>
    ├── <div id="app"></div>
    └── <script>
        ├── // === CONSTANTS ===
        ├── // === STATE ===
        ├── // === LOCALSTORAGE ===   (load/save helpers)
        ├── // === FONT ===           (loadFont, registerWebFont)
        ├── // === RENDERER ===       (renderNameToCanvas)
        ├── // === PDF PIPELINE ===   (stampName, generate, downloadPdf)
        ├── // === UI HELPERS ===     (el, getReadiness)
        ├── // === RENDER ===         (render + all renderX functions)
        └── // === BOOT ===           (boot IIFE)
```

Keep these sections in order. When a function's location is ambiguous, ask: is it about data, or about display? Data goes above UI HELPERS. Display goes below.
