# cleanfork — Keep a clean copy of private code in sync

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/cleanfork**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

cleanfork three-way merges each upstream change into a clean extract of a private codebase, holds the ones that conflict, bring a forbidden word back or need a private module, gates on the extract's own tests, releases it, then carries it to downstream copies.

## Get it

Buy cleanfork at https://dominares.org/tools/cleanfork: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python package and the cleanfork command, a script that builds a throwaway example world, and the full test suite.

Requirements: Python 3.10 or newer and git on your PATH. No other dependencies.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### The problem

You keep a clean, shareable extract of an internal codebase: its prefixes removed, its brand words gone,
its own tests. The internal code keeps moving. Copying files over by hand loses the extract's local fixes,
lets a forbidden word slip back in, and ships changes that import code the extract does not have. A
plain `git merge` does not help: the two sides are not related by history, because the extract was made by
hand.

cleanfork treats each file as a module with three copies and does the merge for you, per file:

- **upstream**: the private source, read at a ref (default `origin/main`) of a local checkout.
- **extract**: the clean copy, a git repository of its own.
- **downstreams**: any number of other apps that hold a copy of the extract.

### What it does

`cleanfork apply` runs two stages and records where it got to.

1. **Extract stage.** For every module, the upstream file's changes since the commit recorded for it are
   three-way merged into the extract (`git merge-file`). The base and the incoming text have the product
   prefixes removed first (`acme_core` becomes `core`), so the merge sees what the extract sees. A merge is
   **held**, not forced, when it:
   - conflicts with a change made in the extract (`held: conflict`);
   - brings a forbidden word back (`held: brand`);
   - adds an import of private code, so it needs a person to port that code first (`held: new dependency`);
   - finds the upstream file deleted (`held: removed upstream`).

   Everything ready is written, the extract's gate commands run, and if they pass the files are committed and
   released as the next minor version tag (`v1.1.0`, then `v1.2.0`). The first release of an untagged extract is `v1.0.0`.
2. **Downstream stage.** Only committed extract text moves on, never an edit still in the working tree. For
   each downstream, the extract's change since the commit that downstream last took is merged into its copy and
   gated by that downstream's commands. The downstream commit says `take extract v1.2.0`; if the extract has
   commits after its last release tag (a hold you resolved by hand, say), it says `take extract v1.2.0+abc1234`
   and no tag is made for them. A module marked `diverged` for a downstream was rewritten there on purpose and
   is skipped.

If a gate fails, the files are put back exactly as they were. If a batch fails, each module is tried alone and
the ones that pass alone still ship, so one broken module does not hold up the rest.

It also refuses to touch a managed file that has uncommitted edits (`held: uncommitted changes`), and commits
only the files it manages, so anything else you had staged stays staged.

#### Commands

| Command | What it does |
|---|---|
| `cleanfork init` | Write a starter `cleanfork.json`. Never overwrites. |
| `cleanfork baselines` | Run once. For each module, record the upstream commit the extract best matches (it must match at least 50%; among equal matches the oldest wins, so an upstream commit that only added code is carried, not skipped), and take each downstream to hold the extract's current commit. Fills only what is missing. |
| `cleanfork plan` | Read-only. Print what would move and what is held. `--diff` shows each ready change, `--report FILE` writes it as JSON, `--held-dir DIR` writes each held merge with its conflict markers for a person to finish, and records the commit it was planned against. |
| `cleanfork apply` | Merge, gate, commit and release what is ready. `--push` pushes what was committed, `--trailer TEXT` is appended to every commit message, `--json` prints the full result. |
| `cleanfork accept MODULE...` | Record that you resolved a held merge by hand (see below). `--downstream NAME` accepts that downstream's copy instead; `--to SHA` names the commit you resolved against. |
| `cleanfork check` | Scan every git-tracked file of the extract for forbidden words (binary files and symlinks are skipped and listed), run the extract's gates, and print how close each module still is to its upstream and downstream copies. `--skip-gates`, `--report FILE`. |

Global option: `-c FILE` (default `cleanfork.json`).

Exit codes: `0` ok. `1` a gate failed during `apply`, `check` found a forbidden word or a failing gate, or
`baselines` could not record a module. `2` a config or git error (the message names the field or command).

#### Resolving a held merge

`apply` and `plan --held-dir DIR` remember which upstream commit each hold was planned against; `--held-dir`
also writes the merged text, conflict markers included. Finish it by hand in the extract and commit it. Then
tell cleanfork you are done with `cleanfork accept MODULE`, which records that the extract is in step with
that commit, not with whatever upstream has become since: a change upstream made while you worked on the hold
is still carried by the next `apply`. (If your edit is exactly the upstream change, the next `apply` notices
by itself and moves on without `accept`.) For a downstream copy you resolved by hand,
`cleanfork accept MODULE --downstream NAME` records that it holds the extract commit its hold was planned
against. With no hold recorded, `accept` refuses unless you name the commit with `--to SHA`. Until you accept,
the module stays held, every run.

#### The config file

One JSON file holds the settings and the state. Relative paths resolve against the folder that holds it.
`baselines` and `apply` write commit ids back into it; commit it with your pipeline.

```json
{
  "upstream":   {"path": "../app", "ref": "origin/main"},
  "extract":    {"path": "../extract", "tag_prefix": "v",
                 "gates": [["{python}", "-m", "pytest", "-q"]],
                 "push": {"remote": "origin", "branch": "main"}},
  "downstreams": {"site": {"path": "../site", "gates": [["{python}", "-m", "pytest", "-q"]]}},
  "strip_prefixes": ["acme_"],
  "private_import_roots": ["packages", "apps"],
  "forbid": ["\\bacme\\s*corp\\b", {"pattern": "\\bstaging\\b", "exclude": ["docs/**"]}],
  "modules": {
    "billing": {"upstream": "packages/acme_core/billing.py", "extract": "src/billing.py",
                "downstream": {"site": "packages/site_core/billing.py"}}
  }
}
```

| Field | Meaning |
|---|---|
| `upstream` | `path` of a local checkout and the `ref` to read (default `origin/main`). Fetch it yourself before you run. |
| `extract` | The clean repository: `path`, `gates`, `push`, and `tag_prefix` for release tags (default `v`). |
| `downstreams` | Name to `{path, gates, push}`. Any number. |
| `strip_prefixes` | Product prefixes removed from upstream text before merging and before comparing. |
| `private_import_roots` | A merge that adds `from <root>...` or `import <root>...` is held as a new dependency. |
| `forbid` | Required. Regexes matched case-insensitively against each line; or `{pattern, paths, exclude, id}` objects (the rule format of branding-leak-guard). `[]` opts out on purpose. |
| `scan_exclude` | Globs `check` does not scan, for example a test that lists the forbidden words. |
| `gate_env`, `gate_timeout` | Environment added to gate commands; seconds before a gate is failed (default 1800). |
| `in_step` | Share of code lines that must still match for `check` to say "in step" (default 0.9). |
| `modules` | Per file: `upstream` path (or `null`), `extract` path (default: the upstream path with the prefixes removed), `downstream` name to path, `diverged` list of downstream names. `upstream_commit`, `synced`, `held_upstream` and `held_downstream` are the state. |

A gate is a command, as a list of arguments or a string. `{python}` stands for the interpreter running
cleanfork. Commands run in the repository they guard, with your environment, and stop at the first failure.
`apply` refuses to land anything in a repository with no gates unless you pass `--ungated`. `--ungated` waives
that only for a repository that has no gate commands: a repository with gates always runs them, and a commit
landed without any says so in its message.

#### Library use

Everything the command does is a function: `cleanfork.load`, `plan`, `apply`, `record_baselines`,
`provenance`, `scan_extract`, `scan_extract_report`, `merge3`, `debrand`, `new_private_imports`, `overlap`, `run_gates`, `next_minor`.

### Limits

- **One upstream per config file.** A second upstream is a second config file. The chain is fixed: upstream,
  then one extract, then any number of downstreams.
- **It does not make the first extract.** You make the first clean copy and add its files to `modules`;
  cleanfork keeps it in step from then on. A new upstream file is not noticed until you add a module for it.
- **Whole text files, merged line by line.** The merge is textual, not semantic: a clean merge can still be
  wrong, which is what the gates are for. Files must be UTF-8 text (anything else is held), and line endings are preserved, so a downstream copy whose line endings differ from the extract's (LF against CRLF) is held as a conflict.
- **The brand check is a line scan.** It reads every tracked file that has no NUL byte, for your regexes, including
  a phrase split across joined string literals. A phrase wrapped across two lines, or written with look-alike
  Unicode characters, is not caught. It does not read images or binaries, git history, or file names, and it only
  knows the words you list. `check` exits 2 if the extract is not a git repository or has nothing to scan. Run it
  on the extract before you publish it (`cleanfork check`).
- **Dependency detection reads Python imports only**, and absolute ones (`from x import`, `import x`); a relative
  import (`from .x import`) is not seen. Comment and docstring
  stripping in `check` knows Python, shell, YAML/TOML and C-style languages.
- **A baseline is a match, not a record.** `baselines` picks the upstream commit the extract most resembles
  (the oldest of equal matches) and refuses a match below 50%; verify it with `plan` before the first `apply`.
  It also takes each downstream to hold the extract's current commit without looking: a downstream really on an
  older extract version has the changes in between never carried, so run `baselines` only when they are in step.
- **No history is carried.** The extract gets one commit per release, not the upstream commits.
- **It runs your gate commands and your `git`.** Gates run with your permissions, and `--push` uses your git
  credentials. It commits to the repositories you point it at; it does not fetch the upstream checkout for you.
- **Uncommitted state.** The config file is rewritten in place; committing it is up to you.

Requires Python 3.10 or newer and `git` on PATH. No other dependencies.

---

Docs mirror of the private repository Dominares-Tools/cleanfork; updated with each release.
