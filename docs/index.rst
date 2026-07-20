pbi-plot-styler
================

A reporting template for Power BI that scales by declaration, and the
CLI that keeps it presentation-ready. Three field-parameter tables
declared in the semantic model let one combo-chart page replace a
growing stack of near-identical pages; ``pbi-plot-styler`` is the one
piece those tables cannot reach themselves, the per-measure visual
styling stored inside each visual's ``visual.json``. Run it after every
model change and every chart it targets stays on-brand, deterministically
and idempotently.

.. toctree::
   :maxdepth: 1

   tutorials/getting-started
   how-to/enforce-styling-in-ci
   how-to/set-a-brand-palette
   reference/cli
   explanation/field-parameter-reporting
