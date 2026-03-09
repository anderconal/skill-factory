# Skill Factory Structure

```
skill-factory/
├── CLAUDE.md                               # Agent instructions for this project
├── update-docs                             # Fetch latest Anthropic/Claude Code skill docs
├── update-docs-copilot                     # Fetch latest GitHub Copilot skill docs
├── skills                                  # Install/uninstall skills (Claude Code + Copilot)
├── scripts/                                # Automation scripts
│   ├── sources.txt                         # URLs to fetch Claude Code docs from
│   ├── sources-copilot.txt                 # URLs to fetch Copilot docs from
│   ├── fetch_anthropic_skill_docs.py       # Fetch latest Anthropic docs
│   └── fetch_copilot_skill_docs.py         # Fetch latest Copilot docs
├── docs/                                   # All knowledge about creating skills
│   ├── knowledge/
│   │   ├── anthropic-skill-docs/           # Official Anthropic skill documentation
│   │   │   ├── overview.md                 # What skills are, why they exist, core concepts
│   │   │   ├── skills.md                   # Implementation syntax, structure, usage patterns
│   │   │   └── best-practices.md           # Proven patterns, common pitfalls, guidelines
│   │   └── copilot-skill-docs/             # Official GitHub Copilot skill documentation
│   │       ├── about-agent-skills.md       # What skills are, core concepts (≈ overview.md)
│   │       ├── create-skills.md            # Implementation syntax, structure (≈ skills.md)
│   │       └── customization-cheat-sheet.md# Options, patterns, guidelines (≈ best-practices.md)
│   ├── create_new_skill-process.md         # Instructions for creating Claude Code skills
│   ├── create_new_copilot_skill-process.md # Instructions for creating Copilot skills
│   ├── map.md                              # This file - repository structure
│   └── project.md                          # Project-specific information
└── output_skills/                          # Created skills organized by category (gitignored)
    ├── testing/                            # tdd, nullables, approval-tests, bdd-with-approvals
    ├── design/                             # hexagonal-architecture, event-modeling, collaborative-design
    ├── practices/                          # refactoring, refinement-loop
    ├── ai/                                 # ai-patterns, creating-process-files
    │   └── claude-code/                    # creating-hooks
    └── developer-tools/                    # writing-bash-scripts, using-uv, git-worktrees
```

## Purpose

- **docs/**: Contains all instructional material the agent uses to create skills
- **output_skills/**: Stores completed skills, each in a category subfolder
- **CLAUDE.md**: Provides context to the agent about this repository's purpose
