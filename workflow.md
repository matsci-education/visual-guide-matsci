# Notebook → deployed page: the workflow

A runbook for converting any Wolfram `.nb` into a MyST page in this book, with its
interactive content deployed to the Wolfram Cloud. Written to be followed by a human or by
an AI agent working unattended.

`wolfram.py` (vendored at the book root) is the authority on everything here; its module
docstring is the short version. [`README.md`](README.md) covers the day-to-day build.

For a worked result, compare `notebooks/Nucleation_Local_Global_Equilibrium_LGMC.nb`
against [`03.nucleation_and_hysteresis.md`](03.nucleation_and_hysteresis.md).

```
your.nb ──(1) prepare──▶ your.nb ──(2) --scaffold──▶ NN.page.md ──(3) edit──▶ NN.page.md
                                                                                  │
                             deployments.json ◀──(5) deploy──── (4) build ────────┘
                                                                     │
                                                          (6) open widgets, look
```

Steps 1–3 are authoring. Steps 4–6 are verification, and **step 6 is not optional** — a
cell can deploy successfully and still render blank.

---

## Decide: which embedding path?

| | `{wolfram}` cells | `{wolfram-notebook}` |
| --- | --- | --- |
| Unit of deployment | one cloud object per tagged expression | the whole `.nb` as one object |
| Page source | markdown you own and edit | a single directive line |
| Setup runs in the build kernel | **yes** | no |
| Definitions reach the reader via | `IncludeDefinitions` at publish time | `SaveDefinitions -> True` baked into the saved output cell |
| Prose | markdown, diffable | whatever the notebook says |
| Choose when | the notebook loads a package, computes anything, or you want real prose | a small self-contained demo |

**Default to `{wolfram}` cells.** Only that path evaluates your setup in the build kernel,
which is what a notebook that `Get`s a package needs. `{wolfram-notebook}` publishes the
notebook *as last saved* — if you have not evaluated it in the front end with
`SaveDefinitions -> True`, the reader gets dead controls.

`{wolfram-notebook}` usage:

```markdown
:::{wolfram-notebook} notebooks/your-notebook.nb
:::
```

It accepts a repo-local `.nb` path or a public cloud URL, plus `:hide-input:` to collapse
Input cells. The rest of this document covers the `{wolfram}` path.

---

## The author contract

Six rules, all enforced by `wolfram.py`.

**1. Tag what you want deployed.** A full-line Wolfram comment marks the expression to
publish:

```wl
(* #| deploy *)
(* #| label: fig:some-name *)
```

Lines *above* the marker are setup; the region *below* it is evaluated and its value
becomes the widget. The label makes the widget a cross-reference target (`@fig:some-name`)
and, with `:caption:`, a numbered figure. Marker lines are hidden from `:echo:` output, and
renaming a label does **not** invalidate a cached deployment.

**2. An untagged cell deploys nothing.** It still evaluates, so it is how you supply
definitions to later cells. This is the default; only tag cells that should produce a
figure.

**3. One marker per cell, and it must be followed by an expression.** A tagged region
ending in `;` evaluates to `Null`, which the plugin refuses to publish.

**4. `SaveDefinitions -> True` on every `Manipulate`.** Costs nothing on this path, and is
load-bearing on the `{wolfram-notebook}` path. See *Definitions* below.

**5. One kernel session per page, document order.** Later cells see earlier definitions.
Digests are **chained** — cell N's digest covers cells 1…N — so editing any cell
invalidates the cached deployment of every deployable cell below it. Prose, captions,
`:echo:`, and label names are not hashed.

**6. Deployment is opt-in.** It happens only when `JUPYTER_BOOK_WOLFRAM_DEPLOY=1` *and*
`wolframscript` is on `PATH`. Otherwise an unresolved cell renders a warning admonition
carrying its digest, and the build still succeeds.

### Directive options

`{wolfram}`: `echo`, `caption`, `url`, `defer`, `width`, `max-height`, `border`,
`interact`, `progress`, `shadow`, `css`, `class`.
`{wolfram-notebook}`: the display options above plus `hide-input`.

`:url:` points a cell at an existing public cloud object instead of deploying.
`:defer: true` skips publishing this cell for now.

---

## Step 1 — Prepare the notebook

Work in the `.nb`, not the generated page. Five things to fix:

**Package and data paths.** The notebook and the build kernel disagree about where they
are:

| | working directory | `NotebookDirectory[]` |
| --- | --- | --- |
| Front end | wherever it was launched | the notebook's folder |
| Build kernel (`wolframscript`) | the book root | **fails** — no notebook exists |

So `Get[FileNameJoin[{NotebookDirectory[], ...}]]` cannot work in both. Resolve a list of
candidates and take the first that exists:

```wl
$packagePath = SelectFirst[
   {Quiet@Check[
     FileNameJoin[{NotebookDirectory[], "library", "YourPackage.wl"}], $Failed],
    FileNameJoin[{Directory[], "notebooks", "library", "YourPackage.wl"}]},
   StringQ[#] && FileExistsQ[#] &,
   $Failed];
If[$packagePath === $Failed,
  Print["YourPackage.wl not found; looked next to the notebook and under ", Directory[]];
  Abort[],
  Get[$packagePath]];
```

`Quiet@Check[...]` is required — `NotebookDirectory[]` raises, it does not return quietly.
`Abort[]` on failure is deliberate: one clear message beats every downstream cell failing
obscurely.

**Mark setup cells `InitializationCell -> True`** (Cell ▸ Cell Properties ▸ Initialization
Cell). `--scaffold` hoists these to the top of the page as hidden `:echo: false` cells.

**Add markers** above each expression that should become a figure.

**Add `SaveDefinitions -> True`** to every `Manipulate`.

**Delete development scratch cells.** `Clear[...]`, `$Version`, empty cells, one-off
probes. This matters more than it looks: `--scaffold` hoists initialization cells to the
top but leaves everything else in notebook order, so a `Clear[f]` that harmlessly *preceded*
`f`'s definition in the notebook lands *after* it on the page and wipes the definition out
of the build session.

> **Editing a `.nb` programmatically.** `Get` → transform → `Put` round-trips the
> expression losslessly (`CompressedData` and box payloads stay inert), but `Put` writes a
> bare `Notebook[...]` without the `(* Content-type: ... *)` header or cache block.
> Re-save through the front end afterwards so the file stays a canonical notebook:
>
> ```wl
> UsingFrontEnd[
>   nb = NotebookOpen[path, Visible -> False]; NotebookSave[nb]; NotebookClose[nb]];
> ```
>
> This also re-parses inserted comment strings into real comment boxes. Verify by
> re-scaffolding and diffing — the output should be unchanged.

---

## Step 2 — Scaffold a page

```bash
./wolfram.py --scaffold notebooks/your-notebook.nb -o NN.your_chapter.md
```

Needs `wolframscript` (it drives the front end to export each cell's exact source).
Add `--force` to overwrite, `--title` to override the page title.

How cells map:

| Notebook cell | Becomes |
| --- | --- |
| `Title`, `Chapter` | `#` (first one becomes the frontmatter title) |
| `Section` / `Subsection` / `Subsubsection` | `##` / `###` / `####` |
| `Text` | markdown prose; `\[Name]` literals become unicode |
| `Item`, `ItemNumbered`, `Subitem`, `SubitemNumbered` | list items |
| `Input`, `Code`, `Program` | `{wolfram}` directive, source verbatim |
| `Code`, or `Input` marked `InitializationCell` | hoisted to top, `:echo: false` |
| TeX-assistant equations | `$...$` inline, or a `:::{math}` block if the cell is only an equation |
| `Output` | dropped |

---

## Step 3 — Edit the page

The scaffold is a **starting point, not a binding**. The page and the notebook are allowed
to drift; the notebook stays where you experiment. Check four things:

**Setup order.** Hoisted cells are ordered by a lexical dependency heuristic that cannot
know a `Get` supplies a package's symbols. Confirm the loader cell is still first.

**Elided payloads.** Pasted graphics and compressed data (`GraphicsBox`, `Graphics3DBox`,
`RasterBox`, `CompressedData`, `TemplateBox`, `DynamicModuleBox`, …) are replaced by a
comment saying how many characters were dropped — they cannot be scaffolded. Re-create them
as `{wolfram}` cells or export them to image files.

**`TODO` comments**, emitted for typeset cells and unrecognised styles. Nothing is silently
dropped.

**Bare display cells.** A cell like `myVariable` that is untagged renders an echoed code
fence and no output — usually noise. Either tag it, set `:echo: false`, delete it, or fold
the value into prose.

Then add `:caption:` to each tagged figure and tighten the prose.

---

## Step 4 — Build without deploying

```bash
myst start          # or: myst build --html
```

Unresolved cells render a warning admonition naming their digest. **Expect
`Could not link citation` warnings** for `@fig:...` at this stage — labels only exist once
a cell has a deployment. They disappear after step 5.

---

## Step 5 — Deploy

Requires a licensed, cloud-connected `wolframscript`.

```bash
JUPYTER_BOOK_WOLFRAM_DEPLOY=1 myst build --html
```

This evaluates the whole page in one kernel session, publishes every unresolved tagged
cell, and rewrites `.jupyter-book-wolfram/deployments.json`. **Commit the manifest** — it
is what lets prose-only contributors build with no Wolfram tooling.

Budget: **600 s for the entire page session** (`DEPLOY_TIMEOUT_SECONDS`); `--scaffold` has
its own 300 s limit. On timeout or a non-zero exit the plugin logs to stderr and leaves
cells unresolved rather than failing the build — so **read the build output**, do not just
check the exit code.

Optional environment variables:

| Variable | Effect |
| --- | --- |
| `JUPYTER_BOOK_WOLFRAM_DEPLOY=1` | enable publishing |
| `JUPYTER_BOOK_WOLFRAM_CLOUD_PATH` | cloud path prefix (default `myst-widgets`) |
| `JUPYTER_BOOK_WOLFRAM_CLOUD_BASE` | non-default cloud base |
| `JUPYTER_BOOK_WOLFRAM_ESM` | override the browser widget URL |

---

## Step 6 — Open every widget and look at it

**Do not skip this, and do not infer success from a clean build.** A cell can publish
successfully and render blank (see *Rendering limits*). Pull the URLs and open them:

```bash
python3 -c "import json;[print(v['url']) for v in json.load(open('.jupyter-book-wolfram/deployments.json'))['deployments'].values()]"
```

The first load of any cloud object often shows **"Initialization timed out"** while the
kernel cold-starts. Reload and allow ~20 s before concluding anything is wrong. Then
actually move a control and confirm the output changes.

---

## Reference: how definitions reach the reader

Two mechanisms, firing at different times. Confusing them is the most common cause of a
dead widget.

| | `IncludeDefinitions -> True` | `SaveDefinitions -> True` |
| --- | --- | --- |
| Set by | the plugin, always, on `CloudPublish` | you, on each `Manipulate` |
| Fires | at deploy time, in the build kernel | when the `Manipulate` itself evaluates |
| Collects | definitions of every symbol in the expression, recursively, across package contexts, reaching inside held expressions | definitions the `Manipulate` body needs |
| Makes work | the `{wolfram}` path | the `{wolfram-notebook}` path |

**`DynamicModule` needs no conversion** — it deploys and renders like anything else, with
or without a `Manipulate` inside. But note the ordering: under `wolframscript` a
`DynamicModule` never evaluates its body, so an inner `Manipulate`'s `SaveDefinitions`
does not run at build time at all. What carries your package into the cloud is
`IncludeDefinitions`. Set `SaveDefinitions` anyway — it is free here and required there.

---

## Reference: rendering and size limits

Neither of these surfaces as a build error.

**Payload size.** `IncludeDefinitions` bakes in every symbol the expression touches,
including large data. A widget holding a few MB works but loads slowly; tens of MB is
painful. If a widget only ever shows a fixed set of frames, store fewer or smaller ones.

**Primitive count inside dynamic updates.** The cloud gives a *dynamic* update a much
smaller budget than a static render, and **silently drops the primitive rather than
erroring**. A graphic built from one primitive per data point can therefore render fine as
a static figure and come back blank inside a `Manipulate`.

Measured on a square lattice drawn as one `Disk` per site: ~4,000 disks updates fine;
~40,000 renders statically but is blank inside a `Manipulate`. The same data as an
`ArrayPlot` raster updates instantly.

**Rule of thumb: vector primitives up to a few thousand elements, raster above.** Past that
the primitives are usually sub-pixel anyway, so the raster is also the better picture.

```wl
(* many thousands of elements, inside a Manipulate: *)
ArrayPlot[data, ImageSize -> 200]     (* not: one Disk/Point/Rectangle per element *)
```

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Warning admonition with a digest | no deployment recorded | deploy (step 5), or `--record` a known URL |
| `Could not link citation "fig:x"` | that cell has no deployment, so no label exists | deploy; harmless before step 5 |
| Widget deploys but renders blank | too many primitives for a dynamic update | switch to a raster |
| "Initialization timed out" | cloud kernel cold start | reload, wait ~20 s |
| Controls render, output never appears | definitions missing in the cloud | check the expression actually references the symbols; add `SaveDefinitions -> True` |
| Every cell after cell N broke | definitions cleared mid-page | remove `Clear[...]`/scratch cells |
| `Get` fails only when deploying | `NotebookDirectory[]` in the build kernel | use the candidate-path resolver |
| "tagged region evaluated to Null" | tagged region ends in `;` | drop the semicolon |
| All figures re-deploy after a small edit | chained digests | expected; edit low on the page where possible |
| `--scaffold` fails | no `wolframscript` | run on a licensed machine |

---

## Command reference

```bash
./wolfram.py --scaffold nb.nb -o page.md   # extract a .nb into an editable page
./wolfram.py --hash < cell.wl              # digest of one cell
./wolfram.py --hash-chain < cells.wl       # chained digests; separate cells with a %% line
./wolfram.py --hash --nb file.nb           # digest of a whole .nb
./wolfram.py --record URL < cell.wl        # record a deployment by code
./wolfram.py --record URL --digest D       # record by digest (copy it from the placeholder)
./wolfram.py --record URL --nb file.nb     # record a deployed local .nb
```

Refresh the vendored plugin:

```bash
curl -O https://raw.githubusercontent.com/matsci-education/jupyterbook-wolfram-plugin/main/wolfram.py && chmod +x wolfram.py
```

---

## Checklist

Notebook:

- [ ] Package/data paths resolve from both the front end and the book root
- [ ] Setup cells marked `InitializationCell -> True`
- [ ] `(* #| label: ... *)` above every expression that should be a figure
- [ ] `SaveDefinitions -> True` on every `Manipulate`
- [ ] No `Clear[...]`, `$Version`, empty, or scratch cells
- [ ] No tagged region ending in `;`
- [ ] Raster rather than per-element primitives for large dynamic graphics
- [ ] If edited programmatically, re-saved through the front end

Page:

- [ ] Loader/setup cells hoisted first and in a working order
- [ ] Elided graphics re-created or exported; `TODO`s resolved
- [ ] No untagged bare display cells
- [ ] `:caption:` on each tagged figure

Deploy:

- [ ] Built clean with deployment enabled; build output read, not just exit code
- [ ] Every widget opened in a browser **and a control moved**
- [ ] `deployments.json` committed
