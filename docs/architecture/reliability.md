---
tags:
  - architecture
---

# Reliability & Autonomy

Three small, related subsystems in `missy/agent/` keep the agent loop resilient and enable it to act without being asked: `FailureTracker` detects when a tool is stuck in a failure loop and nudges the model toward a different approach; `Watchdog` runs background health checks on other subsystems and reports degradation; `ProactiveManager` autonomously initiates agent runs from file-change, resource-threshold, and schedule triggers. They are grouped on one page because each is a single-purpose class with no internal state machine complex enough to warrant its own page — see [Circuit Breaker](circuit-breaker.md) for the more involved failure-isolation state machine these complement.

## Failure Tracker

The `FailureTracker` class (`missy/agent/failure_tracker.py`) counts **consecutive** failures per tool name. After `threshold` (default 3) consecutive failures of the *same* tool, it signals that a strategy-rotation prompt should be injected — asking the model to stop repeating the failing approach and try something else.

```python
from missy.agent.failure_tracker import FailureTracker

tracker = FailureTracker(threshold=3)

should_inject = tracker.record_failure("shell_exec", "permission denied")
# should_inject becomes True once consecutive failures reach 3

if should_inject:
    prompt = tracker.get_strategy_prompt("shell_exec", "permission denied")
    # "The tool 'shell_exec' has failed 3 times consecutively. Last error: ...
    #  Before continuing, please:
    #  1. Analyse why this tool keeps failing
    #  2. List 3 alternative approaches that do not use 'shell_exec'
    #  3. Execute the best alternative
    #  Do not attempt 'shell_exec' again until you have tried an alternative."

tracker.record_success("shell_exec")  # resets the consecutive counter to 0
```

`record_success()` only resets the *consecutive* counter — the lifetime `total` failure count for that tool is preserved for statistics via `get_stats()`. A fresh tracker should be created per top-level task (`reset_all()`) so failure counts from an earlier, unrelated request don't bleed into a new one.

### Integration with the Runtime

`AgentRuntime` (`missy/agent/runtime.py`) instantiates one `FailureTracker(threshold=3)` per run. Inside the tool-call loop, every failed tool result calls `failure_tracker.record_failure(tc.name, tr.content)`; when it returns `True`, the runtime calls `get_strategy_prompt()` and injects the result as a user-role message into the loop's message history before the next provider call, forcing the model to explicitly reconsider its approach rather than retrying the same failing tool indefinitely.

## Watchdog

The `Watchdog` class (`missy/agent/watchdog.py`) is a generic background health monitor: callers `register(name, check_fn)` a zero-argument callable that returns `True`/`False` (or raises), and the watchdog polls all registered checks on a fixed interval from a daemon thread.

```python
from missy.agent.watchdog import Watchdog

wd = Watchdog(check_interval=60.0, failure_threshold=3)
wd.register("memory_store", lambda: memory_store.ping())
wd.register("provider", lambda: provider.health_check())
wd.start()

# ... later ...
report = wd.get_report()
# {"memory_store": {"healthy": True, "consecutive_failures": 0, "last_error": ""}, ...}

wd.stop()
```

Each subsystem's state is tracked as a `SubsystemHealth` dataclass (`name`, `healthy`, `consecutive_failures`, `last_checked`, `last_error`). A check that raises an exception, or returns `False`, marks the subsystem unhealthy and increments `consecutive_failures`; a passing check afterward resets it to healthy and logs recovery. Every check publishes an `AuditEvent` (`event_type="watchdog.health_check"`, category `"plugin"`) on `missy.core.events.event_bus` regardless of outcome, and a healthy→unhealthy transition that reaches `failure_threshold` (default 3) is logged at `ERROR` instead of `WARNING`.

`Watchdog` is a standalone utility class — it is not currently instantiated by `AgentRuntime` itself. It is meant to be created and populated with checks by whatever process hosts the agent long-term (e.g. a service-mode process), which decides which subsystems matter enough to watch.

## Proactive Manager

The `ProactiveManager` class (`missy/agent/proactive.py`) lets Missy start agent runs on its own, driven by configured `ProactiveTrigger`s rather than direct user input. Four trigger types are supported:

| `trigger_type` | Fires when |
|---|---|
| `file_change` | A watched path is modified (via the `watchdog` PyPI package's `Observer` + `PatternMatchingEventHandler`; requires `pip install watchdog`, disabled with a warning if absent) |
| `disk_threshold` | `shutil.disk_usage(disk_path).percent` exceeds `disk_threshold_pct` (default 90.0) |
| `load_threshold` | The 1-minute load average divided by `os.cpu_count()` exceeds `load_threshold` (default 4.0) |
| `schedule` | A repeating interval of `interval_seconds` elapses (default 300) |

```python
from missy.agent.proactive import ProactiveManager, ProactiveTrigger

triggers = [
    ProactiveTrigger(
        name="watch-logs",
        trigger_type="file_change",
        watch_path="/var/log/app",
        watch_patterns=["*.log"],
        prompt_template="A log file changed in /var/log/app at {timestamp}.",
        cooldown_seconds=60,
    ),
    ProactiveTrigger(
        name="disk-full",
        trigger_type="disk_threshold",
        disk_path="/",
        disk_threshold_pct=90.0,
        requires_confirmation=True,
        prompt_template="Disk usage on / has exceeded 90%. Investigate and clean up.",
    ),
]

manager = ProactiveManager(triggers=triggers, agent_callback=my_agent_fn)
manager.start()
# ... later ...
manager.stop()
```

`_fire_trigger()` runs a fixed sequence on every trigger event: check `cooldown_seconds` since the trigger's last fire (default 300s; fires within the cooldown window are silently dropped); build the synthetic prompt from `prompt_template` (supports both `${var}`-style and legacy `{var}`-style substitution of `trigger_name`, `trigger_type`, `timestamp` via `string.Template.safe_substitute`, so a malformed template can't raise); if `requires_confirmation=True`, gate the firing through an `ApprovalGate.request()` call (skipped with an audit `deny` event if no `ApprovalGate` was provided); publish an `agent.proactive.trigger_fired` audit event; then invoke `agent_callback(prompt, session_id)`, where `session_id` is `f"proactive-{trigger.name}"`. Threshold triggers share one polling thread (interval clamped to 5–30s); each schedule trigger and each file-change watch runs on its own thread/observer.

### Integration with the Runtime

`ProactiveManager` is wired up by the `missy gateway start` CLI command (`missy/cli/main.py`), not by `AgentRuntime` directly. When `cfg.proactive.enabled` and `cfg.proactive.triggers` are set (via the `proactive:` config section, parsed into `ProactiveConfig`/`ProactiveTriggerConfig` in `missy/config/settings.py`), the gateway command builds a lightweight `AgentRuntime` and passes `runtime.run(prompt, session_id=session_id)` as the `agent_callback` — so a fired trigger runs a full agent turn, complete with tool use, exactly as if a user had typed the synthetic prompt.

```yaml
proactive:
  enabled: true
  triggers:
    - name: watch-logs
      trigger_type: file_change
      watch_path: /var/log/app
      watch_patterns: ["*.log"]
      prompt_template: "A log file changed in /var/log/app at {timestamp}."
      cooldown_seconds: 60
```

## Related

- [Circuit Breaker](circuit-breaker.md) — the state-machine-based failure isolation these subsystems complement; failure tracking operates *within* the tool loop, the circuit breaker guards *provider* calls
- [Agent Runtime](agent-runtime.md) — orchestrates `FailureTracker` per run and hosts the `AgentRuntime` instance `ProactiveManager` drives
- [Vision Subsystem](vision.md) — `VisionHealthMonitor` implements a similar success/failure-rate health assessment pattern, specialized for camera devices
