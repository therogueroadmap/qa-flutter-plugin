# Appium `scripts/bootstrap.py` / `scripts/teardown.py` contract — v1

This document specifies what the user-owned companion scripts `scripts/bootstrap.py` and `scripts/teardown.py` (in the qa-agent companion directory) must do so the plugin's `qa-flutter-bootstrap` skill can delegate to them safely.

**Applies to:** Appium stack (`qa-agent.yaml` with `project.android_stack: appium`).

**Versioning:** `v1` is the first contract. Previous ad-hoc usage (no `--marker` flag, `bootstrap-status.json` only) is v0 and supported as deprecated fallback for one minor release.

---

## Invocation

### bootstrap.py

```
python scripts/bootstrap.py <path_to_qa-agent.yaml> --marker=<marker_path>
```

- `<path_to_qa-agent.yaml>`: absolute path to the active yaml.
- `--marker=<marker_path>`: absolute path where the script must write the marker file.

### teardown.py

```
python scripts/teardown.py <path_to_qa-agent.yaml> --marker=<marker_path>
```

Same arguments.

---

## Configuration reading

Under v1, `bootstrap.py` reads configuration from the yaml's `autonomous.*` block (unified plugin contract):

```yaml
autonomous:
  backend:
    start_cmd: "..."
    start_cwd: "..."
    health_url: "..."
    ready_timeout_seconds: 120
    stop_cmd: "..."
    stop_timeout_seconds: 30
  device:
    boot_avd: "..."
    boot_timeout_seconds: 90
```

**Backward compatibility during deprecation window:** the script may fall back to its pre-v1 config fields (wherever those were) if `autonomous.backend` is absent, emitting a deprecation warning to stderr. At the next minor release, this fallback must be removed.

For Appium-specific configuration (Appium server port, capabilities, etc.), continue reading from `appium.*` as before — the v1 contract only unifies the backend + device part.

---

## bootstrap.py responsibilities

1. Read the yaml at `<path>`.
2. Start the backend per `autonomous.backend.start_cmd` (or legacy path during deprecation window).
3. Launch / verify the device per `autonomous.device.boot_avd`.
4. Start the Appium server and Flutter build per existing Appium flow.
5. **Write the marker file** at `<marker_path>` with this exact YAML schema:

```yaml
started_at: 2026-04-23T03:00:00Z
caller: android-runner               # or whatever the invoker told qa-flutter-bootstrap
pid_ward: <int>                      # PID of a process that lives for the entire run (backend or appium server)
backend:
  we_started_it: true                # false if found already running
  pid: <int or null>
  stop_cmd: "<stop_cmd from yaml or null>"
  stop_timeout_seconds: 30
  health_url: "<from yaml>"
device:
  id: <e.g. emulator-5554>           # ID that `adb devices` reports
  we_started_it: true                # false if the AVD was already running
  emulator_pid: <int or null>
teardown_failed: false
appium:                              # Appium-specific addendum, optional block
  server_pid: <int>
  server_port: <int>
```

Write the marker **atomically**: write to `<marker_path>.tmp` first, then rename. This prevents `qa-flutter-bootstrap` from reading a half-written marker on a race.

6. Exit 0 on success. Exit non-zero on failure with a clear stderr message.

**On failure**: do NOT leave a marker file. If some subsystems started and others failed, clean them up best-effort before exiting.

---

## teardown.py responsibilities

1. Read the yaml at `<path>`.
2. Read the marker at `<marker_path>` to know what WAS started.
3. Stop Appium server, Flutter build, device (if `device.we_started_it`), backend (if `backend.we_started_it`).
4. On success (all stops completed or safely skipped) → delete the marker file.
5. On partial failure → update the marker file in place with `teardown_failed: true`, exit non-zero.

---

## Legacy `bootstrap-status.json` — deprecated

v0 scripts wrote a `bootstrap-status.json` file to `{reports_output_dir}/bootstrap-status.json` with fields like `api_ready`, `flutter_ready`, `appium_ready`. Under v1, this file is NOT the contract. Writing it is allowed (may be useful for debugging), but the **marker file** is what `qa-flutter-bootstrap` reads.

If a v0 script is invoked with `--marker` and it ignores the flag:
- `qa-flutter-bootstrap` detects the missing marker post-exit.
- Synthesizes a minimal marker from `bootstrap-status.json` as best-effort fallback.
- Emits a deprecation warning pointing the user to this doc.

This fallback is for **one minor release only**. Update `bootstrap.py` / `teardown.py` to v1 before the next release or Appium stack will fail.

---

## Example — minimal compliant `bootstrap.py` skeleton

```python
#!/usr/bin/env python3
"""Appium stack bootstrap script — v1 contract compliant."""
import argparse, os, subprocess, sys, time, yaml

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("config_path")
    parser.add_argument("--marker", required=True)
    args = parser.parse_args()

    with open(args.config_path) as f:
        cfg = yaml.safe_load(f)

    auto = cfg.get("autonomous", {})
    backend = auto.get("backend", {})
    device = auto.get("device", {})

    marker = {
        "started_at": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "caller": "android-runner",
        "pid_ward": os.getpid(),  # refined below if backend/emulator started
        "backend": {
            "we_started_it": False,
            "pid": None,
            "stop_cmd": backend.get("stop_cmd"),
            "stop_timeout_seconds": backend.get("stop_timeout_seconds", 30),
            "health_url": backend.get("health_url"),
        },
        "device": {
            "id": None,
            "we_started_it": False,
            "emulator_pid": None,
        },
        "teardown_failed": False,
    }

    # ... start backend (update marker["backend"]) ...
    # ... start/verify device (update marker["device"]) ...
    # ... start Appium server (add marker["appium"]) ...

    # Atomic write
    tmp = args.marker + ".tmp"
    with open(tmp, "w") as f:
        yaml.safe_dump(marker, f)
    os.rename(tmp, args.marker)
    print("bootstrap.py: OK", file=sys.stderr)
    sys.exit(0)

if __name__ == "__main__":
    main()
```

---

## Testing the contract

After updating `bootstrap.py`, verify the contract with:

```bash
python scripts/bootstrap.py qa-agent.yaml --marker=/tmp/test-marker.yaml
# Expected: exit 0, /tmp/test-marker.yaml exists, YAML-parseable, schema matches above
python -c "import yaml; m = yaml.safe_load(open('/tmp/test-marker.yaml')); assert 'pid_ward' in m and 'backend' in m and 'device' in m; print('OK')"

python scripts/teardown.py qa-agent.yaml --marker=/tmp/test-marker.yaml
# Expected: exit 0, /tmp/test-marker.yaml does NOT exist
```
