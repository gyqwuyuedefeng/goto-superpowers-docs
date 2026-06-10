# Sankey Gradient Links Design

## Goal

Update `tmp/sankey.R` so it generates `/mnt/f/IdeaProjects/goto-software/tmp/sankey.html` as a standalone HTML Sankey chart whose links use gradients based on each link's source and target node colors.

The final visual target is closer to the ggplot2 reference image: colored nodes, soft curved ribbons, and per-link source-to-target color gradients. The current Plotly output with gray links is not acceptable.

## Scope

In scope:

- Replace the Plotly Sankey rendering path with a custom HTML/SVG renderer.
- Allow external JavaScript dependencies.
- Use D3 and `d3-sankey` in the generated HTML.
- Preserve the existing sample nodes and links from `tmp/sankey.R`.
- Keep the output path as `tmp/sankey.html`.
- Redraw responsively when the browser window changes size.
- Show a useful page-level error when D3 dependencies fail to load or the data cannot be rendered.

Out of scope:

- Recreating Plotly hover interactions.
- Building a reusable R package.
- Changing unrelated project files.
- Supporting arbitrary chart types beyond this Sankey example.

## Architecture

`tmp/sankey.R` will become a small generator:

1. Define `nodes` and `links` in R.
2. Serialize both data frames to JSON with `jsonlite`.
3. Write an HTML document to `tmp/sankey.html`.
4. The HTML loads D3 and `d3-sankey` from external CDN URLs.
5. Browser JavaScript renders the chart directly into an SVG.

This avoids depending on Plotly's internal Sankey DOM structure. Link gradients, path geometry, resize behavior, and error handling are controlled by the generated JavaScript.

## Data Model

Nodes keep the current fields:

- `name`: display label, such as `A`.
- `color`: node color, such as `#ff9999`.

Links keep the current fields:

- `source`: 0-based source node index.
- `target`: 0-based target node index.
- `value`: numeric flow value.

The JavaScript converts indexes into `d3-sankey` node references. It validates that each link points to existing nodes and has a finite positive value before rendering.

## Rendering

The SVG renderer will:

- Use `d3.sankey()` to compute node and link layout.
- Draw links first as wide curved paths using `d3.sankeyLinkHorizontal()`.
- Draw nodes above links as filled rectangles using each node's color.
- Draw node labels near or inside nodes depending on available space.
- Draw lightweight bottom layer labels (`L1`, `L2`, `L3`, `L4`) based on node x positions, matching the style of the ggplot2 reference.

Each link gets a unique `<linearGradient>` in SVG `<defs>`.

Gradient rules:

- `x1/y1` use the source node's right-center point.
- `x2/y2` use the target node's left-center point.
- The first stop uses the source node color.
- The second stop uses the target node color.
- Link opacity remains soft enough to show overlapping flows without returning to gray.

## Responsiveness

The page will render into a fixed chart container and compute dimensions from the container width. On `resize`, rendering is debounced and the SVG is rebuilt. Rebuilding the SVG also rebuilds gradients, so link colors remain aligned after layout changes.

## Error Handling

The generated page will show a short message in the chart area if:

- D3 or `d3-sankey` is unavailable.
- The embedded JSON cannot be parsed.
- No valid links remain after validation.

Invalid individual links are skipped and logged to the browser console with the reason.

## Verification

Verification will include:

- Run `Rscript tmp/sankey.R`.
- Confirm `tmp/sankey.html` is regenerated.
- Inspect the generated HTML for D3, `d3-sankey`, and `<linearGradient>` rendering logic.
- Open the HTML when possible and confirm that links visibly transition from source node color to target node color instead of gray.

## Acceptance Criteria

- `tmp/sankey.html` opens as a Sankey diagram without Plotly.
- Link colors are gradients derived from the two endpoint node colors.
- The visual result is closer to the ggplot2 reference than the current gray Plotly output.
- Resizing the browser does not leave gradients detached from links.
- The R script remains a single command generator for the HTML output.
