# Honor fork — redshift-mcp-server

This directory is maintained as a **thin Honor patch layer on top of a pinned
upstream `awslabs/mcp` release**, not as a moving fork of `main`. Every consumer
(the dev/hagent launcher and the Claude Desktop frozen binary) builds from an
immutable Honor tag with locked dependencies, so launches are reproducible and
dev and fleet run the identical engine.

This file documents the maintenance process only. It intentionally does **not**
describe individual patches: each Honor patch is a self-contained commit and
carries its own rationale in its commit message (and its own doc file if it
needs one), so patches can be dropped or reordered without editing this file.

## Naming convention

- Branch: `honor/redshift-mcp-server/<upstream-tag>`
- Tag (immutable, what consumers pin): `redshift-mcp-server/<upstream-tag>-honor.N`

Names are scoped to this server (not the repo) so other awslabs servers in the
fork can be patched and tagged independently without collision.

## Commit structure

- One commit establishes this guide (`HONOR.md`).
- Each Honor change is its own commit on top, self-documenting via its message.

Keeping patches standalone is deliberate: when upstream adopts a change, dropping
the corresponding commit is a clean operation that touches nothing else.

## Consumers pin the tag

- Dev launcher (`joinhonor/redshift-mcp`): builds a locked virtualenv from
  `git+ssh://git@github.com/joinhonor/aws-mcp.git@<tag>#subdirectory=src/redshift-mcp-server`.
- Claude `.pkg` freeze (`joinhonor/h4-claude`): builds the wheel from the same
  tag and installs with `uv sync --locked` before PyInstaller.

Both MUST use the same tag and the same `uv.lock` (parity invariant).

## Bump runbook — moving to a new upstream release

A bump reaches the fork as a pull request into `main`. Push only the feature
branch. The `honor/*` release branch and the tag are cut after the PR merges.

1. `git fetch upstream --tags` (`upstream` = `https://github.com/awslabs/mcp.git`).
2. Pick the target upstream release tag `<new>`.
3. Create a feature branch from the tag:
   `git switch -c <ticket>/redshift-mcp-server-<new> <new>`.
4. Cherry-pick each patch commit from the current release branch
   `honor/redshift-mcp-server/<old>` onto it. Resolve conflicts per patch.
5. **Re-verify each patch is still needed.** For every Honor patch commit, check
   whether upstream already incorporated it at `<new>`; drop any commit upstream
   has adopted. (Because patches are standalone, dropping one is just omitting
   that commit.)
6. Confirm `uv.lock` is present and consistent with `pyproject.toml`
   (`uv lock --check` passes). Then check that the consumers still build. The
   h4-claude PyInstaller spec bundles only the modules and data files it names.
   mcp 2 moved `mcp.server.fastmcp` to `mcp.server.mcpserver`, and from
   2026.11 the server reads its `skills/` Markdown files at import. Freeze the
   new wheel with that spec and run the stdio smoke test in `build-macos.sh`.
7. `python3 -m py_compile awslabs/redshift_mcp_server/redshift.py`, then run the
   suite with `uv run --locked pytest`.
8. Merge `origin/main` into the feature branch, so the PR merges without
   conflicts. Resolve conflicts in favour of the new base and the carried
   patches. Open a PR into `main`, and merge it with a merge commit (not squash),
   so every patch stays a standalone commit.
9. From the merge commit, create the branch `honor/redshift-mcp-server/<new>` and
   the tag `redshift-mcp-server/<new>-honor.1` (increment the suffix for re-cuts
   against the same upstream release). Push both.
10. Bump the pinned tag in `joinhonor/redshift-mcp` (`install.sh` / launcher) and
    `joinhonor/h4-claude` (freeze build).
