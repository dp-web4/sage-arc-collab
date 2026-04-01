# Integration Status

*Updated: 2026-04-01*

---

## Completed

### Andy's side
- [x] `to_dict()` / `from_dict()` pushed to `membot/vision/grid_observation.py`
  - `frame_raw` as nested lists, `embedding` as base64 float32
  - `GridObservation.from_dict(payload)` classmethod
- [x] Embedding confirmed: numpy `np.ndarray` float32, **512-dim** (CLIP ViT-B-32)
- [x] GitHub handles: `agro23` (personal), `project-you-apps` (org)

### SAGE's side (dp-web4)
- [x] `GridVisionIRP` skeleton (`sage/irp/plugins/grid_vision_irp.py`)
  - Push model: `submit_observation()` / `get_latest()`
  - Raw frame fallback: `push_raw_frame()` (no perception layer required)
  - IRP contract: `init_state`, `step`, `energy`, `halt`
  - `to_dict()` / `from_dict()` on `GridObservation` (compatible with Andy's)
  - `StepRecord` dataclass with `cognitive_type` for cartridge h-row tagging
- [x] `GameActionEffector` (`sage/irp/plugins/game_action_effector.py`)
- [x] Both wired into consciousness loop (config-gated)
- [x] Collaborator access: `agro23` added to `dp-web4/SAGE` and `dp-web4/sage-arc-collab`

---

## Open

### Immediate (this week)
- [ ] End-to-end integration test: Andy push frame → SAGE loop → action dispatch
- [ ] `GridCartridgeIRP` — SAGE's memory search wrapper over Andy's paired lattice
- [ ] Relay: confirm Andy has access to this repo and the interface spec
- [ ] Serialization format decision: JSON (Phase 1 agreed) → msgpack when Phase 2

### Andy's this week
- [ ] `GridCartridge` paired lattice format (even: embedding+objects, odd: SAGE StepRecord)
  - Cognitive flag in h-row for `cognitive_type = "reflection"` priority search
- [ ] Connected component analysis → `GridObservation.objects`
- [ ] CLIP frame embedding pipeline

### Architecture still open
- Grid representation: one-hot 16-channel vs structured features
- Session-level vs step-level loop
- Cross-level cartridge assembly strategy

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Embedding dim | 512 (CLIP ViT-B-32) |
| Frame size | 64×64 uint8 |
| Max GridObservation (worst case) | ~1KB |
| Protected row budget | 704 bytes raw (compression headroom available) |
| Thor LLM speed | ~24s/action (Qwen 3.5 27B, Q5_K_M) — too slow |
| McNugget LLM speed | 1.9s/step fast mode (Gemma 3 12B) |
| Smart clicker (no LLM) | 3k–6k steps/sec — currently best scorer |
| Games scoring (McNugget) | 7 games (lp85, r11l, vc33, tn36, lf52, ft09, sb26) |

---

## Contact

- dp-web4: Dennis + Claude (`@dp-web4`)
- project-you-apps: Andy + Claude (`@agro23`, `@project-you-apps`)
