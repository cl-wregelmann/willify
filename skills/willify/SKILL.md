---
name: willify
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
| 3 | (Optional) Reset to merge-base | Clean working tree, prototype stashed |
| 4 | Reimplement via contextless subagent | Fresh implementation from spec alone |

## Process

### 1. Read the Prototype

Read all files that constitute the prototype. Accept:
- Explicit file paths or globs from the user
- A directory
- "Current changes" — run `git diff HEAD` and `git status` to identify modified/added files, then read them

If the scope is ambiguous, ask the user to identify what constitutes the prototype before proceeding.

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

### 3. (Optional) Reset to Merge-Base

Ask the user whether to reset the working tree to the merge-base before reimplementing, or to keep the prototype in place.

**Why:** Removing the prototype from the working tree prevents the reimplementing subagent from reading it, which would undermine the clean-room constraint. If the prototype stays, instruct the subagent explicitly not to read the existing files.

**If resetting:**

1. Identify the merge-base: `git merge-base HEAD <main-branch>` (check for `main`, `master`, or `origin/HEAD`)
2. Stash everything (committed and uncommitted prototype work):
   - `git add -A`
   - `git stash push --include-untracked -m "prototype-distill: prototype stash $(date '+%Y-%m-%d %H:%M')"`
   - If commits exist above merge-base: `git reset --soft <merge-base>` then stash again, or use `git diff <merge-base>` to verify what's captured
3. Confirm to the user: show the stash ref so they can recover with `git stash pop` or `git stash apply stash@{0}`

**If not resetting:** Note in the subagent prompt that existing files are the old prototype and must not be used as a reference.

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
- **Resetting without confirming the stash.** Verify the stash ref before resetting. Losing the prototype is hard to recover from if the stash failed.
- **Passing the original code to the subagent.** Even "for context" — this defeats the clean-room constraint.

## Key Principles

- **The spec describes behavior, not structure.** If the spec could be satisfied by multiple different architectures, it's doing its job.
- **Hard constraints must be justified.** "We use X" is not a justification. "We use X because the API requires it" or "because the user specified it" is.
- **The reset is optional but recommended.** A reimplementing agent that can see the prototype will be anchored by it, consciously or not.
- **The subagent's context window is a clean room.** What goes in shapes what comes out. Spec only.
