# Changelog

## 1.0.1 (2026-10-05)

Fixes from review. Behaviour changes are marked.

- `check` scans every git-tracked file, not an allowlist of extensions: `.sql`, `.svg`, `.vue`, `.c`, `.xml`,
  `.php` and the rest were missed. Binary files (a NUL byte) and symlinks are skipped and listed; a file that
  is not UTF-8 is still scanned. `check` exits 2, and `scan_extract` raises, when the extract is not a git
  repository or has no text file, instead of passing after scanning nothing. New `scan_extract_report`.
- `--ungated` only waives the gate requirement for a repository that has no gate commands. Before, it also
  switched off configured gates. A commit landed without gates says so in its message.
- `baselines` no longer moves past an upstream commit that only added code, which was then never carried:
  equal matches now go to the oldest commit (then to the one most alike line for line), and the match is
  compared unrounded.
- A file that is not UTF-8 text on the upstream side is held (`held: not UTF-8 text`), no longer merged with
  replacement characters. The same for the extract text a downstream starts from.
- Behaviour change: `accept` records the commit the hold was planned against, which `apply` and
  `plan --held-dir` now remember (`held_upstream`, `held_downstream` in the config), not the ref as it stands
  when you run `accept`. An upstream change made while you worked on a hold is carried afterwards. With no
  recorded hold, `accept` refuses unless you pass the new `--to SHA`.
- Behaviour change: extract text committed after the last release tag still moves downstream, now labelled
  `take extract v1.0.0+abc1234` rather than with the old tag alone. The README says "committed", not "released".
- `examples/make_world.py` runs the installed `cleanfork` command when there is one.
- README: `baselines` takes each downstream to hold the extract's current commit without checking, relative
  imports are not seen as dependencies, and a brand phrase wrapped over two lines is not caught.

## 1.0.0 (2026-10-04)

First release.

- `init`, `baselines`, `plan`, `apply`, `accept` and `check` commands, and the same functions as a library.
- Per-module three-way merge of upstream changes into a clean extract (`git merge-file`), with product
  prefixes removed from the base and the incoming text.
- Merges are held, never forced, on a conflict, a forbidden word, a new import of private code, an
  upstream deletion, an unknown baseline, uncommitted edits or a file that is not UTF-8 text.
- Gate commands per repository; a failed gate restores the files, and a failed batch still ships the
  modules that pass alone.
- Release of the extract as the next minor version tag, then the merge of each released change into any
  number of downstream copies, each with its own gates; `diverged` downstream copies are skipped.
- Optional `--push` to each repository's configured remote and branch.
- Provenance report: how much of the extract's code each upstream and downstream copy still holds.
- Brand scan of the extract's tracked files with branding-leak-guard's rule format and its joined-string-literal pass.
- `examples/make_world.py`: a throwaway three-repository world and its real plan.
