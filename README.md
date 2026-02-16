# Gnuplot Studio

A browser-based visual editor for creating publication-quality plots and exporting them as ready-to-run **gnuplot scripts**. No installation required — open `index.html` in any modern browser and start plotting.

Gnuplot Studio bridges the gap between GUI convenience and gnuplot's powerful scripting model. You design your plot visually — choosing data columns, styling lines and points, arranging legends and annotations — and the app generates a clean `.gp` script that reproduces your figure exactly in gnuplot.

---

## What You Can Do

- **Plot data** from CSV, TSV, or whitespace-separated files (the same formats gnuplot reads)
- **Plot mathematical functions** like `sin(x)`, `exp(-x**2)`, or any expression gnuplot understands
- **Style everything visually** — line colors, widths, dash patterns, point types, fill opacity, error bars
- **Export a gnuplot script** that reproduces your figure with one command: `gnuplot plot.gp`
- **Export PNG** directly from the browser for quick sharing
- **Save/load projects** as JSON files to continue editing later

If you've ever spent time tweaking gnuplot commands by trial and error, this tool lets you see the result immediately and get the script when you're done.

---

## Key Features

### Data Handling (Gnuplot-Compatible)

Gnuplot Studio follows the same data model as gnuplot itself:

- **Data sources** hold your raw tabular data (like a `.dat` file in gnuplot)
- **Plots** reference a data source with a `using` clause that selects which columns to plot
- Multiple plots can share the same data source with different `using` columns — just like `plot 'data.dat' using 1:2, '' using 1:3` in gnuplot

The column selectors display **1-indexed** column numbers to match gnuplot convention, and a live preview shows the equivalent gnuplot command as you configure each plot:

```
-> plot 'mydata' using 1:3
```

Import data by pasting directly or loading a file. Lines starting with `#` are treated as comments. If the first row contains non-numeric values, it's used as column headers.

### Plot Types

| Type | Description |
|------|-------------|
| Lines | Connected line segments |
| Points | Scatter plot with configurable point shapes |
| Lines+Points | Both lines and markers |
| Boxes (Bars) | Bar chart with adjustable width and X offset |
| Impulses | Vertical lines from zero to each data point |
| Steps | Staircase plot |
| Fill Curves | Filled area under the curve |

### Styling Options (Per-Plot)

- **Color** picker with a 10-color default palette
- **Line width** (0.5 - 10), **dash type** (solid, dashed, dotted, dash-dot)
- **Point type** (circle, square, triangle, diamond, cross, plus), **point size** (1 - 20)
- **Fill alpha** (0 - 1) for boxes and fill curves
- **Smooth curves**: C-splines, Bezier, and Acsplines interpolation
- **Error bars**: select a Y-error column for yerrorbars / yerrorlines
- **X offset** and **bar width**: arrange side-by-side grouped bar charts by dragging data points or entering values

### Mathematical Functions

Toggle any plot to function mode and enter an expression:

```
sin(x) * exp(-x/5)
```

Supported functions: `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2`, `abs`, `sqrt`, `log` (natural), `log2`, `log10`, `exp`, `floor`, `ceil`, `pow`. Constants: `pi`, `PI`, `e`.

Configure the X domain and sample count (10 - 2000 points).

### Canvas Interactions

| Action | What it does |
|--------|-------------|
| **Scroll wheel** | Zoom in/out around cursor position |
| **Shift + drag** | Pan the view |
| **Drag a data point** | Adjust the plot's X offset (for bar charts) |
| **Drag margin edges** | Resize plot margins visually |
| **Drag the legend** | Reposition the legend box |
| **Drag legend right edge** | Resize legend width (triggers flow reflow) |
| **Double-click** | Add a text annotation at that data coordinate |
| **Hover** | Tooltip showing nearest point's coordinates |

### Gnuplot Script Export

Click **Export .gp** to generate a complete, self-contained gnuplot script. The export includes:

- Terminal and output configuration
- Figure size, margins, and background
- Axis ranges, log scales, and grid settings
- Legend position and styling
- Data blocks (`$data0 << EOD ... EOD`) with your data inline
- Plot commands with full styling: `using`, `with`, `lc rgb`, `lw`, `pt`, `ps`, `dt`, `smooth`, `title`
- Annotations as `set label` commands
- Any custom gnuplot commands you've added

The generated script is ready to run:

```bash
gnuplot plot.gp
```

Supported output terminals: **pngcairo**, **svg**, **pdfcairo**, **epscairo**, **tikz**, **qt** (interactive).

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+N` | Add new data plot |
| `Ctrl+D` | Duplicate selected plot |
| `Delete` | Remove selected plot |
| `Ctrl+Z` | Undo |
| `Ctrl+Y` / `Ctrl+Shift+Z` | Redo |
| `Ctrl+S` | Save project |
| `Ctrl+0` | Reset zoom (fit all data) |
| `?` | Show/hide shortcuts overlay |

### Project Persistence

- **Auto-save**: your session is saved to `localStorage` automatically and restored when you reopen the page
- **Manual save** (`Ctrl+S`): downloads a `.json` project file and writes to `localStorage`
- **Load**: open any saved `.json` project to restore the full editing state

---

## Getting Started

1. Open `index.html` in a browser (Chrome, Firefox, Safari, Edge)
2. A sample plot is created automatically — edit the data in the bottom-left textarea
3. Modify plot styling in the right panel
4. Click **Import Data** to load your own CSV/TSV data
5. Click **Export .gp** when you're happy with the result

No build step, no dependencies, no server. The entire application is three files: `index.html`, `style.css`, `app.js`.

---

## Architecture

This section covers the internal design for developers who want to extend or modify Gnuplot Studio.

### File Structure

```
gnuplot-studio/
  index.html   — DOM structure, modals, shortcuts overlay
  style.css    — Dark theme, three-panel layout, component styles
  app.js       — All application logic (~2,300 lines)
```

The app is intentionally a single-page application with zero dependencies. Everything runs client-side.

### State Model

A single global `state` object is the source of truth:

```
state
  figure          {width, height, bg, terminal}
  margin          {top, right, bottom, left}
  title, xlabel, ylabel, labelFontSize
  xaxis           {auto, min, max, log, grid, locked}
  yaxis           {auto, min, max, log, grid, locked}
  key             {show, pos, box, fontSize, horizontal, x, y, width}
  annotations[]   [{text, x, y, fontSize, color}]
  customCommands  string
  dataSources[]   [{id, name, rawData, headers, rows}]
  plots[]         [{id, name, type, color, lineWidth, pointSize, pointType,
                    dashType, fillAlpha, smooth, visible, dataSourceId,
                    usingX, usingY, usingYerr, hasErrorBars,
                    isFunction, funcExpr, funcXmin, funcXmax, funcSamples,
                    xOffset, barWidth, offsetLocked}]
  selectedPlot    index into plots[]
```

Two transient properties (`state._area` and `state._bounds`) are set during rendering and used by mouse interaction handlers for coordinate transforms.

### Data Flow

```
                  +-----------+
  Import/Paste -> | Parse     | -> dataSource {headers, rows}
  (CSV/TSV)       +-----------+
                       |
                       v
              +----------------+
  UI Change ->| State Update   | -> state.plots[], state.figure, etc.
              +----------------+
                       |
           +-----------+-----------+
           |                       |
           v                       v
    +------------+          +---------------+
    | render()   |          | generateGnuplotScript()
    | (Canvas)   |          | (Export .gp)  |
    +------------+          +---------------+
           |
           v
    +------------+
    | pushUndo() |  (debounced, 300ms)
    +------------+
```

Every state change flows through `render()`, which redraws the canvas. The rendering pipeline:

1. **Compute bounds** — scan all visible plot data to determine axis ranges (or use manual ranges)
2. **Compute plot area** — subtract margins from figure dimensions
3. **Draw background, grid, axes** — with tick marks and labels
4. **Draw each visible plot** — transform data coordinates to canvas pixels, apply smoothing/error bars
5. **Draw title, axis labels, annotations**
6. **Draw legend** — automatic flow layout that packs entries into rows

### Data Sources & the `using` Paradigm

This is the core design decision that mirrors gnuplot. A **data source** is a named block of tabular data (like a file). A **plot** references a data source by ID and specifies which columns to use:

```
plot.dataSourceId  ->  which data source
plot.usingX        ->  column index for X values (0-indexed internally)
plot.usingY        ->  column index for Y values
plot.usingYerr     ->  column index for Y error bars (optional)
```

The gnuplot export converts these to 1-indexed `using` clauses. Multiple plots can reference the same data source — the export deduplicates data blocks automatically.

### Coordinate System

Two coordinate spaces are used throughout:

- **Data coordinates**: the actual X/Y values from the data
- **Canvas coordinates**: pixel positions on the HTML canvas

Conversion functions handle the mapping, including log-scale transforms:

```
dataToCanvas(dx, dy, area, bounds) -> {cx, cy}
canvasToData(cx, cy, area, bounds) -> {dx, dy}
```

The `area` object defines the drawable rectangle (figure minus margins), and `bounds` defines the data range. Log-scale axes use log10 interpolation.

### Rendering

The canvas supports HiDPI displays by scaling to `devicePixelRatio`. All drawing uses the Canvas 2D API.

Plot-specific rendering:
- **Lines/Points/LinesPoints**: standard polyline/marker drawing with clipping
- **Boxes**: centered or offset rectangles with configurable width
- **Impulses**: vertical lines from Y=0 to each data point
- **Steps**: staircase segments
- **Fill Curves**: filled polygon between curve and Y=0
- **Error Bars**: vertical lines with horizontal caps
- **Smooth curves**: cubic spline or Bezier interpolation, densified to 12 sub-segments per interval

The **legend** uses a flow layout algorithm: entries are measured, packed into rows that fit the available width, and drawn as a floating box. The legend can be dragged to a custom position and resized by its right edge.

### Undo/Redo

The undo system takes JSON snapshots of the entire state. Snapshots are pushed to a stack (max 50) on a 300ms debounce after each `render()` call. Duplicate consecutive snapshots are skipped.

Undo pops a snapshot and pushes the current state to the redo stack. Restoration re-parses data sources from raw text and re-syncs all UI elements.

### Gnuplot Script Generation

`generateGnuplotScript()` walks the state and emits gnuplot commands line by line:

1. Terminal and output settings
2. Figure size and margins (converted to gnuplot screen coordinates)
3. Axis configuration (`set xrange`, `set logscale`, `set grid`)
4. Legend configuration (`set key`)
5. Labels and annotations (`set title`, `set xlabel`, `set label`)
6. Custom commands (pass-through)
7. Shared data blocks (deduplicated, one `$dataN << EOD` block per unique data source)
8. Plot command with all series joined by `, \` continuation

Each plot series emits the appropriate gnuplot syntax: `using`, `with`, style options, and `title`.

### Auto-Save & Project Files

Projects are saved as JSON matching the state structure. Auto-save writes to `localStorage` on a 1-second debounce after each render. Manual save also triggers a file download. Loading a project calls `applyProjectData()`, which validates and restores the state, re-parses all data sources, and re-initializes ID counters.

---

## Browser Compatibility

Tested on modern versions of Chrome, Firefox, Safari, and Edge. Requires:
- Canvas 2D API
- ES6+ (template literals, arrow functions, destructuring, `let`/`const`)
- `localStorage` for auto-save
- `Blob` / `URL.createObjectURL` for file downloads

---

## License

See [LICENSE](LICENSE) for details.
