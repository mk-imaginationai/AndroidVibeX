# Contributing to AndroidVibeX

AndroidVibeX is a Claude Code plugin. Every contribution — whether a bug fix, new pattern, or new Android topic — must ship with the skill guidance that teaches it. Code without a corresponding skill entry is incomplete.

---

## Core principle

**New pattern added → skill updated or created.**

If you write Kotlin code demonstrating a new Android pattern, update the relevant section skill so any developer (or Claude) using this plugin learns that pattern too. This is what makes the plugin useful over time.

---

## Project structure

```
AndroidVibeX/
├── CLAUDE.md                     # Android engineering rules (auto-loaded by plugin)
├── skills/
│   ├── sections/                 # Domain skills — loaded automatically when relevant
│   │   ├── android-architecture.md
│   │   ├── android-ui.md
│   │   └── ...
│   └── agents/                   # Agent skills — invoked explicitly by developers
│       ├── android-architect.md
│       ├── android-implementer.md
│       ├── android-reviewer.md
│       └── android-debugger.md
└── package.json                  # Plugin manifest
```

### Section skills vs Agent skills

| Type | Location | Loaded | Purpose |
|---|---|---|---|
| Section | `skills/sections/` | Automatically when relevant | Teach patterns for a specific Android domain |
| Agent | `skills/agents/` | Explicitly invoked | Define a role — architect, implementer, reviewer, debugger |

---

## How to update an existing skill

**When:** You found a better pattern, fixed a guardrail, added a missing example, or corrected an error.

1. **Find the skill.** Each `skills/sections/android-*.md` maps to one domain (see table in README).
2. **Read it fully** before editing — understand where your change fits.
3. **Follow the existing format:**
   - `## Concept` — explain the why, not just the what
   - `## Implementation Patterns` — named patterns with full, runnable Kotlin examples
   - `## Guardrails` — DO / DON'T list, one line each, no hedging
   - `## References` — official Android docs links
4. **Each code example must:**
   - Be self-contained and runnable
   - Use idiomatic Kotlin (see CLAUDE.md)
   - Show the full class, not just a fragment
   - Include a comment above the class explaining *when* to use this pattern
5. **Each guardrail must:**
   - State the rule as a command ("Use X", "Never Y")
   - Include the reason in the same line ("Never `SharedFlow` for events — it replays on resubscription")
6. Open a PR. See [PR process](#pr-process).

---

## How to create a new skill

**When:** You want to add an Android domain not yet covered (e.g., Paging 3, Billing, in-app updates, ML Kit).

### Step 1 — Check if it belongs in an existing skill

Before creating a new file, check if the topic is a subsection of an existing domain:
- New Compose component → `android-ui.md`
- New Room feature → `android-storage.md`
- New coroutine pattern → `android-async.md`

If it clearly stands alone (100+ lines of unique patterns), create a new file.

### Step 2 — Copy the template

```bash
cp skills/SKILL_TEMPLATE.md skills/sections/android-<topic>.md
```

### Step 3 — Fill in the template

See `skills/SKILL_TEMPLATE.md` for the required structure. Every section is mandatory.

Key rules:
- `name:` in frontmatter must match the filename exactly (minus `.md`)
- `description:` must be one sentence stating *when* Claude should load this skill
- Concept section explains the domain and when to use each option
- Every pattern has a name, a one-line rationale, and a full Kotlin code block
- Guardrails cover the most common mistakes — not exhaustive, just critical

### Step 4 — Register the skill in `package.json`

Open `package.json` and add your skill to the `skills` array:

```json
{
  "name": "android-<topic>",
  "type": "section",
  "path": "skills/sections/android-<topic>.md"
}
```

### Step 5 — Update the README table

Add a row to the **Section skills** table in `README.md`:

```markdown
| `android-<topic>` | Brief description of what it covers |
```

### Step 6 — Test your skill

Before opening a PR, verify the skill works:

1. Install the plugin locally:
   ```bash
   claude plugin install .
   ```
2. Open a new Claude Code session in an Android project.
3. Ask Claude to implement something that should trigger your skill.
4. Check that Claude uses the patterns from your skill correctly.
5. If Claude ignores or misapplies the skill, revise the `description:` field — that's what Claude uses to decide whether to load it.

---

## How to add or update CLAUDE.md rules

`CLAUDE.md` contains project-wide Android engineering rules enforced on every task. These are unconditional constraints, not patterns.

**Add a rule when:** Violating it always produces broken or insecure code (e.g., "never use `!!`", "never store secrets in SharedPreferences").

**Do not add a rule for:** Preferences, style, or patterns that have valid exceptions.

Rules must follow the existing format:
- One line per rule
- Imperative voice ("Never use X", "Always use Y")
- Group under the existing category headings

---

## PR process

1. Fork the repo and create a branch: `feat/android-<topic>` or `fix/android-<skill>-<issue>`
2. Make changes following the guidelines above
3. Verify locally (see Step 6 above)
4. Open a PR with:
   - **Title:** `feat: add android-<topic> skill` or `fix: correct <pattern> in android-<skill>`
   - **Body:** what you added/changed and why, plus a one-line example of what Claude now does differently
5. A maintainer will review for: format compliance, Kotlin correctness, guardrail accuracy, and skill trigger accuracy

---

## Quick checklist

Before opening a PR:

- [ ] Skill follows the format: Concept → Implementation Patterns → Guardrails → References
- [ ] All code examples are complete, idiomatic Kotlin, and runnable
- [ ] Each guardrail includes the reason, not just the rule
- [ ] New skill registered in `package.json`
- [ ] README table updated (for new skills)
- [ ] Skill tested locally — Claude uses the patterns correctly
- [ ] CLAUDE.md updated if a new unconditional rule was added
