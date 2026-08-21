# visual-guide-matsci

A Visual Guide to Materials Science — a [MyST](https://mystmd.org) book whose figures are
live Wolfram Language computations, rendered client-side from the Wolfram Cloud.

Built with
[jupyterbook-wolfram-plugin](https://github.com/matsci-education/jupyterbook-wolfram-plugin).

## Building

```sh
myst start        # or: myst build --html
```

**No Wolfram tooling is required.** Every figure resolves from the committed manifest
[`.jupyter-book-wolfram/deployments.json`](.jupyter-book-wolfram/deployments.json), so
contributors who only touch prose need nothing but `mystmd`. A cell with no recorded
deployment renders a warning admonition rather than failing the build.

## Editing Wolfram cells

Editing the code inside a `{wolfram}` cell invalidates its cached deployment, and — because
digests are *chained* down the page — every deployable cell below it too. Those figures
become placeholders until someone re-publishes them on a machine with a licensed
`wolframscript`:

```sh
JUPYTER_BOOK_WOLFRAM_DEPLOY=1 myst build
```

That publishes the changed cells, rewrites the manifest, and leaves the rest alone.
**Commit the manifest** — that is what lets everyone else build without Wolfram.

Prose, `:caption:`, `:echo:`, and marker labels are not hashed, so you can edit those
freely.

## Updating the plugin

`wolfram.py` is a vendored copy, because MyST only loads executable plugins from disk:

```sh
curl -O https://raw.githubusercontent.com/matsci-education/jupyterbook-wolfram-plugin/main/wolfram.py
chmod +x wolfram.py
```

The browser-side widget is *not* vendored — the plugin points MyST at its published URL
and MyST fetches it at build time.

## Layout

| Path | What it is |
| --- | --- |
| `index.md` | Landing page |
| `01.crystal_structures.md` | Bravais lattices, packing factor, Miller indices |
| `02.stress_and_strain.md` | Stress tensor, principal stresses, Mohr's circle |
| `notebooks/` | Source `.nb` files embedded whole via `{wolfram-notebook}` |
| `wolfram.py` | Vendored copy of the plugin |
| `.jupyter-book-wolfram/deployments.json` | Committed manifest of deployed cloud objects |
