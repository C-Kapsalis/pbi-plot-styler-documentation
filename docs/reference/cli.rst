CLI reference
=============

::

    pbi-plot-styler [OPTIONS] REPORT_DIR

Styles every targeted visual in ``REPORT_DIR`` from the measures declared
in the paired semantic model's field-parameter tables.

Argument
-----------

.. list-table::
   :header-rows: 1

   * - Argument
     - Required
     - Description
   * - ``REPORT_DIR``
     - yes
     - Path to a PBIP report folder (the ``*.Report`` directory
       containing ``definition.pbir`` and ``definition/pages/``).

If the report folder's name begins with a dash (``-Sales.Report``), the
shell and option parser read it as a flag. Prefix it with ``./``, or end
option parsing with ``--``:

::

    pbi-plot-styler "./-Sales.Report"
    pbi-plot-styler --dry-run -- "-Sales.Report"

Every page, every time
--------------------------

There is no flag to scope a run to one report page. Every visual under
``definition/pages/**`` whose ``visualType`` matches ``visual_types`` is
checked and, if needed, restyled on every run, across the whole report.
This is deliberate: a field-parameter table's measures can be plotted
from any page, and a partially-styled report is exactly the drift this
tool exists to prevent. To leave a specific visual out of scope entirely,
give it a ``visualType`` outside the configured set instead.

Options
----------

.. list-table::
   :header-rows: 1

   * - Option
     - Type
     - Default
     - Description
   * - ``--model PATH``
     - directory
     - resolved from ``definition.pbir``
     - Semantic-model folder. Accepts the ``*.SemanticModel`` folder or
       its ``definition`` subfolder. When omitted, the model is resolved
       from the report's ``definition.pbir``, following
       ``datasetReference.byPath.path``.
   * - ``--config PATH``
     - file
     - ``./plotstyler.toml`` if present
     - TOML config file. See `Config file`_.
   * - ``--table NAME``
     - repeatable
     - ``x-Plot Specific 1``, ``x-Plot Specific 2``, ``y-Plot Specific``
     - Field-parameter table(s) to read measure references from. Passing
       the flag at least once replaces the whole default set. To change
       just one of the three without respecifying the others, use
       ``--rename-table`` instead.
   * - ``--rename-table OLD=NEW``
     - repeatable
     - none
     - Renames a single table in the current list (the three defaults,
       or whatever ``--table``/config set), leaving the others in place.
       ``OLD`` must match a name already present in that list; a typo'd
       ``OLD`` fails with an error rather than silently doing nothing.
       Example: ``--rename-table "x-Plot Specific 2=Set Lines"`` if your
       project calls that table "Set Lines".
   * - ``--exclude-role NAME``
     - repeatable
     - none
     - Query role (Power BI's internal name, for example ``Y`` for the
       columns of a combo chart, ``Y2`` for the line) whose *currently
       bound* measure is left out of styling, per visual. Needed when
       two field-parameter tables share measures. See `A shared measure
       catalog`_ below.
   * - ``--visual-type NAME``
     - repeatable
     - ``lineClusteredColumnComboChart``, ``lineStackedColumnComboChart``
     - ``visualType`` value(s) to restyle. Passing the flag replaces the
       default set.
   * - ``--line-color HEX``
     - ``#RRGGBB``
     - ``#118DFF``
     - Fill for every series (line and columns) of every styled measure.
   * - ``--palette HEX``
     - repeatable
     - empty
     - Adds a color to the palette. When the palette is non-empty it is
       cycled across measures in sorted order and ``--line-color`` is
       ignored.
   * - ``--label-color HEX``
     - ``#RRGGBB``
     - ``#252423``
     - Data-label text color (set once per visual, in the default labels
       entry).
   * - ``--label-background HEX``
     - ``#RRGGBB``
     - match line color
     - Data-label background per measure. When omitted, each measure's
       background matches its line color.
   * - ``--label-transparency N``
     - integer 0-100
     - ``20``
     - Data-label background transparency. Serialized as a Power BI
       literal ``NL`` (for example, ``20L``).
   * - ``--show-labels`` / ``--hide-labels``
     - flag
     - show
     - Whether data labels are switched on in the default labels entry.
   * - ``--dry-run``
     - flag
     - off
     - Compute all rewrites, print a unified diff per pending visual,
       write nothing.
   * - ``--zip PATH``
     - file
     - not set
     - Also write the styled report folder to ``PATH`` as a zip archive
       (archive root equals the report folder name). Combined with
       ``--dry-run``, the archive contains the styled content while the
       source files stay untouched.
   * - ``--version``
     - flag
     - not set
     - Print version and exit.
   * - ``-h``, ``--help``
     - flag
     - not set
     - Print usage and exit.

All hex colors must be 6-digit ``#RRGGBB`` values.

Config file
--------------

Loaded from ``--config``, or from ``plotstyler.toml`` in the current
working directory when the flag is omitted. Precedence: CLI flags, then
config file, then built-in defaults. All keys are optional; an unknown
section or key raises an error naming it rather than being silently
ignored.

::

    [tables]
    # --table
    field_parameters = ["x-Plot Specific 1", "x-Plot Specific 2", "y-Plot Specific"]
    # --rename-table
    renames = { "x-Plot Specific 2" = "Set Lines" }

    [targets]
    # --visual-type
    visual_types = ["lineClusteredColumnComboChart", "lineStackedColumnComboChart"]
    # --exclude-role
    exclude_current_roles = ["Y"]

    [style]
    line_color = "#118DFF"
    palette = []
    label_color = "#252423"
    label_background = "#118DFF"
    label_transparency = 20
    show_labels = true

What gets rewritten
-----------------------

For each visual whose ``visual.visualType`` is in ``visual_types``, the
styler replaces exactly two arrays under ``visual.objects`` in
``visual.json``:

.. list-table::
   :header-rows: 1

   * - Array
     - Content
   * - ``dataPoint``
     - One entry per measure: ``properties.fill.solid.color.expr.Literal.Value``
       set to ``"'#HEX'"``, ``selector.metadata`` set to
       ``"<Table>.<Measure>"``.
   * - ``labels``
     - Entry 0 (no selector): ``show``, ``color`` (label text),
       ``enableBackground: true``. Then one entry per measure:
       ``backgroundColor``, ``backgroundTransparency`` (an ``"NL"``
       literal), ``selector.metadata``.

Measures are the ``NAMEOF`` references in the configured field-parameter
tables that resolve to a measure in the model (table-qualified
references are checked against that table; bare references resolve to
the measure's home table). Column references and commented-out tuples
(``--``, ``//``) are ignored. Keys are sorted, so output order is
deterministic. Everything else in the file (queries, position, other
``objects`` entries) is preserved verbatim. Files are written with
2-space-indented JSON, LF line endings, and the source file's
end-of-file convention: a trailing newline is kept only when the file
already had one (Power BI Desktop saves ``visual.json`` without one).

Styling is applied uniformly to every series in scope, line and columns
alike. There is no way to style only the line; a measure that needs
different treatment from the rest belongs on a visual type outside
``visual_types``.

A shared measure catalog
-----------------------------

Power BI matches ``selector.metadata`` to a measure **by name**, across
the whole visual, regardless of which query role (which field-parameter
table, which axis) picked it. This only matters when two roles of the
same combo chart draw from field-parameter tables whose measure catalogs
overlap: styling every measure ``x-Plot Specific 2`` (the line) can show
will also color whichever of those measures happens to be currently
selected on the columns via ``x-Plot Specific 1``, even though the
columns were never the target.

``exclude_current_roles`` fixes this without narrowing the catalog
itself: it reads whichever measure is presently bound to the named
query role (for example ``Y``, the columns) on each visual, and leaves
that measure out of that visual's styling. It re-reads the current
binding on every run, so if the viewer changes what is plotted on the
columns, the next run follows automatically. See the worked example in
:doc:`../tutorials/getting-started`.

A related, unfixable case: if the exact same measure is projected on
*both* the column and the line role of one combo chart, Power BI only
keeps one ``selector.metadata`` match per visual, so one of the two
falls back to default rendering, and the PBIP schema offers no per-role
disambiguation. Use a thin wrapper measure (``Revenue (Line) = [Revenue]``)
if the same value genuinely needs to appear on both.

Enforcing style in CI
--------------------------

``--dry-run`` never writes anything; it prints a unified diff per
out-of-style visual and its exit code tells a pipeline whether to pass:

.. list-table::
   :header-rows: 1

   * - Exit code
     - Meaning for a CI check
   * - ``0``
     - Every targeted visual already matches the model and config. Pass.
   * - ``1``
     - At least one visual would change. Fail the build.
   * - ``2``
     - A setup problem (bad config, missing folder, unresolvable model,
       no field-parameter tables found, no measures resolved). Fail the
       build and read the message; this is not styling drift.

A minimal GitHub Actions step:

::

    - name: Check report styling
      run: pbi-plot-styler "My Project.Report" --dry-run

Point ``--config`` at a committed ``plotstyler.toml`` if your colors are
not the defaults, so the CI check and a local run agree. This turns a
style guide into a gate: a pull request that adds or renames a measure
in a field-parameter table without re-running the styler fails until
someone runs the one command.

Setting a brand palette
----------------------------

A committed ``plotstyler.toml`` is the durable way to replace the
default blue:

::

    [style]
    line_color = "#7A4419"
    label_color = "#FFF8F0"
    label_transparency = 30

To cycle a color per measure instead of one flat color, set ``palette``;
when it is non-empty it overrides ``line_color``:

::

    [style]
    palette = ["#7A4419", "#B08968", "#DDB892", "#E6CCB2"]

Measures are sorted, then assigned palette entries in order, so the same
measure gets the same color across runs as long as the measure list
itself does not change order. By default a measure's data-label
background matches its own line color; set ``label_background``
explicitly only if every label should share one background regardless
of series.

Exit codes
-------------

.. list-table::
   :header-rows: 1

   * - Code
     - Meaning
   * - ``0``
     - Success. In apply mode: styling applied (or already in place). In
       ``--dry-run``: no visual needs changes.
   * - ``1``
     - ``--dry-run`` only: at least one visual is out of style (drift
       detected). Use as a CI gate.
   * - ``2``
     - Error: bad invocation, invalid config value, missing report or
       model folder, unresolvable ``definition.pbir``, no
       field-parameter tables found, no measures resolved, an unknown
       config section/key, an unmatched ``--rename-table`` source name,
       or a malformed ``OLD=NEW`` rename.
