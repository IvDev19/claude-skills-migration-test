---
name: skill-creator-per
description: Create new skills, modify and improve existing skills, and measure skill performance. Use when users want to create a skill from scratch, edit, or optimize an existing skill, run evals to test a skill, benchmark skill performance with variance analysis, or optimize a skill's description for better triggering accuracy.
---

<!-- debug: skill dir = ${CLAUDE_SKILL_DIR} -->

# Skill Creator

A skill for creating new skills and iteratively improving them.

The default workflow assumes **Claude.ai or Claude Desktop** (no subagents, file system available via Code Execution). For Cowork or Claude Code, see the environment-specific references at the bottom.

## High-level flow

1. Capture intent — figure out what the skill should do.
2. Draft the SKILL.md.
3. Create 2–3 test prompts and run them.
4. Show the user the outputs, gather feedback.
5. Improve the skill based on feedback.
6. Repeat 3–5 until the user is satisfied.
7. (Optional) Optimize the description for better triggering.

Be flexible about where the user is in this loop. If they show up with a draft already written, skip ahead. If they say "just vibe with me, no formal evals", do that.

## Language

- Match the language the user writes in for the response.
- Skill artifacts themselves (SKILL.md, references, scripts, comments) are written in **English** regardless of user language — this maximizes how well the LLM processes the skill at runtime.
- Keep technical artifacts in English: code, identifiers, file names, error messages, API fields.

## Communicating with the user

The skill creator is used by people across a wide range of familiarity with coding jargon — from non-technical users to experienced engineers. Pay attention to context cues to calibrate communication. In the default case: "evaluation" and "benchmark" are borderline OK; for "JSON" and "assertion" wait for clear cues before using them without a brief explanation.

---

## Step 1: Capture intent

The current conversation may already contain the workflow the user wants to capture ("turn this into a skill"). If so, extract what you can from history first — tools used, sequence of steps, corrections, input/output formats — and have the user confirm before proceeding.

Ask:

1. What should this skill enable Claude to do?
2. When should it trigger? (what user phrases / contexts)
3. What's the expected output format?
4. Do we want test cases? Skills with objectively verifiable outputs (file transforms, data extraction, code generation, fixed workflows) benefit from them. Skills with subjective outputs (writing style, art) often don't. Suggest a default based on the skill type but let the user decide.

## Step 2: Interview and research

Proactively ask about edge cases, input/output formats, example files, success criteria, and dependencies. Don't write test prompts until this is ironed out.

If MCPs are available and useful (searching docs, finding similar skills, looking up best practices), use them. Come prepared with context to reduce burden on the user.

---

## Step 3: Write the SKILL.md

### Frontmatter

```yaml
---
name: skill-name              # kebab-case (lowercase, digits, hyphens only)
description: ...              # see below
---
```

The **description** is the primary triggering mechanism. Include both what the skill does AND specific contexts for when to use it. All "when to use" info goes here, not in the body.

Claude tends to **undertrigger** skills — to not use them when they'd be useful. Counteract this by writing descriptions that are slightly pushy. Instead of "How to build a dashboard for internal data", write "How to build a dashboard for internal data. Use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display company data, even if they don't explicitly say 'dashboard'."

### Skill anatomy

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description)
│   └── Markdown instructions
└── Bundled resources (optional)
    ├── scripts/     - Executable code for deterministic/repetitive tasks
    ├── references/  - Docs loaded into context as needed
    └── assets/      - Files used in output (templates, fonts, etc.)
```

For deeper detail on layout patterns (multi-domain skills, table of contents for large refs, etc.), see `references/skill-anatomy.md`.

### Progressive disclosure

Skills load in three levels:

1. **Metadata** (name + description) — always in context (~100 words).
2. **SKILL.md body** — loaded when the skill triggers (~500 lines ideal).
3. **Bundled resources** — loaded on demand (scripts can execute without loading their source).

**Keep SKILL.md under 500 lines.** Approaching that limit? Add hierarchy and reference files with clear pointers about when to load them.

**The 100-line austerity check.** Before finalizing a skill, ask: could this be 100 lines? If not, audit every line above that threshold — each must either direct action or prevent a mistake. Remove anything that explains rather than instructs, or exists only to feel complete. Not a hard limit; a deliberate pressure check.

### Writing style

- **Prefer the imperative.** "Do X" beats "You should do X".
- **Explain the why, not just the what.** Modern LLMs have good theory of mind. When given reasoning, they generalize beyond rote instructions. If you find yourself writing ALL CAPS `MUST` or `NEVER`, that's a yellow flag — reframe and explain why the requirement matters.
- **Generalize.** Skills will be used across many prompts. Don't overfit to specific examples.
- **Draft, then re-read.** Write a first pass and then look at it with fresh eyes. The second pass is where the noise comes out.

### Safety

Skills must not contain malware, exploit code, or anything that compromises security. A skill's contents should not surprise the user given its description. Roleplay skills are fine; covert exfiltration skills are not.

---

## Step 4: Test the skill (qualitative loop)

This is the heart of the loop in Claude.ai / Desktop. No subagents means no parallel runs — you'll do this sequentially. The user reviews each output and gives feedback inline.

### 4.1 Define test prompts

Write 2–3 realistic test prompts — the kind of thing a real user would say. Share them with the user: "Here are a few test cases I'd like to try. Look right, or want to add more?"

Save to `evals/evals.json` in the skill workspace:

```json
{
  "skill_name": "my-skill",
  "evals": [
    { "id": 1, "prompt": "User task prompt", "expected_output": "What success looks like" }
  ]
}
```

See `references/schemas.md` for the full schema. Assertions (objective checks) can be added later if useful.

### 4.2 Run each test case

For each prompt: read the skill's SKILL.md yourself, then carry out the test prompt following the skill's instructions. Save the output to `<skill-name>-workspace/iteration-1/eval-<id>/outputs/`.

This is less rigorous than running in an independent subagent (you wrote the skill and you're also running it — you have full context). The human review step in 4.3 compensates.

### 4.3 Present outputs and gather feedback

For each test case, show the user:
- The prompt that was run.
- The output. If it's a file the user needs to inspect (`.docx`, `.xlsx`, `.pptx`, `.pdf`), save it to a clear path and tell them where it is so they can download and open it. Use `present_files` if available.

Ask for feedback inline: "How does this look? Anything you'd change?"

Empty feedback means the user thought it was fine. Specific feedback is where the next iteration's improvements come from.

### 4.4 Improve the skill

After the user has reviewed all test cases, revise the skill. Four principles:

1. **Generalize from feedback.** Don't overfit to the test cases. The user iterates on a handful of examples because it's fast, but the skill will be used a million times across different prompts. If a stubborn issue persists, try different framing, different metaphors, different patterns — it's cheap to try.

2. **Keep the prompt lean.** Read transcripts, not just outputs. If the skill is making the model waste time on unproductive steps, cut those parts and see what happens.

3. **Look for repeated work.** If every test run independently wrote a similar helper script, that's a strong signal the skill should bundle that script. Write it once, put it in `scripts/`, point the skill to it.

4. **Explain the why.** Already covered in writing style. Worth repeating here — terse or frustrated feedback from the user often hides a real insight. Try to understand what they actually meant, then encode that understanding (not their literal words) into the instructions.

### 4.5 Re-run and repeat

Apply the improvements, run the test cases again into `iteration-2/`, present results, get feedback. Continue until:

- The user says they're happy.
- Feedback is all empty.
- You're not making meaningful progress.

Take your thinking time. Drafts are cheap.

---

## Updating an existing skill

If the user is improving an existing skill rather than starting from scratch:

- **Preserve the name.** Note the skill's directory name and frontmatter `name` — use them unchanged. If the installed skill is `research-helper`, output `research-helper.skill` (not `research-helper-v2`).
- **Copy to a writeable location before editing.** Installed skill paths are typically read-only. Copy to `/tmp/<skill-name>/`, edit there, and package from the copy.
- **Snapshot the original** before changing it. If you decide to do A/B testing of versions later, you'll need the baseline. `cp -r <skill-path> <workspace>/skill-snapshot/`.

The qualitative loop in Step 4 applies the same way. The only change: when picking a baseline, you can compare against the previous version of the skill (not "without skill"), since the user already considers the current version their starting point.

---

## Description optimization (optional)

The description field is the primary mechanism that determines whether Claude invokes a skill. After the skill works well, offer to optimize the description for better triggering accuracy.

This is **a separate, optional pass** with its own tooling. The fully automated loop requires `claude -p` (Claude Code only). On Claude.ai / Desktop / Cowork you can still do a manual version: write 15–20 trigger eval queries (mix of should-trigger and should-not-trigger), have the user review them, then iterate on the description by hand.

For both the manual variant and the automated loop, see `references/description-optimization.md`.

---

## Validation checklist

Before considering a skill done, verify:

- [ ] `name` in frontmatter is kebab-case and matches the directory name exactly.
- [ ] `description` covers both what the skill does AND when to trigger it.
- [ ] `description` does not contain angle brackets `< >` (rejected by the validator).
- [ ] SKILL.md is under 500 lines (ideally well under).
- [ ] Every reference file mentioned in SKILL.md actually exists.
- [ ] The skill is self-contained — no link escapes its folder (no `../` into a sibling skill). Need another skill's file? Vendor it with `bash scripts/vendor.sh` (verified by `bash scripts/check-vendored.sh`), so the skill can be promoted to its own repo.
- [ ] Every script in `scripts/` is referenced somewhere in SKILL.md or by another script.
- [ ] Instructions use the imperative form where appropriate.
- [ ] Heavy `MUST` / `NEVER` capitalization is replaced with reasoning wherever possible.

Run the validator:

```bash
python -m scripts.quick_validate <path-to-skill>
```

It checks frontmatter validity, line count, broken references, references that escape the skill folder, vendored-copy parity, and orphan scripts.

---

## Publishing

After validation passes, create a GitHub repo for the skill:

1. Repo name: `skill-<name>` under IvDev19 (e.g. skill named `dark-mode-toggle` → repo `IvDev19/skill-dark-mode-toggle`)
2. Initialize with the skill folder as root (SKILL.md at root, plus any references/, scripts/, agents/)
3. Push to `main`
4. Note the repo URL — it will be registered in claude-skills/registry.json when skill-publish runs

Exception: `skill-creator-per` itself predates this convention and keeps its current repo name.

For the naming rationale, the exact git commands, and how the registry entry looks, see `references/publishing.md`.

---

## Packaging

Once validation passes, package the skill into a distributable `.skill` file:

```bash
python -m scripts.package_skill <path-to-skill>
```

The `.skill` file is a zip with a renamed extension. The user installs it via Settings → Skills in Claude.ai / Desktop, or via the equivalent install flow in Cowork or Claude Code.

If you have access to the `present_files` tool, present the resulting `.skill` to the user so they can download it directly.

---

## Environment-specific notes

The workflow above is written for **Claude.ai / Desktop** (no subagents, file system available). Two other environments have variations:

### Cowork

You have subagents and a file system, but typically no browser display. The workflow benefits from running test cases in parallel via subagents and using the eval viewer in static mode. See `references/cowork.md` for the adapted workflow.

### Claude Code

You have subagents, file system, and the `claude -p` CLI. This is the only environment where the full automated description optimization loop runs. You can also run blind A/B comparison between two skill versions and quantitative benchmarking with variance analysis. See `references/claude-code.md` for the full workflow.

---

## Reference files

- `references/schemas.md` — JSON structures for `evals.json`, `grading.json`, `benchmark.json`.
- `references/skill-anatomy.md` — detailed layout patterns: multi-domain skills, table of contents for large refs, etc.
- `references/description-optimization.md` — manual and automated description tuning.
- `references/advanced-eval.md` — quantitative grading, blind comparison, benchmarking with variance analysis.
- `references/cowork.md` — parallel test runs and static eval viewer.
- `references/claude-code.md` — full subagent-based loop with `claude -p`.
- `references/publishing.md` — naming convention, git commands, and registry entry for publishing a new skill repo.
- `agents/grader.md`, `agents/comparator.md`, `agents/analyzer.md` — specialized subagent instructions (read when spawning the relevant subagent in Cowork or Claude Code).

---

Before starting, add the major workflow steps to your TodoList so they don't get dropped mid-execution.
