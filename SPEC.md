# gf: small specs, hard gates

A build spec for a coding model. Paste it into the model, in an empty repository, and build it one step at a time.

`gf` is a checker for repositories where coding agents work. A ticket is split into small specs. A large ticket gets a short design first. A person approves each spec and its tests before any code is written. Every rule is checked by a script, in the terminal and in CI. No model judges anything, and no agent hooks are used.

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
| `design_min_points` | `5` | A ticket with this estimate or more needs a design. 0 turns it off |
| `design_min_specs` | `4` | A ticket with this many specs or more needs a design. 0 turns it off |
| `skills_dirs` | `["skills"]` | Where the skills a spec may name live |
| `gates` | `["git"]` | The kinds of approval this repository accepts (section 5) |

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
.gf/templates/design.md       an empty design
skills/<name>/SKILL.md        how this repository does a recurring kind of work
AGENTS.md                     the working agreement for agents (step 8)
specs/<TICKET>-<slug>/
  story.md                    the ticket: key, title, estimate, list of specs
  design.md                   only for a large ticket
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
points: 3
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
skills: [python-parsing]
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
- `skills` is optional. Each name must exist as `<skills_dir>/<name>/SKILL.md`. The agent reads those files before it writes code.
- A line that holds `[NEEDS CLARIFICATION: <question>]` marks an open question. Write one instead of guessing.
- Approval adds four keys: `approved_by`, `approved_via`, `approved_spec_sha`, `locked_tests`.

**`design.md`**, only when the ticket is large: `points` is at least `design_min_points`, or the ticket has at least `design_min_specs` specs.

```markdown
---
ticket: DATA-123
status: draft
---

## Approach
## Alternatives
## Delivery order
## Risks
```

- All four sections must have text. `status` is `draft` or `approved`.
- Approval adds `approved_by`, `approved_via` and `approved_design_sha`.

**A test tag.** Any line in a test file that holds the word `req` and a requirement id links the file to that requirement. This works in every language:

```
@pytest.mark.req("DATA-123-S1.R1")      # Python
-- req: DATA-123-S1.R1                   SQL
// req: DATA-123-S1.R1                   anything else
```

**A commit.** A commit that changes code names its spec in the message, on a line of its own: `Spec: DATA-123-S1`.

**Hashes.** Use SHA-256 and keep the first 16 hex characters.
- File hash: over the file's bytes, after replacing `\r\n` with `\n`.
- Spec hash: over the spec's text without the approval lines: `status` and every key that starts with `approved_`, plus `locked_tests`. The design hash is computed the same way.
- `locked_tests` is a list of `path@filehash`.

**Globs.** `**` matches any number of folders, `*` matches inside one path segment.

## 5. Rules

Local rules run on the files. Git rules run only with `--ci`, on the commits between the merge base with `origin/<source_branch>` and `HEAD`, merge commits left out.

| Code | Where | Fails when |
|---|---|---|
| D1 | local | A `specs/` folder has no `story.md`, its `ticket` does not match `ticket_pattern`, a spec file is not listed under `## Specs`, or a spec id does not start with the folder's ticket |
| D2 | local | `size` is not in `sizes`, there are more than `max_requirements` requirements or more than `max_scope` scope entries, or a scope entry starts with `*` |
| D3 | local | A large ticket has a spec that is `approved` or `done`, and its `design.md` is missing, not approved, or changed after approval. Or a `design.md` lacks a section |
| S1 | local | Intent, Non-goals, Edge cases or Requirements is missing or empty, or Edge cases comes after Requirements |
| S2 | local | A requirement does not have the form of section 4, or two requirements share a number |
| S3 | local | A spec names a skill that does not exist in `skills_dirs` |
| S4 | local | A spec or design that is `approved` holds an open question |
| T1 | local | A spec is `approved` or `done` and one of its requirements has no tagged test |
| T2 | local | A tag names a requirement that does not exist |
| T3 | local | One test file is tagged for two different specs |
| G1 | local | A spec is `approved` or `done` and `approved_by` is missing or not in `humans`, or `approved_spec_sha` is not the current spec hash |
| G2 | local | A file in `locked_tests` is missing or its hash changed, or a file tagged for an approved spec is not in `locked_tests` |
| G3 | git | The commit that last changed a spec's `approved_spec_sha`, or a design's `approved_design_sha`, was not written by one of `humans` |
| G5 | local | `approved_via` names a gate that is not in `gates` |
| C1 | git | A commit changes a file outside the `scope` of the spec it names. That spec's own test files are allowed |
| C2 | git | A commit changes a file outside `free_paths` and `test_globs`, and names no spec, or names one that is not approved |
| P1 | git | A commit changes `.gf/**`, `skills/**`, `AGENTS.md` or a CI file, and its author is not one of `humans` |
| D4 | git | A commit changes `points` in a `story.md` that already exists on the source branch, and its author is not one of `humans` |

A `draft` spec is checked for D1, D2, S1, S2 and S3 only, plus G2 if it still has `locked_tests`.

**Gates.** A gate is a kind of proof that a person approved, and a way to check it. `gates` lists the kinds this repository accepts, and `approved_via` records the one used. The list lives in the config, never in the spec, so an agent cannot choose a weaker gate. This build has one gate, `git`: the proof is the approval lines, committed by one of `humans` (G1, G3). To add another gate later, for example a comment in your ticket tracker, keep one rule: the approval must name the spec hash and the hashes of the locked tests, so that it approves one exact version.

## 6. Build steps

| # | Build | Done when |
|---|---|---|
| 1 | `config.json`, the front-matter parser, loading of `specs/`. Command `gf.py status` prints one line per spec: id, status, title | `status` lists the spec of section 4 |
| 2 | Rules D1, D2, S1, S2, S3. Command `gf.py check` | Scenarios 1 to 4, and 18 |
| 3 | The tag scanner over `test_globs`. Rules T1, T2, T3 | Scenarios 5 to 7 |
| 4 | The two hashes. Rules G1, G2, G5, S4 | Scenario 8, with the approval fields written by hand, and 19 |
| 5 | Command `gf.py approve <id>`, described below. Rule D3 | Scenarios 9 to 11, and 20 to 22 |
| 6 | `gf.py check --ci [--base <ref>]` with rules C1, C2, G3, P1, D4 | Scenarios 12 to 16, and 23 |
| 7 | `gf.py verify`: runs `check`, then `test_command`, and fails if either fails | Scenario 17 |
| 8 | `AGENTS.md` (the loop below, in your own words) and a CI job, described below | A person reads both |

**`approve`, step 5.** Refuse, with exit code 1, when:
- stdin or stdout is not a terminal;
- one of the environment variables `CI`, `CLAUDECODE` or `GF_AGENT` is set;
- `git config user.email` is not in `humans`;
- the spec would fail `check` once approved, for example because a requirement has no test;
- the spec holds an open question;
- the ticket is large and its design is not approved.

Otherwise ask the person to type the spec id, and stop if it does not match. Then write `status: approved`, `approved_by`, `approved_via: git`, `approved_spec_sha` and `locked_tests` into the front matter. The person commits the file themselves.

`approve <TICKET>` approves that ticket's `design.md` in the same way: the same refusals, except the last, and it writes `approved_design_sha` and no `locked_tests`.

**The CI job, step 8.** CI must not run the checker that the pull request brings, or a pull request could rewrite its own judge. Take the checker from the source branch:

```bash
git fetch origin <source_branch>
mkdir /tmp/gf && git archive origin/<source_branch> .gf | tar -x -C /tmp/gf
GF_ROOT="$PWD" python3 /tmp/gf/.gf/gf.py check --ci
```

`gf.py` reads its config from its own folder, and reads the repository from `GF_ROOT` when that variable is set.

**The loop that `AGENTS.md` describes.**
1. A ticket becomes a folder in `specs/` with a `story.md`.
2. For a large ticket the agent drafts `design.md` and stops. A person approves it.
3. The agent drafts small specs and one tagged test per requirement, runs `check`, and stops. What it is unsure about becomes an open question, not a guess.
4. A person reads them and runs `approve`. This is gate 1.
5. The agent reads the skills the spec names, then writes code inside the spec's scope. Every commit names its spec.
6. `verify` is green. A person reviews and merges. This is gate 2.
7. The agent never runs `approve`, never writes the approval fields, and never edits `.gf/`, `skills/`, the estimate or the CI files.

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
| 18 | A spec with `skills: [nope]` and no such skill | S3 |
| 19 | An approved spec with `approved_via: tracker` while `gates` is `["git"]` | G5 |
| 20 | A draft spec with an open question | `check` OK, `approve` exits 1 |
| 21 | `points: 5`, no design, `approve` of a spec | exit 1, spec still `draft` |
| 22 | The design approved, then the spec; then one word changed in the design | approvals succeed, then D3 |
| 23 | `points` changed from 5 to 2 by an address in `agent_authors` | D4 |

## 8. Out of scope

Leave these out of the first build. Each one can be added later as a spec of its own:
- a budget per ticket and cost reporting in commits;
- reading the ticket and its estimate from Jira or another tracker;
- a second gate, such as an approving comment in the tracker;
- proof from a JUnit report that the locked tests really ran;
- signed commits as proof of who approved;
- tickets that span several repositories, and one design shared by them.

## 9. Known limits

- "A person" means a git author email, and anyone can type an email into git. For real identity, also require a code owner's review on every change under `specs/` and `.gf/`, or add a gate whose proof lives where identity is real.
- The checker verifies that a named skill exists, not that the agent followed it. The tests do that.
- An open question blocks approval only once someone wrote it down. Nothing finds an ambiguity by itself.
