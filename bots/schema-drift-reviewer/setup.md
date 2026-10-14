Prepare this machine so database schemas can be inspected and pull requests drafted.

- `python3` (or `python`) is installed and version is 3.9 or newer - confirm with `python --version` or `python3 --version`.
- `schemadrift` is installed on PATH - confirm with `schemadrift --version`. If it is not installed, install it with `pip install "pg-schema-diff>=0.2.4"`.
- `git` is installed and on PATH - confirm with `git --version`.
- The GitHub CLI `gh` is installed and authenticated - confirm with `gh auth status`.

Do not run migrations or edits during setup. Preparing the toolchain is the whole job.
