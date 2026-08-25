# Issue #110: `ledcolors` validation + stream auth warnings

**GitHub:** [Shaffer-Softworks/hyperhdr-ha#110](https://github.com/Shaffer-Softworks/hyperhdr-ha/issues/110)

**Reported:** Integration ~1.0.0 / 0.10.x, HyperHDR 22.0.0beta2

## Symptoms

1. Repeated HA warnings:
   `Failed HyperHDR ... command: {'command': 'ledcolors', 'error': 'Errors during specific message validation...', 'success': False}`
2. Stream warnings about admin password / missing token (until password reset).
3. Light control still worked; LED Colors camera reported disabled.

## Root causes

### A. Average color RPC (regression from #90 / #94)

[#94](https://github.com/Shaffer-Softworks/hyperhdr-ha/pull/94) switched the Average Color sensor from `calculate-colors` to `ledcolors` + `currentColors`. HyperHDR’s `schema-ledcolors.json` only allows `ledstream-*`, `imagestream-*`, and `testled` — **not** `currentColors`.

Correct API (`schema-current-state.json`):

```json
{"command": "current-state", "subcommand": "average-color", "instance": N}
```

Reply `info` is `{red, green, blue}` from `HyperHdrInstance::getAverageColor()`.

The sensor still called this RPC on every priorities-update even when cameras were disabled.

### B. LED stream auth

- Cameras opened WebSockets eagerly in `async_added_to_hass`.
- On auth failure, `hyperhdr.stream` reconnect-looped forever.
- Warning text referenced `HYPERHDR_ADMIN_PASSWORD` (debug env) instead of HA Options.
- HyperHDR requires admin password length ≥ 8 (`hyperhdr` default is exactly 8).
- `imagestream-start` (LED Colors) needs admin auth; app tokens alone are insufficient.

## Fix

| Layer | Change |
|-------|--------|
| `hyperhdr-py-sickkick` 0.2.2 | `async_get_average_color` → `current-state`/`average-color`; stop auth-fail reconnect; clearer warnings |
| HA sensor | Prefer fixed `async_get_average_color`; parse `red`/`green`/`blue`; coalesce update tasks |
| HA camera | Lazy-start stream on first image/MJPEG request |
| HA options | Reject passwords shorter than 8; empty field clears stored password |

## Verification

1. Reload integration; no `ledcolors` / `currentColors` / `calculate-colors` validation warnings.
2. Debug: outbound RPC is `current-state` + `average-color` + `instance`.
3. Average Color updates with cameras disabled/unused.
4. Wrong admin password → one clear warning, no reconnect spam.
5. Correct password (≥8) + enable LED Colors → stream works when viewed.
6. Options reject short passwords.

## Related

- [#90](https://github.com/Shaffer-Softworks/hyperhdr-ha/issues/90) / [#94](https://github.com/Shaffer-Softworks/hyperhdr-ha/pull/94) — incomplete average-color fix
- [#104](https://github.com/Shaffer-Softworks/hyperhdr-ha/issues/104) — unrelated HTTP-on-19444; confirms correct average-color RPC shape
- Library: [Shaffer-Softworks/hyperhdr-py#12](https://github.com/Shaffer-Softworks/hyperhdr-py/pull/12)
