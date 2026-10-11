# Validation evidence

Recorded 2026-10-12 on macOS arm64. Selected Python: **3.15.0**; uv0.13.0; native Cargo/Rust1.99.0. Commands below ran from the repository root. Exit124 means the documented time bound expired; exit139 means a native segmentation fault. An earlier failed attempt is retained even if a later rerun succeeds.

## Current scope and result

Full existing suite, including --runslow and module doctests:113passed,30pre-existing Windows-only skips,3pre-existing xfails. Pandas3 chronological concatenation, aliases, scalar returns, and serialization metadata were repaired.

Current recorded status: **complete**. Interpreter fallback evidence and native prerequisites are described in [development](development.md). See [architecture](ayu-develop.md) for workflows, components, and existing source links.

## Executed commands

| Step | Exact command | Exit | Elapsed seconds | Evidence |
|---|---|---:|---:|---|
| python315-sync | `uv sync --python 3.15 --all-groups --all-extras` | 0 | None | [check record](maintenance/index.html) |
| python315-lock | `uv lock --upgrade --python 3.15` | 0 | None | [check record](maintenance/index.html) |
| tests | `uv run --no-sync pytest --runslow` | 1 | 106.75 | [check record](maintenance/index.html) |
| build | `uv build` | 0 | 85.71 | [check record](maintenance/index.html) |
| tests-numpy-pandas | `uv run --no-sync pytest --runslow` | 1 | 349.57 | [check record](maintenance/index.html) |
| tests-final | `uv run --no-sync pytest --runslow` | 1 | 312.64 | [check record](maintenance/index.html) |
| tests-pandas-sorted | `uv run --no-sync pytest --runslow` | 0 | 278.69 | [check record](maintenance/index.html) |
| build-final | `uv build` | 124 | 120.01 | [check record](maintenance/index.html) |
| build-verified | `uv build` | 0 | 188.83 | [check record](maintenance/index.html) |

## Test summaries and failures

### tests

Exit 1; elapsed 106.75seconds.

```text
FAILED src/syscore/pandas/find_data.py::syscore.pandas.find_data.get_row_of_series
FAILED src/syscore/pandas/find_data.py::syscore.pandas.find_data.get_row_of_series_before_date
FAILED src/syscore/pandas/frequency.py::syscore.pandas.frequency.get_intraday_pdf_at_frequency
FAILED src/syscore/pandas/merge_data_keeping_past_data.py::syscore.pandas.merge_data_keeping_past_data._calculate_change_in_daily_units
FAILED src/syscore/pandas/merge_data_with_label_column.py::syscore.pandas.merge_data_with_label_column._find_dates_when_series_starts_matching
FAILED src/syscore/pandas/pdutils.py::syscore.pandas.pdutils.get_index_of_columns_in_df_with_at_least_one_value
FAILED src/syscore/tests/test_dateutils.py::Test::test_data - DeprecationWarn...
FAILED src/systems/tests/test_mp_optimise_over_time.py::Test::test_mp_optimisation
FAILED src/systems/tests/test_mp_optimise_over_time.py::Test::test_pickling
FAILED src/sysquant/optimisation/cleaning.py::sysquant.optimisation.cleaning.clean_list_of_weights
FAILED src/sysquant/optimisation/full_handcrafting.py::sysquant.optimisation.full_handcrafting.get_avg_corr
FAILED tests/test_examples.py::TestExamples::test_simple_system_combing_rules
FAILED tests/test_examples.py::TestExamples::test_simple_system_combining_fixed
FAILED tests/test_examples.py::TestExamples::test_simple_system_position_sizing
FAILED tests/test_examples.py::TestExamples::test_simple_system_portfolio_estimated
FAILED tests/test_examples.py::TestExamples::test_simple_system_portfolio_fixed
FAILED tests/test_examples.py::TestExamples::test_simple_system_costs - Value...
FAILED tests/test_examples.py::TestExamples::test_simple_system_config_object
FAILED tests/test_examples.py::TestExamples::test_simple_system_risk_overlay
FAILED tests/test_examples.py::TestExamples::test_simple_system_config_import
FAILED tests/test_examples.py::TestExamples::test_prebaked_simple_system - Va...
FAILED tests/test_examples.py::TestExamples::test_prebaked_from_confg - Value...
FAILED tests/test_examples.py::TestExamples::test_prebaked_chapter15 - ValueE...
======= 23 failed, 90 passed, 30 skipped, 3 xfailed in 105.36s (0:01:45) =======
```

### tests-numpy-pandas

Exit 1; elapsed 349.57seconds.

```text
FAILED src/syscore/pandas/merge_data_with_label_column.py::syscore.pandas.merge_data_with_label_column._find_dates_when_label_changes
FAILED src/syscore/pandas/merge_data_with_label_column.py::syscore.pandas.merge_data_with_label_column._find_dates_when_series_starts_matching
FAILED src/syscore/pandas/merge_data_with_label_column.py::syscore.pandas.merge_data_with_label_column.merge_data_series_with_label_column
FAILED src/syscore/tests/test_dateutils.py::Test::test_data - DeprecationWarn...
FAILED tests/test_examples.py::TestExamples::test_simple_system_portfolio_estimated
======= 5 failed, 108 passed, 30 skipped, 3 xfailed in 347.86s (0:05:47) =======
```

### tests-final

Exit 1; elapsed 312.64seconds.

```text
FAILED tests/test_examples.py::TestExamples::test_simple_system_portfolio_estimated
======= 1 failed, 112 passed, 30 skipped, 3 xfailed in 311.52s (0:05:11) =======
```

### tests-pandas-sorted

Exit 0; elapsed 278.69seconds.

```text
============ 113 passed, 30 skipped, 3 xfailed in 277.32s (0:04:37) ============
```

## Synchronization

All upstream branch and tag refs were fetched by the workspace synchronization procedure. Fork updates use non-forcing pushes; upstream tags also have a private local namespace to preserve divergent tags. The default branch was merged into feature/ayu_develop, preserving the modern package layout and upstream source changes. The workspace synchronization JSON and log retain per-ref outcomes and merge history. This document does not claim that any feature branch was pushed.
