# Issue #90: HyperHDR JSON validation (`calculate-colors`)

**GitHub:** [Shaffer-Softworks/hyperhdr-ha#90](https://github.com/Shaffer-Softworks/hyperhdr-ha/issues/90)

**Reported:** Integration v0.10.5, HyperHDR v21.0.0.0, host `hyperhdr.brunt.ca`

## Symptoms

### Home Assistant

```
WARNING [hyperhdr.client] Failed HyperHDR (hyperhdr.brunt.ca:19444) command:
{'command': '', 'error': 'Errors during message validation, please consult the HyperHDR Log.',
 'success': False, 'tan': 2}
```

Often appears immediately after sensor/light setup and a successful `Connected to HyperHDR server` line.

### HyperHDR server log

```
[JSONCLIENTCONNECTION] While validating schema against json data of 'JsonRpc@...':
[root].command: Unknown enum value (allowed values are: ["color","tunnel","smoothing",...,
"current-state","ledcolors",...,"instance",...])
```

`calculate-colors` is **not** in the allowed `command` enum on HyperHDR v21 / HyperHDR.ng.

## Root cause

1. The **average color** sensor (`HyperHDRAverageColorSensor`) called `async_get_average_color()` from `hyperhdr-py-sickkick`.
2. That library method sends JSON-RPC: `{"command": "calculate-colors", "tan": N}`.
3. HyperHDR.ng validates `command` against a fixed schema; `calculate-colors` is rejected → generic validation error in HA (`command: ""` in the error reply is normal for this failure mode).
4. `tan: 2` is typically the **second** transactional request on the per-instance client after connect (first is often `serverinfo` during `async_client_connect`).

Integration entities otherwise load; the failure is noisy but non-fatal (sensor falls back to priority/stream sources).

## Fix (integration) — superseded by #110

**File:** `custom_components/hyperhdr/sensor.py`

| Before | After (#94, incomplete) | Correct (#110) |
|--------|-------------------------|----------------|
| Always tried `async_get_average_color()` (`calculate-colors`) | Prefer `async_get_current_colors()` (`ledcolors` + `currentColors`) | Prefer `async_get_average_color()` → `current-state` / `average-color` |

`currentColors` is **not** a valid `ledcolors` subcommand. See [docs/issue-110-ledcolors-validation-stream-auth.md](issue-110-ledcolors-validation-stream-auth.md).

## Verification

1. Deploy updated `sensor.py` (or a release that includes the #110 change) and `hyperhdr-py-sickkick>=0.2.2`.
2. Reload the HyperHDR config entry (or restart Home Assistant).
3. Confirm HyperHDR log **no longer** shows `Unknown enum value` or `specific message validation` for `ledcolors`/`currentColors`/`calculate-colors`.
4. Optional HA debug:

   ```yaml
   logger:
     logs:
       hyperhdr.client: debug
   ```

   Confirm `Send to server` lines use `current-state` / `average-color`, not `calculate-colors` or `currentColors`.

## Related

- Dependency: `hyperhdr-py-sickkick>=0.2.2` implements `current-state` / `average-color`.
- Follow-up: [#110](https://github.com/Shaffer-Softworks/hyperhdr-ha/issues/110) corrected the #94 interim approach.

## Investigation notes (2026-04-21)

- HA connects twice: root client (`raw_connection=True`) + per-instance client.
- Light `HyperHDR full state update` debug line is unrelated to the failed RPC; it reflects `serverinfo` / subscription updates already received.
- Allowed commands on reporter’s server include `ledcolors` and `current-state` but not `calculate-colors`.
