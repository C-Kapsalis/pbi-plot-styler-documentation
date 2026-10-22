pbi-plot-styler
===============

These pages document ``pbi-plot-styler`` 1.2.1.

The hassle this replaces
------------------------

Before this tool, giving a chart's line a consistent color meant
opening Power BI Desktop, selecting the line and clustered column
chart, and setting the line's color and its data labels by hand in the
visual's own formatting pane, one measure at a time. Power BI's own
best practice for a scaling report is to drive that chart from field
parameters instead of hand-picking a fixed measure, so the same chart
can plot any measure a viewer selects. That is exactly what makes the
manual approach fall apart: every measure the field parameter can show
needs its own color and label setting, by hand, and every new measure
added to the parameter is one more chart to reopen and reformat.

The `BI-report-template-site
<https://github.com/C-Kapsalis/BI-report-template-site>`_ pattern this
tool assumes is a line and clustered column chart plus a matrix, both
driven by field parameters. Change which measure the line plots, and
without this tool, the new measure renders in whatever color Power BI
defaults to, its data label sitting on no background at all, or on
whatever color a previous measure left behind, sometimes unreadable
against the columns behind it.

``pbi-plot-styler`` fixes this by declaration instead of by hand: it
reads the measures a field-parameter table exposes and rewrites the
line's fill color and its data labels, text color and background,
consistently for every one of them, in one command. Run it again after
the model changes and it repairs itself: a renamed or removed measure's
stale styling is dropped, a new measure is added, and re-running a
report that is already styled is a no-op.

The styler sets only the series color (a hex color, a per-measure pin,
or a theme color), the data labels (shown or hidden, and their text
color), and the data-label background and its transparency. Everything
else about the visual is left untouched, including formatting set by
hand: the styler owns only its own per-measure entries.

The same command line also checks a report: ``pbi-plot-styler check``
reports, with stable ids, field bindings that no longer resolve in the
model, bookmarks the service would reject on import or that point at
missing pages and visuals, inconsistent field-parameter slots, and
missing theme files. It is read-only unless you add ``--fix``, which
repairs the two bookmark problems it can prove safe, and it supports a
baseline so a team can adopt it on a report that already has findings. With ``--rule``, the
same command checks the conventions of the standard canvas page, one
rule at a time: the slicer gates, the analysis category, the theme, the
visual positions, the y-combo groups, the filter panel, the format
selector, the stale placeholders, the answer bookmarks and the bookmark
literals. Several of these rules have fixers of their own.
``--summary`` scores four of them in one compliance report. Both
commands accept a single report or a folder of reports.

Four more commands keep a report in step with its model:
``pbi-plot-styler colors sync`` re-stamps the series colors from the
palette measure the model declares, ``pbi-plot-styler bookmarks build``
renders the answer bookmarks from a base bookmark that is already known
to work, ``pbi-plot-styler rename-measure`` renames a measure in every
visual and bookmark of a report, and ``pbi-plot-styler rename-label``
renames a field-parameter row label in the model and in every report
that remembers it.

New to the tool? Start with :doc:`tutorials/getting-started`, which
styles and checks the example report that ships with the tool.

.. toctree::
   :maxdepth: 1

   tutorials/getting-started
   reference/cli
   reference/convention-rules
   reference/known-issues
