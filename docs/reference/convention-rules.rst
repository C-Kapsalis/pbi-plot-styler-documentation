Convention rules
================

``pbi-plot-styler check PATH`` runs the ``integrity`` rule by default
(bindings, bookmarks, field-parameter slots, theme files; see
:doc:`cli`). The eleven rules on this page check the conventions of the
standard canvas page, and run only when named with ``--rule SLUG``.
``--rule`` is repeatable, and the rules run in the order given:

::

    pbi-plot-styler check "My Project.Report" --rule theme --rule visual-position

A run with ``--rule`` replaces the ``integrity`` checks; name
``--rule integrity`` as well to run both.

The rules share the ``check`` machinery: stable ids, ``--json``,
``--baseline`` and ``--update-baseline``, and the exit codes ``0`` (no
new errors), ``1`` (new errors) and ``2`` (setup problem).
``--update-baseline`` with ``--rule``, ``--report`` or ``--model-name``
rewrites only that scope's entries. ``--fix --dry-run`` prints the diff
of every planned fix and writes nothing. With several reports, ``--fix``
checks every report's prerequisites before writing any.

A rule that cannot judge a report says so instead of passing silently.
In the text output this is a ``skipped`` line, for example:

::

    skipped  Roastery.Report: filter-panel: the report has no group containers

In ``--json`` the same text is an entry of ``skipped``. A model that is
referenced but cannot be loaded is an ``MDL001`` error, not a skip.

The examples on this page run on the reports in the tool repository's
``tests/fixtures`` folder. Several finding lines end with the
visual's absolute path; this page shows that part as
``<absolute path>``.

Vocabulary and settings
-----------------------

Every name a rule depends on is a setting with a built-in default. Put
the sections you need in ``plotstyler.toml`` (read from the current
folder, or from ``--config FILE``); the sections for ``style`` and
``bookmarks build`` can live in the same file. These are the built-in
defaults, except ``[answer_bookmarks] prefixes``, which has no default
and is shown with example values:

::

    [canvas]
    matrix_types = ["pivotTable", "matrix"]
    combo_types = ["lineStackedColumnComboChart"]
    excluded_page_fragments = ["cohort", "ltv"]
    selector_tables = ["x-*", "y-*"]          # exact names or globs
    measure_selector_tables = ["x-*"]
    dimension_selector_tables = ["y-*"]
    exclude_reports = ["M.Report", "-M.Report"]

    [gates]
    table = "f-Filters"                       # used in messages only
    slicer_measure = "f-Slicer"
    list_measure = "f-All"
    mode_map = { Dropdown = "slicer_measure", Basic = "list_measure" }
    name_heuristic = false

    [category]
    table = "z-Map Dimension-Measure Groupings"
    column = "Analysis Category"
    chrome_words = ["info", "overview", "menu", "edit", "year", "quarter", "month", "week", "day"]
    chrome_prefixes = ["<", "++"]

    [placeholders]
    blank_dimension_table = "#Report Measures"
    template_slot_table = "Helper Measures"

    [panel]
    group_name = "y_filters_group"
    filter_tables = ["y-Selector 3", "y-Selector 4", "y-Selector 5", "y-Selector 6"]
    min_slicers = 6                           # main-page thresholds
    min_combos = 1
    min_matrices = 1

    [format_selector]
    table = "z-Selector Format"
    column = "Selector Format"
    item = "Format"
    filter_name = "format_selector_filter"

    [answer_bookmarks]
    prefixes = ["Sales |", "Marketing |"]     # required for answer-bookmarks
    submenu_prefix = "Submenu"
    slot_mode = "all-roles"                   # or "values-only"
    inherited_fingerprints = []

``[answer_bookmarks]`` also takes optional keys with no default:
``main_page`` (a page folder name), ``matrix`` (a visual id),
``slicers`` (a table of field-parameter name to slicer visual id),
``row_dropdowns`` (a list of visual ids) and ``baseline`` (a file,
relative to the config file, used when no ``--baseline`` is given).
When they are not set, the page, the matrix and the slicers are
discovered.

Terms used below:

- **Canvas page**: a page with a visual of a ``matrix_types`` type and
  one of a ``combo_types`` type, whose display name contains none of the
  excluded fragments.
- **Main page**: the page with at least ``min_slicers`` slicers, a combo
  chart and a ``pivotTable`` (most slicers wins, then the larger folder
  name).
- **Selector slicer**: a slicer whose ``Values`` projection is a column
  of a selector table.
- **Semantic name**: a group's ``visualGroup.displayName``, any other
  visual's ``name``.

Report discovery: a ``*.Report`` folder is checked as is; any other
folder is searched at every depth. ``--report NAME`` keeps one report by
folder name (with or without ``.Report``) and overrides
``exclude_reports``; ``--model-name NAME`` keeps the reports whose
``definition.pbir`` points at ``NAME.SemanticModel``. When nothing
matches, the run exits ``2`` and names the filter.

slicer-gates
------------

Every selector slicer stacks with others on the canvas; a visual-level
filter on ``[f-Slicer] > 0`` hides the ones that should not show. A gate
is *active* when one of its conditions compares the measure greater than
0 (kind 1 with ``0L``) or at least 1 (kind 2 with ``1L``).

- ``GAT001``: a selector slicer on a visible canvas page has no active
  gate.
- ``GAT002``: a bookmark capture of a selector slicer on a canvas page
  has no active gate under ``filters.byName``, so applying the bookmark
  removes it.

A slicer or capture is compliant when any filter bound to the gate
measure is active. Selector-table patterns (``[canvas]
selector_tables``) match names ignoring case. With ``[gates]
name_heuristic = true`` the check also treats a slicer whose name
contains both ``slicer`` and ``selector`` as a selector slicer; this
only adds findings and never makes ``--fix`` gate such a slicer.

From ``tests/fixtures/gate-order``:

::

    pbi-plot-styler check ./-Canvas.Report --rule slicer-gates

::

    1/1 visual(s) missing the active gate:
    ERROR    GAT001  -Canvas.Report/definition/pages/7a7b7c7d7e7f70717273/visuals/dropdown01/visual.json: dropdown01  (<absolute path>)

    1 bookmark visualContainer(s) lack the active gate (would erase it on apply):
    ERROR    GAT002  -Canvas.Report/definition/bookmarks/f2f2f2f2f2f2f2f2f2f2.bookmark.json: f2f2f2f2f2f2f2f2f2f2.bookmark.json -> page 7a7b7c7d7e7f70717273 vc dropdown01

    Checked 1 report(s): 2 error(s), 0 warning(s).

A report with no selector slicer prints ``No Report field-parameter
slicers found under <absolute path>`` and passes.

``--fix`` binds the gate to the table that declares ``f-Slicer`` in the
paired model (never to whatever a stale binding names). It repairs the
first bound filter (keeping its name; a bookmark gate's name is its
``byName`` key) only when none is active, or appends one. The filter
alias is always the letter of ``[gates] table`` (``f``), whatever table
declares the measure. A report whose model declares no such table is
left unchanged with a ``REFUSED`` line; the other reports are still
fixed, and the exit code comes from the re-check.

::

    pbi-plot-styler check ./-Canvas.Report --rule slicer-gates --fix

::

      -Canvas.Report: source entity = 'f-Filters'  (declared in Growth.SemanticModel/definition/tables/f-Filters.tmdl)
        patched visual dropdown01
        patched bookmark f2f2f2f2f2f2f2f2f2f2.bookmark.json

    Patched 1/1 visual(s) and 1 bookmark file(s).

    Checked 1 report(s): 0 error(s), 0 warning(s).

The ``-Canvas.Report`` folder name starts with a dash, so the commands
pass it as ``./-Canvas.Report``.

slicer-mode-gates
-----------------

A drop-down slicer gates on ``f-Slicer``, a vertical-list slicer on
``f-All`` (``mode_map``); other modes are out of scope. The host table
is the first table of a sibling ``*.SemanticModel`` that declares both
measures; without one, the run stops and exits ``2``:

::

    !! could not find a table declaring both [f-Slicer] and [f-All] for Roastery.Report

- ``GAT003``: the gate for the slicer's mode is absent or passive
  (visual or bookmark capture).
- ``GAT004``: the gate is active but bound to a table that does not
  declare it.
- ``GAT005``: a file holds gate references (a gate ``Measure``, or the
  ``From`` of a filter that mentions a gate) that name another table.

From ``tests/fixtures/mode-gates``:

::

    pbi-plot-styler check ./-Canvas.Report --rule slicer-mode-gates

::

      gate source entity (from TMDL): 'f-Filters'
    ERROR    GAT004  -Canvas.Report/definition/pages/6c6d6e6f606162636465/visuals/dropdown02/visual.json: wrong_entity Dropdown  -> f-Slicer  6c6d6e6f606162636465/dropdown02
    ERROR    GAT003  -Canvas.Report/definition/pages/6c6d6e6f606162636465/visuals/list01/visual.json: absent       Basic     -> f-All     6c6d6e6f606162636465/list01
    ERROR    GAT005  -Canvas.Report/definition/pages/6c6d6e6f606162636465/visuals/dropdown02/visual.json: 2 gate reference(s) name a table other than 'f-Filters'
    ERROR    GAT005  -Canvas.Report/definition/bookmarks/e0e0e0e0e0e0e0e0e0e0.bookmark.json: 2 gate reference(s) name a table other than 'f-Filters'
      Would patch 2 visual(s) and 0 bookmark entrie(s)
      Would patch 4 stale entity reference(s) -> 'f-Filters' across 2 file(s)

    Checked 1 report(s): 4 error(s), 0 warning(s).

``--fix`` writes the gates and points stale references at the host table. In a stale
filter's ``From`` list, only the alias a gate reference uses, or an
entry naming the stale table, is changed; other sources are left
alone. Both gate fixers write the same shapes and name a new bookmark
gate the same way, so running them in either order gives identical
files. Generated filter names are a stable hash, so reruns agree. In
1.2.1 the closing ``Patched`` counts of this fixer can read lower than
what it wrote; preview with ``--fix --dry-run``, which lists every file
(see :doc:`known-issues`).

category-filter
---------------

``CAT001``: a bookmark listed in a ``bookmarks.json`` group never
filters ``[category.table][category.column]``, so applying it switches
every gate off. Menu chrome is skipped: labels such as ``Year``, labels
that start with ``<`` or ``++``, and decorated labels. When no group
lists any bookmark, every page-level bookmark is checked.

From ``tests/fixtures/gate-order``:

::

    pbi-plot-styler check ./-Canvas.Report --rule category-filter

::

    ERROR    CAT001  -Canvas.Report/definition/bookmarks/f2f2f2f2f2f2f2f2f2f2.bookmark.json: Page-level bookmark has no Analysis Category filter (Expected: a filter in byName or byExpr targeting z-Map Dimension-Measure Groupings[Analysis Category], Got: no such filter found)
    ✅ Rule #3 Validation Results:
       Components checked: 1
       Components passed: 0
       Compliant: False

       Violations found:
       🔴 Backend | View: Page-level bookmark has no Analysis Category filter
    Rule #3 critical violations found:
      - Backend | View: Page-level bookmark has no Analysis Category filter (Expected: a filter in byName or byExpr targeting z-Map Dimension-Measure Groupings[Analysis Category], Got: no such filter found)

    Checked 1 report(s): 1 error(s), 0 warning(s).

The block that starts ``✅ Rule #3 Validation Results`` is the rule's
own summary; it starts with the same mark whether or not the rule
passed, so read ``Compliant``. There is no fixer: set the filter in
Power BI Desktop and update the bookmark.

theme
-----

- ``THM002`` (error): no ``themeCollection``.
- ``THM003`` (warning): no ``customTheme``.
- ``THM004`` (error): ``report.json`` holds quoted hex color literals;
  one finding with the count and up to 8 samples.
- ``THM005`` (error): neither a base nor a custom theme.
- ``THM006`` (error): the base or the custom theme is missing (the
  message names which).

From ``tests/fixtures/shop-combo``, a report with no
``themeCollection`` at all:

::

    pbi-plot-styler check Shop.Report --rule theme

::

    ERROR    THM002  Shop.Report/definition/report.json: Report missing themeCollection configuration (Expected: themeCollection with baseTheme and customTheme, Got: null or missing)
    WARNING  THM003  Shop.Report/definition/report.json: No custom theme defined - may use hardcoded colors (Expected: customTheme configuration, Got: null or missing)
    ERROR    THM005  Shop.Report/definition/report.json: Report has no theme configuration. Without a base or custom theme, ThemeDataColor references cannot resolve and visuals fall back to Power BI defaults.
    ERROR    THM006  Shop.Report/definition/report.json: Theme collection is incomplete - missing: baseTheme, customTheme. Reports must declare both baseTheme and customTheme.
    ✅ Rule #6 Validation Results:
       Components checked: 1
       Components passed: 0
       Compliant: False

       Violations found:
       🔴 report: Report missing themeCollection configuration
       ⚠️ report: No custom theme defined - may use hardcoded colors
       🔴 report: Report has no theme configuration. Without a base or custom theme, ThemeDataColor references cannot resolve and visuals fall back to Power BI defaults.
       🔴 report: Theme collection is incomplete - missing: baseTheme, customTheme. Reports must declare both baseTheme and customTheme.
    Rule #6 critical violations found:
      - themeCollection: Report missing themeCollection configuration (Expected: themeCollection with baseTheme and customTheme, Got: null or missing)
      - themeCollection: Report has no theme configuration. Without a base or custom theme, ThemeDataColor references cannot resolve and visuals fall back to Power BI defaults.
      - baseTheme+customTheme: Theme collection is incomplete - missing: baseTheme, customTheme. Reports must declare both baseTheme and customTheme.

    Checked 1 report(s): 3 error(s), 1 warning(s).

A report that declares only a base theme gets ``THM003`` and ``THM006``
(``missing: customTheme``); see the ``Brewbar.Report`` example in
:doc:`cli`, section *Compliance summary*. In 1.2.1 the ``Violations
found`` line of ``THM006`` always says ``missing: baseTheme,
customTheme``; the ``ERROR THM006`` line names what is really missing
(see :doc:`known-issues`).

A ``THM004`` finding reads, for one literal:

::

    ERROR    THM004  Brewbar.Report/definition/report.json: 1 hard-coded hex color literal(s) in report.json (sample: "#00FF00") (Expected: ThemeDataColor references only, Got: 1 '#RGBHEX' literals)

``--summary`` scores and lists only ``THM002``, ``THM003`` and
``THM004`` in the theme row; ``THM005`` and ``THM006`` run under
``--rule theme``. The rule has no fixer.

visual-position
---------------

``POS001`` (error): a visual has no ``position.x`` or ``position.y``.
For a visual ``table20`` without ``x``, the run prints the finding and
the rule's aggregate line:

::

    ERROR    POS001  Brewbar.Report/definition/pages/shops/visuals/table20/visual.json: 1 visual container(s) missing x or y position: table20
    1 visual container(s) missing x or y position: table20

The rule has no fixer.

y-combo
-------

- ``YCG001``: a group named like a y-combo group does not match
  ``y<N>_combo_group`` exactly.
- ``YCG002``: a ``y<N>_*`` member (at least one character after
  ``y<N>_``) is not inside an existing ``y<N>_combo_group``.

A report without y-combo groups is valid and is skipped:

::

    skipped  Roastery.Report: y-combo naming: No y-combo groups discovered in this report - skip rather than fail (reports without y-combos are valid).
    skipped  Roastery.Report: y-combo hierarchy: No y_combo_group containers in this report

The rule has no fixer.

filter-panel
------------

Skipped when the report has no group containers.

- ``PNL001``: no visual is named ``panel.group_name`` (the message reads
  ``No y_filters_group master container found``).
- ``PNL002`` (warning): it exists but is not a group (the message reads
  ``y_filters_group should be a group container, not specific visual``).

From ``tests/fixtures/bookmark-builder``:

::

    pbi-plot-styler check Newsletter.Report --rule filter-panel

::

    note: Newsletter.Report: binding checks skipped, no local semantic model (Newsletter.SemanticModel is not a folder next to the report)
    ERROR    PNL001  Newsletter.Report/definition/pages: No y_filters_group master container found (Expected: y_filters_group group container (visualGroup.displayName == 'y_filters_group'), Got: not found)
    ✅ Rule #8 Validation Results:
       Components checked: 1
       Components passed: 0
       Compliant: False
    Rule #8 critical violations (--fix not given, no auto-repair):
      - y_filters_group: No y_filters_group master container found (Expected: y_filters_group group container (visualGroup.displayName == 'y_filters_group'), Got: not found)
    Manual: pbi-plot-styler check <absolute path>/Newsletter.Report --rule filter-panel --fix

    Checked 1 report(s): 1 error(s), 0 warning(s).

``--fix`` acts only when the check reports ``PNL001``: a report that
already has a visual named ``panel.group_name`` (whatever its folder is
called) is left as it is, with a ``skipped`` note. It works on the main
page: the master is the visible group object (display name not
starting with ``_``) that holds slicers of at least two filter tables,
most tables first, then the deepest, then the smallest name; a visual
that is not a group is never chosen. It is renamed, child
``parentGroupName`` links and bookmark references follow, and the folder
is renamed last. Visual files it rewrites keep non-ASCII text as ``\u``
escapes. An existing folder with the target name stops the fix
(exit ``2``). When no group qualifies, the fix writes nothing:

::

    pbi-plot-styler check Newsletter.Report --rule filter-panel --fix --dry-run

::

    note: Newsletter.Report: binding checks skipped, no local semantic model (Newsletter.SemanticModel is not a folder next to the report)
    skipped  Newsletter.Report: filter-panel fix: no group covers two or more filter tables
    ERROR    PNL001  Newsletter.Report/definition/pages: No y_filters_group master container found (Expected: y_filters_group group container (visualGroup.displayName == 'y_filters_group'), Got: not found)
    ✅ Rule #8 Validation Results:
       Components checked: 1
       Components passed: 0
       Compliant: False

    Nothing to do (no eligible master groups found, or all already named).

    Checked 1 report(s): 1 error(s), 0 warning(s).

format-selector
---------------

Active only when the paired model has the ``format_selector.table``;
otherwise the report is skipped:

::

    skipped  Roastery.Report: format-selector: the paired model has no 'z-Selector Format' table

``FMT001``: ``report.json`` does not hold exactly one filter on the
format-selector column that selects ``'Format'`` and is hidden and
locked in view mode. The message names the first failing condition.
The selected item is the first non-empty literal across the filter's
``In`` rows. A report without ``definition/report.json`` is skipped as
not applicable; an unreadable one is a finding.

From ``tests/fixtures/universal``:

::

    pbi-plot-styler check M.Report --rule format-selector

::

    1/1 report(s) missing or misconfigured filter:
    ERROR    FMT001  M.Report/definition/report.json: <absolute path>/M.Report/definition/report.json: no report-level filter on z-Selector Format[Selector Format]

    Checked 1 report(s): 1 error(s), 0 warning(s).

``--fix`` replaces the targeting filters with one canonical filter at the
front, keeping the first filter's name:

::

    pbi-plot-styler check M.Report --rule format-selector --fix

::

      <absolute path>/M.Report/definition/report.json: patched (no report-level filter on z-Selector Format[Selector Format])
    fixed    M.Report/definition/report.json: patched (no report-level filter on z-Selector Format[Selector Format])

    Patched 1/1 report(s).

    Checked 1 report(s): 0 error(s), 0 warning(s).

``M.Report`` is in the default ``exclude_reports`` list, so a sweep of
the ``universal`` folder skips it; naming the report folder directly
checks it.

stale-placeholders
------------------

Each dimension selector offers one blank-dimension placeholder column
(``NAMEOF('#Report Measures'[z])``). ``PLH001``: a visual role or
bookmark capture bound to ``y-*`` selectors projects a placeholder those
selectors do not offer. Bookmark captures also have their ``byExpr``
filters and ``orderBy`` checked, and every JSON file of the bookmarks
folder that holds exploration state is read, except
``bookmarks.json``. Roles whose selectors offer several placeholders
are skipped as ambiguous.

A clean report prints:

::

    No stale placeholder bindings under Roastery.Report.

``--fix`` sets the offered placeholder and updates ``queryRef`` /
``nativeQueryRef``.

answer-bookmarks
----------------

Answer bookmarks are the bookmarks whose display name starts with one of
``[answer_bookmarks] prefixes``. The setting is required; without it the
run exits ``2``:

::

    Error: the answer-bookmarks rule needs [answer_bookmarks] prefixes in the config file (the display-name prefixes of the answer bookmarks)

Finding subjects start with the bookmark's display name, so a baseline
entry reads ``<file>::<display name>``.

.. list-table::
   :header-rows: 1

   * - Id
     - Finding
   * - ``ABK000``
     - No bookmark matches the prefixes (the suite would check nothing).
   * - ``ABK001``
     - ``options.targetVisualNames`` is missing or not exactly ``[]``.
   * - ``ABK003``
     - The matrix is not captured, or captured as hidden.
   * - ``ABK004``
     - Matrix rows differ from the row picker's picks, or the capture
       carries an inherited fingerprint.
   * - ``ABK005``
     - The capture keeps ``Values`` in ``activeProjections``, or
       expansion states.
   * - ``ABK006``
     - A captured picker selects nothing.
   * - ``ABK007``
     - The analysis category is not pinned with a literal.
   * - ``ABK008``
     - The bookmark is not a child of a ``Submenu`` group.
   * - ``ABK009``
     - The matrix slot map does not match the declared parameters or the
       picks.
   * - ``ABK010``
     - A row field is missing from the dropdown filters.
   * - ``PGW001``
     - A field-parameter well on the main page projects a raw measure.
   * - ``PGW002``
     - A bookmark navigator on the main page selects a bookmark with no
       file.

Selections are read only from each slicer's saved filter, never from its
cached display items. Page and visual ids are discovered when not
configured; a main page or matrix that cannot be determined is a setup
error (exit ``2``), not a pass. So is a row parameter whose slicer
cannot be determined: the error names the candidates and the
``[answer_bookmarks] slicers`` key to set. A matrix with no row
parameter is noted under ``skipped`` (``ABK004``'s row comparison and
``ABK010`` are not run). Overlapping parameter slots on any page are the
``integrity`` rule's ``FPM001`` / ``FPM002``. A bookmark file or
main-page visual that does not parse is a ``JSN001`` error.

``ABK000`` can never be accepted by a baseline: ``--update-baseline``
does not write it, drops an entry for it, and exits ``1`` while it
occurs. With prefixes that match nothing, the run prints:

::

    bookmarks tested: 0 | failures: 1
    ERROR    ABK000  Newsletter.Report/definition/bookmarks: no whole-page category bookmarks matched [answer_bookmarks] prefixes, so every check passed on an empty set. Either the naming convention changed or the report moved. Prefixes currently expected: ['Letters |']

The text output starts with ``bookmarks tested: <n> | failures: <m>``
(and, with no new finding, ``all visual-level checks pass (ABK001,
ABK003-ABK010, PGW001-PGW002)``); ``--json`` has the count under
``checked``. Each run of findings of one id is headed ``<ID> — NEW
finding(s), not in the baseline:`` and followed by a blank line and that
id's guidance. After the five bookmarks of the ``bookmarks build``
example in :doc:`cli` are built without ``--group``, a config with
``prefixes = ["Letters |"]`` gives, in part:

::

    bookmarks tested: 5 | failures: 6
    ABK008 — NEW finding(s), not in the baseline:
    ERROR    ABK008  Newsletter.Report/definition/bookmarks/486f10645a1f3b6e6f91.bookmark.json: Letters | 5 Readership: not registered under any Submenu group — unreachable in the UI

    A bookmark not registered under its category's Submenu group exists in
    the file tree and is unreachable in the report — no menu entry renders.
    Add its `name` to that group's children in bookmarks.json.

Fix page-level findings (``PGW001``, ``PGW002``) first: they reproduce
in every bookmark, so capture-level findings are only meaningful once
the page is right. The rule has no fixer; most fixes are to rebuild the
bookmark with ``bookmarks build``.

- ``ABK000``: nothing matched, so every other check passed vacuously.
  Fix the ``[answer_bookmarks] prefixes`` setting (or the bookmark
  names).
- ``ABK001``: a bookmark with a list in ``targetVisualNames`` applies
  only to those visuals, so its picker selections and filters are
  ignored; a missing key means the menu entry cannot be clicked. Re-capture
  the bookmark with no visual selected, or set the key to ``[]`` in the
  bookmark file.
- ``ABK003``: the matrix is hidden on the page and shown only by each
  answer bookmark's capture. Re-capture with the matrix visible.
- ``ABK004``: the matrix rows saved in the capture are what Power BI
  shows; rows that differ from the row picker, or a field inherited from
  the bookmark it was cloned from, show another question's answer.
  Rebuild the bookmark with ``bookmarks build``.
- ``ABK005``: active ``Values`` projections or expansion states pin the
  layout of the bookmark it was copied from. Rebuild it.
- ``ABK006``: a picker captured with no selection leaves its visual
  empty. Every main-page selector slicer, row dropdown and configured
  slicer that a bookmark captures must select something. Pick a field
  and re-capture.
- ``ABK007``: without a pinned analysis category the pickers offer every
  field. Set the category filter and re-capture.
- ``ABK008``: only children of a ``Submenu`` group get a menu entry. Add
  the bookmark to a submenu group in ``bookmarks.json``.
- ``ABK009``: each parameter owns a slot range of the well; a pick
  outside its range renders nothing. The message names the picks outside
  their slots. Rebuild the bookmark.
- ``ABK010``: each row field needs a dropdown filter so readers can
  narrow it. The message says what the dropdowns hold (or that they hold
  nothing). Mirror the rows into the dropdowns, or rebuild the bookmark.
- ``PGW001``: a raw measure in a parameter well renders in every
  bookmark and no bookmark can remove it. Remove it from the page's
  ``visual.json``.
- ``PGW002``: the navigator's selected bookmark (a quoted
  20-character lower-case hex id) has no bookmark file on disk. Point it
  at an existing bookmark.

bookmark-literals
-----------------

Each selector table registers the rows its ``NAMEOF`` tuples name (a
bare ``[Measure]`` resolves to the table declaring it). For each
bookmark (or only those named in ``--only-display-names FILE``), every
selection literal of the active section is checked:

- ``BML001``: a picker's literal is registered in another selector, not
  in the one behind this picker.
- ``BML002``: the literal is registered in no selector.

``--only-display-names`` takes a JSON list of names, or the ``bookmarks
build`` entries file (a list of objects with a ``displayName``). The
text output starts with ``bookmarks checked: <n> | literals OK: <ok> |
UNREGISTERED: <bad>``. There is one finding per bookmark, literal and
reason, sorted, with the reason on the next line. From
``tests/fixtures/label-rename/client``:

::

    pbi-plot-styler check Growth.Report --rule bookmark-literals

::

    bookmarks checked: 2 | literals OK: 0 | UNREGISTERED: 2
    ERROR    BML002  Growth.Report/definition/bookmarks/b1b1b1b1b1b1b1b1b1b1.bookmark.json: Growth | One  ->  'x-Selector 1'[Cost/ New Customer]
                    not registered in ANY field parameter
    ERROR    BML002  Growth.Report/definition/bookmarks/b2b2b2b2b2b2b2b2b2b2.bookmark.json: Growth | Two  ->  'x-Selector 1'[Cost/ New Customer]
                    not registered in ANY field parameter

    Checked 1 report(s): 2 error(s), 0 warning(s).

Without a usable paired model the rule cannot judge a report. When no
report of the run was judged, it is a setup error (exit ``2``) naming
``--model``:

::

    Error: bookmark-literals judged no report: it needs the paired semantic model (definition.pbir next to a local *.SemanticModel); pass --model <*.SemanticModel folder>

Otherwise the report is listed under ``skipped``. An unreadable
bookmark file is a ``JSN001`` error. The rule has no fixer.

--summary
---------

``check --summary`` runs ``slicer-gates``, ``category-filter``,
``theme`` and ``filter-panel`` and reports, per rule, how many
components were checked and passed and whether it is compliant (no
error finding at all), plus the overall percentage. It never fails;
``--json`` adds a ``compliance`` object and a ``fix`` field on each
finding. The finding lines, the closing ``Checked`` totals and the
``--json`` ``findings`` and ``summary`` carry only the ids the rows
count, so they agree with the score. Each finding in the report is
followed by a one-line remediation hint, printed as ``- **Fix**:``. A
worked example is in :doc:`cli`, section *Compliance summary*.

Baselines
---------

``--baseline FILE`` accepts the findings it lists; only others fail the
run. Accepted entries of this run's rules and reports that no longer
occur are listed once per id:

::

    <ID>: <n> accepted finding(s) no longer occur — re-capture the baseline (pbi-plot-styler check --update-baseline) to lock it in: [<locations>]

Entries of other rules or reports are never called healed.
