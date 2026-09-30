# ASCIIVD

A standalone browser tool for pushing video through dither, ASCII and monochrome
overlay effects. One file, no dependencies, no server, nothing leaves the machine.

Open `index.html` in Chrome (or Edge — needed for PNG-sequence export; everything
else works in Safari/Firefox too), then drop a video on the window.

## Pipeline

Source → luma → image adjustments → warp → render mode → overlays → output canvas.

- **Image**: brightness, contrast, gamma, black/white points, saturation, blur, sharpen,
  Sobel edges (mix or edges-only), posterize, invert.
- **Render modes**
  - `dither` — Floyd–Steinberg, Atkinson, Jarvis, Stucki, Burkes, Sierra (3/2/lite),
    Bayer 2/4/8, clustered dot 4/8, blue noise, white noise, halftone-dot matrix.
    Multi-tone (2–8 levels), threshold bias, diffusion amount, serpentine scan.
  - `ascii` — shape matching picks the glyph whose pattern best fits each image
    patch rather than matching brightness alone, so edges and diagonals hold together
    (weight it back toward pure tone if you want the softer look).
    12 charsets + custom string, column count, cell ratio, glyph size,
    font, bold, flipped ramp, glyph-level dithering, edge glyphs (`- \ | /` driven
    by the Sobel angle), ink / sampled-source / duotone colouring.
  - `8-bit` — colour quantised to a hardware palette: Game Boy (and Pocket), Virtual
    Boy, NES,
    C64, ZX Spectrum, CGA (three sets), EGA, PICO-8, Master System, Apple II,
    teletext, 1-bit, or a palette derived from the frame itself by median cut (with
    a lock, so it stops shifting once you like it). Dithering happens in colour —
    ordered screens, blue noise, white noise, Floyd–Steinberg or Atkinson — so
    gradients survive four colours. Attribute clash restricts each cell to two
    entries the way a Spectrum did, and the outline pass darkens colour breaks while
    snapping back onto the palette, so no off-palette pixel ever reaches the frame.

    Palettes that are one colour at several brightnesses — Virtual Boy, both Game
    Boys, 1-bit — are detected automatically and matched on *tone* rather than by
    RGB distance, so a bright cyan reaches the top of the ramp instead of falling to
    black for want of red. Detection is geometric (a ramp is a straight line through
    colour space), which is why it catches the Game Boy's greens even though they
    drift towards yellow as they brighten.
  - `lcd` — a monochrome phone screen in the Nokia mould. The picture becomes a 1-bit
    grid of square pixels with a gap between each; lit pixels cast a soft drop shadow
    onto the glass (the liquid-crystal layer sits slightly above the reflector), the
    backlight blooms up from the bottom edge, and the glass carries a fixed grain and
    mottle so it reads as a physical screen rather than a flat fill. Pitch, gap,
    dither (threshold for hard silhouettes, Atkinson/Floyd/Bayer/blue noise for
    footage), shadow offset/blur/strength, glow, texture, and faint unlit segments.
    Screen sets: 3310 green, 5110 lime, grey, blue backlight, amber.
  - `riso` — Risograph emulation. The frame is split into 2–4 ink separations, each
    dithered on its own rotated screen, offset slightly for misregistration and
    printed semi-transparent (multiply) onto paper stock. Three ways to separate:
    `colour` fits the ink mix to the source colour by least squares in absorption
    space, `tonal build` stacks inks by tone the way a 3-colour poster is built, and
    `mono offset` prints the same plate in every ink for pure registration drift.
    Plate wobble drifts each plate frame to frame the way a real drum does, and ink
    bleed plus paper fibre fatten the dots into the stock.
    Ink sets are real Riso drum colours (fluoro pink, blue, yellow, teal, orange,
    purple, green, black) with per-ink colour pickers, plus controls for
    misregistration distance/angle, ink opacity, ink density and blend mode.
  - `halftone` — dot, square, diamond, line, cross, ring, triangle on a rotatable
    screen; shapes are area-corrected so tone reproduction matches the source.
  - `stipple` — dots placed on a fine lattice against a blue-noise threshold so
    spacing stays even, then scattered and sized by local tone: pen-and-ink rather
    than a printer's grid. Density, size, scatter, size variation and grain.
  - `contour` — topographic banding, adjustable levels, line width, fill tint.
  - `threshold`, `grayscale` — plain references.
- **A / B** — transition between the effect and the clean footage inside a single
  export, so a reveal needs one render rather than two and a switch in the edit.
  Dissolve, wipe (four directions), staggered bars, or iris, positioned anywhere in
  the clip with adjustable length and edge softness, running either way round.
- **Blocks** — draw rectangles on the canvas and the effect applies only inside them
  (or only outside, inverted), with feather and rounded corners. Drag on empty canvas
  to add one; while drawing is active every block shows a centre handle to move it
  and four corner handles to resize it, with the cursor changing over each. Click to
  select, `⌫` or double-click to remove, `esc` to finish. Blocks clamp to the frame
  and cannot be resized below a usable minimum. They are stored normalised to the
  frame, so they survive resolution changes, and they stay put when you switch looks.
- **Time** — slit-scan / time displacement: a ring of recent frames lets each row,
  column, ring or tone band be sampled from a different moment. Plus phosphor
  persistence, where trails brighten on dark stock and darken on light stock.
  Exports pre-roll this history so a clip never starts cold.
- **Smear lines** — datamosh streaks. Draw a line on the canvas and every pixel along
  it is dragged out across the frame: drag down for a vertical line that bleeds
  sideways, across for a horizontal one that bleeds up or down. Bleed right/down,
  left/up or both; ragged row ends, row band size, sample wobble, a brightness gate
  (only bright pixels streak), fade-out and a flicker rate that re-rolls the rows.
  *After effect* streaks the finished frame; *before effect* streaks the picture
  first, so the streaks are dithered, typeset or printed with everything else. Lines
  get the same move/resize handles as blocks, and stay put when you change look.
- **Highlight bleed** — blown highlights drag a burning trail to one side of the
  frame, like an overdriven sensor or tape head. Highlight level and knee, streak
  length, direction, source colour or a tint, row flicker, softness, and screen or
  additive blending.
- **Warp** (animated, driven by the video's own timecode so exports are repeatable):
  sine wave (horizontal / vertical / both / radial), radial ripple, glitch band offset.
- **Overlay**: scanlines, grain, vignette, bloom, brightness pulse, frame border.
- **Colour**: ink + paper (any two colours), duotone tint, dry/wet mix, source ghost.
  Nine one-click phosphor palettes — white, amber (P3), green (P1), blue, VT green,
  cyan, ember, Game Boy, and paper (black on cream) — plus a swap ink/paper button.
  Palettes change colour only, so they can be tried against whatever dither, ASCII
  or halftone setup you already have without disturbing it.

**Pixel size** is the chunkiness control — the frame is processed at
`width ÷ pixel size` and blown back up nearest-neighbour, so the dither stays crisp.

## Export

- **Save frame** — PNG of the current frame at full output resolution.
- **Record** — real-time capture off the canvas. Writes **MP4 (H.264)** where the
  browser's MediaRecorder supports it, WebM otherwise; keeps the original audio and
  honours the in/out points. If the render can't hold the export fps, drop the width
  or raise the pixel size.
- **Export MP4** — frame-accurate H.264. Seeks the source frame by frame, so the
  render never has to keep up with real time and nothing is dropped no matter how
  heavy the effect. Audio is re-encoded to AAC and muxed in. Needs WebCodecs
  (Chrome, Edge, recent Firefox/Zen). This is the one for finished pieces.

  Every frame is verified against the decoded source before it is encoded and
  re-grabbed if the browser had not presented it yet, so a slow seek or a heavy
  effect cannot write blank or duplicated frames into the file. Frames are also
  snapshotted through an `ImageBitmap` rather than handed to the encoder as a live
  canvas, which is what made Firefox-based browsers export near-empty video.
- **PNG sequence** — lossless frames written into a folder you pick
  (`frame_00000.png…`), for a colour-managed grade in Resolve/Premiere/FCP.

The MP4 writer is hand-rolled (single offline file, no libraries): progressive
ISO-BMFF, one chunk per track, `ctts` composition offsets so B-frame reordering
survives. AAC keeps its encoder priming, so audio can sit ~20 ms early against
picture — irrelevant for cutaways, worth checking if you're syncing to a beat.

## Presets

42 built-ins (marathon, newsprint, terminal, xerox, blueprint, topographic, ghost
wave, crt, silk, stamp, ripple, edge code, glitch tape, duotone type, photo copy,
riso 3-colour, riso poster, riso duotone, riso ghost, riso press, a/b reveal,
a/b bars, slit scan, time smear, phosphor trails, stipple ink, typeset, game boy,
nes, virtual boy, nokia 3310, nokia snake, nokia blue, datamosh bleed, scorched highlights, mosh + scorch, c64 demo, spectrum clash, pico-8, cga dos, auto 16…).

The tone controls in the IMAGE panel apply per channel as well as to luma, so colour
modes (8-bit, riso colour separation) respond to them the same way the mono ones do.
Warp and time displacement move colour too, so slit-scan works in 8-bit.
Save your own to localStorage, or export/import them as JSON to move between machines.
**Export** saves the current look; **Export saved** saves every preset in your list, and
importing that file adds them all. Saved presets belong to the address the page was opened
from, so `index.html` opened as a file and `http://localhost:1991` keep separate lists —
Export saved in one, Import in the other to carry them across.

## Keys

`space` play/pause · `←/→` step a frame · `[` `]` in/out · `r` record · `s` save frame ·
`1…9` built-in presets · `esc` stop an export or recording · double-click a slider
label to reset it.

While an export is running its own button becomes the stop control (the other export
buttons are disabled so two jobs can't overlap). Stopping is safe: a PNG sequence
keeps every frame already written to disk, and a stopped MP4 is still flushed, muxed
and saved with a `_partial` suffix rather than being thrown away.
