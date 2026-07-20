# perception-cascade

**Tiered perception for any time-series of frames.** Three cooperating
analysis loops — fast narrow ones below, slow wide ones above — with
attention flowing down and records flowing up. Runs entirely on local
models via [Ollama](https://ollama.com). No cloud required.

Born on a fishing vessel analyzing fish-finder echograms
([tzpro-agent](https://github.com/SuperInstance/tzpro-agent) ·
[boat-agent](https://github.com/SuperInstance/boat-agent) docs/17), but
the loops are domain-agnostic: point them at any directory of timestamped
images — sonar screens, security cams, analog gauges, greenhouse monitors —
and re-skin the prompts via environment variables.

## The loops

| Loop | Cadence | Job |
|------|---------|-----|
| **M1 "racehorse"** | 60 s | One frame + gaze → tiny note. Blinders on: small context, fast. Notes are transient unless novel. |
| **M10 "scribe"** | 10 min | Canonical frame + M1 notes → the searchable record (word-based + structured JSON). May steer M1's focus. |
| **H1 "analyst"** | 60 min + on-demand | Day's records → briefing with confidence-tagged recommendations. May steer all loops. |

**The gaze channel** (`gaze.json`) is the only downward path: any tier —
or you, from the CLI — can refocus the loops below. Priority: human > H1 > M10.

**Retention contract:** M1 notes are garbage-collected unless novel
(training ore); M10 records and H1 briefings are never GC'd; discarded
frames get one final read in the evening pass before anything is deleted.
**Nothing is deleted unread.**

## Quick start

```bash
pip install .
ollama pull gemma3:12b   # or any local vision model

export CASCADE_WORKSPACE=/data/my-frames
export CASCADE_CAPTURES=/data/my-frames/captures   # <group>/*.png [+ .json sidecars]
export CASCADE_MODEL_M1=gemma3:12b

python -m perception_cascade.daemon            # run all loops
python -m perception_cascade.minute_loop       # one M1 pass (dev)
python -m perception_cascade.hourly_loop --now # on-demand briefing
python -m perception_cascade.gaze --set "watch the upper third of frame" --ttl 3600
```

## Re-skin for your domain

Prompts are config, not code:

```bash
export CASCADE_M1_PROMPT=/prompts/greenhouse_m1.txt
export CASCADE_M10_PROMPT=/prompts/greenhouse_m10.txt
export CASCADE_H1_PROMPT=/prompts/greenhouse_h1.txt
export CASCADE_FINAL_READ_PROMPT=/prompts/greenhouse_final.txt
```

All knobs (`CASCADE_*`): workspace, captures, output dir, models per
tier, intervals, novelty threshold, token limits, inference timeout. See
`perception_cascade/config.py`.

## The "zeroclaw" constraints

- Single process, own scheduler, own heartbeat file. No external agent
  runtime. Kill-safe between frames; idempotent re-runs.
- Read-only on the captures tree; atomic writes (temp+rename) everywhere.
- Model down = queue quietly. Never crash, never invent analysis.

## Output

```
$CASCADE_OUT/
├── gaze.json              # live attention directive
├── heartbeat.json         # daemon liveness (watchdog this)
├── minute_notes/novel/    # retained M1 notes (training ore)
├── records/               # canonical M10 records (never GC'd)
├── briefings/             # H1 briefings (never GC'd)
└── logs/
```

## License

Proprietary — SuperInstance.
