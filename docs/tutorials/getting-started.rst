Getting started
===================

In this tutorial you style a real combo-chart report with one command,
see why a common setup needs one extra flag to get exactly right, and
apply it. It assumes a PBIP project already built on field parameters
(the pattern `BI-report-template-site
<https://github.com/C-Kapsalis/BI-report-template-site>`_ teaches);
this tool only styles a report that already has that in place, it does
not build the field parameters themselves.

You will need Python 3.10 or later. The worked example is the
repository's own ``examples/coffee-roastery``, a small, complete,
openable PBIP project with fictional data; nothing about it needs a
gateway or a live connection.

1. Install
-------------

::

    pip install .

Confirm it is on the path:

::

    pbi-plot-styler --version

2. See what would change
-----------------------------

From the ``examples/coffee-roastery`` folder:

::

    pbi-plot-styler Roastery.Report --dry-run

::

    Model:    .../Roastery.SemanticModel
    Measures: 6 drive the styling

    --- definition/pages/a1f935c9ad3918c55c49/visuals/5dd25eb672a72da45096/visual.json
    +++ definition/pages/a1f935c9ad3918c55c49/visuals/5dd25eb672a72da45096/visual.json (styled)
    @@ ... @@
    +    "objects": {
    +      "dataPoint": [ ... one entry per measure ... ],
    +      "labels": [ ... ]
    +    }

    1/1 visual(s) would change.

Nothing is written yet; ``--dry-run`` exits ``1`` to signal drift,
which doubles as a CI gate.

3. The setup: two field parameters sharing a measure catalog
------------------------------------------------------------------

This report's combo chart has ``x-Plot Specific 1`` bound to the
columns and ``x-Plot Specific 2`` bound to the line, both offering the
same catalog of measures (``Bags Sold #``, ``Revenue``, ``Wholesale
Margin %``, ``Avg Cupping Score``, ``Sample Requests #``), plus a
placeholder. That sharing is exactly the setup this tool needs to
handle carefully: Power BI matches a styling rule to a measure by name,
not by which selector chose it, so styling "every measure
``x-Plot Specific 2`` can show" would also color whichever of those
measures is currently selected on the columns via
``x-Plot Specific 1``, even though the columns were never meant to be
touched.

The shipped ``plotstyler.toml`` handles this with two settings working
together:

::

    [tables]
    field_parameters = ["x-Plot Specific 2"]

    [targets]
    exclude_current_roles = ["Y"]

    [style]
    line_color = "#107C10"
    label_color = "#FFFFFF"
    label_background = "#107C10"
    label_transparency = 0

``field_parameters`` restricts styling to the measures
``x-Plot Specific 2`` (the line) exposes. ``exclude_current_roles =
["Y"]`` then reads whichever measure is presently bound to the columns
on each visual and leaves it out, every time the tool runs, so the
columns' current pick never inherits the line's green.

4. Apply it
---------------

::

    pbi-plot-styler Roastery.Report

::

    Model:    .../Roastery.SemanticModel
    Measures: 6 drive the styling
      styled definition/pages/a1f935c9ad3918c55c49/visuals/5dd25eb672a72da45096/visual.json

    Restyled 1/1 visual(s); 0 already styled.

Open the report in Power BI Desktop: every measure reachable through
``x-Plot Specific 2`` renders with a green line and a white-on-green
data label, and whichever measure is currently plotted on the columns
is untouched, in Power BI's own default styling.

.. figure:: /_static/screenshots/line-styled-correctly.png
   :alt: The coffee-roastery combo chart with Revenue plotted as blue columns, unstyled, showing plain data labels ("1,040 EUR", "1,266 EUR", and so on), and Bags Sold # plotted as a line with solid green circular data labels ("69", "79", "84", and so on) in white text.

   Revenue, on the columns, stays in Power BI's default styling;
   Bags Sold #, on the line, gets the consistent green fill and label
   background, whichever measure is currently selected on either axis.

Run the command again and it reports ``Restyled 0/1 visual(s)``; the
operation is idempotent. Switch which measure is plotted on the line,
or add a new one to the field parameter, and re-running repairs it,
no diffing or manual rework needed.

Where to next
-----------------

- Every flag and config key, including ``--rename-table`` and
  ``--exclude-role``: :doc:`../reference/cli`.
