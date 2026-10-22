Getting started
===============

In this tutorial you style a real combo-chart report with one command,
see why a common setup needs two settings to get exactly right, apply
them, and then check the report. The tool works on a PBIP project that
is already built on field parameters (the pattern
`BI-report-template-site
<https://github.com/C-Kapsalis/BI-report-template-site>`_ teaches); it
styles a report that has them, it does not build them. The example
used here already has them.

You will need Python 3.10 or later and Git. The worked example is the
repository's own ``examples/coffee-roastery``, a small, complete PBIP
project with fictional data; nothing about it needs a gateway or a live
connection.

Every command on this page is the same in bash and in PowerShell.

1. Install
----------

Clone release 1.2.1 of the repository, then install the package from
inside it:

::

    git clone --branch v1.2.1 https://github.com/C-Kapsalis/pbi-plot-styler.git
    cd pbi-plot-styler
    python -m pip install .

Git prints a note about a "detached HEAD" because you checked out a
release tag; that is expected.

Confirm the command is on the path:

::

    pbi-plot-styler --version

::

    pbi-plot-styler, version 1.2.1

2. See what would change
------------------------

Move into the example:

::

    cd examples/coffee-roastery

Preview the styling:

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
currently show (which this example's settings leave out of the line
set), and how many entries the two arrays the styler owns would hold.
Nothing is written yet. ``--dry-run`` exits ``1`` when something would
change, so the same command works as a CI gate.

3. The setup: two field parameters sharing a measure catalog
------------------------------------------------------------

This report's combo chart has ``x-Plot Specific 1`` bound to the
columns and ``x-Plot Specific 2`` bound to the line. Both offer the same
catalog of measures (``Bags Sold #``, ``Revenue``, ``Wholesale
Margin %``, ``Avg Cupping Score``, ``Sample Requests #``), and
``x-Plot Specific 2`` adds a blank placeholder measure. That sharing is
exactly the setup this tool needs to handle carefully: Power BI matches
a styling entry to a measure by name, not by which field parameter
chose it, so styling "every measure ``x-Plot Specific 2`` can show"
would also color whichever of those measures is currently selected on
the columns through ``x-Plot Specific 1``, even though the columns were
never meant to be touched.

The example ships a ``plotstyler.toml`` in this folder, and the tool
reads it automatically from the current folder. It handles the shared
catalog with two settings working together:

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
(the ``Y`` role) on each visual and leaves it out, every time the tool
runs, so the columns' current pick never inherits the line's green.

4. Apply it
-----------

This step rewrites one ``visual.json`` in your clone. To undo it later,
run ``git checkout -- .`` from this folder.

::

    pbi-plot-styler Roastery.Report

::

      5dd25eb672a72da45096: dataPoint 0 -> 5, labels 0 -> 6 (5 line measures each; 1 columns-role measure(s) excluded; 1 bar-side entry preserved)

    Models resolved: 1
    rebuilt 1/1 combo chart(s).

Open ``Roastery.pbip`` in Power BI Desktop: every measure reachable
through ``x-Plot Specific 2`` renders with a green line and a
white-on-green data label, and whichever measure is currently plotted
on the columns keeps Power BI's own default styling.

.. figure:: /_static/screenshots/line-styled-correctly.png
   :alt: The coffee-roastery combo chart with Revenue plotted as blue columns, unstyled, showing plain data labels ("1,040 EUR", "1,266 EUR", and so on), and Bags Sold # plotted as a line with solid green circular data labels ("69", "79", "84", and so on) in white text.

   Revenue, on the columns, stays in Power BI's default styling;
   Bags Sold #, on the line, gets the green fill and label background,
   whichever measure is currently selected on either axis.

Run the same command again:

::

    pbi-plot-styler Roastery.Report

::

    Models resolved: 1
    rebuilt 0/1 combo chart(s).

Nothing was rewritten (the output starts with a blank line where the
per-visual lines would be): the operation is idempotent. A dry run now
reports ``All 1 combo chart(s) already in sync.`` and exits ``0``. Switch
which measure is plotted on the line, or add a new one to the field
parameter, and running the command again repairs the styling, with no
manual rework.

Formatting you set by hand in Power BI Desktop survives these runs: the
styler rewrites only the per-measure color and label-background entries
it owns, and keeps the column baseline, per-legend-value settings, the
columns' own entries, and any extra property you tuned.

5. Check the report
-------------------

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
the service rejects on import, and so on), and the run then exits
``1``, so it works as a CI gate next to ``--dry-run``.

Those are the ``integrity`` checks, the default. The same command also
runs the conventions of the standard canvas page, one rule at a time,
with ``--rule``:

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

The example declares a base theme but no custom theme, so the run finds
a warning (``THM003``) and an error (``THM006``) and exits ``1``. The
lint lines at the top are the findings; the block after them is the
rule's own summary. In 1.2.1 that block's ``Violations found`` line for
``THM006`` always reads ``missing: baseTheme, customTheme``, even when
only the custom theme is missing; trust the ``ERROR THM006`` line above
it (see :doc:`../reference/known-issues`).

The names the convention rules look for (the gate measure, the category
table, the theme sections) come from the canvas sections of
``plotstyler.toml``, and from the built-in defaults when the file does
not set them. There are twelve rule slugs in all: ``integrity`` and
the eleven convention rules described in
:doc:`../reference/convention-rules`.

6. Score the foundation rules
-----------------------------

``--summary`` runs four of the convention rules (``slicer-gates``,
``category-filter``, ``theme`` and ``filter-panel``) and prints a
compliance report:

::

    pbi-plot-styler check Roastery.Report --summary

::

    skipped  Roastery.Report: filter-panel: the report has no group containers
    No Report field-parameter slicers found under <absolute path>/Roastery.Report
    WARNING  THM003  Roastery.Report/definition/report.json: No custom theme defined - may use hardcoded colors (Expected: customTheme configuration, Got: null or missing)

    🔍 Template Foundation Rules Comprehensive Report:
    ============================================================
    # Template Foundation Rules Validation Report

    **Overall Compliance**: 100.0% (4/4 rules)
    **Violations**: 0 critical, 1 warnings

    ## Rule #1: Field Parameter f-Slicer Filters ✅ PASS
    Components: 0/0 passed

    ## Rule #3: Analysis Category Filter Required ✅ PASS
    Components: 0/0 passed

    ## Rule #6: ThemeDataColor Integration ✅ PASS
    Components: 1/1 passed

    **⚠️ missing_custom_theme** (report)
    - No custom theme defined - may use hardcoded colors
    - **Fix**: Define customTheme to avoid hardcoded color values

    ## Rule #8: y_filters_group Master Container ✅ PASS
    Components: 0/0 passed


    📊 Summary:
       Overall Compliance: True
       Rules Passed: 4/4
       Compliance Score: 100.0%
       Critical Issues: 0
       Warnings: 1

    Checked 1 report(s): 0 error(s), 1 warning(s).

The second line names the report folder by its absolute path, which is
different on your machine. The theme row of the summary scores only
``THM002``, ``THM003`` and ``THM004``, so ``THM006`` from step 5 is not
listed here, and the closing totals count the same findings as the
score. ``--summary`` never fails: it exits ``0`` whatever it finds.

7. Check a folder of reports
----------------------------

Both ``style`` and ``check`` also accept a folder that holds several
reports, at any depth; each report is handled against its own model.
From this folder, ``..`` is the repository's ``examples`` folder:

::

    pbi-plot-styler check ..

::

    Checked 1 report(s): 0 error(s), 0 warning(s).

Where to next
-------------

- Every flag and config key, including ``--exclude-role``, color pins
  and theme colors: :doc:`../reference/cli`.
- Every ``check`` id, ``--fix``, baselines and the JSON output:
  :doc:`../reference/cli`, section *check*.
- The twelve rules ``--rule`` runs, their ids, settings and fixers:
  :doc:`../reference/convention-rules`.
- The four commands that keep a report in step with its model
  (``colors sync``, ``bookmarks build``, ``rename-measure`` and
  ``rename-label``): :doc:`../reference/cli`.
- What 1.2.1 still gets wrong, and how to work around it:
  :doc:`../reference/known-issues`.
