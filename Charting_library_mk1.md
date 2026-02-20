# Charting Library Specification (MK1)

## 1. Introduction

### Purpose

This specification describes a **2D charting library** with a clean separation between the charting API and the rendering backend. It is derived from the `csplotlib` reference implementation written in C# (.NET 2.0) and is intended to be reimplemented in any modern language or framework.

Target languages include but are not limited to:
- Python (matplotlib-backend, Cairo, Qt, Skia)
- TypeScript / JavaScript (Canvas API, WebGL, SVG)
- Rust (wgpu, Skia, tiny-skia)
- Go (Gio, Ebiten, image/draw)
- Swift (CoreGraphics, Metal)
- Kotlin / Java (AWT, Compose, Skia)
- C++ (OpenGL, Qt, Cairo)
- Gleam / Elixir (via NIF or browser)

### Design Goals

1. **Renderer-agnostic API**: All chart logic calls only abstract drawing primitives. No renderer types leak into chart code.
2. **Composable layers**: Multiple chart layers share a single surface and coordinate system.
3. **Coordinate system independence**: All chart logic works in data-space units. Pixel math is centralized in one place.
4. **Interactive**: Mouse events (click, move, drag, wheel) are routed to layers for selection, snapping, and drag-and-drop.
5. **Extensible chart types**: New chart types are added by implementing a single layer interface.

### Non-Goals

- No built-in animation system (layers can trigger redraws manually).
- No data binding or reactive state management.
- No serialization or persistence of chart data.
- No specific rendering backend is required; the GDI+ implementation serves as the reference only.

---

## 2. Scope

### In scope

- All core data types (Coordinate, VColor, VGradient, VRectangle).
- The coordinate system model and its viewport transforms.
- The `IDrawEngine` rendering contract and `BaseDrawEngine` default implementation.
- The `IDrawSurface` surface contract and its paint lifecycle.
- The `ILayer` layer contract and `BaseLayer` coordinate transformation math.
- The `Line` data model and all `LineLayer` / `AdvLineLayer` behavior.
- The `GanttLayer` with `GanttLine`, `GanttBar`, and `VRange`.
- The `ChartLegend` and `LegendItem` system.
- The `BalancedTree` point-lookup structure.
- The GDI+ reference implementation as a concrete mapping guide.

### Out of scope

- OS-level windowing or event loop integration (platform-specific).
- Font metrics and text measurement (delegated to the rendering backend).
- Networking, threading, or animation frameworks.
- Guaranteeing pixel-identical output across renderers (only identical observable semantics).

---

## 3. Type System

### 3.1 Enumerations

All enumerations are renderer-independent.

```text
VFontStyle:
  Normal | Bold | Italic | BoldItalic
  Underline | BoldUnderline | ItalicUnderline | BoldItalicUnderline

VDirection:
  Up | Down | Left | Right

VLineStyle:
  Solid | Dash | Dotted | HalfDotted

VLabelMode:
  ByStep        -- label spacing is a fixed coordinate step
  ByNumber      -- a fixed count of labels is distributed across the range

VPointLocation:
  Smaller | Inside | Greater

VLegendPosition:
  Top | Bottom | Left | Right

VMarkStyle:
  Circle | Box

TransformType:
  ShiftLeft | ShiftRight | MoveUp | MoveDown
  ZoomIn | ZoomOut
  StretchX | CompressX | StretchY | CompressY

UserMarkStyle:
  Dot | Line | Both

VDownsampleMode:
  None        -- draw all visible points (default; existing behaviour)
  PixelExact  -- deduplicate by screen pixel column: skip points that map to
                 the same integer screenX as the previously drawn point.
                 At most pixelWidth draw calls per line regardless of data size.
  MinMax      -- for each pixel column collect all points that land there,
                 then draw the one with the minimum Y and the one with the
                 maximum Y. Preserves peaks and troughs. Best for signals /
                 waveforms where extremes must not be lost.
  LTTB        -- Largest-Triangle-Three-Buckets offline downsampling.
                 Reduces points to at most lttbTarget before the render loop.
                 Preserves the visual shape of the series better than uniform
                 subsampling. See §15.6.
```

### 3.2 Coordinate

A 2D point in either data-space (unit) or pixel-space (screen), identified by context.

```text
Coordinate:
  X: float
  Y: float

  equals(other): bool   -- true iff X == other.X AND Y == other.Y
```

### 3.3 VRectangle

An axis-aligned rectangle defined by two corner coordinates.

```text
VRectangle:
  TopLeft:    Coordinate    -- (minX, minY) in pixel space
  LowerRight: Coordinate    -- (maxX, maxY) in pixel space

  constructor(x1, y1, x2, y2)
  constructor(topLeft: Coordinate, lowerRight: Coordinate)
```

### 3.4 VColor

A renderer-independent color using normalized floating-point channels.

```text
VColor:
  R: float    -- [0.0, 1.0]
  G: float    -- [0.0, 1.0]
  B: float    -- [0.0, 1.0]

  toSimpleGradient(): VGradient
    -- Returns VGradient(R, G, B, 1.0, 1.0, 1.0)

  static fromString(name: string): VColor
    -- Case-insensitive lookup. Supported names:
    --   red, green, blue, black, white, orange, yellow, lightyellow,
    --   pink, royalblue, slateblue, darkviolet
    -- Unknown names fall back to blue.

  -- Named color factories (exact RGB values below):
  static Red():        VColor(1.0, 0.0, 0.0)
  static Green():      VColor(0.0, 1.0, 0.0)
  static Blue():       VColor(0.0, 0.0, 1.0)
  static Black():      VColor(0.0, 0.0, 0.0)
  static White():      VColor(1.0, 1.0, 1.0)
  static Pink():       VColor(1.0, 0.24, 0.59)
  static RoyalBlue():  VColor(0.25, 0.41, 0.88)
  static SlateBlue():  VColor(0.27, 0.24, 0.55)
  static Orange():     VColor(1.0, 0.65, 0.0)
  static Yellow():     VColor(1.0, 1.0, 0.0)
  static LightYellow():VColor(1.0, 0.98, 0.80)
  static DarkViolet(): VColor(0.80, 0.20, 0.47)
```

**Color conversion to 8-bit (for backends):**
```text
r8 = floor(R * 255)
g8 = floor(G * 255)
b8 = floor(B * 255)
```

### 3.5 VGradient

Extends VColor with a second endpoint for gradient fills.

```text
VGradient extends VColor:
  R2: float    -- [0.0, 1.0]
  G2: float    -- [0.0, 1.0]
  B2: float    -- [0.0, 1.0]

  static GrayToWhite(): VGradient(0.8, 0.8, 0.8,  1.0, 1.0, 1.0)
```

---

## 4. Coordinate System

The `CoordinateSystem` defines the visible data-space window (viewport), raster and label spacing, and navigation step sizes. It is shared between layers drawn on the same surface.

```text
CoordinateSystem:
  -- Viewport bounds (data-space units)
  MinX: float
  MaxX: float
  MinY: float
  MaxY: float

  -- Derived (read-only)
  Width:  float  = MaxX - MinX
  Height: float  = MaxY - MinY

  -- Grid and label spacing
  RasterStepX:  float   -- raster snap grid resolution on X
  RasterStepY:  float   -- raster snap grid resolution on Y
  LabelStepX:   float   -- fixed label interval on X (used when LabelMode = ByStep)
  LabelStepY:   float   -- fixed label interval on Y
  LabelNumberX: float   -- label count on X (used when LabelMode = ByNumber)
  LabelNumberY: float   -- label count on Y

  -- Navigation step sizes
  UpDownStep:    float  -- pan increment on Y
  LeftRightStep: float  -- pan increment on X
  ZoomStepX:     float  -- zoom increment applied to X bounds
  ZoomStepY:     float  -- zoom increment applied to Y bounds

  -- Constraints
  AllowNegativeX: bool  -- if false, MinX must stay >= 0 after transforms
  AllowNegativeY: bool  -- if false, MinY must stay >= 0 after transforms

  constructor(minX, maxX, minY, maxY):
    -- Sets all step sizes to 1.0, LabelNumber to 10, AllowNegative to true.
```

### 4.1 Viewport Transforms

All transforms modify MinX/MaxX/MinY/MaxY. If the resulting state is invalid (see §4.2), the change is rolled back atomically.

```text
ShiftLeft:   MinX += LeftRightStep;  MaxX += LeftRightStep
ShiftRight:  MinX -= LeftRightStep;  MaxX -= LeftRightStep
MoveUp:      MinY -= UpDownStep;     MaxY -= UpDownStep
MoveDown:    MinY += UpDownStep;     MaxY += UpDownStep
ZoomIn:      MinX += ZoomStepX;  MaxX -= ZoomStepX;  MinY += ZoomStepY;  MaxY -= ZoomStepY
ZoomOut:     MinX -= ZoomStepX;  MaxX += ZoomStepX;  MinY -= ZoomStepY;  MaxY += ZoomStepY
StretchX:    MinX -= ZoomStepX;  MaxX += ZoomStepX
CompressX:   MinX += ZoomStepX;  MaxX -= ZoomStepX
StretchY:    MinY -= ZoomStepY;  MaxY += ZoomStepY   (see §4.3)
CompressY:   MinY += ZoomStepY;  MaxY -= ZoomStepY   (see §4.3)

transform(type: TransformType):
  -- Save current bounds.
  -- Apply the named mutation.
  -- If coordinateSystemOk() returns false, restore saved bounds.
```

### 4.2 Validity Check

```text
coordinateSystemOk(): bool
  result = (Width > RasterStepX) AND (Height > RasterStepY)
  if NOT AllowNegativeX: result = result AND (MinX >= 0)
  if NOT AllowNegativeY: result = result AND (MinY >= 0)
  return result
```

### 4.3 Implementation Note on StretchY / CompressY

The reference implementation contains a bug: `StretchY` and `CompressY` incorrectly modify `MinX`/`MaxX` instead of `MinY`/`MaxY`. Implementers **should correct this** to modify `MinY`/`MaxY` as described in §4.1. The corrected behavior is what the method names and design intent describe.

---

## 5. IDrawEngine — Rendering Contract

The `IDrawEngine` is the sole interface between chart logic and the rendering backend. All drawing primitives in layers call only these methods.

```text
interface IDrawEngine:

  -- Lifecycle
  beginDraw()
    -- Called before a draw pass begins. Implementations may set up
    -- graphics contexts or begin batches here.

  endDraw()
    -- Called after a draw pass completes. Flush, swap buffers, etc.

  -- Cursor movement (does NOT draw)
  moveTo(x: float, y: float)
  moveTo(c: Coordinate)
    -- Sets internal cursor position. Both overloads must be supported.
    -- Subsequent lineTo calls draw from this position.

  -- Line drawing
  lineTo(c: Coordinate)
  lineTo(x: float, y: float)
    -- Draws a line from the current cursor position to the given point.
    -- Updates cursor to the destination point.

  -- Shape drawing
  circle(position: Coordinate, size: float)
    -- Draws a filled disc. `size` is the bounding box dimension
    -- (i.e., visual radius = size/2). Center of bounding box is at
    -- (position.X - size/2, position.Y - size/2).
    -- Uses the current color as fill.

  drawRectangle(point: Coordinate, width: float, height: float)
    -- Draws a stroked (outline) rectangle. Top-left corner at `point`.

  drawRectangle(start: Coordinate, end: Coordinate)
    -- Draws a stroked (outline) rectangle between two corners.
    -- start = top-left, end = bottom-right.

  drawRectangle(start: Coordinate, end: Coordinate, fillColor: VColor)
    -- Draws a FILLED rectangle between two corners using fillColor.
    -- The width/height are taken as absolute values of (end - start).

  -- Text
  renderText(text: string)
    -- Renders text at cursor position, offset by -10 pixels on Y
    -- (i.e., at pixel position (cursorX, cursorY - 10)).
    -- Uses the current font size and font style.

  -- Styling
  setColor(c: VColor)
    -- Sets the current drawing color for all subsequent draw calls.

  setLineWidth(w: float)
    -- Sets the stroke width for lines and rectangle outlines.

  setDash(style: VLineStyle)
    -- Sets line dash pattern: Solid, Dash, Dotted, HalfDotted.
    -- Mapping to backend dash patterns:
    --   Solid     -> continuous line
    --   Dash      -> evenly-spaced dashes
    --   Dotted    -> densely dotted
    --   HalfDotted -> alternating dash-dot

  setFontSize(size: float)
    -- Sets font size in backend-native points/pixels.

  setFontStyle(style: VFontStyle)
    -- Sets font weight/style combination.

  enableAntialias(enable: bool)
    -- Enables or disables antialiased rendering for lines and shapes.

  -- Clipping
  createClipRegion(x: float, y: float, w: float, h: float)
    -- Restricts all subsequent rendering to the given pixel rectangle.
    -- x, y: top-left corner in pixel space.
    -- w, h: width and height.
    -- Remains active until the next beginDraw/endDraw cycle or
    -- until explicitly reset by the backend.
```

### 5.1 BaseDrawEngine

`BaseDrawEngine` is an abstract base class that provides:
- A `cursor: Coordinate` field updated by `moveTo`.
- Default no-op implementations of all `IDrawEngine` methods.
- A typed reference to its owning surface (`surface: Surface`).

```text
abstract class BaseDrawEngine<S: IDrawSurface> implements IDrawEngine:
  protected cursor: Coordinate
  protected surface: S

  constructor(surface: S)

  moveTo(x, y): cursor = Coordinate(x, y)
  moveTo(c):    cursor = c

  -- All other methods: default no-op (virtual/overridable).
```

Concrete engine implementations extend this class and override only the methods they need to implement.

---

## 6. IDrawSurface — Surface Contract

The surface is the container that owns layers, the engine, legend, and paint lifecycle. It exposes pixel dimensions and margins.

```text
interface IDrawSurface:

  -- Pixel dimensions (read-only)
  pixelWidth:  float
  pixelHeight: float

  -- Margins (read-write)
  -- Margins are in pixels and reduce the effective drawing area.
  topMargin:    float
  bottomMargin: float
  leftMargin:   float
  rightMargin:  float

  -- Engine (read-write; injection point for swapping backends)
  engine: IDrawEngine

  -- Legend
  legend: ChartLegend

  -- Redraw flag
  redrawGraphs: bool    -- if true, layers are redrawn each paint cycle

  -- Methods
  registerLayer(id: string, layer: ILayer)
    -- Associates a layer with this surface. Sets layer.surface = this.
    -- Layers are drawn in insertion order.

  setBackColor(c: VColor)
    -- Sets the background color of the drawing surface.

  redraw()
    -- Triggers a repaint of the surface (schedules or forces a paint cycle).
```

### 6.1 Paint Lifecycle

The surface is responsible for orchestrating each paint frame:

```text
onPaint():
  // 1. Configure the engine for this frame (e.g. inject Graphics context)
  engine.enableAntialias(true)

  // 2. Draw the legend
  drawLegend()

  // 3. Draw all layers
  drawLayers()

drawLegend():
  engine.beginDraw()
  legend.clearAll()
  for each layer in layers:
    layer.exportLegend()        // layers populate legend.items
  drawTopLegend()
  drawRightLegend()
  engine.endDraw()

drawLayers():
  for each layer in layers:
    layer.externDrawCoordinateSystem()    // axes drawn first, outside clip
  for each layer in layers:
    layer.draw()                          // data drawn inside clip

onResize():
  redraw()
```

### 6.2 Legend Drawing

**Top legend** (items placed horizontally, left to right):

```text
drawTopLegend():
  xOffset = legend.xOffset
  yOffset = legend.yOffset
  for each item in legend.getItems(Top):
    start = Coordinate(xOffset, yOffset)
    end   = Coordinate(xOffset + 10, yOffset + 10)
    engine.setColor(item.color ?? VColor.Blue())
    engine.drawRectangle(start, end, item.color ?? VColor.Blue())
    engine.moveTo(start.X + 20, start.Y + 6)
    engine.renderText(item.text)
    xOffset += 30 + (item.text.length * 8)
```

**Right legend** (items placed vertically, top to bottom):

```text
drawRightLegend():
  xOffset = legend.xOffset
  yOffset = legend.yOffset
  for each item in legend.getItems(Right):
    start = Coordinate(pixelWidth - legend.rightOffset, yOffset)
    end   = Coordinate(start.X + 10, start.Y + 10)
    engine.setColor(item.color ?? VColor.Blue())
    engine.drawRectangle(start, end, item.color ?? VColor.Blue())
    engine.moveTo(start.X + 20, start.Y + 6)
    engine.renderText(item.text)
    yOffset += 20
```

---

## 7. ILayer — Layer Contract

A layer is one independently drawable chart component. Layers share the surface and optionally share a coordinate system.

```text
interface ILayer:

  -- Required associations
  coordinateSystem: CoordinateSystem   -- read-write
  surface: IDrawSurface                -- read-write

  -- Coordinate system setup
  setCoordinateSystem(c: CoordinateSystem)
    -- Stores c as the active coordinate system.

  externDrawCoordinateSystem()
    -- Called before draw(). Draws axes and grid outside the clip region.
    -- Default: no-op.

  -- Coordinate transforms (public API)
  unitToScreen(c: Coordinate): Coordinate
  screenToUnit(c: Coordinate): Coordinate

  -- Legend
  exportLegend()
    -- Called by surface to populate surface.legend before rendering.
    -- Default: no-op.

  -- Drawing lifecycle
  clip()
    -- Sets a clip region on the engine. Called before draw().
    -- Default: no-op.

  draw()
    -- Main draw method. Calls engine primitives to render chart data.
    -- Must bracket with engine.beginDraw() / engine.endDraw().

  drawForegroundObjects()
    -- Called after draw() for overlay rendering (e.g., tooltips).
    -- Default: no-op.

  -- Mouse event callbacks (called by surface when mouse events occur)
  mouseX: float          -- current mouse X in pixel space
  mouseY: float          -- current mouse Y in pixel space
  mouseButtonPressed: bool

  mouseClick()
  mouseMove()
  mouseDown()
  mouseUp()
  mouseWheel(direction: VDirection)
```

### 7.1 BaseLayer — Coordinate Transformation

`BaseLayer` is the abstract base class implementing `ILayer`. Its primary role is providing correct coordinate transformations. All layers extend `BaseLayer`.

#### Unit-to-Screen (sX, sY)

Converts a data-space coordinate to a pixel coordinate, accounting for margins.

```text
-- Margin-aware scaling factor (shared by both axes):
marginPctX = 1.0 - (leftMargin + rightMargin) / pixelWidth
marginPctY = 1.0 - (topMargin + bottomMargin) / pixelHeight

sX(c: Coordinate): float
  -- Axis clamping: if the origin is outside the viewport, snap to boundary.
  x = c.X
  if minX > 0 AND x == 0:  x = minX
  if maxX < 0 AND x == 0:  x = maxX

  rel = pixelWidth / (maxX - minX)       -- pixels per unit
  res = rel * (x - minX)                 -- raw pixel position
  return res * marginPctX + leftMargin   -- margin-adjusted

sY(c: Coordinate): float
  y = c.Y
  if minY > 0 AND y == 0:  y = minY
  if maxY < 0 AND y == 0:  y = maxY

  rel = pixelHeight / (maxY - minY)
  res = rel * ((maxY - minY) - (y - minY))   -- = rel * (maxY - y)
  return res * marginPctY + topMargin

-- Shorthand overloads:
sX(x: float): float  = sX(Coordinate(x, 0))
sY(y: float): float  = sY(Coordinate(0, y))
sT(c: Coordinate): Coordinate = Coordinate(sX(c), sY(c))
sT(x, y: float):   Coordinate = sT(Coordinate(x, y))
```

#### Screen-to-Unit (uX, uY)

Converts a pixel coordinate back to data-space, accounting for margins.

```text
uX(pixelX: float): float
  rel = (pixelWidth - leftMargin - rightMargin) / (maxX - minX)
  return (pixelX - leftMargin) / rel + minX

uY(pixelY: float): float
  rel = (pixelHeight - topMargin - bottomMargin) / (maxY - minY)
  return (pixelHeight - pixelY - topMargin) / rel + minY

uT(x, y: float):   Coordinate = Coordinate(uX(x), uY(y))
uT(c: Coordinate): Coordinate = Coordinate(uX(c.X), uY(c.Y))
```

#### Public API

```text
unitToScreen(c): Coordinate  = sT(c)
screenToUnit(c): Coordinate  = uT(c)

origin: Coordinate           = sT(Coordinate(0, 0))   -- pixel position of data origin
```

#### Miscellaneous

```text
invalidate()
  -- Triggers surface.redraw(). Layers call this after state changes.

coordinateSystem: get
  -- Throws if no coordinate system has been set.
```

---

## 8. Legend System

```text
LegendItem:
  position: VLegendPosition
  text: string
  color: VColor

  constructor(position, text, color)

ChartLegend:
  xOffset:      float   -- pixel X offset for top/left legend drawing
  yOffset:      float   -- pixel Y offset for top/left legend drawing
  rightOffset:  float   -- distance from right edge for right legend
  bottomOffset: float   -- distance from bottom edge for bottom legend

  addItem(item: LegendItem)
    -- Dispatches to the correct internal list by item.position.
    -- Only Top and Right positions are fully implemented in reference.

  getItems(position: VLegendPosition): list<LegendItem>
    -- Returns items for the given position.

  clearAll()
    -- Empties all position lists.
```

---

## 9. BalancedTree

A binary search tree indexed by `Coordinate.X` for fast nearest-neighbor point lookup. Used internally by `LineLayer` / `AdvLineLayer` for interactive point snapping.

```text
BalancedTree:
  -- Internal node structure (node contains: point, leftBranch, rightBranch)

  insert(p: Coordinate)
    -- If tree is empty, store p.
    -- If p.X < node.X: recurse into leftBranch.
    -- Else: recurse into rightBranch.

  lookup(c: Coordinate, tolerance: float): Coordinate | null
    -- If tree is empty, return null.
    -- If pointInsideRaster(c, tolerance): return this node's point.
    -- Else if c.X < node.X: recurse leftBranch.
    -- Else: recurse rightBranch.

  clear()
    -- Recursively clears all nodes.

-- Point-in-raster test:
pointInsideRaster(c: Coordinate, tolerance: float): bool
  return (node.X >= c.X - tolerance) AND
         (node.X <= c.X + tolerance) AND
         (node.Y >= c.Y - tolerance) AND
         (node.Y <= c.Y + tolerance)
```

**Note:** `BalancedTree` is defined in the codebase but used for point lookup during snap operations. Implementations may substitute any structure with equivalent `O(log n)` lookup by X (e.g., a sorted array with binary search, or a 2D spatial index).

---

## 10. Line Data Model

`Line` is the data model for a single series of (X, Y) points. It is shared by `LineLayer` and `AdvLineLayer`.

```text
Line:
  -- Identity
  lineKey: string               -- unique identifier within the layer
  lineDescription: string       -- display name (shown in legend)

  -- Data storage
  points: SortedMap<float, Coordinate>   -- keyed by X; unique X values
  relevantPoints: list<Coordinate>       -- points actually rendered last frame

  -- Draw flags
  visible:       bool     -- default true: if false, entire line is skipped
  showLines:     bool     -- default true: connect points with lines
  drawXMarks:    bool     -- default false: vertical drop-lines to X axis
  drawYMarks:    bool     -- default false: horizontal drop-lines to Y axis
  drawPointMark: bool     -- default false: draw a mark at each point
  canSelect:     bool     -- default false: enable point snapping/selection
  showInLegend:  bool     -- default true

  -- Visual styling
  lineWidth:      float         -- default 2
  lineStyle:      VLineStyle    -- default Solid
  lineColor:      VColor | null -- default null (renders as blue)
  xMarkStyle:     VLineStyle    -- default Dotted
  yMarkStyle:     VLineStyle    -- default Dotted
  markRadius:     float         -- default 5
  pointMarkStyle: VMarkStyle    -- default Circle

  -- Performance
  dataDensity:    float            -- default 1: draw every Nth point (1 = all, 2 = every other, etc.)
                                   -- Minimum value is clamped to 1.
  downsampleMode: VDownsampleMode  -- default None
  lttbTarget:     int              -- default 1000: target point count for LTTB mode.
                                   -- Ignored for other modes. Clamped to >= 2.

  -- Events
  enableEvents: bool    -- default false
  onBeforeDrawPoint: callback(DrawPointEventArgs) | null

  constructor(key: string)

  addPoint(c: Coordinate)
    -- If no point exists at c.X, insert. Otherwise update Y.

  addPoints(points: iterable<Coordinate>)
    -- Calls addPoint for each.

  removePoint(x: float)
    -- Removes the point at the given X, if present.

  removePoints(xs: iterable<float>)

  movePoint(point: Coordinate, dx: float, dy: float): Coordinate | null
    -- If dx != 0: remove old point, insert new at (point.X+dx, point.Y+dy).
    -- If dx == 0: update Y in place.
    -- Returns the new Coordinate, or null if point not found or delta is zero.

  cutOffIrrelevantPoints()
    -- Replaces points with the current relevantPoints set.
    -- Effectively discards all off-screen points.

  firstPoint: Coordinate
    -- The point with the smallest X. Returns Coordinate(0, 0) if empty.

  lastPoint: Coordinate
    -- The point with the largest X.

  clone(): Line
    -- Deep copy of all fields and points (excluding event handlers).

  downsample(mode: VDownsampleMode, pixelWidth: float, lttbTarget: int): Line
    -- Returns a NEW Line containing a reduced point set. The original is unchanged.
    -- mode == None:       returns a clone (no reduction).
    -- mode == PixelExact: iterates points sorted by X; keeps a point only if
                           its screenX pixel column differs from the last kept point.
                           Requires the caller to supply pixelWidth and the current
                           coordinate system bounds so that screenX can be computed.
    -- mode == MinMax:     for each integer pixel column [0, pixelWidth) collects all
                           points that map to it, then keeps the min-Y and max-Y point
                           in that column (two points per column maximum).
    -- mode == LTTB:       applies the LTTB algorithm (§15.6) with the given
                           lttbTarget. pixelWidth is unused.
    -- In all cases the returned Line inherits all styling properties of the
       original but has no event handlers and an empty relevantPoints list.

  fireBeforeDrawPointEvent(args: DrawPointEventArgs)
    -- Invokes onBeforeDrawPoint if set and enableEvents is true.

DrawPointEventArgs:
  point:       Coordinate
  line:        Line
  surface:     IDrawSurface
  layer:       ILayer
  pointNumber: int          -- 1-based index of point in the current render pass
  cancel:      bool         -- set to true to skip rendering this point
```

---

## 11. LineLayer

`LineLayer` renders a standard XY line chart with optional grid, labels, interactive cursor, and point snapping. It extends `BaseLayer`.

### 11.1 Configuration

```text
LineLayer:
  -- Display settings
  indicatorSize:       float          -- default 5: tick mark size in pixels
  snapTolerance:       float          -- default 10: pixel radius for point snap
  roundLabels:         bool           -- default false: round label values to 2 decimals
  enableCursor:        bool           -- default false
  horizontalLines:     bool           -- default false: draw horizontal grid lines
  verticalLines:       bool           -- default false: draw vertical grid lines
  strictlyRaster:      bool           -- default false: snap cursor to raster grid
  verticalLineStyle:   VLineStyle     -- default Dotted
  horizontalLineStyle: VLineStyle     -- default Dotted
  labelMode:           VLabelMode     -- default ByStep
  markStyle:           UserMarkStyle  -- default Line

  -- Label overrides (optional; indexable list)
  overrideLabelsX: list<string>    -- if non-empty, use index i for X label at step i
  overrideLabelsY: list<string>

  -- Data
  lines: map<string, Line>

  -- Axis marks (colored markers on axes)
  xMarks: map<float, VColor>    -- position -> color
  yMarks: map<float, VColor>

  -- Interactive state
  cursor:        Coordinate | null   -- current cursor position (data-space)
  selectedPoint: Coordinate | null   -- last snapped/clicked point
```

### 11.2 Public Methods

```text
addLine(line: Line)
removeLine(line: Line)

insertXMark(position: float, color: VColor)
insertYMark(position: float, color: VColor)
removeXMark(position: float)
removeYMark(position: float)
```

### 11.3 Drawing Sequence (draw())

```text
draw():
  engine.beginDraw()
  try:
    engine.setColor(VColor.Blue())
    drawCoordinateSystem()
    drawXIndicators()
    drawYIndicators()
    setClipRegion()
    connectPoints()
    drawUserMarks()
    drawCursor()         // interactive overlay
    selectPoint()        // interactive overlay
  finally:
    engine.endDraw()
```

### 11.4 Draw Coordinate System

Draws X and Y axes as lines from edge to edge:

```text
drawCoordinateSystem():
  // X axis
  engine.moveTo(sT(Coordinate(minX, 0)))
  engine.lineTo(sT(Coordinate(maxX, 0)))
  // Y axis
  engine.moveTo(sT(Coordinate(0, minY)))
  engine.lineTo(sT(Coordinate(0, maxY)))
```

### 11.5 Label Step Calculation

```text
getLabelStep(forX: bool): float
  if labelMode == ByNumber:
    return forX ? (width / labelNumberX) : (height / labelNumberY)
  else:  -- ByStep
    return forX ? labelStepX : labelStepY
```

### 11.6 Draw X Axis Indicators

```text
drawXIndicators():
  engine.setLineWidth(1)
  step = getLabelStep(true)
  for x from minX to maxX by step:
    if x == 0: continue    // skip origin tick

    if NOT verticalLines:
      engine.moveTo(sX(x), sY(0))
      engine.lineTo(sX(x), sY(0) + indicatorSize)
    else:
      // Full vertical grid line
      engine.enableAntialias(false)
      engine.setDash(verticalLineStyle)
      engine.moveTo(sX(x), sY(minY))
      engine.lineTo(sX(x), sY(maxY))
      engine.enableAntialias(true)
      engine.setDash(Solid)

    // Label
    lX = roundLabels ? floor(x) : x   // AdvLineLayer truncates to int; LineLayer rounds to 2dp
    if lX == 0: continue

    offset = (verticalLines AND minY < 0) ? 18.0 : 0.0
    engine.moveTo(sX(x) - (lX.toString().length * 3) + offset - 2,
                  sY(0) + indicatorSize + 15)
    engine.renderText(lX.toString())

  engine.setLineWidth(2)
```

### 11.7 Draw Y Axis Indicators

```text
drawYIndicators():
  engine.setLineWidth(1)
  step = getLabelStep(false)
  for y from minY to maxY by step:
    if y == 0: continue

    if NOT horizontalLines:
      engine.moveTo(sX(0), sY(y))
      engine.lineTo(sX(0) - indicatorSize, sY(y))
    else:
      engine.enableAntialias(false)
      engine.setDash(horizontalLineStyle)
      engine.moveTo(sX(minX), sY(y))
      engine.lineTo(sX(maxX), sY(y))
      engine.enableAntialias(true)
      engine.setDash(Solid)

    lY = roundLabels ? floor(y) : y
    if lY == 0: continue

    offset = (horizontalLines AND minX < 0) ? 10.0 : 0.0
    engine.moveTo(sX(0) - (indicatorSize + 10) - (lY.toString().length * 5),
                  sY(y) + 2 + offset)
    engine.renderText(lY.toString())

  engine.setLineWidth(2)
```

### 11.8 Clip Region

```text
setClipRegion():
  engine.createClipRegion(
    sX(minX),
    sY(minY),
    sX(maxX) - sX(minX),
    sY(maxY) - sY(minY)
  )
```

Note: `sY(minY)` produces the larger pixel Y (top of viewport visually) because Y is inverted.

### 11.9 Connect Points

The main rendering loop across all lines:

```text
connectPoints():
  for each line in lines.values:
    if NOT line.visible: continue
    if line.points.isEmpty: continue

    lastPoint = Coordinate(minX, maxX)   // sentinel
    pointCount = 0

    engine.setColor(line.lineColor ?? VColor.Blue())
    engine.setLineWidth(line.lineWidth)
    engine.setDash(line.lineStyle)
    engine.moveTo(sT(line.firstPoint))

    line.relevantPoints.clear()

    -- Viewport culling: use a sorted-map range query to seek directly to
    -- the first point with X >= minX rather than iterating from the beginning.
    -- This is O(log n) for the seek and O(k) for k visible points,
    -- instead of O(n) for the full scan. See §15.4.
    -- One point before minX is included to preserve line continuity at the
    -- left edge; the break condition handles the right edge as before.

    -- Downsampling: if line.downsampleMode != None, obtain a reduced point
    -- set before the loop (see §15.6). The loop below then operates on the
    -- reduced set. dataDensity culling is still applied afterwards.
    renderPoints = getPointsForRender(line, pixelWidth)

    for each point in renderPoints (sorted by X):
      -- Clipping: points are already within [minX, maxX] when a range query
      -- was used, but the break condition is kept for safety.
      if point.X < minX: continue
      if point.X > maxX AND lastPoint.X > maxX: break

      pointCount += 1

      -- Data density culling (skip if not Nth point)
      if line.dataDensity != 0 AND
         (pointCount % line.dataDensity != 0) AND
         pointCount != 1:
        continue

      -- Per-point event (optional)
      if line.enableEvents:
        args = DrawPointEventArgs(point, line, surface, this)
        args.pointNumber = pointCount
        line.fireBeforeDrawPointEvent(args)
        if args.cancel: continue

      -- Draw line segment
      if line.showLines:
        engine.lineTo(sT(point))
        engine.moveTo(sT(point))

      -- Per-point decorations
      if line.drawXMarks:
        // drop-line to X axis
        engine.enableAntialias(false)
        engine.setLineWidth(1)
        engine.setDash(line.xMarkStyle)
        engine.moveTo(sT(point))
        engine.lineTo(sX(point.X), sY(0))
        engine.moveTo(sT(point))
        engine.setDash(Solid)
        engine.enableAntialias(true)
        engine.setLineWidth(2)

      if line.drawYMarks:
        // drop-line to Y axis
        engine.enableAntialias(false)
        engine.setLineWidth(1)
        engine.setDash(line.yMarkStyle)
        engine.moveTo(sT(point))
        engine.lineTo(sX(0), sY(point.Y))
        engine.moveTo(sT(point))
        engine.setDash(Solid)
        engine.enableAntialias(true)
        engine.setLineWidth(2)

      if line.drawPointMark:
        drawPointMarking(point, line.markRadius, line.pointMarkStyle)

      line.relevantPoints.add(point)
      lastPoint = point

  // Reset state
  engine.setLineWidth(2)
  engine.setDash(Solid)

drawPointMarking(point: Coordinate, radius: float, style: VMarkStyle):
  if style == Circle:
    engine.circle(sT(point), radius)
    engine.moveTo(sT(point))
  else if style == Box:
    start = sT(point)
    end   = sT(point)
    start.X -= radius / 2;  start.Y += radius / 2
    end.X   += radius / 2;  end.Y   -= radius / 2
    engine.drawRectangle(start, end)
```

### 11.10 User Marks

```text
drawUserMarks():
  engine.setDash(Dash)
  engine.setLineWidth(1)
  for each (x, color) in xMarks:
    if markStyle in [Line, Both]:
      engine.setColor(color)
      engine.moveTo(sT(Coordinate(x, minY)))
      engine.lineTo(sT(Coordinate(x, maxY)))
    if markStyle in [Dot, Both]:
      engine.setColor(color)
      engine.circle(sT(Coordinate(x, 0)), 5)
  // same pattern for yMarks (horizontal lines and dots on Y axis)
  engine.setDash(Solid)
  engine.setLineWidth(2)
  engine.setColor(VColor.Blue())
```

### 11.11 Cursor and Selection

```text
drawCursor():
  if NOT enableCursor OR cursor == null: return
  engine.setDash(Dash)
  engine.setColor(VColor.Blue())
  engine.moveTo(sT(cursor))
  engine.lineTo(sX(0), sY(cursor.Y))      // horizontal line to Y axis
  engine.moveTo(sT(cursor))
  engine.lineTo(sX(cursor.X), sY(0))      // vertical line to X axis
  engine.setDash(Solid)

selectPoint():  // LineLayer version: draws box only
  if selectedPoint == null: return
  engine.setColor(VColor.Blue())
  c = Coordinate(sX(selectedPoint.X) - 10, sY(selectedPoint.Y) - 10)
  engine.drawRectangle(c, 20, 20)
```

### 11.12 Mouse Interaction

```text
mouseClick():
  if NOT mouseInsideCoordinateSystem(): return
  snapToCoordinate(Coordinate(mouseX, mouseY))
  if snapPoint != null:
    selectedPoint = snapPoint
    cursor = null
  else:
    if strictlyRaster:
      cursor = raster(uT(mouseX, mouseY))
    else:
      cursor = uT(mouseX, mouseY)
  surface.redraw()

mouseInsideCoordinateSystem(): bool
  ct = uT(mouseX, mouseY)
  return ct.X >= minX AND ct.X <= maxX AND
         ct.Y >= minY AND ct.Y <= maxY

snapToCoordinate(pixel: Coordinate):
  snapPoint = pointInsideSnapRaster(pixel)
  if snapPoint != null: surface.redraw()

pointInsideSnapRaster(pixel: Coordinate): Coordinate | null
  for each line in lines.values:
    if NOT line.canSelect: continue
    for each c in line.relevantPoints:
      if sX(c.X) in [pixel.X - snapTolerance, pixel.X + snapTolerance] AND
         sY(c.Y) in [pixel.Y - snapTolerance, pixel.Y + snapTolerance]:
        return c
  return null
```

### 11.13 Raster Snapping

```text
raster(point: Coordinate): Coordinate
  return Coordinate(rasterX(point.X), rasterY(point.Y))

rasterX(x: float): float
  step = rasterStepX
  rest = x mod step
  if rest < step / 2: return x - rest
  else: return (x - rest) + step

rasterY(y: float): float
  // same algorithm with rasterStepY
```

### 11.14 Legend Export

```text
exportLegend():
  for each line in lines.values:
    surface.legend.addItem(LegendItem(Top, line.lineDescription, line.lineColor))
```

---

## 12. AdvLineLayer

`AdvLineLayer` is a superset of `LineLayer` with three additional capabilities:
1. **Right-side Y-axis** (optional).
2. **Drag-and-drop point editing**.
3. **Rectangular zoom-to-selection**.

It also moves `drawCoordinateSystem()` and axis indicators out of `draw()` into `externDrawCoordinateSystem()`, so axes are drawn before the clip region is applied.

### 12.1 Additional Configuration

```text
AdvLineLayer (extends LineLayer behavior):
  yAxisRight:      bool    -- default false: draw Y axis on right side
  dragDrop:        bool    -- default false: enable drag-and-drop
  enableSelection: bool    -- default false: enable rubber-band zoom

  -- Selection state
  selectedLine:  Line | null
  currentPoint:  Coordinate | null   -- current mouse position during selection drag

  -- Events
  onGraphMouseUp: callback | null    -- fired after zoom-to-selection completes
```

### 12.2 Right Y-Axis

When `yAxisRight == true`:
- The Y axis is drawn at `X = maxX` (right edge) instead of `X = 0`.
- Y axis indicators are drawn to the right of that axis line.
- Y axis labels appear to the right of the indicators.

```text
// Axis line:
engine.moveTo(sX(maxX) - 1, sY(minY))
engine.lineTo(sX(maxX) - 1, sY(maxY))

// Y indicator ticks (right side):
engine.moveTo(sX(maxX), sY(y))
engine.lineTo(sX(maxX) + indicatorSize, sY(y))

// Y label position:
engine.moveTo(sX(maxX) + indicatorSize, sY(y) + 2 + offset)
```

### 12.3 externDrawCoordinateSystem()

In `AdvLineLayer`, axis drawing is separated from data drawing:

```text
externDrawCoordinateSystem():
  engine.setColor(VColor.Blue())
  drawCoordinateSystem()
  drawXIndicators()
  drawYIndicators()
```

`draw()` then omits those steps and calls only:
```text
  setClipRegion()
  connectPoints()
  drawUserMarks()
  drawCursor()
  selectPoint()    // AdvLineLayer version: also renders coordinate label
  drawSelection()
```

### 12.4 Selection Indicator (AdvLineLayer)

In `AdvLineLayer`, `selectPoint()` also renders the point's coordinate:

```text
selectPoint():
  if selectedPoint == null: return
  engine.setColor(VColor.Blue())
  c = Coordinate(sX(selectedPoint.X) - 10, sY(selectedPoint.Y) - 10)
  engine.drawRectangle(c, 20, 20)
  c.X += 25;  c.Y += 10
  engine.moveTo(c)
  engine.renderText("X:{round(selectedPoint.X,1)} Y:{round(selectedPoint.Y,1)}")
```

### 12.5 Drag-and-Drop

```text
mouseMove():
  // Rubber-band selection tracking
  if enableSelection AND mouseButtonPressed:
    currentPoint = Coordinate(mouseX, mouseY)
    surface.redraw()

  // Drag-and-drop point
  if NOT dragDrop: return
  if selectedPoint == null OR selectedLine == null: return
  if selectedPoint.X == 0 OR selectedPoint.X == maxX: return
  if NOT mouseButtonPressed: return
  if NOT mouseInsideCoordinateSystem(): return

  xN = rasterX(uX(mouseX))
  yN = rasterY(uY(mouseY))
  dX = xN - selectedPoint.X
  dY = yN - selectedPoint.Y

  if (selectedPoint.X + dX) in (0, maxX) AND
     (selectedPoint.Y + dY) in (0, maxY):
    newCoord = selectedLine.movePoint(selectedPoint, dX, dY)
    if newCoord != null: selectedPoint = newCoord

  surface.redraw()
```

### 12.6 Rubber-Band Zoom

```text
mouseDown():
  if enableSelection:
    selectedPoint = Coordinate(mouseX, mouseY)   // anchor corner in pixel space

mouseUp():
  if enableSelection AND currentPoint != null AND selectedPoint != null:
    // Convert pixel corners to data-space
    minX_new = uX(selectedPoint.X)
    maxX_new = uX(currentPoint.X)
    minY_new = uY(currentPoint.Y)
    maxY_new = uY(selectedPoint.Y)

    coordinateSystem.minX = minX_new
    coordinateSystem.maxX = maxX_new
    coordinateSystem.minY = minY_new
    coordinateSystem.maxY = maxY_new

    if onGraphMouseUp != null: onGraphMouseUp()
    surface.redraw()

drawSelection():
  if enableSelection AND mouseButtonPressed:
    engine.setLineWidth(1)
    engine.drawRectangle(selectedPoint, currentPoint)   // pixel-space rectangle
    engine.setLineWidth(2)
```

---

## 13. GanttLayer

`GanttLayer` renders a horizontal bar (Gantt) chart. The Y axis maps to integer row indices; each row contains one `GanttLine` with zero or more `GanttBar` elements. The layer enforces non-negative Y values.

### 13.1 Supporting Types

```text
VRange:
  from: float
  to:   float

  intersects(other: VRange): bool
    return (other.from > from AND other.from < to) OR
           (from > other.from AND from < other.to)

  containsPoint(px: float): bool
    return px >= from AND px <= to

GanttLine:
  index:       int       -- auto-assigned by layer on add (0-based)
  key:         string    -- unique identifier
  description: string    -- displayed as row label
  lineHeight:  float     -- default 20 pixels
  tag:         any       -- user-defined payload
  bars:        map<string, GanttBar>

  addBar(key: string, bar: GanttBar)
  removeBar(key: string)

GanttBar:
  range:       VRange    -- data-space X extent of the bar
  barHeight:   float     -- half-height in pixels; default 10
  fillColor:   VColor | null
  borderColor: VColor | null
  borderStyle: VLineStyle -- default Solid
  borderSize:  float     -- border line width; default 1
  hasBorder:   bool      -- default false
  pixelDim:    VRectangle -- set during draw; stores rendered pixel bounds
  parent:      GanttLine -- back-reference; set by addBar
  tag:         any

  constructor(from: float, to: float)

  intersects(other: GanttBar): bool
    return range.intersects(other.range)
```

### 13.2 Configuration

```text
GanttLayer:
  labelMode:    VLabelMode  -- default ByStep
  roundLabels:  bool        -- default true

  -- Selection state
  selectedLine: GanttLine | null
  selectedBar:  GanttBar  | null

  -- Internal storage
  lines:        SortedMap<int, GanttLine>    -- keyed by index
  ganttLines:   map<string, GanttLine>       -- keyed by key
  lineYRanges:  map<int, VRange>             -- pixel Y ranges of rendered lines
  relevantLines: list<GanttLine>             -- lines actually drawn last frame

  -- Events
  onBarSelected:      callback(BarSelectedEventArgs) | null
  onButtonPressed:    callback(GanttLine) | null
  onCheckedChanged:   callback(GanttLine) | null
```

### 13.3 Coordinate System Constraint

`GanttLayer` forces `AllowNegativeY = false` on its coordinate system:

```text
setCoordinateSystem(c):
  base.setCoordinateSystem(c)
  c.allowNegativeY = false
```

### 13.4 Drawing Sequence

```text
draw():
  engine.beginDraw()
  engine.setLineWidth(1)
  engine.enableAntialias(false)
  try:
    drawCoordinateSystem()
    drawRangeIndicators()
    drawGanttLines()
    drawSelectedBar()
  finally:
    engine.endDraw()
```

### 13.5 Axes

```text
drawCoordinateSystem():
  engine.setLineWidth(2)
  engine.setColor(VColor.Blue())
  // X axis (bottom of gantt area)
  engine.moveTo(sT(Coordinate(minX, maxY)))
  engine.lineTo(sT(Coordinate(maxX, maxY)))
  // Y axis (left side)
  engine.moveTo(sT(Coordinate(minX, minY)))
  engine.lineTo(sT(Coordinate(minX, maxY)))
  engine.setColor(VColor.Black())
  engine.setLineWidth(1)
```

### 13.6 Range Indicators (X Axis Labels)

```text
drawRangeIndicators():
  step = getLabelStep()
  maxYPos = getMaximumRelevantYPosition()   // pixel Y of lowest visible line bottom

  engine.setDash(Dotted)
  for x from minX to maxX by step:
    if x == 0: continue
    engine.setColor(VColor.Black())
    engine.moveTo(sX(x), sY(maxY))
    engine.lineTo(sX(x), maxYPos)           // dotted vertical tick

    lX = roundLabels ? round(x, 2) : x
    engine.moveTo(sX(x) - (lX.toString().length * 3.5), sY(maxY) - 8)
    engine.renderText(lX.toString())
```

### 13.7 Drawing Gantt Lines and Bars

```text
drawGanttLines():
  minY_pixel = sY(maxY)   // top pixel of gantt area
  lineYRanges.clear()
  relevantLines.clear()

  for i from minY to maxY (integer steps):
    if i > lines.count - 1: break
    line = lines[i]

    // Skip if partially clipped at bottom
    if (minY_pixel + line.lineHeight) >= sY(minY): break

    // Skip out-of-range rows
    if line.index < minY: continue
    if line.index > maxY: break

    // Draw bars at the vertical center of this row
    midY = minY_pixel + (line.lineHeight / 2)
    drawBars(line, midY)

    // Draw row label/button
    // Platform-specific: reference uses a Windows Forms Button widget.
    // Language-specific implementations should render an interactive
    // clickable label at pixel position (leftMargin, minY_pixel).
    renderRowLabel(line, minY_pixel)

    minY_pixel += line.lineHeight

    // Draw row separator
    engine.setDash(Dotted)
    engine.setColor(VColor.Black())
    engine.moveTo(sX(minX), minY_pixel)
    engine.lineTo(sX(maxX), minY_pixel)

    relevantLines.add(line)
    lineYRanges[line.index] = VRange(minY_pixel - line.lineHeight, minY_pixel)

drawBars(line: GanttLine, midYPixel: float):
  for each bar in line.bars.values:
    yStart = midYPixel - bar.barHeight
    yEnd   = midYPixel + bar.barHeight
    xStart = sX(bar.range.from)
    xEnd   = sX(bar.range.to)

    start = Coordinate(xStart, yStart)
    end   = Coordinate(xEnd, yEnd)

    engine.drawRectangle(start, end, bar.fillColor)

    if bar.hasBorder:
      engine.setColor(bar.borderColor)
      engine.setLineWidth(bar.borderSize)
      engine.setDash(bar.borderStyle)
      engine.drawRectangle(start, end)
      engine.setDash(HalfDotted)
      engine.setLineWidth(1)

    // Store rendered pixel bounds for hit-testing
    bar.pixelDim = VRectangle(start, end)
```

### 13.8 Row Label Rendering

In the reference implementation, row labels are rendered as Windows Forms `Button` widgets parented to the surface control. This is inherently platform-specific.

**For portable reimplementations**, render the row label as:
- A clickable region at `(leftMargin, rowPixelY)` with width 130 px and height `lineHeight - 2 px`.
- Text: `line.description`.
- When clicked, fire `onButtonPressed(line)`.

The abstract label rendering contract is:

```text
renderRowLabel(line: GanttLine, rowPixelY: float)
  -- Render a clickable label for this row.
  -- Position: x=leftMargin (or fixed pixel offset), y=rowPixelY + 2
  -- Size: width=130px, height=lineHeight-2
  -- Text: line.description
  -- Click fires: onButtonPressed(line)
```

### 13.9 Selected Bar Highlight

```text
drawSelectedBar():
  if selectedBar == null: return
  if lineYPos(selectedBar.parent) != Inside: return
  if NOT lineYRanges.containsKey(selectedBar.parent.index): return

  xStart = selectedBar.pixelDim.topLeft.X - 2
  yStart = selectedBar.pixelDim.topLeft.Y + 2
  xEnd   = selectedBar.pixelDim.lowerRight.X + 2
  yEnd   = selectedBar.pixelDim.lowerRight.Y - 2

  engine.setColor(VColor.Blue())
  engine.setLineWidth(2)
  engine.setDash(Solid)
  engine.drawRectangle(Coordinate(xStart, yStart), Coordinate(xEnd, yEnd))
  engine.setLineWidth(1)
```

### 13.10 Mouse Interaction

```text
mouseClick():
  selectedLine = findLine(mouseY)
  selectedBar  = findBar(selectedLine, mouseX)
  surface.redraw()
  if onBarSelected != null:
    onBarSelected(BarSelectedEventArgs(selectedBar))

mouseWheel(direction):
  if direction == Up:   coordinateSystem.transform(MoveUp)
  if direction == Down: coordinateSystem.transform(MoveDown)
  invalidate()

findLine(pixelY: float): GanttLine | null
  for each line in relevantLines:
    if lineYRanges[line.index].containsPoint(pixelY): return line
  return null

findBar(line: GanttLine, pixelX: float): GanttBar | null
  if line == null: return null
  for each bar in line.bars.values:
    if bar.range.containsPoint(uX(pixelX)): return bar
  return null
```

### 13.11 Layer API

```text
addLine(line: GanttLine)
  -- Assigns line.index = current line count.
  -- Inserts into both ganttLines (by key) and lines (by index).

removeLine(key: string)
  -- Removes from both maps.
```

### 13.12 Custom Events

```text
BarSelectedEventArgs:
  bar: GanttBar | null

// Event types:
BarSelectedHandler:   (args: BarSelectedEventArgs) -> void
ButtonPressed:        (line: GanttLine) -> void
CheckedChanged:       (line: GanttLine) -> void
```

---

## 14. GDI+ Reference Implementation

This section documents how each abstraction maps to GDI+ (`System.Drawing`). Use this as a translation guide when implementing a backend.

### 14.1 GDIDrawingEngine

```text
GDIDrawingEngine extends BaseDrawEngine<GDISurface>:

  -- Internal GDI state
  context:   Graphics   -- injected each frame from PaintEventArgs.Graphics
  pen:       Pen        -- initialized with Color.Blue, width 2
  font:      Font       -- initialized with Arial 10pt Regular
  fontStyle: FontStyle
  fontSize:  float

  constructor(surface):
    pen = new Pen(Color.Blue, 2)
    fontStyle = FontStyle.Regular
    fontSize = 10
    font = new Font("Arial", 10, Regular)

  lineTo(c):
    context.DrawLine(pen, PointF(cursor.X, cursor.Y), PointF(c.X, c.Y))
    // Note: cursor is updated by base class moveTo/lineTo

  circle(position, size):
    context.FillPie(pen.Brush,
      position.X - size/2, position.Y - size/2,
      size, size, 0, 360)

  drawRectangle(point, width, height):
    context.DrawRectangle(pen, point.X, point.Y, width, height)

  drawRectangle(start, end):
    context.DrawRectangle(pen, start.X, start.Y, end.X-start.X, end.Y-start.Y)

  drawRectangle(start, end, fillColor):
    savedColor = pen.Color
    pen.Color = GDIColor(fillColor)
    context.FillRectangle(pen.Brush, start.X, start.Y,
      abs(end.X - start.X), abs(end.Y - start.Y))
    pen.Color = savedColor

  renderText(text):
    context.DrawString(text, font, pen.Brush, PointF(cursor.X, cursor.Y - 10))

  setColor(c):
    pen.Color = GDIColor(c)      // VColor -> System.Drawing.Color

  setLineWidth(w):
    pen.Width = w

  setFontSize(size):
    fontSize = size;  createFont()

  setFontStyle(style):
    fontStyle = gdiStyle(style);  createFont()

  setDash(style):
    Solid    -> pen.DashStyle = Solid
    Dash     -> pen.DashStyle = Dash
    Dotted   -> pen.DashStyle = Dot
    HalfDotted -> pen.DashStyle = DashDot

  enableAntialias(enable):
    context.SmoothingMode = enable ? AntiAlias : None

  createClipRegion(x, y, w, h):
    context.SetClip(RectangleF(x, y, w, h))

  createFont():
    font = new Font("Arial", fontSize, fontStyle)

-- Color conversion:
GDIColor(c: VColor): Color
  return Color.FromArgb(floor(c.R*255), floor(c.G*255), floor(c.B*255))

ToVColor(c: Color): VColor
  return VColor(c.R/255.0, c.G/255.0, c.B/255.0)
```

### 14.2 GDISurface

```text
GDISurface extends UserControl, implements IDrawSurface:

  -- State
  layers:   OrderedMap<string, ILayer>   -- insertion order preserved
  engine:   GDIDrawingEngine
  legend:   ChartLegend                  -- lazy-initialized

  constructor():
    doubleBuffered = true
    redrawGraphs = true
    engine = new GDIDrawingEngine(this)
    // Wire mouse events -> dispatch to all layers

  pixelWidth:  this.Width
  pixelHeight: this.Height

  onPaint(e: PaintEventArgs):
    (engine as GDIDrawingEngine).context = e.Graphics
    engine.enableAntialias(true)
    drawLegend()
    drawLayers()

  onResize(e):
    redraw()

  registerLayer(id, layer):
    layer.surface = this
    layers[id] = layer

  redraw():
    this.Invalidate(true)   // triggers OS repaint cycle

  setBackColor(c: VColor):
    this.BackColor = Color.FromArgb(floor(255*c.R), floor(255*c.G), floor(255*c.B))

-- Mouse event dispatch pattern:
  onMouseMove(e):
    for each layer in layers.values:
      layer.mouseX = e.X;  layer.mouseY = e.Y
      layer.mouseMove()

  onMouseClick(e):
    for each layer in layers.values:
      layer.mouseX = e.X;  layer.mouseY = e.Y
      layer.mouseClick()

  onMouseDown(e):
    for each layer in layers.values:
      layer.mouseButtonPressed = true
      layer.mouseDown()

  onMouseUp(e):
    for each layer in layers.values:
      layer.mouseButtonPressed = false
      layer.mouseUp()

  -- Mouse wheel events should be similarly dispatched; the reference
  -- implementation handles this per-layer (GanttLayer.mouseWheel).
```

---

## 15. Algorithms Summary

### 15.1 Unit-to-Screen (canonical formulas)

```text
-- Given: surface margins, pixel dimensions, coordinate system bounds
marginPctX = 1.0 - (leftMargin + rightMargin) / pixelWidth
marginPctY = 1.0 - (topMargin + bottomMargin) / pixelHeight

screenX(x):
  rel = pixelWidth / (maxX - minX)
  return (rel * (x - minX)) * marginPctX + leftMargin

screenY(y):
  rel = pixelHeight / (maxY - minY)
  return (rel * (maxY - y)) * marginPctY + topMargin
```

### 15.2 Screen-to-Unit (canonical formulas)

```text
unitX(px):
  rel = (pixelWidth - leftMargin - rightMargin) / (maxX - minX)
  return (px - leftMargin) / rel + minX

unitY(py):
  rel = (pixelHeight - topMargin - bottomMargin) / (maxY - minY)
  return (pixelHeight - py - topMargin) / rel + minY
```

### 15.3 Raster Snap

```text
rasterSnap(v, step):
  rest = v mod step
  if rest < step / 2: return v - rest
  else: return (v - rest) + step
```

### 15.4 Point Culling in ConnectPoints

```text
-- Preferred (O(log n) seek):
visiblePoints = line.points.rangeQuery(minX, maxX)
  -- Use the sorted map's range-seek to find the first point with X >= minX.
  -- Include one point to the left of minX if present, to preserve line
  -- continuity at the left viewport edge.
  -- Iterate forward; stop when X > maxX.

-- Fallback (O(n) scan, reference implementation behaviour):
for each point in line.points (sorted by X):
  if point.X < minX: skip
  if point.X > maxX AND lastPoint.X > maxX: stop
```

Implementations backed by a sorted container with O(log n) range seek (e.g.
`SortedDict.irange` in Python, `SortedSet.GetViewBetween` in C#, `BTreeMap::range`
in Rust) should prefer the range-query form. The O(n) fallback is kept for
compatibility with unsorted or unindexed backing stores.

### 15.5 Label Step

```text
labelStep(forX, mode, cs):
  if mode == ByNumber:
    return forX ? cs.width / cs.labelNumberX : cs.height / cs.labelNumberY
  else:
    return forX ? cs.labelStepX : cs.labelStepY
```

### 15.6 Downsampling Algorithms

All three algorithms operate on the full point set of a `Line` and return a
reduced list of `Coordinate` values. They are called by `getPointsForRender`
inside `connectPoints` when `line.downsampleMode != None`, and also by the
standalone `Line.downsample()` method.

#### PixelExact

```text
pixelExact(points: sorted list<Coordinate>, screenX: Coordinate -> float):
  result = []
  lastPixel = null
  for each point in points:
    px = floor(screenX(point))
    if px != lastPixel:
      result.add(point)
      lastPixel = px
  return result
```

Upper bound on output size: `pixelWidth` points per line.

#### MinMax

```text
minMax(points: sorted list<Coordinate>, screenX: Coordinate -> float):
  buckets: map<int, (minPoint, maxPoint)>
  for each point in points:
    col = floor(screenX(point))
    if col not in buckets:
      buckets[col] = (point, point)
    else:
      if point.Y < buckets[col].minPoint.Y: buckets[col].minPoint = point
      if point.Y > buckets[col].maxPoint.Y: buckets[col].maxPoint = point

  result = []
  for each col in buckets (sorted):
    result.add(buckets[col].minPoint)
    if buckets[col].maxPoint != buckets[col].minPoint:
      result.add(buckets[col].maxPoint)
  return result
```

Upper bound on output size: `2 * pixelWidth` points per line.
The result list is sorted by the X of `minPoint`; implementations must
re-sort by Coordinate.X if min and max are interleaved across columns.

#### LTTB (Largest-Triangle-Three-Buckets)

```text
lttb(points: list<Coordinate>, target: int): list<Coordinate>
  -- target is clamped to [2, points.count].
  if points.count <= target: return points

  result = [points.first]           // always keep the first point
  bucketSize = (points.count - 2) / (target - 2)

  prevSelected = points.first
  for i from 0 to target - 3:      // iterate over middle buckets
    -- Current bucket
    bucketStart = floor((i + 1) * bucketSize) + 1
    bucketEnd   = min(floor((i + 2) * bucketSize) + 1, points.count)

    -- Next bucket: compute its average point
    nextStart = bucketEnd
    nextEnd   = min(floor((i + 3) * bucketSize) + 1, points.count)
    avgX = mean(points[nextStart..nextEnd].X)
    avgY = mean(points[nextStart..nextEnd].Y)
    avgPoint = Coordinate(avgX, avgY)

    -- Pick the point in the current bucket that forms the largest triangle
    -- with prevSelected and avgPoint
    maxArea = -1
    selected = points[bucketStart]
    for each point in points[bucketStart..bucketEnd]:
      area = abs( (prevSelected.X - avgPoint.X) * (point.Y      - prevSelected.Y)
                - (prevSelected.X - point.X)    * (avgPoint.Y   - prevSelected.Y) )
      if area > maxArea:
        maxArea = area
        selected = point

    result.add(selected)
    prevSelected = selected

  result.add(points.last)           // always keep the last point
  return result
```

Output size: exactly `min(target, points.count)` points.

#### getPointsForRender

```text
getPointsForRender(line: Line, pixelWidth: float): iterable<Coordinate>
  visiblePoints = line.points.rangeQuery(minX, maxX)   // §15.4
  match line.downsampleMode:
    None:       return visiblePoints
    PixelExact: return pixelExact(visiblePoints, sX)
    MinMax:     return minMax(visiblePoints, sX)
    LTTB:       return lttb(visiblePoints, line.lttbTarget)
```

Note: `dataDensity` culling is applied **after** `getPointsForRender` in the
`connectPoints` loop (§11.9). Using both simultaneously is valid but unusual;
`downsampleMode` is generally the preferred mechanism for large datasets.

---

## 16. Known Bugs in Reference Implementation

Implementers should be aware of and correct these issues:

1. **StretchY / CompressY modify X instead of Y** (§4.3). The correct behavior modifies `MinY`/`MaxY`.

2. **BalancedTree.RightBranch setter assigns to leftBranch**:
   ```
   set { this.leftBranch = value; }   // bug: should be this.rightBranch
   ```
   Correct implementations should assign to `rightBranch`.

3. **BalancedTree is not self-balancing** despite its name. The structure degrades to a linked list for sorted inputs. Implementers may substitute an AVL tree, red-black tree, or a simple sorted array with binary search.

4. **GanttLayer creates UI widgets (WinForms Button) inside the draw loop**, which is both platform-specific and inefficient (new controls created every repaint). Reimplementations should decouple label rendering from the draw loop, creating/updating control representations once and rendering them from state.

---

## 17. Acceptance Checklist

Use this checklist to verify a correct implementation:

- [ ] `Coordinate.equals` returns true iff X and Y match exactly.
- [ ] `VColor` channels are in [0.0, 1.0]; 8-bit conversion uses floor(channel * 255).
- [ ] `CoordinateSystem.transform` is atomic: if the result is invalid, the original bounds are restored.
- [ ] `StretchY`/`CompressY` modify `MinY`/`MaxY` (not MinX/MaxX).
- [ ] `screenX(minX)` maps to the left margin pixel. `screenX(maxX)` maps to `pixelWidth - rightMargin`.
- [ ] `screenY(maxY)` maps to the top margin pixel. `screenY(minY)` maps to `pixelHeight - bottomMargin`.
- [ ] `unitX(screenX(x)) == x` for any x in [minX, maxX].
- [ ] `unitY(screenY(y)) == y` for any y in [minY, maxY].
- [ ] Axis clamping: if `minX > 0`, `screenX(0)` returns `screenX(minX)`.
- [ ] `circle(position, size)` draws a filled disc; visual radius is `size/2`.
- [ ] `renderText` renders at `(cursor.X, cursor.Y - 10)`.
- [ ] `drawRectangle(start, end, fillColor)` uses `abs(end-start)` for dimensions.
- [ ] `connectPoints`: out-of-viewport points before the range are skipped; after the range, the loop breaks. Off-screen rendering does not occur.
- [ ] `dataDensity = 1` renders every point; `dataDensity = N` renders every Nth point (starting from point 1).
- [ ] `beforeDrawPoint` event `cancel = true` prevents that point from rendering.
- [ ] `GanttLayer` rejects negative Y coordinate systems (`allowNegativeY = false`).
- [ ] `GanttLayer.addLine` auto-assigns `line.index` as the current count before insertion.
- [ ] `mouseClick` in `GanttLayer` selects both the line and bar under the cursor.
- [ ] Mouse events are dispatched to **all** registered layers (not just the first).
- [ ] Legend items are cleared and repopulated on every paint cycle.
- [ ] `AdvLineLayer.mouseUp` with `enableSelection = true` updates the coordinate system bounds from the rubber-band rectangle.

---

## 18. Minimal Implementation Plan

Recommended build order:

1. **Core types**: `Coordinate`, `VColor`, `VGradient`, `VRectangle`, all enums.
2. **CoordinateSystem** with transform and validity check.
3. **IDrawEngine** contract and `BaseDrawEngine` stub.
4. **IDrawSurface** contract.
5. **BaseLayer** with coordinate transform math. Verify against §15 formulas.
6. **`Line` data model** with sorted point storage and mutation methods.
7. **`LineLayer`** with full drawing sequence. Test against a concrete engine.
8. **A concrete `IDrawEngine`** for your target platform (Canvas, Cairo, Skia, etc.).
9. **A concrete `IDrawSurface`** for your target UI framework.
10. **`AdvLineLayer`** (builds on LineLayer logic).
11. **`GanttLayer`** with `GanttLine`, `GanttBar`, `VRange`.
12. **`ChartLegend`** / `LegendItem` and legend draw pass.
13. **Mouse event routing** in surface.
14. **`BalancedTree`** or equivalent point-lookup structure.
