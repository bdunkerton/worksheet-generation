# Stamp Machine

### A single-file HTML app that stamps student names onto preschool worksheets.

_100% vibe-planned with Claude, and 99% hand coded by me!_

_Check out the [build guide](/stamp-machine-v2-build-guide.md) to see how we got here!_

## Why this exists

My mom is a preschool teacher. She hand-writes student names onto stacks of practice worksheets all the time so the kids can trace them. That's a lot of names and a lot of writing. This app does it for her!

## How it works

1. **Upload a template PDF** — the worksheet you want to stamp names onto.
2. **Draw a bounding box** — click and drag on the preview to mark where the name should go.
3. **Type a list of names** — one per line, press Enter to add.
4. **Generate** — get back a single PDF with one page per name, each name printed inside your bounding box in a light grey traceable font.

## Features

- **Light grey traceable font** — names render in Quicksand at low opacity so students can trace right over them.
- **Auto-fit** — names stretch and squeeze to fill the bounding box.
- **Memory** — remembers the names you typed and the last bounding box you drew, so you don't have to redo setup on every visit.
- **Single file** — everything (font, libraries, code) is bundled into one `stamp-machine.html`. No build step, no install.

## Usage

Open `stamp-machine.html` in a browser. That's it.

## Scope

This is intentionally narrow:

- One template PDF at a time
- One bounding box per template
- Names are stamped in the same spot on every page
- Designed for one-off teacher use, not batch automation

## Tech

- [pdf.js](https://mozilla.github.io/pdf.js/) for rendering the template preview
- [pdf-lib](https://pdf-lib.js.org/) for generating the output PDF
- [Quicksand](https://fonts.google.com/specimen/Quicksand) (base64-embedded in the HTML) for the traceable font
- SVG-based text rendering with `textLength` for natural width fitting
