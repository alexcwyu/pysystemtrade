# pysystemtrade: portable verification records

Full raw logs are retained in the workspace maintenance evidence, not distributed here. Hashes identify those original logs. A successful build or selected test subset does not imply a full test-suite pass. Commands containing <workspace> describe the original execution environment.

[Machine-readable records](verification.json) · [Repository validation summary](../validation.md)

## Executed commands

| Check | Exit | Duration | Command | Outcomes |
| --- | ---: | ---: | --- | --- |
| build | 0 | unrecordeds | `uv build` | Successfully built dist/pysystemtrade-1.8.3.tar.gz<br>Successfully built dist/pysystemtrade-1.8.3-py3-none-any.whl |
| build-final | 124 | unrecordeds | `uv build` | See exit status; no test-count assertion. |
| build-verified | 0 | unrecordeds | `uv build` | Successfully built dist/pysystemtrade-1.8.3.tar.gz<br>Successfully built dist/pysystemtrade-1.8.3-py3-none-any.whl |
| python315-lock | 0 | unrecordeds | `uv lock --upgrade --python 3.15` | See exit status; no test-count assertion. |
| python315-sync | 0 | unrecordeds | `uv sync --python 3.15 --all-groups --all-extras` | See exit status; no test-count assertion. |
| tests | 1 | unrecordeds | `uv run --no-sync pytest --runslow` | ======= 23 failed, 90 passed, 30 skipped, 3 xfailed in 105.36s (0:01:45) ======= |
| tests-final | 1 | unrecordeds | `uv run --no-sync pytest --runslow` | ======= 1 failed, 112 passed, 30 skipped, 3 xfailed in 311.52s (0:05:11) ======= |
| tests-numpy-pandas | 1 | unrecordeds | `uv run --no-sync pytest --runslow` | ======= 5 failed, 108 passed, 30 skipped, 3 xfailed in 347.86s (0:05:47) ======= |
| tests-pandas-sorted | 0 | unrecordeds | `uv run --no-sync pytest --runslow` | ============ 113 passed, 30 skipped, 3 xfailed in 277.32s (0:04:37) ============ |
