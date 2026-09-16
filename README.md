<img src="./public/logo.png" alt="Demo" width="100" height="100">

# SVG/PNG Frames → Lottie / dotLottie

**[Demo: https://frames-to-lottie.pages.dev/](https://frames-to-lottie.pages.dev/)**

The core concept is that we take a ZIP file containing ordered SVG and/or PNG frames and turns those frames into a Lottie animation. SVG frames are converted into native Lottie shape layers, while PNG frames are embedded as image assets. The result can be exported either as a self-contained `.json` Lottie file or as a `.lottie` ZIP container.

## How it works

The flow is essentially:

````text
ZIP file
   │
   ▼
ingestZip()
   │
   ├── SVG → read SVG text + dimensions
   │
   └── PNG → read bytes + detect dimensions
   │
   ▼
ordered frame descriptors
   │
   ▼
buildLottie()
   │
   ├── SVG → parse <path> → Lottie shapes
   │
   └── PNG → base64 image asset + image layer
   │
   ▼
Lottie JSON
   │
   ├── .json → self-contained, PNGs inline as base64
   │
   └── buildDotLottie() → .lottie ZIP with /images/*



## 1. Ingesting the ZIP

`ingestZip()` opens the ZIP with JSZip, finds all `.svg` and `.png` files, and sorts them naturally by filename.

```js
const zip = await JSZip.loadAsync(file);

const entries = Object.values(zip.files)
  .filter((f) => !f.dir && /\.(svg|png)$/i.test(f.name))
  .sort((a, b) => naturalCompare(a.name, b.name));
````

That means filenames such as:

```text
frame1.svg
frame2.svg
frame3.svg
...
frame10.svg
```

are ordered numerically rather than lexicographically.

Each entry is converted into a common frame descriptor:

```js
{
  id: entry.name,
  type: "svg", // or "png"
  name: baseName,
  text,        // SVG only
  bytes,       // PNG only
  width,
  height,
  thumbUrl
}
```

SVG dimensions come from the SVG root:

```js
const w =
  parseFloat(
    (root.getAttribute("width") || "1000").replace(/[a-zA-Z%]/g, ""),
  ) || 1000;

const h =
  parseFloat(
    (root.getAttribute("height") || "1000").replace(/[a-zA-Z%]/g, ""),
  ) || 1000;
```

PNG dimensions are determined by decoding the image:

```js
const { width, height } = await pngDimensions(bytes);
```

A thumbnail URL is also created so the frames can be previewed in a UI.

---

## 2. Converting SVG paths into Lottie shapes

SVG frames are not stored as raster images. Instead, their `<path>` elements are converted into Lottie shape data.

The supported path commands are:

```text
M  Move
L  Line
C  Cubic Bézier
Z  Close path
```

The parser tokenizes the `d` attribute:

```js
const tokenRe = /[MLCZ]|-?\d*\.?\d+(?:[eE][-+]?\d+)?/g;
const tokens = d.match(tokenRe) || [];
```

A cubic Bézier command such as:

```text
C c1x c1y c2x c2y x y
```

becomes Lottie vertex and tangent information:

```js
cur.out[cur.out.length - 1] = [c1x - prev[0], c1y - prev[1]];

cur.verts.push([ex, ey]);

cur.in.push([c2x - ex, c2y - ey]);
```

So an SVG curve is represented by:

```js
{
  c: sp.closed,
  i: sp.in,
  o: sp.out,
  v: sp.verts
}
```

where:

- `v` = vertices
- `i` = incoming Bézier handles
- `o` = outgoing Bézier handles
- `c` = whether the path is closed

Each SVG path is then wrapped in a Lottie shape group:

```js
{
  ty: "gr",
  it: [
    {
      ty: "sh",
      ks: {
        a: 0,
        k: {
          c: sp.closed,
          i: sp.in,
          o: sp.out,
          v: sp.verts
        }
      }
    },

    // fill / stroke / transform...
  ]
}
```

The parser intentionally handles a restricted SVG subset. Arc and quadratic commands are not supported:

```text
A  Arc
Q  Quadratic Bézier
T  Smooth quadratic
```

Those paths need to be converted to cubic Bézier paths before ingestion.

---

## 3. Preserving SVG paint properties

For each path, the converter reads basic SVG styling:

```js
const fill = el.getAttribute("fill");
const stroke = el.getAttribute("stroke");
const strokeWidth = el.getAttribute("stroke-width");
```

An SVG fill becomes a Lottie fill:

```js
{
  ty: "fl",
  c: { a: 0, k: [...hexToRgb1(fill), 1] },
  o: { a: 0, k: 100 },
  r: lottieFillRule
}
```

Colors are converted from hexadecimal SVG colors into Lottie's normalized RGB format:

```js
function hexToRgb1(hex) {
  let h = hex.replace("#", "");

  if (h.length === 3) {
    h = h
      .split("")
      .map((c) => c + c)
      .join("");
  }

  return [
    parseInt(h.substr(0, 2), 16) / 255,
    parseInt(h.substr(2, 2), 16) / 255,
    parseInt(h.substr(4, 2), 16) / 255,
  ];
}
```

A stroke becomes:

```js
{
  ty: "st",
  c: { a: 0, k: [...hexToRgb1(stroke), 1] },
  o: { a: 0, k: 100 },
  w: { a: 0, k: parseFloat(strokeWidth) || 1 }
}
```

The code also maps SVG fill rules:

```js
const lottieFillRule = fillRule === "evenodd" ? 2 : 1;
```

---

## 4. Building the animation timeline

`buildLottie()` treats every input frame as a fixed-duration animation frame.

If:

```js
frames.length === 10;
hold === 3;
```

then the animation has:

```js
const totalFrames = frames.length * hold;
```

So every source frame occupies three Lottie frames.

Each layer gets opacity keyframes:

```js
const opKeyframes = [];

if (start > 0) {
  opKeyframes.push({ t: 0, s: [0], h: 1 });
}

opKeyframes.push({ t: start, s: [100], h: 1 });
opKeyframes.push({ t: end, s: [0], h: 1 });
```

Conceptually:

```text
opacity
100% ────────┐
             │
             └──────── 0%
       start       end
```

The `h: 1` values make those opacity changes hold rather than interpolate smoothly.

The animation's basic timing information is:

```js
{
  fr,              // frame rate
  ip: 0,           // in point
  op: totalFrames, // out point
  w: size,
  h: size
}
```

---

## 5. SVG frames become vector layers

For an SVG frame, the code parses its paths:

```js
const { shapeItems, width } = svgToShapeItems(frame.text);
```

Then scales the SVG to the requested animation size:

```js
const scale = size / width;
```

The result is a Lottie shape layer:

```js
{
  ty: 4,
  nm: frame.name,
  shapes: shapeItems,

  ks: {
    ...baseKs,

    p: { a: 0, k: [0, 0, 0] },

    a: { a: 0, k: [0, 0, 0] },

    s: {
      a: 0,
      k: [scale * 100, scale * 100, 100]
    }
  }
}
```

`ty: 4` is Lottie's shape-layer type.

Because the SVG is converted into paths, the resulting animation remains vector-based rather than containing a rasterized screenshot of the SVG.

---

## 6. PNG frames become image layers

PNG frames take a different path.

Their raw bytes are converted to Base64:

```js
function arrayBufferToBase64(buf) {
  let binary = "";
  const bytes = new Uint8Array(buf);
  const chunk = 0x8000;

  for (let i = 0; i < bytes.length; i += chunk) {
    binary += String.fromCharCode.apply(null, bytes.subarray(i, i + chunk));
  }

  return btoa(binary);
}
```

The PNG is then registered as a Lottie image asset:

```js
assets.push({
  id: assetId,
  w: frame.width,
  h: frame.height,
  u: "",
  p: "data:image/png;base64," + b64,
  e: 1,
});
```

The actual animation layer references that asset:

```js
{
  ty: 2,
  refId: assetId,

  ks: {
    ...baseKs,

    p: {
      a: 0,
      k: [size / 2, size / 2, 0]
    },

    a: {
      a: 0,
      k: [
        frame.width / 2,
        frame.height / 2,
        0
      ]
    },

    s: {
      a: 0,
      k: [scale * 100, scale * 100, 100]
    }
  }
}
```

`ty: 2` is Lottie's image-layer type.

---

## 7. Why the SVG and PNG paths are different

The important distinction is:

```text
SVG
 └── <path>
      └── Lottie vector shape

PNG
 └── image bytes
      └── Lottie image asset
```

SVG data becomes part of the `shapes` array directly.

PNG data cannot become Lottie vector geometry, so it is stored as an image asset and referenced with `refId`.

---

## 8. Generating the self-contained Lottie JSON

`buildLottie()` returns a normal Lottie JSON object:

```js
return {
  v: "5.9.6",
  fr,
  ip: 0,
  op: totalFrames,
  w: size,
  h: size,
  nm: name,
  ddd: 0,
  assets,
  layers,
};
```

For PNG frames, the image data is embedded directly inside the JSON:

```text
data:image/png;base64,...
```

This makes the JSON completely self-contained: there are no external image files to resolve.

The trade-off is that Base64 increases the size of the JSON.

---

## 9. Converting the Lottie JSON into `.lottie`

`buildDotLottie()` creates a ZIP archive using JSZip:

```js
const zip = new JSZip();
```

Before writing the animation, embedded PNG assets are extracted from their Base64 data and written as actual files:

```js
const imagesFolder = zip.folder("images");

imagesFolder.file(filename, b64, { base64: true });
```

The animation asset is then rewritten from:

```js
{
  id: "img_0",
  p: "data:image/png;base64,...",
  e: 1
}
```

to something like:

```js
{
  id: "img_0",
  w: 1920,
  h: 1920,
  u: "images/",
  p: "img_0.png",
  e: 0
}
```

So the `.lottie` archive contains roughly:

```text
animation.lottie
├── manifest.json
├── animations/
│   └── animation-name.json
└── images/
    ├── img_0.png
    ├── img_1.png
    └── ...
```

The manifest identifies the animation:

```js
{
  version: "1",
  generator: "svg-lottie-studio",
  animations: [
    {
      id: name,
      speed: 1,
      loop: true
    }
  ]
}
```

---

## 10. Overall data flow

The whole system can be summarized as:

```js
const frames = await ingestZip(file);

const lottie = buildLottie(frames, "my-animation", {
  fr: 30,
  hold: 2,
  size: 512,
});

const dotLottieBlob = await buildDotLottie(lottie, "my-animation");
```

The transformation is:

```text
ZIP
 │
 ├── frame01.svg
 ├── frame02.svg
 ├── frame03.png
 └── frame04.png
       │
       ▼
Frame descriptors
       │
       ▼
┌─────────────────────────────┐
│ SVG                         │
│ XML → paths → Lottie shapes │
└─────────────────────────────┘
       │
       ├──────────────────────┐
       ▼                      ▼
 Vector layer             Image layer
       │                      │
       └──────────┬───────────┘
                  ▼
              Lottie JSON
                  │
          ┌───────┴────────┐
          ▼                ▼
      .json             .lottie
   Base64 PNGs       PNG files in ZIP
```

## Key limitation

The SVG parser is deliberately simple. It understands absolute `M`, `L`, `C`, and `Z` path commands and basic `fill`, `stroke`, `stroke-width`, and `fill-rule` attributes. It does not implement the full SVG specification.

In particular:

```text
A / a  → arcs
Q / q  → quadratic Bézier curves
T / t  → smooth quadratic curves
```

are not handled by `parsePathD()` and should be converted to cubic Bézier paths before passing the SVG into this pipeline.

The result is therefore best thought of as a **frame-sequence converter for path-based SVGs and PNGs**, rather than a complete SVG-to-Lottie renderer.
