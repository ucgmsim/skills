---
name: release
description: Cut a new GitHub release (and trigger PyPI publish) for a ucgmsim repo — source_modelling, workflow, IM_calculation, qcore, high-frequency (hf_port), etc. — with curated release notes in the house style. Use when the user asks to cut, make, tag or publish a release.
---

# Cut a ucgmsim release

Invoking `/release` authorises creating and publishing the release, which triggers the PyPI upload. Do not push commits or edit files. If the user passes a tag or target commit, use it.

## Steps

1. **Orient.**
   - `gh repo view --json nameWithOwner,defaultBranchRef`. The default branch is `main` or `master`; in **ucgmsim/workflow** it is `pegasus`.
   - `git fetch --tags origin` and `git fetch origin <default>`. Local tags are often stale, so trust `gh release list --limit 5`, not `git tag`.
2. **Pick the tag.** Use calendar versioning, `vYYYY.MM.N`, with a zero-padded month and `N` counting from 1 within the month.
   - If the latest release is from this month, increment `N` (e.g. `v2026.10.4` → `v2026.10.5`).
   - Otherwise start at `.1` (e.g. `v2026.11.1`).
3. **Collect the changes since the last release.**
   - `git log --oneline <last-tag>..origin/<default>`
   - `gh pr list --state merged --base <default> --search "merged:>=<last release date>" --json number,title,author`
   - For every PR that is not trivial, read its body with `gh pr view N --json title,body`. The PR's "What was wrong / What changed / For reviewers" sections carry the impact you need: which outputs change, which don't, and by how much.
4. **Check that the release is shippable.**
   - `gh run list --branch <default> --limit 4` must show CI green on the target commit. If it isn't, stop and report.
   - Check how the version is set. If `pyproject.toml` has `dynamic = ["version"]` with setuptools-scm, the tag is enough. If there is a static version, stop and ask before bumping it.
5. **For workflow only, work out the dependency bumps.**
   - Run `git diff <last-tag> origin/pegasus -- uv.lock pyproject.toml`.
   - List the ucgmsim packages (and any major third-party bumps) that changed version, were added or were removed.
   - Read those packages' own release notes to describe each effect.
6. **Write the notes** to `$CLAUDE_JOB_DIR/tmp/notes.md` (or a scratch file), following the style below.
7. **Publish** with the **full** 40-character SHA. A short SHA fails with `Invalid target_commitish`.
   ```sh
   sha=$(git rev-parse origin/<default>)
   gh release create vYYYY.MM.N --target "$sha" --title vYYYY.MM.N \
     --notes-file notes.md --generate-notes --notes-start-tag <last-tag>
   ```
   `--generate-notes` appends GitHub's "What's Changed", "New Contributors" and "Full Changelog" after your notes. The notes file must therefore end with `---` and a blank line.
8. **Verify.**
   - `gh release view <tag> --json url,body` to check the rendered notes.
   - `gh run list --workflow publish-PyPI.yml --limit 2` to confirm the publish run started.
   - Report the URL and the publish run ID; the upload takes about 10 minutes. The publish is not done until that run succeeds, so don't call it done earlier.

## Release notes style

```markdown
## Summary

**<Headline sentence naming the user-visible changes.>** <Optional second sentence for lesser changes, and/or whether results change.>

### Breaking changes
- **<What changed, as API/behaviour>** (#N). <What users must do / what now raises.>

### Bug fixes
- **<What was wrong, past tense, specific>** (#N). <Consequence, and what output now changes or doesn't.>
- <Minor fix, unbolded> (#N).

### Features
- <New capability, with `identifiers`> (#N). <Whether existing results change.>

---

```

### Summary
- Write one or two sentences. Lead with what users notice, not with the PR list.
- **Always say whether results or output change**, e.g. "IM values are unchanged.", "Simulation output is unchanged.", "Output is bit-identical when…", "**pSA values change**…".
- Bold the headline when behaviour, results or the API change. If users must act, bold that call to action, e.g. "**Run `migrate` on realisations with … LF and BB results change.**"
- Workflow summaries often open with "This release …" and end on the bolded action or impact sentence.
- For a release with only internal changes: "Internal release that <reason> (#N). There are no behaviour changes." For packaging-only releases: "This release adds Python 3.15 wheels. <Output> is unchanged."

### Sections
- The order is **Breaking changes → Bug fixes → Features**. Leave out any empty section; never write "None".
- **Breaking changes** cover removed or renamed API, new required arguments, new exceptions, moved extras (`[hdf5]`, `[cli]`), and default changes that alter results. Name the replacement, e.g. "Use `site_calculation.amplification` instead."
- **Bug fixes:**
  - Bold the lead for any fix that affects results. Describe the old wrong behaviour in the past tense, precisely: "Hypocentre depth was `top + d / sin(dip)` instead of `top + d * sin(dip)`".
  - Then state the consequence and quantify it where the PR gives numbers ("about 13% at T = 0.05 s", "268 faults").
  - Say who is affected and who isn't ("SRFs read by the parser already have contiguous rows, so their output does not change.").
  - Minor fixes stay unbolded and take one sentence.
- **Features** name the new function, CLI or config key in backticks, and add "Results are unchanged" when that's true.
- **Leave out** test-only PRs, lockfile refreshes, CI and workflow changes, dependabot dev-tool bumps and docstring-only fixes. These still appear in the generated "What's Changed" list. A pure performance change can go under Features.

### Prose
- Use short, plain sentences and NZ/UK spelling ("behaviour", "honour").
- Put code, modules, keys, CLI names and file names in backticks.
- Put the PR reference `(#N)` at the end of the first sentence of a bullet. Group several PRs as `(#239, #242)`, and write cross-repo references as `ucgmsim/rupture_generator#42`.
- Use `→` for changes of value or version: "CFL 0.9 → 1.3".
- Don't use emojis. Don't add a "Contributors" or "Thanks" section; the generated notes cover that.

## ucgmsim/workflow exceptions

- **Default branch `pegasus`.** Target its full SHA, never the branch name.
- **The section order is Breaking changes → Features → Bug fixes → Significant dependency bumps.**
- **Results-affecting defaults are breaking.** In-place changes to default parameters (SW4 versions, CFL, refinement depths, Vs floors), schema keys that are removed, and changed empirical IM inputs all go under Breaking changes. Each says which outputs change (LF / HF / BB / IMs) and which realisations are affected.
- **Nested bullets are normal.** Use them to list the options of a new config key (`"predetermined"` (the default): …), the sub-changes of one breaking change, or migration steps. Name required migrations: "Run `migrate`, or delete the keys."
- **Cylc and pipeline notes** go with their feature, e.g. "The cylc graph needs an `extract-subset` task after `im-calc`."
- **`### Significant dependency bumps`** comes from the `uv.lock` and `pyproject.toml` diff:
  - Format: `- **<package> <old> → <new>:** <what it brings and whether workflow results change>.` Group minor bumps that don't change results into one bullet: "**oq_wrapper 2026.5.2 → 2026.10.1, im-calculation … , hf-simulation …** (#217). Workflow's results are unchanged."
  - Mark additions `**<package> <version> (new),**` and breaking bumps `**pandas 2 → 3 (breaking for environments that pin pandas<3),**`.
  - Note dropped direct dependencies and vendored code, e.g. "**pyfftw is no longer a direct dependency**".
  - Only list ucgmsim packages (qcore-utils, source-modelling, im-calculation, hf-simulation, oq_wrapper, nzcvm, velocity-modelling, rupture-generator, site-calculation, ucgmsim-imdb, …) and major third-party bumps. Leave out routine patch bumps.

## Reference releases

- `gh release view -R ucgmsim/workflow v2026.10.9`: the full workflow form.
- `gh release view -R ucgmsim/IM_calculation v2026.10.2`: a fix that changes results, with numbers.
- `gh release view -R ucgmsim/qcore v2026.10.1`: a release made mostly of breaking changes.
- `gh release view -R ucgmsim/high-frequency v2026.10.1`: breaking API changes with migration.
- `gh release view -R ucgmsim/source_modelling v2026.11.1`: a small bug-fix release.
