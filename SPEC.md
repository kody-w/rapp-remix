# `rapp-remix/1.0`

A **remix** is a recording that declares more than one way to be replayed. A player picks
the highest tier the current environment supports, and must say which one it picked.

```jsonc
{
  "schema": "rapp-remix/1.0",
  "id": "my-demo",
  "title": "What I built",
  "tiers": {
    "transcript": { /* tier 3 - see below */ },
    "scene":      { /* tier 2 */ },
    "video":      { /* tier 1 */ }
  }
}
```

A remix MUST declare at least one tier. A remix SHOULD declare `video`, because it is the
only tier that always plays.

---

## Resolution

A player resolves top-down and stops at the first tier it can actually run:

```
transcript -> scene -> video
```

"Can actually run" means **probed**, not assumed:

| Tier | Probe |
|---|---|
| `transcript` | `GET {engine}/health` returns 200 within the timeout |
| `scene` | `HEAD {app}` returns 2xx **and** the iframe document is reachable (same origin) |
| `video` | at least one `source` whose `canPlayType()` is not `""` |

Two rules that exist because breaking them produces demos that lie:

1. **A player MUST surface the resolved tier and the reason.** Silent downgrade is
   forbidden. "Recorded - no engine at localhost:7071" is a valid reason; showing recorded
   output styled as live is not.
2. **A player MUST NOT fabricate a tier it could not probe.** If the engine did not answer,
   it did not answer.

---

## Tier 3 - `transcript`

The session itself. Replaying it re-runs the engine.

```jsonc
{
  "schema": "rapp-remix-transcript/1.0",
  "engine": {
    "kind": "rapp-brainstem",
    "version": "0.6.16",
    "model": "claude-haiku-4.5",
    "endpoint": "POST /chat"
  },
  "replay": {
    "live":     { "url": "http://localhost:7071/chat",
                  "field": "user_input",
                  "history": "conversation_history" },
    "recorded": { "note": "played back on captured latencies" }
  },
  "turns": [
    {
      "i": 0,
      "user":   "Remember this fact: ...",
      "reply":  "...what the engine said WHEN CAPTURED...",
      "agents": ["ManageMemory"],
      "logs":   "[ManageMemory] Successfully stored fact memory...",
      "ms":     5236
    }
  ]
}
```

### Live replay is not reproduction

Each turn's `user` string is sent again, in order, with `conversation_history` accumulated
from **this** run - not from the recording. The engine is stateful, so replies will differ.

**A differing reply is a result, not an error.** In the reference capture, turn 0 stores a
fact; replayed, the engine answers *"I've already got that one!"* - which is the correct
behaviour and is precisely the information a video cannot carry.

A player SHOULD show recorded and live side by side. That turns any transcript into a
regression test for free.

### `ms` is a measurement

`ms` is real wall-clock latency at capture. In recorded mode a player SHOULD honour it, so
the replay feels like the thing rather than like a text dump. It is also how you notice the
engine got slower.

### Authorisation

A RAPP brainstem refuses cross-origin browser writes without `X-Brainstem-Secret`
(`brainstem.py`, `_reject_cross_origin_unsafe_request`). This is CSRF protection and is
correct. A player MUST:

- keep the secret in `localStorage` on the user's own device;
- send it only to the configured engine origin;
- never place it in a URL, a query string, or a committed file.

A remix that ships a secret is a leaked credential, not a demo.

---

## Tier 2 - `scene`

Drive the real application. This is
[`rapp-vision-live/1.0`](https://github.com/kody-w/rapp-vision); the short version:

```jsonc
{
  "app": "../localFirstTools/apex-driving-simulator.html",
  "ready": { "selector": "#startBtn" },
  "actions": [
    { "at": 0.2, "do": "click",   "selector": "#startBtn" },
    { "at": 1.2, "do": "keydown", "code": "KeyW" }
  ]
}
```

Four rules, each of which was learned by watching a replay fail:

1. **`at` is measured from `ready`, not from scene start.** Boot time varies with the
   machine; measured here it moved 3-6s between runs, which was the difference between a
   clean lap and a barrier.
2. **`ready` means clickable, not visible.** A control rendered `disabled` has a bounding
   box and swallows events.
3. **Address controls by `selector`, not label,** when the label mutates. A camera button
   that renames itself Chase/Cockpit/Hood cannot be found by text twice.
4. **Do not script anything integrated per frame.** A physics sim reached 11.7 m/s on one
   run and 24.7 m/s on the next from an identical script. Ask for what is robust to timing.

---

## Tier 1 - `video`

```jsonc
{
  "sources": [
    { "src": "media/demo.webm", "type": "video/webm" },
    { "src": "media/demo.mp4",  "type": "video/mp4"  }
  ],
  "poster": "thumbs/demo.jpg",
  "chapters": [ { "t": 0, "label": "Intro" } ]
}
```

**Always ship WebM alongside MP4.** Not only for coverage: headless Chromium has no H.264
decoder, so the WebM is the one that makes your artifact verifiable in CI.

---

## Conformance

A conforming player:

- probes before resolving, and never assumes a tier;
- reports the resolved tier and the reason, visibly;
- honours `ms` in recorded mode;
- keeps engine credentials on-device;
- degrades rather than fails.

MIT © Kody Wildfeuer.
