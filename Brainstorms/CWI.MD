# Captions with Intention — Design Document

**Date:** 2026-03-18
**Status:** Approved

## Background

Based on the [Caption with Intention case study](https://www.captionwithintention.org/) developed in partnership with the Chicago Hearing Society, this feature extends the platform's existing caption system with three accessibility dimensions:

1. **Attribution** — per-speaker color coding so viewers immediately know who is speaking
2. **Synchronization** — word-level karaoke reveal: each word transitions to the speaker color as it is spoken
3. **Intonation** — variable word size (a multiplier) to convey volume, emphasis, and vocal nuance

---

## Approach

**JSON-first format** as the internal source of truth. SRT remains supported for import. WebVTT export available as a compatibility/portability layer. A custom renderer replaces both native `<track>` and Shaka's caption rendering, working identically for regular video and HLS streams.

---

## 1. Data Format

```json
{
  "version": "1.0",
  "speakers": [
    { "id": "s1", "name": "Alice", "color": "#E63946" },
    { "id": "s2", "name": "Bob",   "color": "#457B9D" }
  ],
  "cues": [
    {
      "id": "cue_1",
      "speakerId": "s1",
      "start": 1.24,
      "end": 4.10,
      "words": [
        { "text": "I",      "start": 1.24, "end": 1.40, "size": 1.0 },
        { "text": "really", "start": 1.40, "end": 1.80, "size": 1.4 },
        { "text": "mean",   "start": 1.82, "end": 2.10, "size": 1.8 },
        { "text": "it.",    "start": 2.12, "end": 2.50, "size": 1.0 }
      ]
    }
  ]
}
```

### Key decisions

- `speakers` array defined once at the top; cues reference by `id`
- Colors stored on the speaker object, not on individual cues
- `start`/`end` in seconds (float) at both cue and word level
- `size` is a multiplier: `1.0` = normal, `>1.0` = louder/emphasis, `<1.0` = soft
- No visual styling in the format — the renderer owns all CSS decisions
- Cue-level `start`/`end` redundant with word timestamps but used for fast seeking and fallback rendering

### SRT import result

Cues import with timing intact. `speakerId` is `null`, no word timestamps, `size: 1.0` on all words. The editor then guides the client to assign speakers and optionally add word-level timing and intonation.

---

## 2. Editor

### Upload flow (SRT import)

1. Client uploads `.srt` — timing preserved, content imported
2. Editor prompts to create speakers (name + color)
3. Client assigns cues to speakers (individually or in bulk by time range)
4. Optional: add intonation emphasis per word

Word-level timing cannot be recovered from SRT; karaoke sync is skipped unless the client manually adjusts word boundaries.

### Create-from-scratch flow

Timeline editor with:
- Video preview pane
- Waveform scrubber
- Speaker management panel (name, color, add/remove)
- Cue list with speaker assignment

Clicking a cue opens a word-level editor:
- Words displayed as chips on a mini-timeline
- Drag word boundaries to adjust timing
- Slider per word to set emphasis (size multiplier)

### Out of scope (v1)

- Auto-transcription / speech-to-text
- Font customization beyond speaker color
- Multiple speakers overlapping in a single cue

---

## 3. Renderer

A transparent overlay on top of the video element, fully custom, replacing both native `<track>` and Shaka's caption pipeline.

### Playback behavior

At each frame the renderer:
1. Finds the active cue for the current timestamp
2. Displays cue text in the speaker's color
3. Reveals words in real-time — each word starts neutral and transitions to the speaker color exactly when spoken (karaoke effect)
4. Scales word size inline using the `size` multiplier

### Graceful degradation

| Condition | Behavior |
|---|---|
| No word timestamps (SRT import) | Whole cue displays in speaker color immediately — no karaoke, but attribution and intonation still work |
| No speaker assigned | Renders in white at standard position — identical to current captions |

### HLS compatibility

The renderer is decoupled from the streaming layer. Shaka handles the HLS stream; its caption pipeline is ignored entirely. The renderer works identically for regular video and HLS.

---

## Next Steps

1. Implement the JSON format and storage schema
2. Build the custom renderer
3. Reimplement the caption editor with CWI support
4. SRT import converter
5. WebVTT export (compatibility layer)
