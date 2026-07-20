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
       the flag at least once replaces the whole default set.
   * - ``--visual-type NAME``
     - repeatable
     - ``lineClusteredColumnComboChart``
     - ``visualType`` value(s) to restyle. Passing the flag replaces the
       default set (for example, add ``lineStackedColumnComboChart``).
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
config file, then built-in defaults. All keys are optional.

::

    [tables]
    # --table
    field_parameters = ["x-Plot Specific 1", "x-Plot Specific 2", "y-Plot Specific"]

    [style]
    line_color = "#118DFF"
    palette = []
    label_color = "#252423"
    label_background = "#118DFF"
    label_transparency = 20
    show_labels = true

    [targets]
    visual_types = ["lineClusteredColumnComboChart"]

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
       field-parameter tables found, or no measures resolved.
