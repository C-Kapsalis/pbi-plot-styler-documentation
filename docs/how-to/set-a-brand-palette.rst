Set a brand color palette
=========================

Replace the default blue with your own brand colors, either once per
run with flags or, more durably, with a committed config file.

Drop a config file next to your project
---------------------------------------------

Create ``plotstyler.toml`` in the folder you run the styler from:

::

    [style]
    line_color = "#7A4419"
    label_color = "#FFF8F0"
    label_transparency = 30

- ``line_color`` sets the fill for every measure's series (line and
  columns alike) when no palette is given.
- ``label_color`` sets the data-label text color.
- ``label_transparency`` sets the data-label background transparency, an
  integer from 0 (opaque) to 100 (fully transparent).

The styler picks this file up automatically; pass ``--config PATH`` to
use a different one, or pass the equivalent flags (``--line-color``,
``--label-color``, ``--label-transparency``) to override it for one run.

Give each measure its own color
-------------------------------------

To cycle a palette across measures instead of using one flat color, add
a ``palette`` list. When it is non-empty it overrides ``line_color``:

::

    [style]
    palette = ["#7A4419", "#B08968", "#DDB892", "#E6CCB2"]

Measures are sorted, then assigned palette entries in order, so the same
measure always gets the same color across runs as long as the measure
list itself does not change order.

Match label backgrounds to each series
--------------------------------------------

By default, a measure's data-label background matches its own line
color, which keeps labels readable against their series without any
extra configuration. Set ``label_background`` explicitly only if you
want every label to share one background color regardless of series:

::

    [style]
    label_background = "#FFFFFF"

Check the result
--------------------

::

    pbi-plot-styler "My Project.Report" --dry-run

Confirm the diff shows your colors, not the defaults, then drop
``--dry-run`` to apply.
