---
tags:
  - cli
---

# Approvals Commands

Inspect approval requests raised by the `ApprovalGate` — Missy's human-in-the-loop confirmation mechanism for sensitive operations.

## missy approvals list

List pending approval requests.

```bash
missy approvals list
```

!!! note
    Approval requests only exist for the lifetime of a running `missy gateway start` process — the `ApprovalGate` holds pending approvals in memory, not on disk. Running `missy approvals list` from a separate, short-lived CLI invocation has no gateway session to inspect, so it currently prints an informational message rather than a live list:

    ```
    No active gateway session; approvals are processed during missy gateway start.
    ```

When a tool call triggers an approval request during a gateway run, the channel that raised it (e.g. Discord) receives a message with an approval ID and risk level, and an operator responds with `approve <id>` or `deny <id>` within the configured timeout (default 60 seconds). Unanswered requests time out and the action is treated as denied.
