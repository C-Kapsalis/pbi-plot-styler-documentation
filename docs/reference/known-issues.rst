Known issues in 1.2.1
=====================

Each issue below was reproduced with ``pbi-plot-styler`` 1.2.1. Each
entry says what you see, what is actually true, and how to work around
it.

A dry run names a column measure it will style
----------------------------------------------

When no query role is excluded (no ``--exclude-role`` and no ``[targets]
exclude_current_roles``), a ``style --dry-run`` block still prints
``columns-role measure(s) kept OUT of the line set: <measure>`` for the
measure bound to the columns. The write run styles that measure anyway,
and its result line says ``0 columns-role measure(s) excluded``.

To see what will happen, read the ``objects.dataPoint[]`` count of the
dry run, or set ``exclude_current_roles = ["Y"]`` when the columns must
keep their own styling (see :doc:`../tutorials/getting-started`, step 3).

A relative page or visual path styles nothing
---------------------------------------------

``style`` accepts a page folder or a ``visual.json`` to restyle only
part of a report, but in 1.2.1 only an absolute path works. A relative
path ends with ``Error: no targeted visuals found ...`` and exit ``2``.
Pass the absolute path, as shown in :doc:`cli`, section *Styling one
page*.

A report whose model cannot be loaded passes the style gate
-----------------------------------------------------------

When ``style`` cannot resolve or load a report's semantic model (no
``definition.pbir``, a model folder that is missing, or a model not
saved as TMDL), it prints ``SKIPPED - no paired SemanticModel`` for each
combo chart and exits ``0``, even for a single report and even with
``--model``. A dry run then ends with ``All <n> combo chart(s) already
in sync (<n> skipped).``

Run ``pbi-plot-styler check`` next to the dry run in CI: it reports a
model that is referenced but cannot be loaded as ``MDL001`` and exits
``1``. A model folder that does not exist at all is only a ``note`` in
``check``, so also read the ``Models resolved`` line of the dry run.

The THM006 summary line names both themes
-----------------------------------------

Under ``--rule theme``, the ``Violations found`` block prints the
``THM006`` line as ``Theme collection is incomplete - missing:
baseTheme, customTheme`` even when only one theme is missing. The
``ERROR THM006`` finding line, the ``critical violations`` block and
``--json`` name the theme that is actually missing.

The slicer-mode-gates fixer reports low counts
----------------------------------------------

``check --rule slicer-mode-gates --fix`` can close with counts lower
than what it wrote. On the repository's ``tests/fixtures/mode-gates``
report it prints ``Patched 2 visual(s) and 0 bookmark entrie(s) across
0 file(s)`` and ``Patched 0 stale entity reference(s) -> 'f-Filters'
across 0 file(s)``, while it rewrites two visuals and one bookmark and
the re-check finds nothing left. Run ``--fix --dry-run`` first: its
``would fix`` lines and its diff list every file.

Some help texts describe older behavior
---------------------------------------

``pbi-plot-styler style --help`` says that ``--dry-run`` prints a
unified diff; it prints a block per visual (see :doc:`cli`, section
*What the run prints*). It also says that ``--table`` overrides a
default ``x-Plot Specific 1`` / ``x-Plot Specific 2`` / ``y-Plot
Specific`` trio, and that ``--rename-table`` renames a default table;
the default is the set of field parameters detected in the model (see
:doc:`cli`, section *Field parameter detection*).
