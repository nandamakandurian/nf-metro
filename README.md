# nf-metro

**[Documentation](https://pinin4fjords.github.io/nf-metro/latest/)**

> **Fork note** — this is the DurianPay fork of
> [`pinin4fjords/nf-metro`](https://github.com/pinin4fjords/nf-metro). It adds the
> `%%metro line_offset:` directive (per-map track spacing) and an
> [AI-authoring quickstart](#authoring-maps-with-an-ai-agent) so an agent can go
> from a domain description to a valid `.mmd` without reading layout internals.
> Used by `durian-oracle`'s `wiki/flows/` layer.

Generate metro-map-style SVG diagrams from Mermaid graph definitions with `%%metro` directives. Designed for visualizing bioinformatics pipeline workflows (e.g., nf-core pipelines) as transit-style maps where each analysis route is a colored "metro line."

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/pinin4fjords/nf-metro/main/examples/rnaseq_light_animated.svg">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/pinin4fjords/nf-metro/main/examples/rnaseq_light_animated.svg">
  <img alt="nf-core/rnaseq metro map" src="https://raw.githubusercontent.com/pinin4fjords/nf-metro/main/examples/rnaseq_auto_light.png">
</picture>

## Installation

### pip (PyPI)

```bash
pip install nf-metro
```

### Conda (Bioconda)

```bash
conda install bioconda::nf-metro
```

### Container (Seqera Containers)

A pre-built container is available via [Seqera Containers](https://seqera.io/containers/):

```bash
docker pull community.wave.seqera.io/library/pip_nf-metro:611b1ba39c6007f1
```

### Development

```bash
pip install -e ".[dev]"
```

Requires Python 3.10+.

## Quick start

Render a metro map from a `.mmd` file:

```bash
nf-metro render examples/simple_pipeline.mmd -o pipeline.svg
```

Validate your input without rendering:

```bash
nf-metro validate examples/simple_pipeline.mmd
```

Inspect structure (sections, lines, stations):

```bash
nf-metro info examples/simple_pipeline.mmd
```

## Authoring maps with an AI agent

> This section is a self-contained quickstart for an LLM agent that has been
> handed a domain (a pipeline, a money flow, an ops process) and needs to emit a
> **valid, well-laid-out `.mmd`** without reading the layout source. Read this
> section + the directive table + one example and you have enough to author.

### The whole model in three primitives

A metro map is exactly three kinds of thing:

1. **Lines** — the colored *routes*. A line is anything a reader traces
   end-to-end and asks "where does *this one* go?". (A pipeline variant, a
   payment method, an exception class.) Declared once globally.
2. **Sections + stations** — the *places*. A `subgraph` is a section (a stage /
   service / actor); the nodes inside are stations. A section with one station is
   fine and common.
3. **Edges** — the *hops*. `a -->|line1,line2| b` says "these lines travel from
   a to b". The line list on an edge is an assertion: exactly those routes take
   this hop.

That is the entire conceptual surface. Everything else is layout hinting.

### Author nodes by port-role (the reliable pattern)

The single most common cause of an ugly map is inflow and outflow sharing a
side, or a line passing *under* a node it shouldn't touch. Avoid both by giving
every node a **role** and setting its `entry:`/`exit:` hints from the role:

| Role | Ports | Use for |
|------|-------|---------|
| **source** | exit right only | upstream origins (providers, inputs, "already here") |
| **through** | entry left → exit right | a step *every* line on it passes through |
| **branch** | entry left → exit right, **members only** | a step only *some* lines take; non-members route around it |
| **sink** | entry left only | terminal states (dead-ends, outputs) |

Two invariants this enforces, and that you should be able to defend for every node:

- **Direction**: inflow and outflow never share a side.
- **Ownership**: a node sits only on the lines it actually serves; give each node
  its own `subgraph` and route *only* member lines through it. Non-members bypass.

A clean way to author at scale is to keep the role model as data (lines, typed
nodes, edges) and emit the `.mmd` from it, so the invariants hold by
construction rather than by hand. See
[`examples/`](examples/) for hand-written maps and the directive table below for
every knob.

### Minimal complete example

```
%%metro title: Tiny flow
%%metro style: light
%%metro legend: right
%%metro line: a | Route A | #0064B0
%%metro line: b | Route B | #E2231A

graph LR
    subgraph src [Source]
        %%metro exit: right | a,b
        src_n[inputs]
    end
    subgraph work [Process]
        %%metro entry: left | a,b
        %%metro exit: right | a,b
        work_n[do the thing]
    end
    subgraph done [Done]
        %%metro entry: left | a,b
        done_n[result]
    end

    src_n -->|a,b| work_n
    work_n -->|a,b| done_n
```

### The author → validate → render loop

```bash
nf-metro validate flow.mmd   # parse + structural checks, no output. Fix errors first.
nf-metro info flow.mmd        # echo parsed sections / lines / stations to confirm intent
nf-metro render flow.mmd -o flow.svg  --theme light
nf-metro render flow.mmd -o flow.html --format html --theme light   # interactive: click a legend line to isolate it
```

Always `validate` before `render`. If a dense map with few lines looks squished,
spread the parallel tracks with `%%metro line_offset: <px>` (see directive table).

### Layout knobs you will actually reach for

- `%%metro grid: <section> | col,row` — pin a section when auto-layout places it
  awkwardly. Most maps need none.
- `%%metro line_offset: <px>` — per-map track separation for parallel bundles /
  forks. Raise it (e.g. `11`) to de-squish dense maps.
- `%%metro compact_offsets: true` — compact per-station offsets; good for dense
  maps with few lines.
- `%%metro line_order: span` — give longest-spanning lines the inner tracks.
- `%%metro note: <station_id> | <text>` — attach a smaller detail line below a
  station (the table / topic / config key / queue / state it touches), so the node
  label stays readable while the precise detail rides underneath. Use `\n` for
  multiple small lines.
- `%%metro card: <section_id> | <line>` — describe a stage's internal sub-steps as a
  **formatted text block inside the box** (header/bold/italic/underline/divider/
  alignment), with the line running straight through one pass-through station. Prefer
  this over stacking several stations in a box (which forces the line to weave). The
  box auto-sizes to the card.
- Line style 4th field — `dashed` / `dotted` to mark exceptional routes
  (unconfirmed, fallback, side-paths) so the eye separates them from the happy path.

## CLI reference

### `nf-metro render`

Render a Mermaid metro map definition to SVG or interactive HTML.

```
nf-metro render [OPTIONS] INPUT_FILE
```

| Option | Default | Description |
|--------|---------|-------------|
| `-o`, `--output PATH` | `<input>.<format>` | Output file path |
| `--format [svg\|html]` | `svg` | Output format: `svg` or interactive `html` |
| `--theme [nfcore\|light]` | `nfcore` | Visual theme |
| `--width INTEGER` | auto | SVG width in pixels |
| `--height INTEGER` | auto | SVG height in pixels |
| `--x-spacing FLOAT` | `60` | Horizontal spacing between layers |
| `--y-spacing FLOAT` | `40` | Vertical spacing between tracks |
| `--max-layers-per-row INTEGER` | auto | Max layers before folding to next row |
| `--animate / --no-animate` | off | Add animated balls traveling along lines |
| `--debug / --no-debug` | off | Show debug overlay (ports, hidden stations, edge waypoints) |
| `--logo PATH` | none | Logo image path (overrides `%%metro logo:` directive) |
| `--line-order [definition\|span]` | from file | Line ordering strategy: `definition` preserves `.mmd` order, `span` sorts by section span (longest first) |
| `--straight-diamonds / --no-straight-diamonds` | on | Keep top branch of diamond fork-joins on the main track. Use `--no-straight-diamonds` for symmetric fan-out. |
| `--center-ports / --no-center-ports` | off | Centre inter-section ports on the shorter of the two connected sections |
| `--section-x-gap FLOAT` | `50` | Horizontal gap between sections |
| `--section-y-gap FLOAT` | `40` | Vertical gap between sections |
| `--from-nextflow` | off | Convert Nextflow `-with-dag` mermaid input before rendering |
| `--title TEXT` | none | Pipeline title (used with `--from-nextflow`) |

The `--logo` flag lets you use the same `.mmd` file with different logos for dark/light themes:

```bash
nf-metro render pipeline.mmd -o pipeline_dark.svg --theme nfcore --logo logo_dark.png
nf-metro render pipeline.mmd -o pipeline_light.svg --theme light --logo logo_light.png
```

#### Interactive HTML output

`--format html` produces a self-contained `.html` file with the SVG inlined plus a small JS/CSS layer (no external dependencies, no network):

```bash
nf-metro render pipeline.mmd --format html -o pipeline.html
```

The page provides:

- **Drag to pan**, **scroll to zoom** (Cmd/Ctrl+scroll in embedded mode).
- **Hover a station** to see its label, section, and the lines passing through it.
- **Click a line in the legend** to isolate it. Stations and sections not carrying that line disappear and the view zooms to the bounding box of what remains. Click again, hit `Esc`, or use the **Reset** button to restore.
- **Embed&hellip;** opens a copy-snippet panel with three options:
  - **Inline HTML** - a self-contained `<div>` you paste into any HTML host (MkDocs, Confluence, Notion, blog templates). Keeps full interactivity, no iframe.
  - **iframe** - a one-liner pointing at the hosted `.html` file.
  - **Static SVG** - the raw `<svg>` markup for contexts that strip scripts.

GitHub READMEs strip `<script>` tags, so embed there as a static SVG (or link out to a hosted version). Most static-site generators and internal wikis run the inline-HTML snippet as-is.

### `nf-metro validate`

Check a `.mmd` file for errors without producing output.

```
nf-metro validate INPUT_FILE
```

### `nf-metro info`

Print a summary of the parsed map: sections, lines, stations, and edges.

```
nf-metro info INPUT_FILE
```

## Examples

The [`examples/`](examples/) directory contains ready-to-render `.mmd` files:

| Example | Description |
|---------|-------------|
| [`simple_pipeline.mmd`](examples/simple_pipeline.mmd) | Minimal two-line pipeline with no sections |
| [`rnaseq_auto.mmd`](examples/rnaseq_auto.mmd) | nf-core/rnaseq with fully auto-inferred layout |
| [`rnaseq_sections.mmd`](examples/rnaseq_sections.mmd) | nf-core/rnaseq with manual grid overrides |

### Topology gallery

The [`examples/topologies/`](examples/topologies/) directory has 15 examples covering a range of layout patterns. See the [topology README](examples/topologies/README.md) for descriptions and rendered previews.

A few highlights:

| | | |
|:---:|:---:|:---:|
| **Wide Fan-Out** | **Section Diamond** | **Variant Calling** |
| ![Wide Fan-Out](examples/topologies/wide_fan_out.png) | ![Section Diamond](examples/topologies/section_diamond.png) | ![Variant Calling](examples/topologies/variant_calling.png) |
| **Fold Serpentine** | **Multi-Line Bundle** | **RNA-seq Lite** |
| ![Fold Double](examples/topologies/fold_double.png) | ![Multi-Line Bundle](examples/topologies/multi_line_bundle.png) | ![RNA-seq Lite](examples/topologies/rnaseq_lite.png) |

## Input format

Input files use a subset of Mermaid `graph LR` syntax extended with `%%metro` directives. The format has three layers: **global directives** that configure the overall map, **section directives** inside `subgraph` blocks that control section layout, and **edges** that define connections between stations.

### Walkthrough: nf-core/rnaseq

The full example is at [`examples/rnaseq_sections.mmd`](examples/rnaseq_sections.mmd). Here's how each part works.

#### Global directives

```
%%metro title: nf-core/rnaseq
%%metro logo: examples/nf-core-rnaseq_logo_dark.png
%%metro style: dark
```

- `title:` sets the map title (shown top-left unless a logo is provided)
- `logo:` embeds a PNG image in place of the text title
- `style:` selects a theme (`dark` or `light`)

#### Lines (routes)

Each metro line represents a distinct path through the pipeline. Lines are defined with an ID, display name, and color:

```
%%metro line: star_rsem | Aligner: STAR, Quantification: RSEM | #0570b0
%%metro line: star_salmon | Aligner: STAR, Quantification: Salmon (default) | #2db572
%%metro line: hisat2 | Aligner: HISAT2, Quantification: None | #f5c542
%%metro line: pseudo_salmon | Pseudo-aligner: Salmon, Quantification: Salmon | #e63946
%%metro line: pseudo_kallisto | Pseudo-aligner: Kallisto, Quantification: Kallisto | #7b2d3b
```

In the rnaseq pipeline, each line corresponds to a parameter-driven analysis route. All five lines share the preprocessing section, then diverge based on aligner choice.

#### Grid placement

Sections are placed on a grid automatically via topological sort, but explicit positions can be set:

```
%%metro grid: postprocessing | 2,0,2
%%metro grid: qc_report | 1,2,1,2
```

The format is `section_id | col,row[,rowspan[,colspan]]`. In this example:
- `postprocessing` is pinned to column 2, row 0, spanning 2 rows vertically
- `qc_report` is pinned to column 1, row 2, spanning 2 columns horizontally

#### Legend

```
%%metro legend: bl
```

Position the legend: `tl`, `tr`, `bl`, `br` (corners), `bottom`, `right`, or `none`.

#### Sections

Sections are Mermaid `subgraph` blocks. Each section is laid out independently, then placed on the grid:

```
graph LR
    subgraph preprocessing [Pre-processing]
        %%metro exit: right | star_salmon, star_rsem, hisat2
        %%metro exit: bottom | pseudo_salmon, pseudo_kallisto
        cat_fastq[cat fastq]
        fastqc_raw[FastQC]
        ...
    end
```

**Section directives:**

- `%%metro entry: <side> | <line_ids>` - declares which lines enter this section and from which side (`left`, `right`, `top`, `bottom`)
- `%%metro exit: <side> | <line_ids>` - declares which lines exit and to which side
- `%%metro direction: <dir>` - section flow direction: `LR` (default), `RL` (right-to-left), or `TB` (top-to-bottom)

Entry/exit hints control port placement on section boundaries. A section can have exit hints on multiple sides (e.g., preprocessing exits right for aligners and bottom for pseudo-aligners), but all lines from a section leave through a single exit port. If all exit hints point to one side, that side is used; otherwise it defaults to `right`.

#### Section directions

Most sections flow left-to-right (`LR`, the default). Two other directions are useful for layout:

**Top-to-bottom (`TB`)** - used for the Post-processing section, which acts as a vertical connector carrying lines downward:

```
    subgraph postprocessing [Post-processing]
        %%metro direction: TB
        %%metro entry: left | star_salmon, star_rsem, hisat2
        %%metro exit: bottom | star_salmon, star_rsem, hisat2
        samtools[SAMtools]
        picard[Picard]
        ...
    end
```

**Right-to-left (`RL`)** - used for the QC section, which flows backward to create a serpentine layout:

```
    subgraph qc_report [Quality control & reporting]
        %%metro direction: RL
        %%metro entry: top | star_salmon, star_rsem, hisat2
        rseqc[RSeQC]
        preseq[Preseq]
        ...
    end
```

#### Stations and edges

Stations use Mermaid node syntax. Edges carry comma-separated line IDs to indicate which routes use that connection:

```
        cat_fastq[cat fastq]
        fastqc_raw[FastQC]

        cat_fastq -->|star_salmon,star_rsem,hisat2,pseudo_salmon,pseudo_kallisto| fastqc_raw
```

All five lines pass through this edge. Later, lines diverge:

```
        star -->|star_rsem| rsem
        star -->|star_salmon| umi_tools_dedup
        hisat2_align -->|hisat2| umi_tools_dedup
```

Here different lines take different paths through the section, creating the visual fork in the metro map.

#### Inter-section edges

Edges between stations in different sections go outside all `subgraph`/`end` blocks:

```
    %% Inter-section edges
    sortmerna -->|star_salmon,star_rsem| star
    sortmerna -->|hisat2| hisat2_align
    sortmerna -->|pseudo_salmon| salmon_pseudo
    sortmerna -->|pseudo_kallisto| kallisto
    stringtie -->|star_salmon,star_rsem,hisat2| rseqc
```

These are automatically rewritten into port-to-port connections with junction stations at fan-out points. You just specify the source and target stations directly.

### Directive reference

| Directive | Scope | Description |
|-----------|-------|-------------|
| `%%metro title: <text>` | Global | Map title |
| `%%metro logo: <path>` | Global | Logo image (replaces title text) |
| `%%metro style: <name>` | Global | Theme: `dark`, `light` |
| `%%metro line: <id> \| <name> \| <color> [\| <style>]` | Global | Define a metro line. Optional style: `solid` (default), `dashed`, `dotted` |
| `%%metro grid: <section> \| <col>,<row>[,<rowspan>[,<colspan>]]` | Global | Pin section to grid position |
| `%%metro legend: <position>` | Global | Legend position: `tl`, `tr`, `bl`, `br`, `bottom`, `right`, `none` |
| `%%metro line_order: <strategy>` | Global | Line ordering for track assignment: `definition` (default) or `span` (longest-spanning lines get inner tracks) |
| `%%metro file: <station> \| <label>` | Global | Mark a station as a file terminus with a document icon |
| `%%metro files: <station> \| <label>` | Global | Mark a station with a stacked-documents icon (e.g. paired files) |
| `%%metro dir: <station> \| <label>` | Global | Mark a station with a folder icon (e.g. output directory) |
| `%%metro compact_offsets: true` | Global | Use compact per-station offsets instead of global line-priority slots (better for dense maps with few lines) |
| `%%metro line_offset: <px>` | Global | Per-line track separation for parallel bundles & forks (overrides the `OFFSET_STEP` default). Larger values de-squish dense maps. _(DurianPay fork addition.)_ |
| `%%metro note: <station_id> \| <text>` | Global | Secondary annotation rendered as **smaller muted text below the station label** (use `\n` for multiple small lines). Keeps a readable label on the node and the precise detail (table, topic, config key, queue, state) just beneath it. _(DurianPay fork addition.)_ |
| `%%metro card: <section_id> \| <line>` | Global | Repeatable. Adds one line to a section's **rich-text description card**, rendered inside the section box above the (single) pass-through station. Per-line markdown: `# header`, `## subheader`, `**bold**`, `*italic*`, `__underline__`, `---` divider, optional leading `\|c`/`\|r` alignment (default left). The box auto-sizes (width + height) to fit. Use this to describe a stage's internal sub-steps as text instead of stacking fake stations. _(DurianPay fork addition.)_ |
| `%%metro legend_min_height: <pixels>` | Global | Minimum legend content height in pixels (useful for single-line maps where the logo would otherwise be tiny) |
| `%%metro entry: <side> \| <lines>` | Section | Entry port hint |
| `%%metro exit: <side> \| <lines>` | Section | Exit port hint |
| `%%metro direction: <dir>` | Section | Flow direction: `LR`, `RL`, `TB` |

## License

[MIT](LICENSE)
