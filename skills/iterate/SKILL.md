---
name: iterate
description: Iterates on a prototype by distilling it into a minimal behavioral spec and reimplementing from scratch. Use when the user says "reimplement my prototype", "clean-room rewrite", "spec out what this does and redo it", "distill and reimplement", "start fresh from a spec", or "iterate on my prototype clean".
tags:
  - prototyping
  - refactoring
  - spec-driven
  - clean-room
---

# Prototype Distill

Extract the essential behavior from a prototype as a minimal, implementation-neutral spec, then reimplement it from scratch via a contextless subagent. The goal is to shed implementation baggage without losing intent.

## Quick Reference

| Step | Job | Output |
|------|-----|--------|
| 1 | Read the prototype | Understanding of what it does |
| 2 | Extract minimal spec | Behavioral spec with hard constraints only |
| 3 | Reset to merge-base | Clean working tree, prototype stashed |
| 4 | Reimplement via contextless subagent | Fresh implementation from spec alone |

## Process

### 1. Read the Prototype

Identify the merge-base (`git merge-base HEAD <main-branch>`) and run `git diff <merge-base>` and `git status` to find all files changed since the branch diverged — committed and uncommitted. Read them all. This is always the prototype scope.

### 2. Extract the Minimal Spec

Produce a spec that describes **what** the prototype does, not **how** it does it. The spec is the only thing the reimplementing agent will see.

**Include:**
- **Behaviors** — what the code does, expressed as inputs → outputs, preconditions → postconditions, or user-observable effects. One behavior per item. Be precise.
- **Interfaces** — external-facing contracts that must be preserved: function signatures (if callers exist outside the prototype), API routes, CLI arguments, data formats consumed or produced by third-party systems
- **Hard constraints** — non-negotiable implementation details only:
  - Third-party API contracts (specific SDK calls, required field names, auth flows)
  - Required dependencies explicitly mandated by the user or the environment
  - Specific behavior the user has called out as fixed

**Exclude everything else:**
- Internal design (classes, modules, abstractions, patterns)
- File structure and naming
- Algorithms and data structures (unless the user has specified them)
- Incidental implementation choices

**Spec format:**

```
## Behaviors
1. [Precise description of observable behavior]
2. ...

## Interfaces (if any)
- [Signature/contract that must be preserved exactly]

## Hard Constraints (if any)
- [Non-negotiable implementation detail + why it's non-negotiable]

## Non-Goals
- The reimplementation is NOT constrained by: [list of things explicitly freed]
```

State the non-goals explicitly. If the original used React, and React is not a hard constraint, say so. This is what gives the reimplementing agent permission to choose differently.

Show the spec to the user and ask for confirmation before proceeding. If the user adds details, incorporate them — but push back on any additions that encode "how" rather than "what" unless the user marks them as non-negotiable.

### 3. Reset to Merge-Base

Reset the working tree to the merge-base before reimplementing. This removes the prototype from the working tree so the reimplementing subagent cannot read it, enforcing the clean-room constraint.

1. Identify the merge-base: `git merge-base HEAD <main-branch>` (check for `main`, `master`, or `origin/HEAD`)
2. Get the current branch name: `git rev-parse --abbrev-ref HEAD`
3. Stash everything (committed and uncommitted prototype work):
   - `git add -A`
   - `git stash push --include-untracked -m "iterate/<branch> $(date '+%Y-%m-%d %H:%M')"`
   - If commits exist above merge-base: `git reset --soft <merge-base>` then stash again
4. Confirm to the user: show the stash ref so they can recover with `git stash pop` or `git stash apply stash@{0}`

### 4. Reimplement via Contextless Subagent

Launch a general-purpose subagent. Pass it **only**:
- The confirmed spec from Step 2
- The working directory path
- Whether any existing files are old prototype (and should be ignored) or are a clean slate

Do NOT pass:
- The original code or file contents
- Any description of the original implementation
- Your own reading notes from Step 1

**Subagent prompt structure:**

```
You are implementing a fresh solution from a specification. Do not look at or reference any existing implementation files unless told they are retained infrastructure.

Working directory: <path>

[Paste the confirmed spec verbatim]

Implement this from scratch. Choose your own approach — language, structure, and design are unconstrained unless listed under Hard Constraints above.
```

Report back to the user with a summary of what the subagent produced and how it differs from the original approach (if observable).

## Common Mistakes

- **Leaking "how" into the spec.** "Uses a React component tree" is how. "Renders a filterable list of items" is what. If an implementation detail isn't in Hard Constraints, it shouldn't be in the spec.
- **Skipping user confirmation on the spec.** The spec is the contract. If it's wrong, the reimplementation will be wrong. Always show it before proceeding.
- **Forgetting non-goals.** A spec without explicit non-goals leaves the subagent to infer constraints from what it sees. Name what's freed.
- **Not confirming the stash ref.** Verify the stash ref before resetting. Losing the prototype is hard to recover from if the stash failed.
- **Passing the original code to the subagent.** Even "for context" — this defeats the clean-room constraint.

## Key Principles

- **The spec describes behavior, not structure.** If the spec could be satisfied by multiple different architectures, it's doing its job.
- **Hard constraints must be justified.** "We use X" is not a justification. "We use X because the API requires it" or "because the user specified it" is.
- **The reset is mandatory.** A reimplementing agent that can see the prototype will be anchored by it, consciously or not.
- **The subagent's context window is a clean room.** What goes in shapes what comes out. Spec only.
