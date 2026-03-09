---
name: capturing-preferences
description: Captures user behavioral corrections as persistent rules, writing to CLAUDE.md or installed skills. Activates on 'always X', 'never Y', 'remember...', 'from now on...', or when the same correction repeats 3 times. Asks scope and destination before writing.
---

STARTER_CHARACTER = 🧠

Detects behavioral corrections from the user and persists them as durable rules so they never need repeating.

When this skill activates, inform the user:
> "🧠 I'm tracking this as a potential standing preference. If you correct me on this 3 times, I'll propose writing it as a permanent rule."

## Detection

**Explicit triggers** — activate immediately on first occurrence:
- "remember that...", "remember to..."
- "always do X", "never do Y"
- "from now on...", "in the future..."
- "make sure you always...", "please stop..."

**Implicit triggers** — activate on the 3rd repetition of the same correction:
Apply the rule of three: first time = act on it, second time = track it, third time = abstract it.
Track pending corrections in `.claude/corrections-pending.md` (create if it doesn't exist; add to `.gitignore` if the project uses git).

Format for tracking:
```
- [correction summary] | count: 2 | last seen: [date]
```

On the 3rd occurrence, surface the pattern:
> "I've noticed you've corrected me on [X] 3 times. This seems like a standing preference worth persisting."

## Confirmation Workflow

Before writing anything, always go through these steps in order:

**1. Extract the rule**
Distill the correction into a clear, imperative statement. Show it to the user:
> "The rule I'd capture is: [imperative statement]. Does that sound right?"

**2. Ask scope**
> "Should this apply just to you in this session, to this project (shared via git), or globally to all your Claude Code sessions?"
- Personal/local only → `.claude/corrections-pending.md` promoted to CLAUDE.md locally, not committed
- Project (shared) → `.claude/CLAUDE.md` committed to git
- Global → `~/.claude/CLAUDE.md`

**3. Review (project or global scope only)**

If scope is project or global, run a peer review before proceeding. Spawn a subagent with this prompt:

> "You are a critical senior engineer reviewing a proposed rule to be added to a shared configuration. Rule: '[rule]'. Installed skills context: [list installed skills and their descriptions]. Evaluate: Does this rule conflict with any installed skill? Does it contradict established engineering principles? Is it too broad, too narrow, or ambiguously worded? Return: a verdict (looks good / concerns found), a brief explanation, and if concerns exist, a suggested reformulation."

Present the result to the user:
> "🔍 Peer review result: [verdict]
> [explanation]
> [suggested reformulation if any]
> Do you want to proceed as-is, adopt the suggested reformulation, or cancel?"

For project scope, also add after writing:
> "💡 Since this goes into a shared file, consider opening a PR or discussing it with your team before merging."

**4. Ask destination**
Check for installed skills that are topically relevant:
- Global skills: `~/.claude/skills/`
- Project skills: `.claude/skills/`

Offer options:
> "Where should I write this?
> 1. CLAUDE.md ([project/global])
> 2. [Skill name] skill — [reason it's relevant]
> 3. Both"

**5. Confirm before writing**
Show exactly what will be written and where:
> "I'll add this to `[file path]`:
> ```
> [exact content]
> ```
> Confirm? (yes/no)"

Only write after explicit confirmation.

## Writing Format

**In CLAUDE.md** — add or append to a `## Preferences` section:
```markdown
## Preferences

- Always make small, focused commits (one concern per commit)
- Never modify config files without asking first
```

**In an existing skill** — append to the most relevant section as a constraint or note:
```markdown
## Constraints

- Always make small commits — user preference
```

**After writing** — remove the entry from `corrections-pending.md` if it was tracked there.

## What NOT to Do

- Don't write without explicit confirmation
- Don't merge unrelated corrections into one rule
- Don't invent destination files — only write to files that exist or CLAUDE.md
- Don't track corrections that are task-specific (only standing preferences)
