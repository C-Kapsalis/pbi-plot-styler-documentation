CLI reference
=============

::

    pbi-plot-styler [style] [OPTIONS] REPORT_DIR
    pbi-plot-styler check [OPTIONS] PATH
    pbi-plot-styler colors sync [OPTIONS]
    pbi-plot-styler colors reset [OPTIONS]
    pbi-plot-styler bookmarks build [OPTIONS] [ENTRIES]
    pbi-plot-styler rename-measure [OPTIONS] OLD NEW
    pbi-plot-styler rename-label [OPTIONS] CLIENT_DIR
    pbi-plot-styler --version | --help

``pbi-plot-styler`` has six commands:

- ``style`` (the default) styles every targeted visual from the measures
  declared in the paired semantic model's field-parameter tables. When
  the first argument is not a command name, ``style`` runs, so
  ``pbi-plot-styler REPORT_DIR`` and ``pbi-plot-styler style REPORT_DIR``
  are the same command. See `style`_.
- ``check`` is a read-only integrity gate: it reports broken bindings,
  bookmarks that cannot be imported or that dangle, inconsistent
  field-parameter slots, and missing theme files. With ``--rule`` it
  checks the conventions of the standard canvas page instead. See
  `check`_ and :doc:`convention-rules`.
- ``colors sync`` and ``colors reset`` keep a report's series colors in
  step with the palette measure the model declares. See `colors`_.
- ``bookmarks build`` renders one answer bookmark per entry of a JSON
  file, from a base bookmark that is already known to work. See
  `bookmarks`_.
- ``rename-measure`` renames a measure in every visual and bookmark of a
  report, without touching the model. See `rename-measure`_.
- ``rename-label`` renames a field-parameter row label in the model and in
  every report that remembers it. See `rename-label`_.

A report folder literally named like a command must be passed as
``./<name>`` (or after an explicit ``style``): ``check``, ``style``,
``colors``, ``bookmarks``, ``rename-measure`` and ``rename-label`` all
take that name.

Where each command writes:

- ``style`` and ``check --fix`` write inside the report folders they are
  given; ``style --zip`` also writes the archive you name.
- ``colors`` and ``bookmarks build`` write inside the ``--report``
  folder.
- ``rename-measure`` writes in every ``*.Report`` folder below the
  parent folder of ``--model``.
- ``rename-label`` writes in the models and reports under
  ``CLIENT_DIR``.

When each command writes:

- ``colors`` and ``rename-label`` are dry runs unless ``--apply`` is
  given.
- ``check`` writes only with ``--fix``, and never with ``--fix
  --dry-run``.
- ``style``, ``rename-measure`` and ``bookmarks build`` write unless
  ``--dry-run`` is given.

Every example on this page runs unchanged in bash and in PowerShell,
unless a separate PowerShell form is shown. Paths may use ``/`` in both
shells. Quote any argument that contains a space or a ``#``.

``--version`` belongs to the top-level command, not to ``style``:

::

    pbi-plot-styler --version

::

    pbi-plot-styler, version 1.2.1

The top level and every command take ``-h`` / ``--help``.

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
       that contains one or more of them at any depth, or a page folder
       or a ``visual.json`` inside a report (only the visuals under it
       are restyled). In 1.2.1 a page folder or a ``visual.json`` must
       be given as an absolute path; see `Styling one page`_.

When ``REPORT_DIR`` is a folder of reports, every ``*.Report`` folder
under it is found recursively and styled against its own paired model
(each model is loaded once). A discovered folder without
``definition/pages`` (a report not saved in the PBIR format) is skipped;
``check`` says so on stderr, ``style`` says nothing.

If the report folder's name begins with a dash (``-Sales.Report``), the
option parser reads it as a flag. Prefix it with ``./``, or end option
parsing with ``--``:

::

    pbi-plot-styler "./-Sales.Report"
    pbi-plot-styler --dry-run -- "-Sales.Report"

Both forms work in bash and in PowerShell 7.

Styling one page
~~~~~~~~~~~~~~~~

To restyle only the visuals of one page, or one visual, pass the page
folder or the ``visual.json`` instead of the report. In 1.2.1 the path
must be absolute: a relative page path finds no visual and the run
exits ``2`` with ``no targeted visuals found`` (see
:doc:`known-issues`). Let the shell make the path absolute.

bash:

::

    pbi-plot-styler "$PWD/Roastery.Report/definition/pages/a1f935c9ad3918c55c49" --dry-run

PowerShell:

::

    pbi-plot-styler (Resolve-Path Roastery.Report/definition/pages/a1f935c9ad3918c55c49).Path --dry-run

Run from the repository's ``examples/coffee-roastery`` folder, both
print the same block and closing line as a dry run of the whole report,
because that page holds the report's only combo chart.

What the run prints
~~~~~~~~~~~~~~~~~~~

The examples in this section run from the repository's
``examples/coffee-roastery`` folder, which holds a ``plotstyler.toml``.

A write run prints one line per visual it wrote, then a blank line, the
number of models the run resolved, and the count of the combo charts:

::

    pbi-plot-styler Roastery.Report

::

      5dd25eb672a72da45096: dataPoint 0 -> 5, labels 0 -> 6 (5 line measures each; 1 columns-role measure(s) excluded; 1 bar-side entry preserved)

    Models resolved: 1
    rebuilt 1/1 combo chart(s).

The counts in brackets are the measures styled on that visual, the
measures left out through an excluded role (``--exclude-role``), and
the existing column-side entries kept as they were.

A dry run prints a block per visual that would change, instead of a
diff:

::

    pbi-plot-styler Roastery.Report --dry-run

::

      5dd25eb672a72da45096: WOULD CHANGE
        columns-role measure(s) kept OUT of the line set: Sales Measures.Revenue
        objects.dataPoint[]: 0 -> 5 entries
        objects.labels[]: 0 -> 6 entries

    Models resolved: 1
    1/1 combo chart(s) need rebuild (re-run without --dry-run).

The ``columns-role`` line names the measures bound to the column role
(``Y``) and any excluded measures. In 1.2.1 it is printed even when no
role is excluded, and then the measure it names is styled after all;
read the ``dataPoint`` count, or the write run's ``excluded`` count, to
see what happens (see :doc:`known-issues`).

When the field-parameter tables come from detection (no ``--table`` and
no ``[tables] field_parameters``), the run first prints one
``Field parameters:`` line; see `Field parameter detection`_.

The closing line is ``rebuilt <written>/<total> combo chart(s).`` after a
write, ``<pending>/<total> combo chart(s) need rebuild (re-run without
--dry-run).`` after a dry run that found drift, and ``All <total> combo
chart(s) already in sync.`` after a dry run that found none. When some
visuals were skipped, ``(<n> skipped)`` follows the count. A report the
run cannot pair with a model prints one line per combo visual:

::

      5dd25eb672a72da45096: SKIPPED - no paired SemanticModel (missing definition.pbir / no x-Selector measures)

Those visuals count in the total:

::

    Models resolved: 0
    All 1 combo chart(s) already in sync (1 skipped).

In 1.2.1 such a run exits ``0``, even for a single report or with
``--model`` (see :doc:`known-issues`). ``--zip`` adds a final ``Wrote
<path>`` line.

A ``visual.json`` that does not parse is reported on stderr as
``ERROR reading <path>: <error>`` and left alone; it counts as skipped:

::

      ERROR reading Bad.Report/definition/pages/a1f935c9ad3918c55c49/visuals/5dd25eb672a72da45096/visual.json: Expecting property name enclosed in double quotes: line 2 column 1 (char 2)

    Models resolved: 1
    rebuilt 0/1 combo chart(s) (1 skipped).

A write run then exits ``2``; a dry run prints ``0/1 combo chart(s) need
rebuild (1 skipped) (re-run without --dry-run).`` and exits ``1``.

Every page, every time
~~~~~~~~~~~~~~~~~~~~~~

When you name the report folder, every visual under
``definition/pages/**`` whose ``visualType`` matches ``visual_types``
is checked and, if needed, restyled on every run, across the whole
report. There is no option to leave pages out. This is deliberate: a
field-parameter table's measures can be plotted from any page, and a
partially-styled report is exactly the drift this tool exists to
prevent. (To work on one page at a time, name the page folder instead
of the report; see `Styling one page`_.) To leave a specific visual out of scope entirely, give it a
``visualType`` outside the configured set instead.

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
     - the model's own field parameters
     - Field-parameter table(s) to read measure references from. A name
       may be a shell-style pattern (``x-*``), matched against the
       model's tables case-insensitively. Passing the flag at least
       once replaces the whole set. To change just one of the detected
       tables without listing the others again, use ``--rename-table``
       instead. See `Field parameter detection`_.
   * - ``--rename-table OLD=NEW``
     - repeatable
     - none
     - Renames a single table in the current list (the detected ones, or
       whatever ``--table``/config set), leaving the others in place.
       ``OLD`` must match a name already present in that list; a misspelled
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
       `Series colors`_.
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
     - Data-label text color, set in the no-selector labels entry. The
       default only seeds a visual that has no labels entry yet. See
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
       literal ``NL`` (for example, ``20L``). Set explicitly, it is
       enforced on every measure; left alone, it only seeds a measure
       that has none.
   * - ``--seed-label-transparency N``
     - integer 0-100
     - not set
     - Seeds the data-label background transparency where a measure has
       none, and never overwrites an existing value. It cannot be
       combined with ``--label-transparency``.
   * - ``--show-labels`` / ``--hide-labels``
     - flag
     - show
     - Whether data labels are switched on in the no-selector labels
       entry.
   * - ``--rebind-stale-line``
     - flag
     - off
     - Before styling, point a line (``Y2``) measure projection whose
       name the model no longer declares at the current measure: leading
       symbols are stripped, then an exact name or a single
       case-insensitive match in the same table wins. The column role
       (``Y``) then counts as column side unless ``--exclude-role`` is
       given. Off by default.
   * - ``--dry-run``
     - flag
     - off
     - Compute all rewrites, print what would change, write nothing.
   * - ``--zip PATH``
     - file
     - not set
     - Also write the styled report folder(s) to ``PATH`` as a zip
       archive (rooted at the folder name of ``REPORT_DIR``). Combined
       with ``--dry-run``, the archive contains the styled content while
       the source files stay untouched.
   * - ``-h``, ``--help``
     - flag
     - not set
     - Print usage and exit.

All hex colors must be 6-digit ``#RRGGBB`` values.

Field parameter detection
~~~~~~~~~~~~~~~~~~~~~~~~~

With no ``--table`` and no ``[tables] field_parameters`` in the config
file, the tool detects the model's field parameters from its TMDL: a
calculated table whose column carries ``extendedProperty
ParameterMetadata`` and whose rows are ``NAMEOF(...)`` tuples. A
parameter called ``Bar Metrics`` is in the set; a table called
``x-Selector 1`` that carries no parameter metadata is not. The name
patterns (``x-*``, ``y-*``) only rank and label the detected tables;
they never replace the detection.

Each detected parameter is classified by what its rows point at: columns
make a **dimension** parameter, measures make a **metric** one, and a
table that points at both is flagged as mixed. The measures the styling
drives come from the metric parameters; the column wells and the
excluded roles work on the whole detected set, so a ``Y`` role bound
through a parameter you did not configure still excludes that
parameter's whole catalog.

The run prints one line naming the tables it uses. For the
``coffee-roastery`` model, styled from a folder that holds no
``plotstyler.toml``, the line reads:

::

    Field parameters: x-Plot Specific 1, x-Plot Specific 2, y-Plot Specific (dimension: y-Plot Specific; metric: x-Plot Specific 1, x-Plot Specific 2)

A model with no dimension parameter prints an empty list
(``dimension: ;``).

An explicit ``--table`` (or a config list) restricts the expansion to
that list, and no such line is printed.

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
    # the query role drawn as the line, and the one drawn as the columns
    line_role = "Y2"
    column_role = "Y"

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
    # the properties the seeded no-selector labels entry carries
    label_default_properties = { show = "true", color = "#252423", enableBackground = "true" }
    # false leaves an existing no-selector labels entry exactly as found
    touch_existing_label_default = true
    # --seed-label-transparency; not together with label_transparency
    # label_transparency_seed = 20

    # --color, one pin per measure
    [style.colors]
    "Sales Measures.Revenue" = "#7A4419"

The same file also carries the sections the other commands read. ``style``
accepts them and ignores them, so one ``plotstyler.toml`` serves the
whole tool. ``check`` reads the canvas sections, and ``bookmarks build``
reads ``[bookmarks]``:

.. list-table::
   :header-rows: 1

   * - Section
     - Read by
     - Keys
   * - ``[canvas]``
     - ``check``
     - ``matrix_types``, ``combo_types``, ``excluded_page_fragments``,
       ``selector_tables``, ``measure_selector_tables``,
       ``dimension_selector_tables``, ``exclude_reports``
   * - ``[gates]``
     - ``check``
     - ``table``, ``slicer_measure``, ``list_measure``, ``mode_map``,
       ``name_heuristic``
   * - ``[category]``
     - ``check``
     - ``table``, ``column``, ``chrome_words``, ``chrome_prefixes``
   * - ``[placeholders]``
     - ``check``
     - ``blank_dimension_table``, ``template_slot_table``
   * - ``[panel]``
     - ``check``
     - ``group_name``, ``filter_tables``, ``min_slicers``, ``min_combos``,
       ``min_matrices``
   * - ``[format_selector]``
     - ``check``
     - ``table``, ``column``, ``item``, ``filter_name``
   * - ``[answer_bookmarks]``
     - ``check``
     - ``prefixes``, ``main_page``, ``matrix``, ``slicers``,
       ``row_dropdowns``, ``inherited_fingerprints``, ``submenu_prefix``,
       ``slot_mode``, ``baseline``
   * - ``[bookmarks]``
     - ``bookmarks build``
     - ``base``, ``matrix``, ``chart``, ``row_picker``,
       ``row_dropdowns``, ``category_column``, ``calendar_table``,
       ``calendar_hierarchy``, ``pickers``, ``wells``, ``legacy_measures``
   * - ``[caption]``
     - nothing
     - accepted for compatibility; no rule reads it

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

``[style] label_default_properties`` replaces the built-in property set
of the seeded no-selector labels entry (``show``, ``color`` and
``enableBackground``) with the properties you name, each written as a
DAX literal. ``[style] touch_existing_label_default = false`` leaves an
existing no-selector labels entry exactly as it was found, instead of
forcing ``enableBackground`` on it.

Series colors
~~~~~~~~~~~~~

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

A related case the tool cannot fix: if the exact same measure is
projected on *both* the column and the line role of one combo chart,
Power BI only
keeps one ``selector.metadata`` match per visual, so one of the two
falls back to default rendering, and the PBIP schema offers no per-role
disambiguation. Use a thin wrapper measure (``Revenue (Line) = [Revenue]``)
if the same value genuinely needs to appear on both.

Enforcing style in CI
~~~~~~~~~~~~~~~~~~~~~

``--dry-run`` never writes anything; it prints what would change and its
exit code tells a pipeline whether to pass (see `Style exit codes`_). A
minimal GitHub Actions step:

::

    - name: Check report styling
      run: pbi-plot-styler "My Project.Report" --dry-run
    - name: Check report integrity
      run: pbi-plot-styler check "My Project.Report"

Point ``--config`` at a committed ``plotstyler.toml`` if your colors are
not the defaults, so the CI check and a local run agree. This turns a
style guide into a gate: a pull request that adds or renames a measure
in a field-parameter table without re-running the styler fails until
someone runs the one command.

Keep the ``check`` step next to it: in 1.2.1 a dry run whose model
cannot be loaded skips the report and exits ``0``, while ``check``
reports that model as ``MDL001`` and fails.

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
     - Error: bad invocation, invalid config value (for example a color
       that is not ``#RRGGBB``), a path that does not exist
       (``ERROR: <absolute path> does not exist``), a ``--model`` folder
       that does not exist, a ``visual.json`` that does not parse in a
       write run (``ERROR reading <path>: <error>``), no ``*.Report``
       folder found, none of the configured field-parameter tables
       found, no measures resolved, no visual of a targeted type found,
       a color pin that matches no measure, an unknown config
       section/key, an unmatched ``--rename-table`` source name,
       ``label_transparency`` together with ``label_transparency_seed``,
       a malformed ``OLD=NEW`` or ``KEY=HEX`` pair, or a rebuild that
       would alter a preserved entry.

A report whose model cannot be resolved or loaded is skipped, not an
error, in 1.2.1: the run exits ``0`` (see `What the run prints`_ and
:doc:`known-issues`).

check
-----

::

    pbi-plot-styler check [OPTIONS] PATH

Checks the report at ``PATH`` (a ``*.Report`` folder, or a folder of
them, found recursively) and prints one line per finding. Read-only
unless ``--fix`` is given. Without ``--rule``, ``check`` runs the
``integrity`` rule; with ``--rule SLUG`` it runs that convention rule
instead. ``check`` reads the canvas sections of ``plotstyler.toml``
(see `Config file`_).

From the repository's ``examples/coffee-roastery`` folder:

::

    pbi-plot-styler check Roastery.Report

::

    Checked 1 report(s): 0 error(s), 0 warning(s).

A clean run prints only the closing totals and exits ``0``.

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
   * - ``--rule SLUG``
     - repeatable
     - ``integrity``
     - Run this rule instead of the default ``integrity`` checks. The
       twelve slugs are listed in `Convention rules`_.
   * - ``--config PATH``
     - file
     - ``./plotstyler.toml`` if present
     - TOML config file with the canvas sections. See
       `Config file`_.
   * - ``--report NAME``
     - text
     - not set
     - Check only the report folder with this name (with or without the
       ``.Report`` suffix). Overrides the excluded-reports list.
   * - ``--model-name NAME``
     - text
     - not set
     - Check only the reports whose ``definition.pbir`` points at
       ``<NAME>.SemanticModel``.
   * - ``--json``
     - flag
     - off
     - Print one JSON document instead of text (see `JSON output`_).
   * - ``--fix``
     - flag
     - off
     - Run the fixers of the selected rules, then check again from disk;
       the exit code comes from that re-check. ``integrity`` fixes
       ``BMK001`` and ``BMK002``; ``slicer-gates``,
       ``slicer-mode-gates``, ``filter-panel``, ``format-selector`` and
       ``stale-placeholders`` have fixers too (see
       :doc:`convention-rules`). Fixers write only the files that need a
       change.
   * - ``--dry-run``
     - flag
     - off
     - With ``--fix``: print the diff of every change the fixers would
       make and write nothing. Exits ``1`` when there is something to
       fix. Without ``--fix`` it is a usage error.
   * - ``--summary``
     - flag
     - off
     - Run the four foundation rules (``slicer-gates``,
       ``category-filter``, ``theme``, ``filter-panel``) and print a
       compliance report after the findings. It replaces ``--rule``.
       Never fails: it exits ``0`` whatever it finds (a setup problem
       still exits ``2``). See `Compliance summary`_.
   * - ``--only-display-names FILE``
     - file
     - not set
     - ``bookmark-literals``: check only the bookmarks the JSON file
       names (a list of names, or a ``bookmarks build`` entries file).
   * - ``--baseline FILE``
     - file
     - not set
     - Accepted-findings file. Only findings not in it fail the run.
   * - ``--update-baseline``
     - flag
     - off
     - Write every current finding to the ``--baseline`` file and exit
       ``0`` (``ABK000`` is never written, and the run exits ``1`` while
       it occurs).

A ``--report`` or ``--model-name`` filter that matches nothing exits
``2`` and names the filter. A sweep over a folder that holds no
``*.Report`` folder prints ``no *.Report folders under <path>; nothing to
check`` and exits ``0``.

A sweep also skips the reports named by ``[canvas] exclude_reports``
(by default ``M.Report`` and ``-M.Report``), the API-only report family.
``--report`` overrides that exclusion.

Convention rules
~~~~~~~~~~~~~~~~

``--rule`` takes the slug of a rule. A run with several slugs runs each
of them, in the order given, and every rule reports through the same
machinery: stable ids, ``--json``, ``--baseline`` and
``--update-baseline``, and the same exit codes. A rule that cannot judge
a report says so in a ``skipped`` line instead of passing silently. Each
rule, its settings, its output and its fixer are described in
:doc:`convention-rules`.

.. list-table::
   :header-rows: 1

   * - Slug
     - Checks
     - Ids
   * - ``integrity``
     - Field bindings resolve in the paired model, bookmarks are
       importable and point at real pages and visuals, field-parameter
       slots are consistent, theme files exist. The default. Fixable
       (``BMK001``, ``BMK002``).
     - ``JSN001``, ``BND001``, ``BND002``, ``BMK001`` to ``BMK005``,
       ``FPM001``, ``FPM002``, ``THM001``
   * - ``slicer-gates``
     - Every selector slicer on a visible canvas page carries an active
       ``[f-Slicer] > 0`` filter, and a bookmark capture of one keeps
       it. Fixable.
     - ``GAT001``, ``GAT002``
   * - ``slicer-mode-gates``
     - The gate matches the slicer's mode: a drop-down list needs
       ``[f-Slicer] > 0``, a vertical list needs ``[f-All] > 0``. The
       gate is bound to the table that declares the measure. Fixable.
     - ``GAT003``, ``GAT004``, ``GAT005``
   * - ``category-filter``
     - A page-level bookmark sets the analysis-category filter.
     - ``CAT001``
   * - ``theme``
     - The report declares a theme collection, a base and a custom
       theme, and no hex color literals in ``report.json``.
     - ``THM002`` to ``THM006``
   * - ``visual-position``
     - Every visual has an x and a y position.
     - ``POS001``
   * - ``y-combo``
     - The y-combo groups are named ``y<N>_combo_group``, and every
       member is inside one.
     - ``YCG001``, ``YCG002``
   * - ``filter-panel``
     - The filter slicers live in the ``y_filters_group`` master
       container. Fixable.
     - ``PNL001``, ``PNL002``
   * - ``format-selector``
     - A hidden, locked report filter selects the ``Format`` item of the
       format-selector table. Fixable.
     - ``FMT001``
   * - ``stale-placeholders``
     - A saved state does not name a blank-dimension placeholder its
       axis does not offer. Fixable.
     - ``PLH001``
   * - ``answer-bookmarks``
     - The whole-page answer bookmarks capture the matrix, their own
       rows, the pinned analysis category, a place in the bookmarks
       menu, and the slot map of their picks.
     - ``ABK000``, ``ABK001``, ``ABK003`` to ``ABK010``, ``PGW001``,
       ``PGW002``, ``JSN001``
   * - ``bookmark-literals``
     - Every picker literal a bookmark captures is a row of the selector
       behind that picker.
     - ``BML001``, ``BML002``, ``JSN001``

Compliance summary
~~~~~~~~~~~~~~~~~~

``check --summary`` runs four of the rules (``slicer-gates``,
``category-filter``, ``theme`` and ``filter-panel``), prints their
findings, and then prints the compliance report: per rule, its number
and name, ``PASS`` or ``FAIL``, the components checked and passed, and
each finding with its type and a ``Fix`` hint; then the overall
compliance, the number of rules passed, the score and the counts of
critical issues and warnings. A rule passes when it has no error
finding.

``--summary`` never fails: it exits ``0`` whatever it finds. Only a
setup problem (a missing path, for example) exits ``2``.

Each row counts only some of its rule's ids. The theme row scores
``THM002``, ``THM003`` and ``THM004``; ``THM005`` and ``THM006`` run
only under ``--rule theme``. Since 1.2.1 the finding lines printed above
the report, the closing ``Checked`` totals, and the ``findings`` and
``summary`` of ``--json`` carry only the ids the rows count, so they
agree with the score. A summary run with ``--baseline`` does not call
the accepted entries of the other ids healed, and ``--update-baseline``
keeps them.

The repository's ``tests/fixtures/brewbar`` report declares a base
theme but no custom one. From that folder, ``--rule theme`` finds two
problems:

::

    pbi-plot-styler check Brewbar.Report --rule theme

::

    WARNING  THM003  Brewbar.Report/definition/report.json: No custom theme defined - may use hardcoded colors (Expected: customTheme configuration, Got: null or missing)
    ERROR    THM006  Brewbar.Report/definition/report.json: Theme collection is incomplete - missing: customTheme. Reports must declare both baseTheme and customTheme.
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

The summary scores the theme row on ``THM003`` alone and does not list
``THM006``:

::

    pbi-plot-styler check Brewbar.Report --summary

::

    skipped  Brewbar.Report: filter-panel: the report has no group containers
    No Report field-parameter slicers found under <absolute path>/Brewbar.Report
    ERROR    CAT001  Brewbar.Report/definition/bookmarks/bm02.bookmark.json: Page-level bookmark has no Analysis Category filter (Expected: a filter in byName or byExpr targeting z-Map Dimension-Measure Groupings[Analysis Category], Got: no such filter found)
    WARNING  THM003  Brewbar.Report/definition/report.json: No custom theme defined - may use hardcoded colors (Expected: customTheme configuration, Got: null or missing)

    🔍 Template Foundation Rules Comprehensive Report:
    ============================================================
    # Template Foundation Rules Validation Report

    **Overall Compliance**: 75.0% (3/4 rules)
    **Violations**: 1 critical, 1 warnings

    ## Rule #1: Field Parameter f-Slicer Filters ✅ PASS
    Components: 0/0 passed

    ## Rule #3: Analysis Category Filter Required ❌ FAIL
    Components: 0/1 passed

    **🔴 missing_analysis_category** (Shops table)
    - Page-level bookmark has no Analysis Category filter
    - **Fix**: In Power BI Desktop, with this bookmark active, set the Analysis Category filter on the page (via the z-Map Dimension-Measure Groupings table), then UPDATE the bookmark so it captures that state.

    ## Rule #6: ThemeDataColor Integration ✅ PASS
    Components: 1/1 passed

    **⚠️ missing_custom_theme** (report)
    - No custom theme defined - may use hardcoded colors
    - **Fix**: Define customTheme to avoid hardcoded color values

    ## Rule #8: y_filters_group Master Container ✅ PASS
    Components: 0/0 passed


    📊 Summary:
       Overall Compliance: False
       Rules Passed: 3/4
       Compliance Score: 75.0%
       Critical Issues: 1
       Warnings: 1

    Checked 1 report(s): 1 error(s), 1 warning(s).

It exits ``0`` although ``CAT001`` is an error. The second line names
the report folder by its absolute path. With ``--json``, the same
numbers arrive in a ``compliance`` object (see `JSON output`_).

Checks
~~~~~~

Ids are stable: an id never changes meaning, so a committed baseline
keeps working across releases. The table below is the tool's own
``CHECKS`` table, in the order the tool holds it. It is generated, not
written by hand: read it out of the installed package with

::

    python -c "from pbi_plot_styler.check import CHECKS; [print(i, s, t) for i, (s, t) in CHECKS.items()]"

or regenerate the whole table (each id, its severity, its short title
and the rule that emits it) with

::

    from pbi_plot_styler.check import CHECKS
    from pbi_plot_styler.rules import rules

    primary = {}
    for slug, rule in rules().items():
        for check_id in rule.ids:
            primary.setdefault(check_id, slug)

    for check_id, (severity, title) in CHECKS.items():
        print(check_id, severity, primary.get(check_id, "every report"), title)

.. list-table::
   :header-rows: 1

   * - Id
     - Severity
     - Rule
     - Finding
   * - ``JSN001``
     - error
     - ``integrity``
     - invalid JSON
   * - ``MDL001``
     - error
     - ``every report``
     - the paired semantic model cannot be loaded
   * - ``BND001``
     - error
     - ``integrity``
     - unresolved field reference
   * - ``BND002``
     - error
     - ``integrity``
     - unresolved queryRef / metadata string
   * - ``BMK001``
     - error
     - ``integrity``
     - unknown bookmark options key
   * - ``BMK002``
     - error
     - ``integrity``
     - bookmark integer stored as string
   * - ``BMK003``
     - error
     - ``integrity``
     - bookmark names a missing page
   * - ``BMK004``
     - error
     - ``integrity``
     - bookmark captures a missing visual
   * - ``BMK005``
     - error
     - ``integrity``
     - bookmarks.json lists a missing bookmark
   * - ``FPM001``
     - error
     - ``integrity``
     - overlapping field-parameter slots
   * - ``FPM002``
     - error
     - ``integrity``
     - field-parameter slots exceed projections
   * - ``THM001``
     - warning
     - ``integrity``
     - theme file not found
   * - ``ABK000``
     - error
     - ``answer-bookmarks``
     - no answer bookmark matches the configured prefixes
   * - ``ABK001``
     - error
     - ``answer-bookmarks``
     - answer bookmark is scoped to some visuals
   * - ``ABK003``
     - error
     - ``answer-bookmarks``
     - answer bookmark does not show the matrix
   * - ``ABK004``
     - error
     - ``answer-bookmarks``
     - matrix capture holds another bookmark's fields
   * - ``ABK005``
     - error
     - ``answer-bookmarks``
     - matrix capture has a stale shape
   * - ``ABK006``
     - error
     - ``answer-bookmarks``
     - answer bookmark picker selects nothing
   * - ``ABK007``
     - error
     - ``answer-bookmarks``
     - answer bookmark does not pin the analysis category
   * - ``ABK008``
     - error
     - ``answer-bookmarks``
     - answer bookmark is not under a submenu
   * - ``ABK009``
     - error
     - ``answer-bookmarks``
     - matrix slot map does not match the picks
   * - ``ABK010``
     - error
     - ``answer-bookmarks``
     - row dimension has no dropdown filter
   * - ``PGW001``
     - error
     - ``answer-bookmarks``
     - raw measure in a field-parameter well
   * - ``PGW002``
     - error
     - ``answer-bookmarks``
     - bookmark navigator names a missing bookmark
   * - ``BML001``
     - error
     - ``bookmark-literals``
     - picker literal registered only in another selector
   * - ``BML002``
     - error
     - ``bookmark-literals``
     - picker literal not registered in any selector
   * - ``CAT001``
     - error
     - ``category-filter``
     - bookmark does not set the analysis category
   * - ``GAT001``
     - error
     - ``slicer-gates``
     - selector slicer has no active gate
   * - ``GAT002``
     - error
     - ``slicer-gates``
     - bookmark drops a selector slicer's gate
   * - ``GAT003``
     - error
     - ``slicer-mode-gates``
     - slicer gate missing or passive for its mode
   * - ``GAT004``
     - error
     - ``slicer-mode-gates``
     - slicer gate bound to the wrong table
   * - ``GAT005``
     - error
     - ``slicer-mode-gates``
     - stale gate table reference
   * - ``THM002``
     - error
     - ``theme``
     - no theme collection
   * - ``THM003``
     - warning
     - ``theme``
     - no custom theme
   * - ``THM004``
     - error
     - ``theme``
     - hex colour literal in report.json
   * - ``THM005``
     - error
     - ``theme``
     - neither a base nor a custom theme
   * - ``THM006``
     - error
     - ``theme``
     - base or custom theme missing
   * - ``POS001``
     - error
     - ``visual-position``
     - visual has no x or y position
   * - ``YCG001``
     - error
     - ``y-combo``
     - y-combo group badly named
   * - ``YCG002``
     - error
     - ``y-combo``
     - y-combo member outside its group
   * - ``PNL001``
     - error
     - ``filter-panel``
     - filter-panel master group missing
   * - ``PNL002``
     - warning
     - ``filter-panel``
     - filter-panel master is not a group
   * - ``FMT001``
     - error
     - ``format-selector``
     - format-selector filter missing or incomplete
   * - ``PLH001``
     - error
     - ``stale-placeholders``
     - stale blank-dimension placeholder

The eleven ids of the ``integrity`` rule, in prose:

- ``JSN001``: a report file (``report.json``, ``page.json``,
  ``visual.json``, a bookmark) is not valid JSON.
- ``BND001``: a field reference does not resolve in the paired model.
  It is a ``Measure``, ``Column`` (including inside an ``Aggregation``)
  or ``HierarchyLevel`` whose table, field, hierarchy or level does not
  exist, or a ``From`` entity of a filter. ``SourceRef.Source``
  aliases are resolved through the enclosing ``From`` list.
- ``BND002``: a ``queryRef`` or ``selector.metadata`` string does not
  resolve.
- ``BMK001``: a bookmark ``options`` key is outside the allowed set:
  ``targetVisualNames``, ``applyOnlyToTargetVisuals``,
  ``suppressData``, ``suppressActiveSection``, ``suppressDisplay``.
  The file is valid JSON, but the service rejects the whole report on
  import. Fixable.
- ``BMK002``: a bookmark ``howCreated``, ``ComparisonKind`` or
  ``Version`` is a string, not a JSON integer. Fixable when the string
  is numeric.
- ``BMK003``: a bookmark's ``explorationState.activeSection``, or a key
  of its ``sections``, names a page that does not exist.
- ``BMK004``: a key of ``sections.<page>.visualContainers`` (or
  ``visualContainerGroups``) is not a visual on that page.
- ``BMK005``: ``bookmarks.json`` lists a bookmark (top level or a
  group's ``children``) that has no ``.bookmark.json`` file.
- ``FPM001``: two field parameters of one query role claim the same
  slot (``queryState.<role>.fieldParameters[]`` ranges overlap); the
  later one wins, so the earlier one's picks never render.
- ``FPM002``: a field parameter's slot range (``index`` plus
  ``length``) runs past the role's projections.
- ``THM001``: a ``report.json`` ``themeCollection`` entry has no item in
  its resource package, or its file is missing under
  ``StaticResources``.

``MDL001`` is not a rule's finding: the command itself raises it when a
report names a semantic model that cannot be loaded. The rules that
would have used it are skipped, and their notes say why.

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

This section describes the fixers of the ``integrity`` rule; the
convention rules' fixers are described in :doc:`convention-rules`.

The example below starts from a copy of the repository's
``tests/fixtures/brewbar`` report in which the ``bm01`` bookmark was
given an unknown ``options`` key and a ``howCreated`` stored as a
string. ``check`` finds both:

::

    pbi-plot-styler check Brewbar.Report

::

    ERROR    BMK001  Brewbar.Report/definition/bookmarks/bm01.bookmark.json: options.applyToAllVisuals is not an allowed bookmark option; the service rejects the whole report on import
    ERROR    BMK002  Brewbar.Report/definition/bookmarks/bm01.bookmark.json: howCreated is the string '1'; it must be a JSON integer

    Checked 1 report(s): 2 error(s), 0 warning(s).

Preview the fix with ``--fix --dry-run``. It prints what it would fix
and a unified diff, writes nothing, and exits ``1``:

::

    pbi-plot-styler check Brewbar.Report --fix --dry-run

::

    would fix Brewbar.Report/definition/bookmarks/bm01.bookmark.json: removed options.applyToAllVisuals
    would fix Brewbar.Report/definition/bookmarks/bm01.bookmark.json: converted 1 string integer(s)
    --- a/Brewbar.Report/definition/bookmarks/bm01.bookmark.json
    +++ b/Brewbar.Report/definition/bookmarks/bm01.bookmark.json
    @@ -2,8 +2,7 @@
       "displayName": "Online only",
       "name": "bm01",
       "options": {
    -    "targetVisualNames": [],
    -    "applyToAllVisuals": true
    +    "targetVisualNames": []
       },
       "explorationState": {
         "version": "1.3",
    @@ -13,7 +12,7 @@
             {
               "name": "bf1",
               "type": "Categorical",
    -          "howCreated": "1",
    +          "howCreated": 1,
               "expression": {
                 "Column": {
                   "Expression": {
    ERROR    BMK001  Brewbar.Report/definition/bookmarks/bm01.bookmark.json: options.applyToAllVisuals is not an allowed bookmark option; the service rejects the whole report on import
    ERROR    BMK002  Brewbar.Report/definition/bookmarks/bm01.bookmark.json: howCreated is the string '1'; it must be a JSON integer

    Checked 1 report(s): 2 error(s), 0 warning(s).

Then apply it. The run checks again from disk, and its exit code comes
from that re-check:

::

    pbi-plot-styler check Brewbar.Report --fix

::

    fixed    Brewbar.Report/definition/bookmarks/bm01.bookmark.json: removed options.applyToAllVisuals
    fixed    Brewbar.Report/definition/bookmarks/bm01.bookmark.json: converted 1 string integer(s)

    Checked 1 report(s): 0 error(s), 0 warning(s).

``--fix`` edits the bookmark text surgically and never re-serializes
the file, so the change reviews as a one-line diff. Each edit is proven
by parsing the text before and after and requiring that the only
difference is the removed key or the converted integers. If that cannot
be proven (for example, the key cannot be located exactly once), the fix
is refused, the file is left untouched, a ``REFUSED`` line explains why,
and the finding still counts.

Over several reports, ``--fix`` checks every report's prerequisites
before writing anything, so a setup problem never leaves half a fix.

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
baseline. Rerun with ``--update-baseline`` after healing findings to
shrink it.

``FPM001``'s location carries the slot as well as the two field
parameters that claim it (``<file>::<role>:<earlier>|<later>#slotN``),
so each overlapping slot is a finding of its own, and the baseline
holds an entry for each. A baseline recorded before the slot became
part of the location needs ``--update-baseline`` for those entries.

``--update-baseline`` with ``--rule``, ``--report`` or ``--model-name``
rewrites only the entries of that scope; the entries of other rules and
other reports are kept. ``ABK000`` is never written to a baseline: it
means the suite checked nothing.

On the same ``Brewbar.Report`` copy as in `Fixes`_, before the fix,
recording the baseline prints the findings and then:

::

    Wrote 2 finding(s) to check-baseline.json.

The file lists each accepted finding by id and location:

::

    {
      "version": 1,
      "findings": [
        {
          "id": "BMK001",
          "location": "Brewbar.Report/definition/bookmarks/bm01.bookmark.json::options.applyToAllVisuals"
        },
        {
          "id": "BMK002",
          "location": "Brewbar.Report/definition/bookmarks/bm01.bookmark.json::explorationState.filters.byExpr.howCreated"
        }
      ]
    }

A run with the baseline then passes:

::

    pbi-plot-styler check Brewbar.Report --baseline check-baseline.json

::

    Checked 1 report(s): 0 error(s), 0 warning(s), 2 accepted by the baseline, 0 healed.

After ``--fix``, the same command lists the healed entries, one line per
id:

::

    BMK001: 1 accepted finding(s) no longer occur — re-capture the baseline (pbi-plot-styler check --update-baseline) to lock it in: ['Brewbar.Report/definition/bookmarks/bm01.bookmark.json::options.applyToAllVisuals']
    BMK002: 1 accepted finding(s) no longer occur — re-capture the baseline (pbi-plot-styler check --update-baseline) to lock it in: ['Brewbar.Report/definition/bookmarks/bm01.bookmark.json::explorationState.filters.byExpr.howCreated']

    Checked 1 report(s): 0 error(s), 0 warning(s), 0 accepted by the baseline, 2 healed.

JSON output
~~~~~~~~~~~

``--json`` prints one object with ``tool``, ``version``, ``rules`` (the
slugs the run used), ``reports``, ``findings`` (each with ``id``,
``severity``, ``file``, ``subject``, ``location``, ``message``,
``fixable`` and ``baselined``), ``checked`` (what each rule judged),
``healed``, ``fixed``, ``planned`` (what ``--fix --dry-run`` would
write), ``refused``, ``skipped``, ``notes``, ``summary`` (``errors``,
``warnings``, ``new``, ``baselined``, ``healed``) and ``exit_code``.

A ``--summary`` run changes two things:

- each finding also carries ``fix``, the one-line remediation hint the
  text report prints after ``Fix``;
- a ``compliance`` object follows ``exit_code``, with
  ``total_rules``, ``compliant_rules``, ``compliance_percentage``,
  ``critical_violations``, ``warning_violations``, ``info_violations``,
  ``overall_compliance`` and ``rules``. Each entry of ``rules`` has
  ``rule`` (the slug), ``title``, ``number``, ``report_name``,
  ``checked``, ``passed``, ``compliant``, ``errors``, ``warnings`` and
  ``findings``; each of those findings has ``id``, ``subject``,
  ``severity``, ``type``, ``description``, ``component`` and ``fix``.

In a summary run, ``findings`` and ``summary`` hold only the ids the
compliance rows count (see `Compliance summary`_). For example, on the
``examples/coffee-roastery`` report, ``check Roastery.Report --summary
--json`` lists one finding, ``THM003``, with ``"fix": "Define
customTheme to avoid hardcoded color values"``, and its ``compliance``
object starts:

::

    "compliance": {
      "total_rules": 4,
      "compliant_rules": 4,
      "compliance_percentage": 100.0,
      "critical_violations": 0,
      "warning_violations": 1,
      "info_violations": 0,
      "overall_compliance": true,

Check exit codes
~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Code
     - Meaning
   * - ``0``
     - No new error findings (warnings never fail the run), or
       ``--update-baseline`` wrote the baseline, or a ``--summary``
       run, or a folder that holds no ``*.Report`` folder (``nothing to
       check``).
   * - ``1``
     - At least one error finding that is not in the baseline; after
       ``--fix``, one left by the re-check. ``--fix --dry-run`` also
       exits ``1`` when there is something to fix.
   * - ``2``
     - Setup problem: missing path, a ``--report`` or ``--model-name``
       filter that matched nothing, missing or malformed baseline file,
       ``--update-baseline`` without ``--baseline``, ``--dry-run``
       without ``--fix``, an unknown ``--rule`` slug, or a rule that
       cannot start (for example ``answer-bookmarks`` without
       ``[answer_bookmarks] prefixes``; see :doc:`convention-rules`).

``--summary`` never fails: whatever it finds, it exits ``0``. Only a
setup problem makes it exit ``2``.

colors
------

::

    pbi-plot-styler colors sync [OPTIONS]
    pbi-plot-styler colors reset [OPTIONS]

``colors`` keeps a report's series colors in step with the palette
measure the model declares. The color rule lives in the model as a
palette measure (by default ``'w-Colors'[c-Measure Color]``) of the form:

.. code-block:: text

    VAR _order = SELECTEDVALUE ( 'x-Selector 1'[x-Selector 1 Order] )
    VAR _slot = MOD ( _order + 0, 3 ) + 1
    RETURN SWITCH ( _slot, 1, "#111111", 2, "#222222", 3, "#333333" )

A measure's color is the slot its row's sort order lands on in the
picker field parameter named by ``SELECTEDVALUE``. The palette must
declare exactly ``modulus`` slots.

``sync`` stamps the column-series fills of the charts and the value and
header colors of the matrices bound to that picker, for every
registered measure, including the measures a future bookmark selects. It
processes bookmark formatting overrides as well as base visual
formatting. The default scope is every matrix that uses the palette's
picker in a value role, plus the charts with a bookmarked multi-measure
selection; ``--all-visuals`` includes every supported chart and table.

``reset`` reverts what ``sync`` wrote: chart series go back to the
report theme, and the matrix overrides lose their font color and their
style flags.

Both are dry runs unless ``--apply`` is given.

Colors options
~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Option
     - Type
     - Default
     - Description
   * - ``--report PATH``
     - directory
     - required
     - The ``*.Report`` folder.
   * - ``--model PATH``
     - directory
     - from ``definition.pbir``
     - The ``*.SemanticModel`` folder.
   * - ``--apply``
     - flag
     - off
     - Write the changes. Without it nothing is written.
   * - ``--measure NAME``
     - text
     - ``c-Measure Color``
     - The palette measure.
   * - ``--palette-table NAME``
     - text
     - ``w-Colors``
     - The table searched first for the palette measure.
   * - ``--default-color-id N``
     - integer
     - ``9``
     - The theme data color ``reset`` restores the chart series to.
   * - ``--visual-type NAME``
     - repeatable
     - ``barChart``, ``columnChart``, ``clusteredColumnChart``,
       ``stackedColumnChart``, ``hundredPercentStackedColumnChart``,
       ``clusteredBarChart``, ``stackedBarChart``,
       ``hundredPercentStackedBarChart``, ``lineChart``,
       ``areaChart``, ``stackedAreaChart``,
       ``lineClusteredColumnComboChart``, ``lineStackedColumnComboChart``,
       ``pivotTable``, ``tableEx``
     - Limit the change to these visual types, on visuals and on
       bookmark captures.
   * - ``--min-series N``
     - integer
     - ``2``
     - (``sync`` only) report a bookmark that pins at least this many
       bars.
   * - ``--all-visuals``
     - flag
     - off
     - (``sync`` only) include every chart and table, beyond the picker
       matrices and the bookmarked multi-measure charts.

What colors prints
~~~~~~~~~~~~~~~~~~

Every run opens with the palette and the number of registered measures.
A multi-series bookmark is then listed as a block, one line per series,
with its role, its sort order and its color, and the run closes with
the number of entries rewritten and added and the files it would change.
From the repository's ``tests/fixtures/color-sync`` folder:

::

    pbi-plot-styler colors sync --report Shop.Report

::

    palette   : x-Selector 1 -> MOD(order+0, 3)+1 -> 3 slots
    registered: 2 measures on the picker

      Sales | Both  [combo01]
          bar  Sales Measures.Revenue                                       order=1      #222222
          bar  Sales Measures.Orders                                        order=2      #333333
    would write 1 file(s): 0 colour entry/entries rewritten, 2 added
       Shop.Report/definition/pages/aa11bb22cc33dd44ee55/visuals/combo01/visual.json

    (dry run -- pass --apply to write)

The run exits ``1``. With ``--apply`` the first word is ``wrote``, the
dry-run note is gone, and the run exits ``0``. A dry run that finds
nothing to change prints ``would write 0 file(s): 0 colour entry/entries
rewritten, 0 added``, no file list and no note, and exits ``0``.

``reset`` prints the same two header lines, then the count and the
files:

::

    pbi-plot-styler colors reset --report Shop.Report

::

    palette   : x-Selector 1 -> MOD(order+0, 3)+1 -> 3 slots
    registered: 2 measures on the picker

    would restore 2 colour entry/entries in 1 file(s)
       Shop.Report/definition/pages/aa11bb22cc33dd44ee55/visuals/combo01/visual.json

    (dry run -- pass --apply to write)

With ``--apply`` the line reads ``restored 2 colour entry/entries in 1
file(s)``.

A visual that uses the picker but cannot carry a per-measure color
prints an ``UNSUPPORTED:`` line; two series that share a slot print a
``COLLISIONS`` block; a series the picker does not register prints a
``NOT REGISTERED`` block.

Colors exit codes
~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Code
     - Meaning
   * - ``0``
     - The changes were written, or there was nothing to change.
   * - ``1``
     - A dry run found changes to make. Rerun with ``--apply``.
   * - ``2``
     - Invalid input: the palette or the picker cannot be read, a
       ``NAMEOF`` cannot be resolved, or no measure is registered. The
       error is printed as ``FAIL: <reason>`` and nothing is written.
       For example, a model with no ``w-Colors`` table prints
       ``FAIL: <model>/definition/tables/w-Colors.tmdl not found``.
       A missing or unreadable ``--report`` folder is a usage error,
       also exit ``2``.

bookmarks
---------

::

    pbi-plot-styler bookmarks build [OPTIONS] [ENTRIES]

``bookmarks build`` renders one bookmark per entry of ``ENTRIES`` (a
JSON list) into the report, from a deep copy of a base bookmark that is
already known to work. Each written bookmark gets a fresh
20-hex-character name.

The entry keys are ``displayName`` (required), ``base``, ``pickers``,
``show``, ``hide``, ``show_groups``, ``hide_groups``, ``target_visuals``,
``analysis_category``, ``date_filter``, ``drill_level``,
``unhide_matrix``, ``matrix_sort``, ``group`` and ``clear_filters``.
Any other key is ignored.

The settings live in the ``[bookmarks]`` section of the config file (see
`Config file`_): ``base``, ``matrix``, ``chart``, ``row_picker``,
``row_dropdowns``, ``category_column``, ``calendar_table`` (default
``__Calendar``), ``calendar_hierarchy`` (default ``Calendar
Hierarchy``), ``pickers`` (a table of picker key to slicer visual id)
and ``wells`` (a table of well name to the picker keys that feed it).

Bookmarks options
~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Option
     - Type
     - Default
     - Description
   * - ``ENTRIES``
     - file
     - required, except with ``--self-test``
     - The JSON list of entries.
   * - ``--report PATH``
     - directory
     - required
     - The report folder the bookmarks are written into.
   * - ``--config PATH``
     - file
     - ``./plotstyler.toml`` if present
     - Config file with a ``[bookmarks]`` section.
   * - ``--base ID``
     - text
     - ``[bookmarks] base``
     - The bookmark id to clone as the base state. Prefer a bookmark
       proven to work in the target report.
   * - ``--group NAME``
     - text
     - not set
     - The ``bookmarks.json`` group to register the new bookmarks under.
   * - ``--keep-ids``
     - flag
     - off
     - Re-render: an existing bookmark with the entry's display name is
       the base and keeps its name and its place in ``bookmarks.json``;
       only new ones are registered, at the front of the group.
   * - ``--legacy-layout``
     - flag
     - off
     - Single-page layout: the measures picker fills the matrix values
       (three or more) and the unused pickers are emptied.
   * - ``--dry-run``
     - flag
     - off
     - Build everything, write nothing.
   * - ``--self-test``
     - flag
     - off
     - Build one synthetic entry from the base bookmark in memory, check
       it and print the self-test report; nothing is written.

What bookmarks prints
~~~~~~~~~~~~~~~~~~~~~

The examples in this section run from the repository's
``tests/fixtures/bookmark-builder`` folder, which holds a report
(``Newsletter.Report``), a ``builder.toml`` with a ``[bookmarks]``
section, and an entries file (``newsletter_bookmarks.json``) of five
entries.

A build prints one line per bookmark, in entry order: the absolute path
of the file it wrote, for example

::

    pbi-plot-styler bookmarks build newsletter_bookmarks.json --report Newsletter.Report --config builder.toml

::

    <absolute path>/Newsletter.Report/definition/bookmarks/b18dcb3f08c0fb08173d.bookmark.json

for each of the five entries. The names are fresh on every build. A dry
run prints ``would write <path>`` instead, and writes nothing.

With ``--keep-ids``, each line says whether the id was kept or is new,
and the display name. Run on the same report right after the build
above:

::

    pbi-plot-styler bookmarks build newsletter_bookmarks.json --report Newsletter.Report --config builder.toml --keep-ids

::

    kept b18dcb3f08c0fb08173d Letters | 1 Overview
    kept e42d6b3493be74d9815b Letters | 2 Reach
    kept 6e3b3b6a5eda26f63b73 Letters | 3 Interest
    kept ba1527bcfdab5ec58ddb Letters | 4 Response
    kept 486f10645a1f3b6e6f91 Letters | 5 Readership

A new bookmark's line starts with ``NEW`` instead of ``kept``. Notes (a
moved matrix capture, for example) go to stderr.

``--self-test`` builds one synthetic entry from the base bookmark in
memory, checks it and prints the report, without writing anything. It
needs no entries file:

::

    pbi-plot-styler bookmarks build --report Newsletter.Report --config builder.toml --self-test

::

    self-test PASS
      generated: 9c098db8853024d5872a
      measures: ['Self Test #']
      matrix projections.Rows count: 0

The generated name is different on every run. With ``--legacy-layout``
the self-test prints ``self-test PASS`` only.

An invalid entry, a missing picker, chart or matrix capture, a filter or
a base stops the run with exit ``2`` before a file is written.

Bookmarks exit codes
~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Code
     - Meaning
   * - ``0``
     - The bookmarks were written, or nothing was to write.
   * - ``1``
     - Never returned by this command.
   * - ``2``
     - The entries, the base or the settings cannot be applied. Nothing
       was written.

rename-measure
--------------

::

    pbi-plot-styler rename-measure [OPTIONS] OLD NEW

Renames a measure in every ``visual.json`` and ``*.bookmark.json`` of
every ``*.Report`` folder under the model's parent folder, at any depth.
The model side (the measure declaration and its DAX) is not edited.

Rewritten (case-insensitively): the ``Measure`` bindings, the
``queryRef`` / ``metadata`` strings ending in the name, the
``nativeQueryRef``, and each ``[name]`` inside a saved selection
literal. The labels follow only while they still equal the old label
(``--label-from`` / ``--label-to``) and a sibling binds the measure:
cached display items, projection labels and value maps. ``Column``
bindings are never touched, and neither are the references or the label
of a column projection that shares the measure's name. A name with an
apostrophe or a closing bracket is matched and written in its escaped
literal form, and each file keeps its byte-order mark and its line
endings.

Rename-measure options
~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Option
     - Type
     - Default
     - Description
   * - ``OLD``, ``NEW``
     - text
     - required
     - The old and the new measure name.
   * - ``--model PATH``
     - directory
     - required
     - The ``*.SemanticModel`` folder; every ``*.Report`` below its
       parent folder (at any depth) is a report root.
   * - ``--label-from TEXT``
     - text
     - ``OLD``
     - The old label.
   * - ``--label-to TEXT``
     - text
     - ``NEW``
     - The new label.
   * - ``--dry-run``
     - flag
     - off
     - Count the references; write nothing.

The dry-run count equals the number of rewrites a real run makes. From
the repository's ``examples/coffee-roastery`` folder:

::

    pbi-plot-styler rename-measure Revenue "Total Revenue" --model Roastery.SemanticModel --dry-run

::

    DRY RUN: 7 reference(s) to [Revenue] in 1 report(s).

A real run prints one line per file it rewrote, with the number of
references in it, and the total:

::

    pbi-plot-styler rename-measure Revenue "Total Revenue" --model Roastery.SemanticModel

::

      Roastery.Report/definition/pages/a1f935c9ad3918c55c49/visuals/17b0cc8e254693190c00/visual.json: 3
      Roastery.Report/definition/pages/a1f935c9ad3918c55c49/visuals/5dd25eb672a72da45096/visual.json: 4
    APPLIED: 7 reference(s) renamed to [Total Revenue] in 1 report(s).

The same functions are importable from ``pbi_plot_styler.rename``:
``report_roots``, ``count_tree``, ``rewrite_tree``,
``count_measure_refs`` and ``rename_measure``.

rename-label
------------

::

    pbi-plot-styler rename-label [OPTIONS] CLIENT_DIR

Renames a field-parameter row label in the model and every report that
remembers it. It edits the selector tables of every ``*.SemanticModel``
directly under ``CLIENT_DIR`` and the visuals, pages and bookmarks of
every ``*.Report`` below it. Only the quoted label (``"OLD"``) is
replaced, never the bare text. With several pairs, the longest old
label goes first.

Rename-label options
~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Option
     - Type
     - Default
     - Description
   * - ``CLIENT_DIR``
     - directory
     - required
     - The client folder holding the model and the reports.
   * - ``--pair OLD NEW``
     - repeatable
     - required
     - The old and the new row label.
   * - ``--table-pattern GLOB``
     - repeatable
     - ``x-*``
     - The selector tables to edit.
   * - ``--apply``
     - flag
     - off
     - Write the changes.

The run prints one line per pair, the count right-aligned, then a blank
line, then ``DRY RUN`` or ``APPLIED`` with the number of files and the
files themselves, in visiting order (the model's selector tables first,
then each report's visuals, bookmarks and page files). From the
repository's ``examples/coffee-roastery`` folder, before any rename:

::

    pbi-plot-styler rename-label . --pair "Revenue" "Total Revenue"

::

      6  "Revenue" -> "Total Revenue"

    DRY RUN: 4 file(s)
        Roastery.SemanticModel/definition/tables/x-Plot Specific 1.tmdl
        Roastery.SemanticModel/definition/tables/x-Plot Specific 2.tmdl
        Roastery.Report/definition/pages/a1f935c9ad3918c55c49/visuals/17b0cc8e254693190c00/visual.json
        Roastery.Report/definition/pages/a1f935c9ad3918c55c49/visuals/5dd25eb672a72da45096/visual.json

A JSON file that would no longer parse stops the run with nothing
written: the rename is all or nothing. The paths are relative to the
client folder and always use ``/``, on Windows too.
