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

1. `git fetch upstream --tags` (`upstream` = `https://github.com/awslabs/mcp.git`).
2. Pick the target upstream release tag `<new>`.
3. `git switch -c honor/redshift-mcp-server/<new> <new>`.
4. Carry the Honor patches forward onto the new base — e.g.
   `git rebase --onto <new> <old-tag> honor/redshift-mcp-server/<old>`, or
   cherry-pick each patch commit. Resolve conflicts per patch.
5. **Re-verify each patch is still needed.** For every Honor patch commit, check
   whether upstream already incorporated it at `<new>`; drop any commit upstream
   has adopted. (Because patches are standalone, dropping one is just omitting
   that commit.)
6. Confirm `uv.lock` is present and consistent with `pyproject.toml`
   (`uv lock --check` passes) and that the pinned `mcp` is a 1.x that still ships
   `mcp.server.fastmcp`.
7. `python3 -m py_compile awslabs/redshift_mcp_server/redshift.py`.
8. Tag: `git tag redshift-mcp-server/<new>-honor.1` (increment the suffix for
   re-cuts against the same upstream release).
9. Push branch + tag, then bump the pinned tag in `joinhonor/redshift-mcp`
   (`install.sh` / launcher) and `joinhonor/h4-claude` (freeze build).
