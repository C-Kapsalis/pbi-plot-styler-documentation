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

Clone the repository, then install the package from inside it:

::

    git clone https://github.com/c-kapsalis/pbi-plot-styler.git
    cd pbi-plot-styler
    pip install .

Confirm it is on the path:

::

    pbi-plot-styler --version

::

    pbi-plot-styler, version 1.2.0

2. See what would change
-----------------------------

From the ``examples/coffee-roastery`` folder:

::

    pbi-plot-styler Roastery.Report --dry-run

::

      5dd25eb672a72da45096: WOULD CHANGE
        columns-role measure(s) kept OUT of the line set: Sales Measures.Revenue
        objects.dataPoint[]: 0 -> 5 entries
        objects.labels[]: 0 -> 6 entries

    Models resolved: 1
    1/1 combo chart(s) need rebuild (re-run without --dry-run).

The block names the visual that would change, the measure the columns
role currently holds (which the tool leaves out of the line set), and
how many entries the two owned arrays would grow. Nothing is written
yet; ``--dry-run`` exits ``1`` to signal drift, which doubles as a CI
gate.

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

      5dd25eb672a72da45096: dataPoint 0 -> 5, labels 0 -> 6 (5 line measures each; 1 columns-role measure(s) excluded; 1 bar-side entry preserved)

    Models resolved: 1
    rebuilt 1/1 combo chart(s).

Open the report in Power BI Desktop: every measure reachable through
``x-Plot Specific 2`` renders with a green line and a white-on-green
data label, and whichever measure is currently plotted on the columns
is untouched, in Power BI's own default styling.

.. figure:: /_static/screenshots/line-styled-correctly.png
   :alt: The coffee-roastery combo chart with Revenue plotted as blue columns, unstyled, showing plain data labels ("1,040 EUR", "1,266 EUR", and so on), and Bags Sold # plotted as a line with solid green circular data labels ("69", "79", "84", and so on) in white text.

   Revenue, on the columns, stays in Power BI's default styling;
   Bags Sold #, on the line, gets the consistent green fill and label
   background, whichever measure is currently selected on either axis.

Run the command again and it reports ``rebuilt 0/1 combo chart(s)``; the
operation is idempotent. Switch which measure is plotted on the line,
or add a new one to the field parameter, and re-running repairs it,
no diffing or manual rework needed.

Formatting you set by hand in Power BI Desktop survives these runs: the
styler rewrites only the per-measure color and label-background entries
it owns, and keeps the column baseline, per-legend-value settings, the
columns' own entries, and any extra property you tuned.

5. Check the report
-----------------------

Styling is one half of keeping a report healthy; the other is catching
the breakages Power BI only reveals at import or render time, such as a
visual still bound to a renamed measure or a bookmark pointing at a
deleted page. ``check`` is read-only:

::

    pbi-plot-styler check Roastery.Report

::

    Checked 1 report(s): 0 error(s), 0 warning(s).

It exits ``0`` here. Each problem it finds is one line with a stable id
(``BND001`` for an unresolved field, ``BMK001`` for a bookmark option
the service rejects on import, and so on), and the run exits ``1``, so
it works as a CI gate next to ``--dry-run``.

Those are the ``integrity`` checks. The same command also runs the
conventions of the standard canvas page, one rule at a time:

::

    pbi-plot-styler check Roastery.Report --rule theme

::

    WARNING  THM003  Roastery.Report/definition/report.json: No custom theme defined - may use hardcoded colors (Expected: customTheme configuration, Got: null or missing)
    ERROR    THM006  Roastery.Report/definition/report.json: Theme collection is incomplete - missing: customTheme. Reports must declare both baseTheme and customTheme.
    ✅ Rule #6 Validation Results:
       Components checked: 1
       Components passed: 0
       Compliant: False

       Violations found:
       ⚠️ report: No custom theme defined - may use hardcoded colors
       🔴 report: Theme collection is incomplete - missing: baseTheme, customTheme. Reports must declare both baseTheme and customTheme.
    Rule #6 critical violations found:
      - customTheme: Theme collection is incomplete - missing: customTheme. Reports must declare both baseTheme and customTheme.

    Checked 1 report(s): 1 error(s), 1 warning(s).

The names the rules look for — the gate measure, the category table, the
theme table — come from the canvas sections of ``plotstyler.toml``, and
from the built-in defaults when the file does not set them. This run
finds two findings on the example: no custom theme, and a theme
collection that names one that is not there. It exits ``1``. The twelve
rule slugs are listed in the reference.

Both commands also accept a folder that holds several reports, at any
depth; each report is handled against its own model:

::

    pbi-plot-styler check ..

Where to next
-----------------

- Every flag and config key, including ``--exclude-role``, color pins
  and theme colors: :doc:`../reference/cli`.
- Every ``check`` id, the twelve convention rules, ``--fix``, and
  baselines: :doc:`../reference/cli`, section *check*.
- The four commands that keep a report in step with its model:
  ``colors sync``, ``bookmarks build``, ``rename-measure`` and
  ``rename-label``: :doc:`../reference/cli`.
