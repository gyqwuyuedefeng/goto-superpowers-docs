# Sankey Gradient Links Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current Plotly Sankey generator with a D3/d3-sankey HTML generator whose links use source-to-target color gradients.

**Architecture:** `tmp/sankey.R` remains the single generator command. R defines the data, serializes it to JSON, and writes a complete HTML file. The generated page loads D3 and `d3-sankey`, computes the layout in browser JavaScript, draws SVG nodes and links, creates one `<linearGradient>` per link, and rebuilds on resize.

**Tech Stack:** R, `jsonlite`, generated HTML, SVG, D3 v7, `d3-sankey` v0.12.3.

---

## File Structure

- Modify: `tmp/sankey.R`
  - Responsibility: define the sample Sankey nodes/links and write a standalone D3-based `tmp/sankey.html`.
- Generated only: `tmp/sankey.html`
  - Responsibility: render the Sankey chart in the browser. Do not commit this generated file unless the repository already tracks generated previews.

No new reusable source files are needed. The implementation is intentionally contained in the one existing script because the current artifact is a small example generator.

---

### Task 1: Establish the Failing Smoke Check

**Files:**
- Modify: none
- Test target: `tmp/sankey.html`

- [ ] **Step 1: Generate the current HTML**

Run:

```bash
Rscript tmp/sankey.R
```

Expected: command completes and prints `已保存: tmp/sankey.html` or an equivalent saved message.

- [ ] **Step 2: Run the smoke check and verify it fails on the current Plotly output**

Run:

```bash
Rscript -e 'html <- paste(readLines("tmp/sankey.html", warn = FALSE), collapse = "\n"); stopifnot(grepl("d3-sankey", html, fixed = TRUE)); stopifnot(grepl("linearGradient", html, fixed = TRUE)); stopifnot(grepl("sankeyLinkHorizontal", html, fixed = TRUE)); stopifnot(!grepl("plotly", html, ignore.case = TRUE)); stopifnot(grepl("addEventListener(\"resize\"", html, fixed = TRUE)); cat("sankey html smoke checks passed\n")'
```

Expected: FAIL, because the current file is Plotly-based and does not contain `d3-sankey` rendering.

---

### Task 2: Replace Plotly Generator With D3 HTML Generator

**Files:**
- Modify: `tmp/sankey.R`
- Test target: `tmp/sankey.html`

- [ ] **Step 1: Replace the full content of `tmp/sankey.R`**

Use this complete file content:

```r
library(jsonlite)

nodes <- data.frame(
  name = c("A", "B", "C", "D", "E"),
  color = c("#ff9999", "#66b3ff", "#99ff99", "#ffcc99", "#c2c2f0"),
  stringsAsFactors = FALSE
)

links <- data.frame(
  source = c(0, 0, 1, 1, 2, 3),
  target = c(1, 2, 2, 3, 4, 4),
  value = c(8, 4, 6, 3, 5, 2)
)

nodes_json <- jsonlite::toJSON(
  nodes,
  dataframe = "rows",
  auto_unbox = TRUE,
  pretty = TRUE
)

links_json <- jsonlite::toJSON(
  links,
  dataframe = "rows",
  auto_unbox = TRUE,
  pretty = TRUE
)

html <- paste0(c(
  "<!doctype html>",
  "<html lang=\"en\">",
  "<head>",
  "  <meta charset=\"utf-8\">",
  "  <meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">",
  "  <title>D3 Sankey Gradient Links</title>",
  "  <style>",
  "    body {",
  "      margin: 0;",
  "      background: #ffffff;",
  "      color: #222222;",
  "      font-family: Arial, Helvetica, sans-serif;",
  "    }",
  "    .page {",
  "      max-width: 760px;",
  "      margin: 0 auto;",
  "      padding: 10px 12px;",
  "    }",
  "    #chart {",
  "      width: 100%;",
  "      min-height: 320px;",
  "    }",
  "    svg {",
  "      display: block;",
  "      width: 100%;",
  "      height: auto;",
  "      overflow: visible;",
  "    }",
  "    .link {",
  "      fill: none;",
  "      stroke-linecap: butt;",
  "      mix-blend-mode: multiply;",
  "    }",
  "    .node rect {",
  "      stroke: rgba(30, 30, 30, 0.55);",
  "      stroke-width: 1;",
  "      shape-rendering: crispEdges;",
  "    }",
  "    .node text {",
  "      pointer-events: none;",
  "      fill: #222222;",
  "      font-size: 11px;",
  "    }",
  "    .layer text {",
  "      fill: #333333;",
  "      font-size: 10px;",
  "    }",
  "    .layer line {",
  "      stroke: #333333;",
  "      stroke-width: 1;",
  "    }",
  "    .error {",
  "      padding: 24px;",
  "      border: 1px solid #f0b8b8;",
  "      background: #fff5f5;",
  "      color: #8a1f1f;",
  "      font-size: 14px;",
  "    }",
  "  </style>",
  "</head>",
  "<body>",
  "  <main class=\"page\">",
  "    <div id=\"chart\" aria-label=\"Sankey diagram with gradient links\"></div>",
  "  </main>",
  "  <script src=\"https://cdn.jsdelivr.net/npm/d3@7/dist/d3.min.js\"></script>",
  "  <script src=\"https://cdn.jsdelivr.net/npm/d3-sankey@0.12.3/dist/d3-sankey.min.js\"></script>",
  "  <script>",
  "    const rawNodes = ",
  nodes_json,
  ";",
  "    const rawLinks = ",
  links_json,
  ";",
  "",
  "    const chart = document.getElementById(\"chart\");",
  "    let resizeTimer = null;",
  "",
  "    function showError(message) {",
  "      chart.innerHTML = \"\";",
  "      const error = document.createElement(\"div\");",
  "      error.className = \"error\";",
  "      error.textContent = message;",
  "      chart.appendChild(error);",
  "    }",
  "",
  "    function validateData(nodes, links) {",
  "      const cleanNodes = nodes.map((node, index) => ({",
  "        id: index,",
  "        name: String(node.name || `Node ${index + 1}`),",
  "        color: /^#[0-9a-f]{6}$/i.test(String(node.color || \"\")) ? String(node.color) : \"#999999\"",
  "      }));",
  "",
  "      const cleanLinks = [];",
  "      links.forEach((link, index) => {",
  "        const source = Number(link.source);",
  "        const target = Number(link.target);",
  "        const value = Number(link.value);",
  "",
  "        if (!Number.isInteger(source) || !Number.isInteger(target)) {",
  "          console.warn(`Skipping link ${index}: source and target must be integer indexes.`);",
  "          return;",
  "        }",
  "        if (source < 0 || source >= cleanNodes.length || target < 0 || target >= cleanNodes.length) {",
  "          console.warn(`Skipping link ${index}: source or target index is out of range.`);",
  "          return;",
  "        }",
  "        if (!Number.isFinite(value) || value <= 0) {",
  "          console.warn(`Skipping link ${index}: value must be a positive number.`);",
  "          return;",
  "        }",
  "",
  "        cleanLinks.push({ source, target, value, index });",
  "      });",
  "",
  "      if (!cleanLinks.length) {",
  "        throw new Error(\"No valid Sankey links to render.\");",
  "      }",
  "",
  "      return { nodes: cleanNodes, links: cleanLinks };",
  "    }",
  "",
  "    function render() {",
  "      if (!window.d3 || !d3.sankey || !d3.sankeyLinkHorizontal) {",
  "        showError(\"D3 or d3-sankey failed to load. Check the CDN scripts and network access.\");",
  "        return;",
  "      }",
  "",
  "      let data;",
  "      try {",
  "        data = validateData(rawNodes, rawLinks);",
  "      } catch (error) {",
  "        showError(error.message);",
  "        return;",
  "      }",
  "",
  "      chart.innerHTML = \"\";",
  "",
  "      const width = Math.max(chart.clientWidth || 700, 360);",
  "      const height = Math.max(320, Math.min(520, Math.round(width * 0.58)));",
  "      const margin = { top: 10, right: 22, bottom: 38, left: 18 };",
  "",
  "      const svg = d3.select(chart)",
  "        .append(\"svg\")",
  "        .attr(\"viewBox\", `0 0 ${width} ${height}`)",
  "        .attr(\"role\", \"img\");",
  "",
  "      const graph = d3.sankey()",
  "        .nodeId(d => d.id)",
  "        .nodeWidth(18)",
  "        .nodePadding(24)",
  "        .nodeAlign(d3.sankeyJustify)",
  "        .extent([[margin.left, margin.top], [width - margin.right, height - margin.bottom]])({",
  "          nodes: data.nodes.map(d => ({ ...d })),",
  "          links: data.links.map(d => ({ ...d }))",
  "        });",
  "",
  "      const defs = svg.append(\"defs\");",
  "      graph.links.forEach((link, index) => {",
  "        const gradient = defs.append(\"linearGradient\")",
  "          .attr(\"id\", `link-gradient-${index}`)",
  "          .attr(\"gradientUnits\", \"userSpaceOnUse\")",
  "          .attr(\"x1\", link.source.x1)",
  "          .attr(\"y1\", link.y0)",
  "          .attr(\"x2\", link.target.x0)",
  "          .attr(\"y2\", link.y1);",
  "",
  "        gradient.append(\"stop\")",
  "          .attr(\"offset\", \"0%\")",
  "          .attr(\"stop-color\", link.source.color)",
  "          .attr(\"stop-opacity\", 0.82);",
  "",
  "        gradient.append(\"stop\")",
  "          .attr(\"offset\", \"100%\")",
  "          .attr(\"stop-color\", link.target.color)",
  "          .attr(\"stop-opacity\", 0.82);",
  "      });",
  "",
  "      svg.append(\"g\")",
  "        .attr(\"class\", \"links\")",
  "        .selectAll(\"path\")",
  "        .data(graph.links)",
  "        .join(\"path\")",
  "        .attr(\"class\", \"link\")",
  "        .attr(\"d\", d3.sankeyLinkHorizontal())",
  "        .attr(\"stroke\", (d, index) => `url(#link-gradient-${index})`)",
  "        .attr(\"stroke-width\", d => Math.max(1, d.width))",
  "        .attr(\"stroke-opacity\", 0.74);",
  "",
  "      const node = svg.append(\"g\")",
  "        .attr(\"class\", \"nodes\")",
  "        .selectAll(\"g\")",
  "        .data(graph.nodes)",
  "        .join(\"g\")",
  "        .attr(\"class\", \"node\");",
  "",
  "      node.append(\"rect\")",
  "        .attr(\"x\", d => d.x0)",
  "        .attr(\"y\", d => d.y0)",
  "        .attr(\"width\", d => d.x1 - d.x0)",
  "        .attr(\"height\", d => Math.max(1, d.y1 - d.y0))",
  "        .attr(\"fill\", d => d.color);",
  "",
  "      node.append(\"text\")",
  "        .attr(\"x\", d => (d.x0 + d.x1) / 2)",
  "        .attr(\"y\", d => (d.y0 + d.y1) / 2)",
  "        .attr(\"dy\", \"0.35em\")",
  "        .attr(\"text-anchor\", \"middle\")",
  "        .text(d => d.name);",
  "",
  "      const layers = Array.from(",
  "        d3.rollup(",
  "          graph.nodes,",
  "          values => d3.mean(values, d => (d.x0 + d.x1) / 2),",
  "          d => d.depth",
  "        ),",
  "        ([depth, x]) => ({ depth, x, label: `L${depth + 1}` })",
  "      ).sort((a, b) => a.depth - b.depth);",
  "",
  "      const layer = svg.append(\"g\")",
  "        .attr(\"class\", \"layers\")",
  "        .selectAll(\"g\")",
  "        .data(layers)",
  "        .join(\"g\")",
  "        .attr(\"class\", \"layer\")",
  "        .attr(\"transform\", d => `translate(${d.x}, ${height - 14})`);",
  "",
  "      layer.append(\"line\")",
  "        .attr(\"x1\", 0)",
  "        .attr(\"x2\", 0)",
  "        .attr(\"y1\", -7)",
  "        .attr(\"y2\", -3);",
  "",
  "      layer.append(\"text\")",
  "        .attr(\"text-anchor\", \"middle\")",
  "        .attr(\"dy\", \"0.9em\")",
  "        .text(d => d.label);",
  "    }",
  "",
  "    window.addEventListener(\"resize\", () => {",
  "      window.clearTimeout(resizeTimer);",
  "      resizeTimer = window.setTimeout(render, 120);",
  "    });",
  "",
  "    render();",
  "  </script>",
  "</body>",
  "</html>"
), collapse = "\n")

output_file <- "/mnt/f/IdeaProjects/goto-software/tmp/sankey.html"
dir.create(dirname(output_file), recursive = TRUE, showWarnings = FALSE)
writeLines(html, output_file, useBytes = TRUE)
cat("Saved: tmp/sankey.html\n")
```

- [ ] **Step 2: Generate the new HTML**

Run:

```bash
Rscript tmp/sankey.R
```

Expected: command completes and prints `Saved: tmp/sankey.html`.

---

### Task 3: Verify Generated HTML Contract

**Files:**
- Modify: none
- Test target: `tmp/sankey.html`

- [ ] **Step 1: Run the smoke check again**

Run:

```bash
Rscript -e 'html <- paste(readLines("tmp/sankey.html", warn = FALSE), collapse = "\n"); stopifnot(grepl("d3-sankey", html, fixed = TRUE)); stopifnot(grepl("linearGradient", html, fixed = TRUE)); stopifnot(grepl("sankeyLinkHorizontal", html, fixed = TRUE)); stopifnot(!grepl("plotly", html, ignore.case = TRUE)); stopifnot(grepl("addEventListener(\"resize\"", html, fixed = TRUE)); cat("sankey html smoke checks passed\n")'
```

Expected: PASS and prints `sankey html smoke checks passed`.

- [ ] **Step 2: Inspect the relevant generated HTML lines**

Run:

```bash
rg -n "d3-sankey|linearGradient|sankeyLinkHorizontal|addEventListener\\(\"resize\"|Plotly|plotly" tmp/sankey.html
```

Expected:

- Matches for `d3-sankey`, `linearGradient`, `sankeyLinkHorizontal`, and `addEventListener("resize"`.
- No matches for `Plotly` or `plotly`.

- [ ] **Step 3: Confirm git diff is scoped**

Run:

```bash
git diff -- tmp/sankey.R tmp/sankey.html
```

Expected: `tmp/sankey.R` is replaced with the D3 generator. `tmp/sankey.html` may appear only if it is tracked; if it is untracked, it will not appear in this diff.

---

### Task 4: Visual Browser Check

**Files:**
- Modify: none
- Test target: `tmp/sankey.html`

- [ ] **Step 1: Open the generated file in a browser**

Open:

```text
/mnt/f/IdeaProjects/goto-software/tmp/sankey.html
```

Expected visual result:

- Nodes A-E use the configured colors.
- Links are not gray.
- Each link visibly transitions from the source node color to the target node color.
- The layout resembles the ggplot2 reference more than the current Plotly output: thinner nodes, soft curved links, and bottom layer labels.

- [ ] **Step 2: Resize the browser window**

Expected: the chart redraws without detached gradients, missing links, or overlapping node labels.

---

### Task 5: Commit Implementation

**Files:**
- Modify: `tmp/sankey.R`
- Generated: `tmp/sankey.html` only if already tracked

- [ ] **Step 1: Check tracked status for generated HTML**

Run:

```bash
git ls-files -- tmp/sankey.html
```

Expected:

- If output is empty, do not add `tmp/sankey.html`.
- If output is `tmp/sankey.html`, include it in the commit.

- [ ] **Step 2: Commit the scoped implementation**

If `tmp/sankey.html` is not tracked, run:

```bash
git add tmp/sankey.R
git commit -m "feat: render sankey links with d3 gradients"
```

If `tmp/sankey.html` is tracked, run:

```bash
git add tmp/sankey.R tmp/sankey.html
git commit -m "feat: render sankey links with d3 gradients"
```

Expected: commit succeeds with only the Sankey generator change and, if tracked, its generated HTML update.
