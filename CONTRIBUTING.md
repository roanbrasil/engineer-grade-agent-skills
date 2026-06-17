# Contributing

## Skill structure

Every skill is a directory under `skills/` with at minimum a `SKILL.md` file:

```
skills/
└── my-skill/
    ├── SKILL.md         ← required
    └── examples/        ← optional: runnable code examples
        ├── java/
        ├── kotlin/
        ├── python/
        └── rust/
```

## SKILL.md format

```markdown
---
name: my-skill
description: One-line description used to trigger this skill. Start with a verb. Be specific about WHEN it activates.
---

# My Skill

## When to Use

- Situation A
- Situation B

**When NOT to use:** Situation C (use X instead)

## Core Concepts

[Explanation with ASCII diagrams where helpful]

## [Topic 1]

[Content with code examples in multiple languages]

## [Topic 2]

...

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| ... | ... | ... |

## Checklist

- [ ] Thing 1 done
- [ ] Thing 2 done
```

## Quality bar

Every skill must have:

- [ ] Clear trigger description (when does this skill activate?)
- [ ] "When NOT to use" section
- [ ] At least 2 language examples (prefer Java/Kotlin + Python)
- [ ] At least 3 named anti-patterns with explanations
- [ ] Practical checklist at the end
- [ ] ASCII diagram for any non-trivial architecture concept

## Adding a command

Commands go in `commands/` as `<name>.md`:

```markdown
# /command-name — Short Description

What this command does in 2-3 sentences.

## Skills activated

- `skill-name` — why it's activated here

## Usage

\`\`\`
/command-name <describe what you want>
\`\`\`

## Examples

- `/command-name do thing A`
- `/command-name do thing B`
```

## Naming conventions

- Skill names: `kebab-case`, descriptive nouns or noun phrases
- Command names: action verbs where possible (`architect`, `verify`, `harden`)
- File names: always `SKILL.md` (uppercase), `commands/<name>.md` (lowercase)
