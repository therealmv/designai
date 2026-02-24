# CLAUDE.md

This file provides guidance to AI assistants (Claude and others) working on the **DesignAI** repository.

_Last updated: 2026-02-24_

---

## Project Overview

**DesignAI** is a repository currently in its initial state — no tech stack, runtime, or source code has been added yet. As the project evolves, this file must be kept up to date with the actual architecture, conventions, and workflows in use.

- **Repository:** `therealmv/designai`
- **Active branch pattern:** `claude/<task-id>`
- **Default branch:** `master`

---

## Repository Structure

The repository is at its starting point. The only file present is this guidance document.

```
designai/
└── CLAUDE.md          # This file — guidance for AI assistants
```

> Update this section whenever files or directories are added.

---

## Development Workflow

### Branching Strategy

- All work is done on feature branches following the pattern `claude/<task-id>`.
- Example: `claude/claude-md-mm0s5e6ozny0c5y6-O1Kdq`
- Never push directly to `master` without explicit permission.
- Always create a branch locally if it does not exist before pushing.

### Commit Conventions

Use clear, descriptive commit messages in the imperative mood:

```
Add user authentication module
Fix null pointer in design renderer
Update CLAUDE.md with new project structure
```

- Keep the subject line under 72 characters.
- Add a body when the "why" is not self-evident.
- Reference issue numbers where applicable (e.g., `Fixes #42`).

### Push Procedure

```bash
git push -u origin <branch-name>
```

- Branch names must start with `claude/` for CI to accept them.
- On network failure, retry up to 4 times with exponential backoff (2 s, 4 s, 8 s, 16 s).

---

## Environment Setup

> Update this section once a package manager and runtime are chosen.

### Prerequisites

- Document required runtime versions (Node.js, Python, etc.) here.
- Document required global tools here.

### Installation

```bash
# Replace once project tech stack is confirmed
npm install
```

### Environment Variables

Create a `.env` file at the project root (never commit it). Document required keys here:

| Variable | Description | Required |
|----------|-------------|----------|
| _(none yet)_ | | |

---

## Build & Run

> Populate once build scripts are defined.

```bash
# Development server
npm run dev

# Production build
npm run build

# Run tests
npm test
```

---

## Testing

> Update once a testing framework is chosen.

- Write tests alongside source files or in a dedicated `tests/` / `__tests__/` directory.
- All tests must pass before merging.
- Aim for meaningful coverage of business logic; avoid testing implementation details.

---

## Code Style & Conventions

> Update once linters/formatters are configured.

- Follow the configured linter rules — do not disable rules without a documented reason.
- Use a consistent formatter (Prettier, Black, etc.) and run it before committing.
- Prefer explicit over implicit; favour readability over cleverness.
- Delete dead code rather than commenting it out.

### General Principles

1. **Minimal changes** — only modify what is necessary to complete a task.
2. **No over-engineering** — avoid abstractions that have only one use.
3. **No backwards-compatibility shims** — if something is unused, remove it entirely.
4. **Security first** — never commit secrets; validate all external input at system boundaries.

---

## Key Files & Directories

| Path | Purpose |
|------|---------|
| `CLAUDE.md` | AI assistant guidance (this file) |

> Add rows here as the project grows.

---

## AI Assistant Notes

- **Always read a file before editing it.**
- **Do not create files unless strictly necessary** — prefer editing existing ones.
- **Do not add comments, docstrings, or type annotations** to code you did not change.
- **Do not introduce error handling** for scenarios that cannot realistically occur.
- **Keep solutions simple** — the minimum complexity needed for the current task is correct.
- **Update this CLAUDE.md** whenever the project structure, tooling, or conventions change significantly.
