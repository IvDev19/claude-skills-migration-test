---
name: skill-promote
description: Bootstrap a local-only library skill into its own external IvDev19 repo — create the repo, push the skill folder as its root, and flip the registry entry to active with a sync baseline. Use when the user wants to promote or publish one or more local-only skills to standalone repos, with phrases like "promote this skill", "publish the local-only skills", "give this skill its own repo", or running /skill-promote with skill names.
---

<!-- debug: skill dir = ${CLAUDE_SKILL_DIR} -->

# skill-promote

Give a skill that only lives in `claude-skills` its own external repo. For a
`local-only` skill there is no external source yet — `claude-skills` *is* the
truth. Promotion seeds a new `IvDev19/skill-<name>` repo from the local folder and
flips authority to it, so `skill-publish` and `skill-sync` (which both treat the
external repo as the source of truth) can take over from there.

This is the one place data flows **local → external**. It's a one-time bootstrap
per skill, distinct from `skill-publish` (mirror an existing external repo in) and
`skill-sync` (keep them aligned). Run it from inside a `claude-skills` checkout —
the registry commit lands there. The `scripts/…` commands are claude-skills'
repo-root scripts.

## Steps

1. **Choose which skills to promote.** Eligibility comes from `registry.json`: read it and
   take only `status: local-only` entries. Exclude every `active` skill (already has a repo)
   and every `internal` one (skill-promote, skill-publish, skill-sync — no external home by
   design); if the user names one of those, say so and skip it. Always filter against the
   registry *before* listing or acting.
   - Names (or `all`) given as the argument → use those, filtered to `local-only`.
   - Nothing given → present the eligible skills as a checklist, one per line, each a literal
     checkbox the user ticks by changing `[ ]` to `[x]`:
     ```
     [ ] 1. caveman
     [ ] 2. diagnose
     [ ] 3. …
     ```
     Process only the ticked skills. Don't default to all.

2. **Pre-flight each chosen skill.** A promoted skill becomes a standalone repo
   with `source_path: "."`, so anything reaching outside its own folder breaks.
   Check for relative references that escape the folder (e.g. `../other-skill/…`).
   If found, don't promote it as-is — flag it and offer to resolve first by vendoring
   the referenced files with `bash scripts/vendor.sh`. `improve-codebase-architecture`
   is the known case: it vendors `CONTEXT-FORMAT.md` and `ADR-FORMAT.md` from
   grill-with-docs. Then run `bash scripts/check-vendored.sh skills/<name>` — a stale
   vendored copy would ship outdated shared content, so refresh it before promoting.

3. **Show the plan and get explicit go-ahead.** List exactly which repos will be
   created (`IvDev19/skill-<name>`), what gets pushed, and the registry edits.
   Creating public repos is outward-facing and hard to undo — never create one
   before the user has approved this plan.

4. **For each approved skill, build its README, then create and seed the repo.** Write the
   skill's `README.md` to the standard structure (see **README** below), then run
   `bash scripts/gen-provenance.sh skills/<name>` to append/refresh its Content-dependencies
   block as the final section. Both land in `skills/<name>/`, so they ship *as part of the
   content* and the mirror baseline stays exact. Then create `IvDev19/skill-<name>`, push the
   *complete contents* of `skills/<name>/` as the repo root, and capture the commit SHA. The
   mechanics (git and Code-section paths, exactly what gets pushed) are in
   `references/bootstrapping.md`.

5. **Flip the registry entry to `active`.** Set `repo` to the new SSH URL, `branch`
   (`main`), `source_path` (`.`), `status: active`, and the baseline:
   `last_synced_commit` (the push SHA) and `last_synced_hash`
   (`bash scripts/skill-hash.sh skills/<name>`). At this moment local == external
   == baseline — the clean starting point `skill-sync` expects.

6. **Commit, open the PR, and merge it.** A promote changes `registry.json` and the skill's
   `README.md` (its provenance block too, when it has vendored deps). The `.claude/commands/`
   body (generated from `SKILL.md`) is unaffected. Commit with
   `feat: promote <names> to external repos`, open a PR to `main`, then **merge it** — the
   merge is part of this flow, not a hand-off, and needs no extra confirmation (the plan was
   approved in step 3). Confirm with the merged PR URL.

7. **Hand off.** Those skills are now `active`: `skill-publish` re-mirrors them on
   demand and `skill-sync` keeps them aligned, with the external repo as the truth.

8. **Offer the next one.** Once the PR is merged, close the loop conversationally instead of
   just stopping:

   > Done — `<name>` is published and registered as `active`. Want to promote another?

   Then offer the same three paths as skill-sync's "Choosing scope", and only build a list
   if asked:

   1. **Pick from a list** — *then* read `registry.json` and present the promotable
      (`local-only`) skills as a literal checklist (`[ ] 1. skill-name`, one per line, ticked
      with `[x]`); process only the ticked skills.
   2. **Name them** — the user types the skill names; process exactly those.
   3. **One inline** — the user just names a single skill; proceed directly with it.

   Don't generate the list until path 1 is chosen. For the chosen skills, pick the loop back
   up at **step 2 (pre-flight)** — same eligibility rule (only `local-only`; say so and skip
   any that aren't), no need to re-invoke `/skill-promote`. If they decline or don't respond,
   end the session cleanly.

## README

Every promoted skill ships a `README.md` in its folder — baseline-safe, and it travels with
the mirror. It's the claude-skills library README at single-skill scope; the canonical model
to follow is that file itself — https://github.com/IvDev19/claude-skills/blob/main/README.md
(consult it when a section's shape is unclear). Build it from the skill's own `SKILL.md`;
never invent features or behaviour. If a `README.md` already exists, read it, keep any genuine
human prose, and rewrite it to this structure so every promoted skill reads the same way
regardless of when it was made.

Sections, in order:

1. **Header** — `# skill-<name>`, then the one-line `description` from the frontmatter.
2. **Intro** — a short paragraph: what the skill does and when to use it, expanded from the
   description.
3. **How it works** — the skill's main loop or decision flow in a few sentences, inferred from
   the SKILL.md body. No invented behaviour.
4. **Usage** — how to invoke it (`/skill-<name>` with any arguments), what it expects, and what
   it produces.
5. **Files in this repo** — a small table: each file or directory and its purpose. `SKILL.md`
   always; add rows for `references/`, `scripts/`, `agents/` only when they exist.
6. **Install** — a short block: clone into a claude-skills library and publish the command with
   the library's sync step. Keep it minimal.
7. **Content dependencies** — *only* when the skill has a `vendored-from:` provenance block.
   Don't write it by hand: after sections 1–6, `bash scripts/gen-provenance.sh skills/<name>`
   generates it as the final section (and keeps it current later). Omit the section entirely
   when there's no block.

Match the library README's tone — direct and technical, no marketing — and let length scale
with the skill: a three-file skill needs a shorter README than an eight-file one. Don't pad.

## Guardrails

- Never create a repo before the plan is shown and approved.
- Push the skill folder's contents *exactly* — never write a file only into the external repo,
  or local and external diverge and the baseline is a lie. The generated `README.md` is safe
  precisely because it's written into `skills/<name>/` first (local == external); anything else
  you want shipped follows the same rule.
- Don't promote a skill with unresolved cross-folder references; fix it first.
- Keep all skill content in English.

## Reference

- `references/bootstrapping.md` — what gets pushed and why, the create-and-push
  mechanics (git and Code-section), and the registry baseline this writes.
