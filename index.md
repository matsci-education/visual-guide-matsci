---
title: A Visual Guide to Materials Science
---

Materials science is a visual subject. Crystal structures are three-dimensional, stress
states rotate, and phase diagrams are read by eye — but textbooks flatten all of it onto
paper. This guide keeps the pictures moving.

Every figure below is a live Wolfram Language computation, deployed to the Wolfram Cloud
and embedded in the page. Drag the sliders; nothing is a screenshot, and you need no
Wolfram installation to read along.

## Contents

- [](01.crystal_structures.md) — Bravais lattices, the cubic family, and atomic packing.
- [](02.stress_and_strain.md) — the stress tensor, principal stresses, and Mohr's circle.

## How this book is built

Pages are [MyST markdown](https://mystmd.org). The `{wolfram}` directive comes from
[jupyterbook-wolfram-plugin](https://github.com/matsci-education/jupyterbook-wolfram-plugin),
which evaluates each tagged cell with `wolframscript`, publishes the result as a public
cloud object, and records it in a committed manifest. Contributors who only touch prose
need no Wolfram tooling at all — the cached deployments render for everyone.
