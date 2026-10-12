CLI reference
=============

::

    pbi-plot-styler [style] [OPTIONS] REPORT_DIR
    pbi-plot-styler check [OPTIONS] PATH
    pbi-plot-styler --version | --help

``pbi-plot-styler`` has two commands:

- ``style`` (the default) styles every targeted visual from the measures
  declared in the paired semantic model's field-parameter tables. When
  the first argument is not a command name, ``style`` runs, so
  ``pbi-plot-styler REPORT_DIR`` and ``pbi-plot-styler style REPORT_DIR``
  are the same command.
- ``check`` is a read-only integrity gate: it reports broken bindings,
  bookmarks that cannot be imported or that dangle, inconsistent
  field-parameter slots, and missing theme files. See `check`_.

A report folder literally named ``check`` or ``style`` must be passed as
``./check`` (or after an explicit ``style``).

style
-----

Argument
~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Argument
     - Required
     - Description
   * - ``REPORT_DIR``
     - yes
     - A PBIP report folder (the ``*.Report`` directory containing
       ``definition.pbir`` and ``definition/pages/``), or any folder
       that contains one or more of them at any depth.

When ``REPORT_DIR`` is a folder of reports, every ``*.Report`` folder
under it is found recursively and styled against its own paired model
(each model is loaded once). Output then starts each report with a
``Report:`` line, diff paths are relative to ``REPORT_DIR``, and a
discovered folder without ``definition/pages`` (a report not saved in
the PBIR format) is skipped with a note on stderr.

If the report folder's name begins with a dash (``-Sales.Report``), the
shell and option parser read it as a flag. Prefix it with ``./``, or end
option parsing with ``--``:

::

    pbi-plot-styler "./-Sales.Report"
    pbi-plot-styler --dry-run -- "-Sales.Report"

Every page, every time
~~~~~~~~~~~~~~~~~~~~~~

There is no flag to scope a run to one report page. Every visual under
``definition/pages/**`` whose ``visualType`` matches ``visual_types`` is
checked and, if needed, restyled on every run, across the whole report.
This is deliberate: a field-parameter table's measures can be plotted
from any page, and a partially-styled report is exactly the drift this
tool exists to prevent. To leave a specific visual out of scope entirely,
give it a ``visualType`` outside the configured set instead.

Style options
~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Option
     - Type
     - Default
     - Description
   * - ``--model PATH``
     - directory
     - resolved from ``definition.pbir``
     - Semantic-model folder, used for every report found. Accepts the
       ``*.SemanticModel`` folder or its ``definition`` subfolder. When
       omitted, each report's model is resolved from its own
       ``definition.pbir``, following ``datasetReference.byPath.path``.
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
       columns of a combo chart, ``Y2`` for the line) whose measures are
       left out of styling, per visual: the measure bound to it, or, when
       the role is bound through a field parameter, every measure that
       parameter offers. Their existing formatting entries are kept
       verbatim. See `A shared measure catalog`_.
   * - ``--visual-type NAME``
     - repeatable
     - ``lineClusteredColumnComboChart``, ``lineStackedColumnComboChart``
     - ``visualType`` value(s) to restyle. Passing the flag replaces the
       default set.
   * - ``--line-color HEX``
     - ``#RRGGBB``
     - ``#118DFF``
     - Fill for every series (line and columns) of every styled measure,
       unless a pin, the palette, or a theme color applies. See
       `Colors`_.
   * - ``--palette HEX``
     - repeatable
     - empty
     - Adds a color to the palette. When the palette is non-empty, each
       measure takes the color of its stable slot, and ``--line-color``
       and ``--theme-color-id`` are ignored.
   * - ``--color TABLE.MEASURE=HEX``
     - repeatable
     - none
     - Pins one measure's series color, for example
       ``--color "Sales Measures.Revenue=#7A4419"``. Beats the palette,
       the theme color and ``--line-color``. Merged per key over
       ``[style.colors]``. A pin that matches no field-parameter measure
       is an error.
   * - ``--theme-color-id N``
     - integer, 0 or more
     - not set
     - Writes the report theme's data color ``N`` (0-based) as a
       ``ThemeDataColor`` expression instead of ``--line-color``, so the
       chart follows a theme change.
   * - ``--theme-color-percent P``
     - number, -1 to 1
     - ``0``
     - Lightens (positive) or darkens (negative) the theme color.
   * - ``--label-color HEX``
     - ``#RRGGBB``
     - ``#252423``
     - Data-label text color, set in the no-selector labels entry. See
       `Explicit versus default settings`_.
   * - ``--label-background HEX``
     - ``#RRGGBB``
     - match series color
     - Data-label background per measure. When omitted, each measure's
       background matches its series color (hex or theme color).
   * - ``--label-transparency N``
     - integer 0-100
     - ``20``
     - Data-label background transparency. Serialized as a Power BI
       literal ``NL`` (for example, ``20L``).
   * - ``--show-labels`` / ``--hide-labels``
     - flag
     - show
     - Whether data labels are switched on in the no-selector labels
       entry.
   * - ``--dry-run``
     - flag
     - off
     - Compute all rewrites, print a unified diff per pending visual,
       write nothing.
   * - ``--zip PATH``
     - file
     - not set
     - Also write the styled report folder(s) to ``PATH`` as a zip
       archive (rooted at the folder name of ``REPORT_DIR``). Combined
       with ``--dry-run``, the archive contains the styled content while
       the source files stay untouched.
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
~~~~~~~~~~~

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
    # --theme-color-id / --theme-color-percent
    theme_color_id = 9
    theme_color_percent = 0
    label_color = "#252423"
    label_background = "#118DFF"
    label_transparency = 20
    show_labels = true

    # --color, one pin per measure
    [style.colors]
    "Sales Measures.Revenue" = "#7A4419"

What gets rewritten
~~~~~~~~~~~~~~~~~~~

For each visual whose ``visual.visualType`` is in ``visual_types``, the
styler owns a narrow slice of two arrays under ``visual.objects`` in
``visual.json``:

.. list-table::
   :header-rows: 1

   * - Array
     - Owned by the styler
   * - ``dataPoint``
     - One entry per styled measure, selected by ``selector.metadata``
       set to ``"<Table>.<Measure>"``. Inside it, only
       ``properties.fill`` (a ``Literal`` hex value or a
       ``ThemeDataColor``).
   * - ``labels``
     - The same per-measure entries, owning ``backgroundColor`` and
       ``backgroundTransparency`` (an ``"NL"`` literal). Plus the
       no-selector entry's ``enableBackground: true``, without which
       Power BI ignores every per-measure background.

Everything else is preserved verbatim and in order:

- the no-selector ``dataPoint`` default (the column render baseline,
  such as ``borderShow`` or a hand-picked column ``fill``); the styler
  never creates one;
- ``selector.data`` entries, which address a legend value rather than a
  measure (for example ``showSeries: false``);
- the metadata entries of excluded measures (``exclude_current_roles``);
- every property of an owned entry other than the owned ones, such as a
  hand-set ``fillTransparency`` or a per-measure label ``color``.

The rebuilt array is laid out as ``[defaults] + [excluded measures] +
[styled measures] + [data-scoped entries]``, styled entries sorted by
key. Metadata entries for measures that no configured field parameter
offers any more (renamed or retired measures) are dropped, so stale
formatting heals. If a rebuild would change any preserved entry, the
styler refuses to write and exits ``2``.

Measures are the ``NAMEOF`` references in the configured field-parameter
tables that resolve to a measure in the model (table-qualified
references are checked against that table; bare references resolve to
the measure's home table). Column references and commented-out tuples
(``--``, ``//`` and ``/* */``) are ignored. Everything else in the file
(queries, position, other ``objects`` entries) is preserved verbatim. A
visual already in style is not rewritten at all; a rewritten file keeps
2-space-indented JSON, the source file's line endings (LF or CRLF), and
its end-of-file convention: a trailing newline is kept only when the
file already had one (Power BI Desktop saves ``visual.json`` without
one).

Explicit versus default settings
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some label settings respect what a visual already carries unless you
ask for them explicitly, with a flag or a config key:

.. list-table::
   :header-rows: 1

   * - Setting
     - When the visual has no value
     - When the visual already has a value
   * - ``show_labels``, ``label_color``
     - Seeded in a new no-selector labels entry.
     - Kept; overwritten only when set explicitly.
   * - ``label_transparency``
     - Seeded on the measure's labels entry.
     - Kept; overwritten only when set explicitly.

``fill`` and ``backgroundColor`` are always owned and always written.

Colors
~~~~~~

A measure's series color is the first that applies:

1. its pin in ``[style.colors]`` (or ``--color``);
2. the ``palette`` entry of its stable slot;
3. ``theme_color_id``, written as
   ``{"ThemeDataColor": {"ColorId": N, "Percent": P}}``;
4. ``line_color``.

A measure's **slot** comes from the model, not the alphabet: the
configured field-parameter tables are taken in order, each table's rows
in their sort-ordinal order (the third member of each tuple), and a
measure registered more than once keeps the slot of its first
registration. With a palette of ``n`` colors, slot ``s`` gets color
``s mod n``. Adding a measure at the end of a field parameter, or
renaming one, therefore leaves every other measure's color in place;
pin a measure with ``[style.colors]`` when its color must never move.
By default a measure's data-label background matches its series color.

A shared measure catalog
~~~~~~~~~~~~~~~~~~~~~~~~

Power BI matches ``selector.metadata`` to a measure **by name**, across
the whole visual, regardless of which query role (which field-parameter
table, which axis) picked it. This only matters when two roles of the
same combo chart draw from field-parameter tables whose measure catalogs
overlap: styling every measure ``x-Plot Specific 2`` (the line) can show
will also color whichever of those measures happens to be currently
selected on the columns via ``x-Plot Specific 1``, even though the
columns were never the target.

``exclude_current_roles`` fixes this without narrowing the catalog
itself. For each named query role (for example ``Y``, the columns) on
each visual it excludes:

- a measure bound directly to the role, read from its current
  projection, re-read on every run; and
- when the role is bound through a field parameter (a ``Column``
  projection on a field-parameter table), every measure that field
  parameter offers, since any of them may be the current pick.

Excluded measures are not styled on that visual, and their existing
formatting entries, such as a hand-picked column color, are kept
verbatim. A measure projected on both the line and the columns counts as
excluded. See the worked example in :doc:`../tutorials/getting-started`.

A related, unfixable case: if the exact same measure is projected on
*both* the column and the line role of one combo chart, Power BI only
keeps one ``selector.metadata`` match per visual, so one of the two
falls back to default rendering, and the PBIP schema offers no per-role
disambiguation. Use a thin wrapper measure (``Revenue (Line) = [Revenue]``)
if the same value genuinely needs to appear on both.

Enforcing style in CI
~~~~~~~~~~~~~~~~~~~~~

``--dry-run`` never writes anything; it prints a unified diff per
out-of-style visual and its exit code tells a pipeline whether to pass
(see `Style exit codes`_). A minimal GitHub Actions step:

::

    - name: Check report styling
      run: pbi-plot-styler "My Project.Report" --dry-run

Point ``--config`` at a committed ``plotstyler.toml`` if your colors are
not the defaults, so the CI check and a local run agree. This turns a
style guide into a gate: a pull request that adds or renames a measure
in a field-parameter table without re-running the styler fails until
someone runs the one command.

Style exit codes
~~~~~~~~~~~~~~~~

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
       model folder, unresolvable ``definition.pbir``, no ``*.Report``
       folder found, no field-parameter tables found, no measures
       resolved, no visual of a targeted type found, a color pin that
       matches no measure, an unknown config section/key, an unmatched
       ``--rename-table`` source name, a malformed ``OLD=NEW`` or
       ``KEY=HEX`` pair, or a rebuild that would alter a preserved
       entry.

check
-----

::

    pbi-plot-styler check [OPTIONS] PATH

Checks the report at ``PATH`` (a ``*.Report`` folder, or a folder of
them, found recursively) and prints one line per finding. Read-only
unless ``--fix`` is given. ``check`` does not read ``plotstyler.toml``.

Check options
~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Option
     - Type
     - Default
     - Description
   * - ``--model PATH``
     - directory
     - resolved from ``definition.pbir``
     - Semantic-model folder used for the binding checks of every report
       found.
   * - ``--json``
     - flag
     - off
     - Print one JSON document instead of text (see `JSON output`_).
   * - ``--fix``
     - flag
     - off
     - Repair the fixable findings (``BMK001``, ``BMK002``) in place,
       then check again.
   * - ``--baseline FILE``
     - file
     - not set
     - Accepted-findings file. Only findings not in it fail the run.
   * - ``--update-baseline``
     - flag
     - off
     - Write every current finding to the ``--baseline`` file and exit
       ``0``.

Checks
~~~~~~

Ids are stable: an id never changes meaning, so a committed baseline
keeps working across releases.

.. list-table::
   :header-rows: 1

   * - Id
     - Severity
     - Finding
   * - ``JSN001``
     - error
     - A report file (``report.json``, ``page.json``, ``visual.json``, a
       bookmark) is not valid JSON.
   * - ``BND001``
     - error
     - A field reference does not resolve in the paired model: a
       ``Measure``, ``Column`` (including inside an ``Aggregation``) or
       ``HierarchyLevel`` whose table, field, hierarchy or level does
       not exist, or a ``From`` entity of a filter. ``SourceRef.Source``
       aliases are resolved through the enclosing ``From`` list.
   * - ``BND002``
     - error
     - A ``queryRef`` or ``selector.metadata`` string does not resolve.
   * - ``BMK001``
     - error
     - A bookmark ``options`` key is outside the allowed set:
       ``targetVisualNames``, ``applyOnlyToTargetVisuals``,
       ``suppressData``, ``suppressActiveSection``, ``suppressDisplay``.
       The file is valid JSON, but the service rejects the whole report
       on import. Fixable.
   * - ``BMK002``
     - error
     - A bookmark ``howCreated``, ``ComparisonKind`` or ``Version`` is a
       string, not a JSON integer. Fixable when the string is numeric.
   * - ``BMK003``
     - error
     - A bookmark's ``explorationState.activeSection``, or a key of its
       ``sections``, names a page that does not exist.
   * - ``BMK004``
     - error
     - A key of ``sections.<page>.visualContainers`` (or
       ``visualContainerGroups``) is not a visual on that page.
   * - ``BMK005``
     - error
     - ``bookmarks.json`` lists a bookmark (top level or a group's
       ``children``) that has no ``.bookmark.json`` file.
   * - ``FPM001``
     - error
     - Two field parameters of one query role claim the same slot
       (``queryState.<role>.fieldParameters[]`` ranges overlap); the
       later one wins, so the earlier one's picks never render.
   * - ``FPM002``
     - error
     - A field parameter's slot range (``index`` plus ``length``) runs
       past the role's projections.
   * - ``THM001``
     - warning
     - A ``report.json`` ``themeCollection`` entry has no item in its
       resource package, or its file is missing under
       ``StaticResources``.

Bindings are resolved case-insensitively. A ``"Table.Field"`` string is
split at a known table name, longest first, never at the last dot, since
measure names may contain dots (``Sales Measures.Revenue v1.2 (Adj.)``);
an aggregation wrapper such as ``Sum(Orders.Amount)`` and a
``Hierarchy.Level`` suffix are understood. Report-level measures
(``"Schema": "extension"``, declared in ``reportExtensions.json``) are
accepted. When a report's model is not a local folder, the binding
checks are skipped with a note and every other check still runs.

Fixes
~~~~~

``--fix`` edits the bookmark text surgically and never re-serializes
the file, so the change reviews as a one-line diff. Each edit is proven
by parsing the text before and after and requiring that the only
difference is the removed key or the converted integers. If that cannot
be proven (for example, the key cannot be located exactly once), the fix
is refused, the file is left untouched, a ``REFUSED`` line explains why,
and the finding still counts.

Baselines
~~~~~~~~~

To adopt the gate on a report that already has findings, record them
once and commit the file:

::

    pbi-plot-styler check "My Project.Report" --baseline check-baseline.json --update-baseline

From then on, ``--baseline check-baseline.json`` fails only on findings
that are not in it, and lists accepted findings that no longer occur as
``healed``. Findings are keyed by check id and location: the file path
(relative to ``PATH``, or to its parent when ``PATH`` is itself a
report, so it starts with the report folder name) plus the offending
name, never an array index. Unrelated edits do not invalidate the
baseline. Rerun with
``--update-baseline`` after healing findings to shrink it.

JSON output
~~~~~~~~~~~

``--json`` prints one object with ``tool``, ``version``, ``reports``,
``findings`` (each with ``id``, ``severity``, ``file``, ``subject``,
``location``, ``message``, ``fixable`` and ``baselined``), ``healed``,
``fixed``, ``refused``, ``notes``, ``summary`` (``errors``,
``warnings``, ``new``, ``baselined``, ``healed``) and ``exit_code``.

Check exit codes
~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Code
     - Meaning
   * - ``0``
     - No new error findings (warnings never fail the run), or
       ``--update-baseline`` wrote the baseline.
   * - ``1``
     - At least one error finding that is not in the baseline.
   * - ``2``
     - Setup problem: missing path, no ``*.Report`` folder found,
       missing or malformed baseline file, or ``--update-baseline``
       without ``--baseline``.
