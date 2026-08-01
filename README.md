# rapp-remix

**One recording. Three fidelities. Pick the highest one your machine can afford.**

A video is a recording you can only watch. A remix is a recording you can re-run.

**Live:** https://kody-w.github.io/rapp-remix/

---

## The problem

You built something and you want to show it. Today you have exactly one option: record a
video. The video is honest about what happened once, on your machine, in the past, and
nothing else. A viewer cannot pause it and take the wheel. They cannot check whether it
still works. They cannot tell the difference between a real capture and a mock-up.

So you pick between a video (watchable, dead) and a live demo (alive, fragile, needs you
present). The pattern here says: stop picking. Ship all three, in one artifact, and let
the player negotiate.

---

## Three tiers

| | Tier 1 - Video | Tier 2 - Scene | Tier 3 - Transcript |
|---|---|---|---|
| What ships | `.mp4` + `.webm` | scene JSON, ~4 KB | transcript JSON, ~8 KB |
| What runs | a decoder | the real app | the real engine |
| Fidelity | what happened once | what the app does now | what the agent decides now |
| Same answer twice? | always | usually | no, and that is the point |
| Requires | nothing | the app, same origin | a live engine and a key |
| Degrades to | - | Tier 1 | Tier 2, then Tier 1 |

Tier 3 is the interesting one. Replaying a transcript against a live brainstem does not
reproduce the recorded replies, because the engine has learned things since. In our own
capture, turn 0 stored a fact. Replay it and the agent answers:

> *"I've already got that one!"*

A video can never tell you that. A transcript that returns a different answer is not a
broken recording. It is a measurement.

---

## Why three, not one

Because each tier fails in a different place, and you do not control your viewer's machine.

- No engine reachable? Tier 3 falls back to Tier 2.
- App moved, or you are offline on a plane? Tier 2 falls back to Tier 1.
- Tier 1 always plays. It is the floor, not the goal.

The player states which tier it resolved and why, every time. A demo that silently
downgrades is lying to you.

---

## Cartridges

This is the same shape as a [`rapp-cart`](https://github.com/kody-w/rapp-carts): a small
file you hand to a runtime, which then does the work. A remix cartridge is a cart whose
payload is a *session* rather than a program. Insert it and the session runs again.

Cartridge to console. Transcript to engine. Scene to app. Video to decoder.
Same contract, four rungs of one ladder.

---

## Try Tier 3 for real

Tier 3 needs a running RAPP brainstem. Cross-origin browser calls to it are refused by
design, because it is CSRF-guarded, so you must present the per-install secret:

```bash
curl -s localhost:7071/health          # is it up?
cat ~/.brainstem/.brainstem_secret     # your key - never commit this
```

Paste that into the player's Engine panel. It is stored in your own `localStorage` and
sent to nobody but your own brainstem on your own machine.

### The hosted page cannot reach your brainstem, and says so

Measured, not assumed: from `https://kody-w.github.io/rapp-remix/` the browser blocks the
call to `http://localhost:7071` as mixed content, so the page resolves to **Tier 2** and
prints the reason. That is the pattern working, not failing.

To actually run Tier 3, serve it over plain HTTP next to your engine:

```bash
git clone https://github.com/kody-w/rapp-remix && cd rapp-remix
python3 -m http.server 8000
open http://localhost:8000
```

Then paste your secret. `/health` is secret-guarded too, not only `/chat` — the brainstem
treats any request carrying an `Origin` header as a foreign browser request, so without the
secret it returns 403, which reads like *"engine is down"* when it means *"engine is there
and does not know you."* The probe distinguishes the two.

---

## Files

| | |
|---|---|
| [`SPEC.md`](SPEC.md) | `rapp-remix/1.0` - all three tier formats |
| [`index.html`](index.html) | the player; one file, no build |
| [`transcript.json`](transcript.json) | a real capture against a real brainstem |

`transcript.json` was not written by hand. Every turn is an actual `POST /chat`, with the
real reply, the real `agent_logs`, and the real wall-clock latency. Two turns fired agents
(`ManageMemory`, and `HackerNews` over the network at 10.7s).

---

## In the ecosystem

- [rapp-vision](https://github.com/kody-w/rapp-vision) - Tier 1 and Tier 2, as a YouTube
- [rapp-carts](https://github.com/kody-w/rapp-carts) - the cartridge spec
- [rapp-rock-tumbler](https://github.com/kody-w/rapp-rock-tumbler) - how any of this gets verified

---

MIT · zero-server · offline-first · owned by [@kody-w](https://github.com/kody-w)
