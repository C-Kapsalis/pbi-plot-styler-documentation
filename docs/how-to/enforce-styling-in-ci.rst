Enforce styling in CI
=====================

Add a read-only gate so a pull request that adds a measure without
re-running the styler fails loudly, instead of shipping an unstyled
chart.

Add the check
-----------------

In your pipeline, run the styler in dry-run mode against the report
folder:

::

    pbi-plot-styler "My Project.Report" --dry-run

``--dry-run`` never writes anything. It prints a unified diff per
out-of-style visual and exits with a code that tells the pipeline
whether to pass or fail:

.. list-table::
   :header-rows: 1

   * - Exit code
     - Meaning for the pipeline
   * - ``0``
     - Every targeted visual already matches the model and config. Pass.
   * - ``1``
     - At least one visual would change. Fail the build.
   * - ``2``
     - A setup problem (bad config, missing folder, unresolvable model).
       Fail the build and read the message; this is not a styling drift.

A minimal GitHub Actions step
---------------------------------

::

    - name: Check report styling
      run: pbi-plot-styler "My Project.Report" --dry-run

Point ``--config`` at a committed ``plotstyler.toml`` if your brand
colors are not the defaults, so the CI check and a local run agree.

What this catches
---------------------

Any pull request that adds or renames a measure in a field-parameter
table without running the styler afterward: the new measure would plot
unstyled, or a renamed measure would leave a dead entry behind. The
dry-run diff shows exactly which visuals and which measures are out of
step, so the fix is always "run the styler and commit the result," not
a guessing game.
