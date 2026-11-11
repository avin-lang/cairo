# cairo

[Cairo](https://cairographics.org) 1.18 for [Avin](https://github.com/avin-lang/avin): 2D drawing on
images in memory, written out as PNG files, and on PDF and SVG documents. The package binds Cairo's
C library directly through Avin's C bindings, which check every binding against Cairo's own headers
when the program is built.

## Installing Cairo

The package finds Cairo with `pkg-config` (the modules `cairo`, `cairo-pdf` and `cairo-svg`):

- macOS: `brew install cairo`
- Debian and Ubuntu: `apt install libcairo2-dev`

## Using it

Name it in your package's `avin.hocon`:

```hocon
dependencies {
  cairo { git = "github.com/avin-lang/cairo", version = "0.1.0" }
}
```

and import from the module `dev.avinlang.cairo`.

## An example

Three translucent squares over a black cross, written to `setsourcergba.png`:

```avin
import dev.avinlang.cairo.{Argb32, CairoError, imageSurface}

draw(path: string) -> Result[unit, CairoError] =
    val surface = imageSurface(Argb32, 120, 120)?
    val cr = surface.context()?
    cr.scale(120, 120)
    cr.setSourceRgb(0, 0, 0)
    cr.moveTo(0, 0)
    cr.lineTo(1, 1)
    cr.moveTo(1, 0)
    cr.lineTo(0, 1)
    cr.setLineWidth(0.2)
    cr.stroke()
    cr.rectangle(0, 0, 0.5, 0.5)
    cr.setSourceRgba(1, 0, 0, 0.80)
    cr.fill()
    cr.rectangle(0, 0.5, 0.5, 0.5)
    cr.setSourceRgba(0, 1, 0, 0.60)
    cr.fill()
    cr.rectangle(0.5, 0, 0.5, 0.5)
    cr.setSourceRgba(0, 0, 1, 0.40)
    cr.fill()
    cr.status()?
    surface.writeToPng(path)

main()
    draw("setsourcergba.png") match
        Ok(_) -> println("wrote setsourcergba.png")
        Err(e) -> println(s"cairo: ${e.message}")
```

## What is bound

- **Surfaces**: `imageSurface(format, width, height)`, `pdfSurface(path, width, height)` and
  `svgSurface(path, width, height)`, each answering `Err` with Cairo's failure; an image surface's
  `width`, `height`, `stride` and `format`; `writeToPng`, `flush`, `finish` and `showPage`; and a PDF
  surface's `restrictToVersion`, `setSize`, `setMetadata`, `setCustomMetadata`, `setPageLabel`,
  `setThumbnailSize` and `addOutline`. `formatStrideForWidth` gives the row stride of any format.
- **Contexts**, made by `Surface.context()`: `save` and `restore`; `setSourceRgb`, `setSourceRgba`,
  `setSource` and `source`; `setLineWidth` and `lineWidth`; the path (`newPath`, `moveTo`, `lineTo`,
  `rectangle`, `arc`, `closePath`, `hasCurrentPoint`, `inFill`, `inStroke`); `stroke`, `fill` and
  `paint`, and the `_preserve` forms; `translate`, `scale` and `rotate`; text with
  `selectFontFace`, `setFontSize` and `showText`; `tagBegin` and `tagEnd` for a PDF's structure;
  `target`; and `showPage`.
- **Enumerations**, each a type of variants: `Status` (with `describe`), `Format`, `Content`,
  `Operator`, `Antialias`, `FillRule`, `LineCap`, `LineJoin`, `Extend`, `Filter`, `FontSlant` and
  `FontWeight`. A context's `setOperator`, `setAntialias`, `setFillRule`, `setLineCap` and
  `setLineJoin` take them and `operator`, `antialias`, `fillRule`, `lineCap` and `lineJoin` answer
  them; a pattern has `setExtend`, `extend`, `setFilter` and `filter`; a surface has `content`; and
  `statusKind()` answers a context's, surface's or pattern's status as a `Status`.
- **Geometry on a context**: `currentPoint`; `userToDevice`, `deviceToUser` and their `_Distance`
  forms; `fillExtents`, `strokeExtents`, `pathExtents` and `clipExtents`; `inClip`, `clip`,
  `clipPreserve` and `resetClip`; and `setDash` and `dashCount`. Cairo's `cairo_get_dash` is not
  bound: Avin's C bindings cannot express an array C fills whose length comes from another call.
- **Patterns**: `solidRgb`, `solidRgba`, `linearGradient` and `radialGradient`, with
  `addColorStopRgb` and `addColorStopRgba`.
- `version()` and `versionString()`.

Surfaces, contexts and patterns are handles on Cairo's counted objects: each is given back to Cairo
when it is collected, or at once by `close()`. A context holds its surface, and a context painting
with a pattern holds the pattern, so either lives as long as anything draws with it.

Cairo keeps a failure on the object it happened to, which then ignores what it is told; `status()`
answers it on any surface, context or pattern, the functions that make one answer it at once, and
`writeToPng` and `finish` answer it before and after they write.

## Testing

```
avin test
```

The tests write their files into temporary directories and read them back: PNG headers for the
size, depth and colour type of what was drawn, and PDF and SVG documents for their headers,
metadata, outlines, page labels and structure.
