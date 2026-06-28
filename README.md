# Stamp Machine
https://github.com/user-attachments/assets/526f7bc6-dd6f-4781-a964-628a88630c86

### A single-file HTML app that stamps student names onto preschool worksheets.

_100% vibe-planned with Claude, and 99% hand coded by me!_

_Check out the [build guide](/stamp-machine-v2-build-guide.md) to see how we got here!_

## Why this exists

My mom is a preschool teacher. She hand-writes student names onto stacks of practice worksheets all the time so her students can trace them. That's a lot of names and a lot of writing. This app does it for her!

## How it works

1. **Upload a template PDF** — the worksheet you want to stamp names onto.
2. **Draw a bounding box** — click and drag on the preview to mark where the name should go.
3. **Type a list of names** — add the names of your students!
4. **Generate** — the stamper creates an SVG of each name stamped into the bounding box area.

## Features

- **Light grey traceable font** — names render in Google's [Quicksand font](https://fonts.google.com/specimen/Quicksand?preview.script=Latn) at low opacity so students can trace right over them.
- **Auto-fit** — the magic is using an SVG to make sure any length of name fits into the bounding box.
- **Memory** — remembers the names you typed and the last bounding box you drew, so you don't have to redo setup on every visit as you upload new templates.
- **Single file** — I just think it's cool! A fun way to solve the problem.
## Tech

- [pdf.js](https://mozilla.github.io/pdf.js/) for rendering the template preview
- [pdf-lib](https://pdf-lib.js.org/) for generating the output PDF
- [Quicksand](https://fonts.google.com/specimen/Quicksand) (base64-embedded in the HTML) for the traceable font
- SVG-based text rendering with `textLength` for natural width fitting

## Usage
Open `stamp-machine.html` in a browser. That's it!
