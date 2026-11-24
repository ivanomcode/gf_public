# gf: small specs, hard gates

A build spec for a coding model. Paste it into the model, in an empty repository, and build it one step at a time.

`gf` is a checker for repositories where coding agents work. A ticket is split into small specs. A person approves each spec and its tests before any code is written. Every rule is checked by a script, in the terminal and in CI. No model judges anything, and no agent hooks are used.

## 0. How to work through this spec

1. Read sections 1 to 5 once. Do not write code yet.
2. Build the steps of section 6 in order, one per commit.
3. After each step, run the check named in its "Done when" column. If it fails, fix it before the next step.
4. Build only what is written here. Section 8 lists what to leave out.

## 1. Adapt first

Change these values before the build. They all end up in `.gf/config.json`; nothing else in the spec depends on your team.

| Setting | Default | Meaning |
|---|---|---|
| `ticket_pattern` | `[A-Z]+-[0-9]+` | What a ticket key looks like, for example `DATA-123` |
| `source_branch` | `main` | The branch features start from and are compared with |
| `humans` | `[]` | Git emails of the people who may approve |
| `agent_authors` | `[]` | Git emails of bot accounts |
| `sizes` | `["XS", "S", "M"]` | Allowed spec sizes |
| `max_requirements` | `5` | Most requirements in one spec |
| `max_scope` | `6` | Most entries in a spec's `scope` |
| `test_globs` | `["tests/**"]` | Where test files live |
| `free_paths` | `["specs/**", "README.md", "docs/**"]` | Paths that need no spec |
| `test_command` | `["python3", "-m", "pytest", "-q"]` | How the tests run |

## 2. Constraints

- Python 3.10 or later, standard library only. One file: `.gf/gf.py`.
- Deterministic: the checker uses no network, no clock, no randomness and no model. The same repository state always gives the same output.
- Every error is one line, `CODE path: message`. Errors are sorted. The last line is `gf: OK (0 errors)` or `gf: FAIL (<n> errors)`.
- Exit code 0 when there is no error, 1 otherwise.
- Files are read as UTF-8, and `\r\n` is treated as `\n` everywhere.

## 3. Files

```
.gf/gf.py                     the checker
.gf/config.json               the settings of section 1
.gf/templates/spec.md         an empty spec
AGENTS.md                     the working agreement for agents (step 8)
specs/<TICKET>-<slug>/
  story.md                    the ticket: key, title, list of specs
  S1-<slug>.md                one small spec
  S2-<slug>.md
selftest.py                   the scenarios of section 7
```

## 4. Formats

Front matter is the block between the first two `---` lines. Parse it by hand: one `key: value` per line, and a list is written on one line as `[a, b]`.

**`story.md`**

```markdown
---
ticket: DATA-123
title: Add sales_channel to the sales dataset
---

## Specs
- DATA-123-S1 · add the column to the table
```

**A spec, `S1-<slug>.md`**

```markdown
---
id: DATA-123-S1
title: Add the column to the table
status: draft
size: S
scope: [src/loader/transform.py]
---

## Intent
One or two sentences.

## Non-goals
- What this spec will not do.

## Edge cases
- Listed before the requirements, on purpose.

## Requirements
- R1: WHEN the channel is empty, THE SYSTEM SHALL write None.
- R2: THE SYSTEM SHALL keep the other fields unchanged.
```

- `status` is `draft`, `approved` or `done`.
- A requirement line matches `- R<n>: <text>`. The text must contain `THE SYSTEM SHALL`, alone or after a clause that starts with `WHEN`, `WHILE`, `IF` or `WHERE` and ends with a comma.
- Approval adds three keys: `approved_by`, `approved_spec_sha`, `locked_tests`.

**A test tag.** Any line in a test file that holds the word `req` and a requirement id links the file to that requirement. This works in every language:

```
@pytest.mark.req("DATA-123-S1.R1")      # Python
-- req: DATA-123-S1.R1                   SQL
// req: DATA-123-S1.R1                   anything else
```

**A commit.** A commit that changes code names its spec in the message, on a line of its own: `Spec: DATA-123-S1`.

**Hashes.** Use SHA-256 and keep the first 16 hex characters.
- File hash: over the file's bytes, after replacing `\r\n` with `\n`.
- Spec hash: over the spec's text without the lines `status`, `approved_by`, `approved_spec_sha` and `locked_tests`.
- `locked_tests` is a list of `path@filehash`.

**Globs.** `**` matches any number of folders, `*` matches inside one path segment.

## 5. Rules

Local rules run on the files. Git rules run only with `--ci`, on the commits between the merge base with `origin/<source_branch>` and `HEAD`, merge commits left out.

| Code | Where | Fails when |
|---|---|---|
| D1 | local | A `specs/` folder has no `story.md`, its `ticket` does not match `ticket_pattern`, a spec file is not listed under `## Specs`, or a spec id does not start with the folder's ticket |
| D2 | local | `size` is not in `sizes`, there are more than `max_requirements` requirements or more than `max_scope` scope entries, or a scope entry starts with `*` |
| S1 | local | Intent, Non-goals, Edge cases or Requirements is missing or empty, or Edge cases comes after Requirements |
| S2 | local | A requirement does not have the form of section 4, or two requirements share a number |
| T1 | local | A spec is `approved` or `done` and one of its requirements has no tagged test |
| T2 | local | A tag names a requirement that does not exist |
| T3 | local | One test file is tagged for two different specs |
| G1 | local | A spec is `approved` or `done` and `approved_by` is missing or not in `humans`, or `approved_spec_sha` is not the current spec hash |
| G2 | local | A file in `locked_tests` is missing or its hash changed, or a file tagged for an approved spec is not in `locked_tests` |
| G3 | git | The commit that last changed a spec's `approved_spec_sha` was not written by one of `humans` |
| C1 | git | A commit changes a file outside the `scope` of the spec it names. That spec's own test files are allowed |
| C2 | git | A commit changes a file outside `free_paths` and `test_globs`, and names no spec, or names one that is not approved |
| P1 | git | A commit changes `.gf/**`, `AGENTS.md` or a CI file, and its author is not one of `humans` |

A `draft` spec is checked for D and S rules only, plus G2 if it still has `locked_tests`.

## 6. Build steps

| # | Build | Done when |
|---|---|---|
| 1 | `config.json`, the front-matter parser, loading of `specs/`. Command `gf.py status` prints one line per spec: id, status, title | `status` lists the spec of section 4 |
| 2 | Rules D1, D2, S1, S2. Command `gf.py check` | Scenarios 1 to 4 |
| 3 | The tag scanner over `test_globs`. Rules T1, T2, T3 | Scenarios 5 to 7 |
| 4 | The two hashes. Rules G1, G2 | Scenario 8, with the approval fields written by hand |
| 5 | Command `gf.py approve <spec-id>`, described below | Scenarios 9 to 11 |
| 6 | `gf.py check --ci [--base <ref>]` with rules C1, C2, G3, P1 | Scenarios 12 to 16 |
| 7 | `gf.py verify`: runs `check`, then `test_command`, and fails if either fails | Scenario 17 |
| 8 | `AGENTS.md` (the loop below, in your own words) and a CI job, described below | A person reads both |

**`approve`, step 5.** Refuse, with exit code 1, when:
- stdin or stdout is not a terminal;
- one of the environment variables `CI`, `CLAUDECODE` or `GF_AGENT` is set;
- `git config user.email` is not in `humans`;
- the spec would fail `check` once approved, for example because a requirement has no test.

Otherwise ask the person to type the spec id, and stop if it does not match. Then write `status: approved`, `approved_by`, `approved_spec_sha` and `locked_tests` into the front matter. The person commits the file themselves.

**The CI job, step 8.** CI must not run the checker that the pull request brings, or a pull request could rewrite its own judge. Take the checker from the source branch:

```bash
git fetch origin <source_branch>
mkdir /tmp/gf && git archive origin/<source_branch> .gf | tar -x -C /tmp/gf
GF_ROOT="$PWD" python3 /tmp/gf/.gf/gf.py check --ci
```

`gf.py` reads its config from its own folder, and reads the repository from `GF_ROOT` when that variable is set.

**The loop that `AGENTS.md` describes.**
1. A ticket becomes a folder in `specs/` with a `story.md`.
2. The agent drafts small specs and one tagged test per requirement, runs `check`, and stops.
3. A person reads them and runs `approve`. This is gate 1.
4. The agent writes code inside the spec's scope. Every commit names its spec.
5. `verify` is green. A person reviews and merges. This is gate 2.
6. The agent never runs `approve`, never writes the approval fields, and never edits `.gf/` or the CI files.

## 7. Self-test

`selftest.py` builds a throwaway git repository in a temporary folder for each scenario, runs `gf.py` in it, and prints `PASS` or `FAIL` per scenario and a final count. It exits with 1 if any scenario fails. Two runs give the same output.

| # | Scenario | Expected |
|---|---|---|
| 1 | The spec of section 4, as a draft | OK |
| 2 | `size: L` | D2 |
| 3 | The Edge cases section removed | S1 |
| 4 | A requirement written as "should handle empty values" | S2 |
| 5 | A tag for `DATA-123-S1.R9` | T2 |
| 6 | One test file tagged for S1 and S2 | T3 |
| 7 | An approved spec with an untested requirement | T1 |
| 8 | An approved spec, then one character changed in a locked test | G2 |
| 9 | `approve` run without a terminal | exit 1, spec still `draft` |
| 10 | `approve` in a pseudo-terminal (`pty`), with the right id typed | spec `approved`, tests locked |
| 11 | An approved spec, then a requirement edited | G1 |
| 12 | Approved by a person; code committed in scope, naming the spec | `check --ci` OK |
| 13 | A commit that changes a file outside the scope | C1 |
| 14 | A code commit that names no spec | C2 |
| 15 | The approval committed by an address in `agent_authors` | G3 |
| 16 | `.gf/gf.py` changed by an address in `agent_authors` | P1 |
| 17 | The code missing, so the tests fail | `verify` exits 1 |

## 8. Out of scope

Leave these out of the first build. Each one can be added later as a spec of its own:
- a budget per ticket and cost reporting in commits;
- reading the ticket from Jira or another tracker;
- proof from a JUnit report that the locked tests really ran;
- signed commits as proof of who approved;
- tickets that span several repositories.

## 9. Known limit

"A person" means a git author email, and anyone can type an email into git. For real identity, also require a code owner's review on every change under `specs/` and `.gf/`.
