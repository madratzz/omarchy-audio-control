# Local fork notes (madratzz)

This fork carries one UI tweak: gentler scroll-wheel volume steps on the mixer
slider. Scrolling now changes volume by a fifth of a drag step per wheel notch
(120 units), and high-resolution wheels scale in proportion.

## Where the edit lives

Only in the copy the shell actually loads:

```
runtime/<buildId>/qml/components/AudioSlider.qml
```

**Do not edit the top-level `qml/` copy.**

## Why the top-level `qml/` copy must stay untouched

- `packaging/build-release.py` `source_id()` computes a fingerprint over the
  top-level `qml/`, `backend/src`, `scripts/`, and the packaging files.
  `runtime/` is not part of it.
- `packaging/launch-backend.py` uses the bundled `bin/omarchy-audio-service`
  only when that fingerprint matches the binary's build ID. Otherwise it tries
  to build a new backend with `cargo`.
  - With no Rust installed, the panel shows: *"This CPU needs a native audio
    backend…"*
  - With Rust installed, the build succeeds, but the panel still hangs on
    "Preparing audio service…". The rebuilt backend listens on
    `$XDG_RUNTIME_DIR/omarchy-audio-control/backend-<new id>.sock`, while the
    panel has the shipped ID hard-coded in
    `runtime/<buildId>/qml/core/ReleasePaths.js` and looks for
    `backend-<shipped id>.sock` (`QLocalSocket::ServerNotFoundError`).

Keeping the top-level `qml/` identical to upstream preserves the fingerprint,
so the bundled backend is used and the socket names match. Rust is not needed.

## After pulling upstream updates

An update replaces `runtime/` with a new build ID directory, so reapply the
tweak to the new `runtime/<id>/qml/components/AudioSlider.qml`. In `onWheel`:

```qml
// Scale by notches (120 units) and use a fifth of a drag step, so high-resolution wheels stay gentle.
var delta = wheel.angleDelta.y / 120 * root.step * 0.2
if (delta === 0) return
```

replacing:

```qml
var delta = wheel.angleDelta.y > 0 ? root.step : -root.step
```

Then run `omarchy restart shell`. If a stale backend from a mismatched build
is still running, stop it (`pkill -x omarchy-audio-s`) before restarting.
