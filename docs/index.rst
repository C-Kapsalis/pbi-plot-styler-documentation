pbi-plot-styler
================

The hassle this replaces
-----------------------------

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

Only three things are stylable: the line's color, the data-label text
color, and the data-label background. Everything else about the visual
is left untouched.

.. toctree::
   :maxdepth: 1

   tutorials/getting-started
   reference/cli
