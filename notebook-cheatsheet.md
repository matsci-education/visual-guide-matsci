# Notebook cheat sheet

Writing a `.nb` that scaffolds into a page and deploys without hand-fixing.

Full runbook, with the reasoning and a troubleshooting table:
[`workflow.md`](workflow.md). Day-to-day build: [`README.md`](README.md). The authority on
all of it is `wolfram.py`.

## Every cell is one of four things

| Cell | Mark it | What happens |
| --- | --- | --- |
| Loads a package, defines setup | `InitializationCell -> True` | hoisted to the top of the page, hidden (`:echo: false`) |
| Defines helpers | *nothing* | evaluates in the page session, renders no figure |
| Should become a figure | `(* #| label: fig:name *)` | deployed as a widget |
| Development scratch | **delete it** | — |

Deleting scratch is not tidiness. `--scaffold` hoists initialization cells to the top but
leaves everything else in notebook order, so a `Clear[f]` that harmlessly *preceded* `f`'s
definition in the notebook lands *after* it on the page and empties the definition out of
the shared build session.

## The marker

```wl
(* #| deploy *)                  (* publish this expression        *)
(* #| label: fig:hysteresis *)   (* publish + cross-reference target *)
```

- On its own line. One per cell.
- Everything **above** it is setup; the region **below** it is deployed.
- That region must not end in `;` — a `Null` value is refused.
- Renaming a label does not invalidate the cached deployment. Moving the marker does.

Reference it in prose as `@fig:hysteresis`; add `:caption:` to the page cell for a
numbered figure.

## Load a package so both kernels find it

`NotebookDirectory[]` is defined in the front end and **fails** under `wolframscript`,
where the working directory is the book root instead. Resolve candidates:

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

`Quiet@Check` is required — `NotebookDirectory[]` raises rather than returning quietly.
`Abort[]` gives one clear message instead of every later cell failing obscurely.

## `SaveDefinitions -> True` on every `Manipulate`

```wl
Manipulate[body, {i, 1, 10, 1}, SaveDefinitions -> True]
```

It is not what makes the `{wolfram}` path work — `IncludeDefinitions -> True`, which the
plugin always passes to `CloudPublish`, is. That one reaches inside held expressions and
across package contexts at deploy time. Under `wolframscript` a `DynamicModule` never
evaluates its body, so an inner `Manipulate`'s `SaveDefinitions` never runs at all.

Set it anyway: it costs nothing here, and on the `{wolfram-notebook}` path it is the only
thing that carries definitions to the reader.

`DynamicModule` needs no conversion. It deploys and renders like anything else.

## Two limits, neither of which is a build error

**Payload.** `IncludeDefinitions` bakes in every symbol the expression touches, large data
included. A few MB loads slowly; tens of MB hurts. Store fewer or smaller frames.

**Primitives inside dynamic updates.** The cloud gives a *dynamic* update a much smaller
budget than a static render, and **drops the primitive silently rather than erroring**. One
primitive per data point renders fine as a static figure and comes back **blank** inside a
`Manipulate`.

Measured on a square lattice, one `Disk` per site: ~4,000 updates fine; ~40,000 renders
statically but is blank in a `Manipulate`. As an `ArrayPlot` raster it updates instantly.

> **Rule of thumb:** vector primitives up to a few thousand elements, raster above.
> Past that they are sub-pixel anyway, so the raster is also the better picture.

## The pipeline

```bash
./wolfram.py --scaffold notebooks/your.nb -o NN.chapter.md   # 1. extract
myst start                                                   # 2. build, no deploy
JUPYTER_BOOK_WOLFRAM_DEPLOY=1 myst build --html              # 3. publish + record
```

Then **4. open every widget and move a control.** A cell can publish successfully and
render blank. Expect "Initialization timed out" on first load; reload and wait ~20 s.

Commit `.jupyter-book-wolfram/deployments.json` — it is what lets prose-only contributors
build with no Wolfram tooling.

Budget: 600 s for the whole page session. Deploy failures are logged to stderr and leave
cells unresolved rather than failing the build, so read the output, not the exit code.

## Preflight

- [ ] Package paths resolve from the front end **and** the book root
- [ ] Setup cells marked `InitializationCell -> True`
- [ ] A marker above every expression that should be a figure
- [ ] `SaveDefinitions -> True` on every `Manipulate`
- [ ] No `Clear[...]`, `$Version`, empty, or scratch cells
- [ ] No tagged region ending in `;`
- [ ] Raster, not per-element primitives, for large dynamic graphics
- [ ] If edited programmatically, re-saved through the front end (`Put` drops the
      notebook header — reopen and `NotebookSave`)
