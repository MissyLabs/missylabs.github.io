---
tags:
  - cli
---

# Patches Commands

Manage prompt self-tuning patches — short guidance snippets the agent proposes and appends to its system prompt after observing repeated outcomes. Patches require operator approval before they take effect. Backed by `PromptPatchManager`, persisted at `~/.missy/patches.json`.

## missy patches list

List all prompt patches, regardless of status.

```bash
missy patches list
```

Shows a table with each patch's ID, type (`tool_usage_hint`, `error_avoidance`, `workflow_pattern`, `domain_knowledge`, `style_preference`), status (`proposed`, `approved`, `rejected`, `expired`), success rate, and a preview of its content.

## missy patches approve

Approve a proposed patch so it is injected into future system prompts.

```bash
missy patches approve PATCH_ID
```

## missy patches reject

Reject a proposed patch, discarding it.

```bash
missy patches reject PATCH_ID
```

!!! note
    Approved patches are automatically retired (moved to `expired`) if their success rate drops below 40% after at least 5 applications.
