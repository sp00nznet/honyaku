# honyaku

> **Automated Japanese → English translation for 2D console games, on top of static recompilation.**
> An LLM plays the game, the runtime captures every line of text with the exact context it appeared in, the script gets translated and reviewed, and English goes back in as a live overlay — no ROM patching, no pointer tables, no 8-character name limits.

**Status: planning. No code yet.** This README is the plan and the progress tracker.

---

## Why this is worth doing

There are thousands of Japan-only 2D games with no English release and no fan translation, because fan translation is brutal manual labour: build a character table by hand, find the script in a compressed blob, reverse the pointer format, translate with no idea who's speaking, then fight to fit English into a Japanese string's byte budget.

Every one of those steps gets easier — several of them get *deleted* — when the game is already a native C program you control, which is exactly what the `recomp` toolchains produce.

The two claims this project rests on:

1. **You should never OCR the screen.** The game renders text by writing font tile indices into a tilemap. Read that data structure and you get the exact character sequence, deterministically, for free. OCR of a 160×144 4-shade framebuffer is strictly worse than reading the thing that produced it.
2. **You should never patch the ROM.** The recompiled game is C. Intercept the tilemap write, suppress the Japanese glyphs, and draw English into the framebuffer yourself with a proportional font. All the classic constraints — pointer tables, DTE/dictionary compression, bank space, fixed-width fields — belong to the ROM-patching workflow, and we simply aren't in it.

Together those turn per-game archaeology into per-game *configuration*.

---

## What's already built (this repo reuses, it doesn't rebuild)

| Piece | Where | What it gives us |
|---|---|---|
| Headless C ABI | `gb/pokemon/red/rom_bridge.c` | `create/step/read/read_range/write/set_buttons/framebuffer/snapshot/restore` over the recompiled engine |
| PyBoy-shaped shim | `gb/pokemon/bot/pyboy_shim.py` | that ABI as a drop-in PyBoy subset — existing GB tooling drives our engine unmodified |
| Coverage-guided driving | `gb/pokemon/bot/` | an RL agent already walks Red across maps on the recompiled engine |
| Control plane | [`recomp-harness-mcp`](https://github.com/sp00nznet/recomp-harness-mcp) | discover/build/recompile/run ~90 harnesses across 21 platforms |
| Ground-truth capture | `gbrecomp/tools/capture_ground_truth.py` | PyBoy execution traces, already part of the recompile flow |

`snapshot`/`restore` is the sleeper feature here. It makes the explorer a tree search instead of a series of doomed 40-minute linear runs.

---

## Pipeline

```
  JP game binary
        │
        ▼
  ┌───────────────┐   existing recomp toolchain (gbrecomp / gbarecomp / …)
  │  recompile    │
  └───────┬───────┘
          ▼
  ┌────────────────────────────────────────────────┐
  │  rom_headless.dll  — native game + tap points  │
  └───┬─────────────────────────────┬──────────────┘
      │ tilemap + VRAM reads        │ tilemap writes (suppress + redraw)
      ▼                             ▲
  ┌─────────────┐              ┌────┴────────┐
  │  CAPTURE    │              │   INJECT    │  VWF overlay into framebuffer
  │ tiles→chars │              └────┬────────┘
  └──────┬──────┘                   │
         │  string + screenshot     │  EN line + box rect
         │  + speaker + last N      │
         ▼                          │
  ┌─────────────┐   ┌────────────┐  │
  │   EXPLORE   │──►│ TRANSLATE  │──┘
  │ savestate   │   │ ctx + glos │
  │ tree, LLM   │   │ + review   │
  │ at frontier │   └─────┬──────┘
  └─────────────┘         │
         ▲                ▼
         │          ┌───────────┐
         └──────────│    QA     │  replay trace, screenshot every text event,
                    └───────────┘  vision model flags overflow / clipping / untranslated
```

Five stages. Each is independently useful and independently testable.

### 1. CAPTURE — tiles to characters

Dump unique 8×8 tiles from VRAM across a playthrough and dedupe. A game's font is a contiguous-ish run of a few hundred of them. Upscale each glyph, batch them to a vision model, get back the kana/kanji it depicts. That's the character table — the artifact fan translators spend days on — built once per game for pennies.

At runtime, text extraction is then a table lookup over the tilemap. Deterministic, exact, and costs nothing per frame. Detect a text event as a contiguous run of glyph tiles landing in the BG or window map; the run's bounding box is also the rect we'll draw English into later.

Two sources of script, and we want both:

- **Static** — relative-search the ROM with the finished table. Gets the bulk of the script and the pointer table. Complete, free, and completely context-free.
- **Dynamic** — capture at runtime. Gets *where* the line appeared, what was on screen, who was speaking, what came before, and catches strings the game composes at runtime (names, counts, branches) that no static dump will ever show as a unit.

Join them on string hash. Static gives coverage; dynamic gives the context that makes the translation good.

### 2. EXPLORE — an LLM playing as a coverage fuzzer

The agent is not trying to beat the game. Its reward is **new unique text strings discovered**.

- Maintain a frontier of savestates (that ABI is already there).
- Random/scripted input covers the overwhelming majority of frames and costs nothing — mash through overworld and dialogue.
- Invoke the LLM only when the frontier stalls: a menu it can't parse, a puzzle, a boss, a branch. It sees a screenshot and picks an action.
- Expand whichever state has yielded the most new text recently; backtrack on stalls.

Keeping the model out of the hot loop is what keeps the token bill in the tens of dollars per game instead of the thousands.

### 3. TRANSLATE — context is the whole game

Model choice matters far less than what you hand it. Per string:

- the screenshot from the moment it appeared,
- the preceding N lines of the same conversation,
- a speaker guess (from portrait tiles / the name box / scene),
- the locked glossary of character names, place names, items, and recurring terms,
- **the box dimensions**, so it writes to a real length budget instead of being truncated later.

Then a second pass: a Japanese-fluent review that sees the source, the draft, and the same context, and fixes register, honorifics, and gendered speech. Glossary conflicts get **flagged, not silently resolved** — the one thing that reliably wrecks a long script is the same character being named three different things in act three.

### 4. INJECT — English as a live overlay

Hook the tilemap write path in the runtime. When a run of glyph tiles matching a known string lands on screen: suppress it, and render the English line into the framebuffer with a variable-width proportional font at that rect. Word wrapping, line breaks, and growing the box are all ours.

Consequences worth being explicit about, because they're the reason to do it this way:

- Text compression (DTE, dictionary, Huffman) becomes irrelevant — we read the decompressed output.
- Pointer tables become irrelevant — we never rewrite the ROM.
- Length limits become irrelevant — the box is ours.
- A proportional font fits roughly twice the English of the original fixed-width cells.

A real ROM-patch export is a separate backend for people who want a `.ips`. It is explicitly **not** on the critical path.

### 5. QA — automated, visual, unattended

Replay the explorer's recorded input trace against the translated build. Screenshot every text event. A vision model checks each one for overflow, clipping, mojibake, or Japanese that never got replaced. This is the step that lets the whole thing run overnight without a human watching.

---

## The eval harness (build this first)

Pick a game that shipped in **both** Japanese and English. Translate the JP version with the pipeline, diff against the official English script.

That is an objective quality score with zero human labelling, and it regression-tests every prompt and model change. Gen-1 Pokémon is the obvious first target: the JP and English versions both exist, the English script is ground truth, and Red already boots headless on this toolchain with an agent driving it.

Without this, "is the translation good?" is an argument. With it, it's a number.

---

## Phasing

Deliberately narrow. One console, one game, end to end, before any abstraction exists.

| Phase | Deliverable | Done when |
|---|---|---|
| **P0** | Lift `rom_bridge.c` out of `pokemon/red/` into `gbrecomp` as a generic headless target | any GB harness builds `rom_headless.dll` without per-game edits |
| **P1** | GB vertical slice: capture → translate → inject, one JP game | English text renders in-game, scored against the official EN script |
| **P2** | The explorer | unattended run discovers >90% of the strings a static dump finds |
| **P3** | Second console (GBA, via `gbarecomp`) | the adapter seam gets designed *here*, not before — two implementations, then the interface |
| **P4** | QA loop + optional ROM-patch export | overnight run produces a reviewed script and a flagged-issues list |

Everything up to P3 is a single console with hardcoded specifics. That is on purpose: the abstraction that survives is the one written after the second implementation, not the one guessed at before the first.

---

## Layout (planned)

```
honyaku/
├── core/            console-agnostic
│   ├── bridge.py    ctypes over the headless ABI
│   ├── tiles.py     VRAM tile + tilemap readers
│   ├── glyphs.py    unique-glyph dedupe, vision-LLM table building
│   ├── capture.py   tilemap → string events
│   ├── explore.py   savestate-tree coverage explorer
│   ├── translate.py context assembly, glossary, review pass
│   ├── render.py    VWF overlay renderer
│   └── qa.py        replay + visual check
├── adapters/        per-console specifics (exists only once P3 forces it)
├── eval/            JP→EN scoring against official localisations
└── work/            per-game data — gitignored, never committed
    └── <game-id>/   game.toml, table.json, script.jsonl, tl.jsonl
```

---

## Known hard parts

Listed because they're the parts that decide whether this works, not the parts that are fun.

- **Kanji at 8×8 are genuinely ambiguous.** Mitigated by the fact that Game Boy JP games are overwhelmingly kana-only. Platforms that use kanji (SNES, GBA, PS1) draw at 12×12 or larger, where recognition is fine. The awkward middle is small.
- **Text drawn as sprites, not BG tiles.** Some games put dialogue in OAM. Capture has to scan OAM too, not just the tilemaps.
- **Bitmap-mode text.** GBA modes 3/4 and most 3D-era games composite text into a bitmap with no tile indices to read. The tilemap approach does not apply and those need a different capture path. Out of scope; noting it so nobody is surprised at P3.
- **Knowing when a text box is *done*.** Text types out character by character. The capture layer needs to debounce on a stable frame rather than emitting a partial string per frame.
- **Runtime-composed strings.** `<PLAYER> obtained <ITEM>!` captures as one flat string with the substitutions already made. Detect the template by clustering near-identical captures and re-slotting the variable spans, or the translation memory fills up with thousands of near-duplicates.
- **Glossary drift over a long script.** Handled by locking terms on first use and flagging conflicts for review rather than letting the model re-decide.

---

## Progress

| Date | Note |
|---|---|
| 2026-09-03 | Repo created. Surveyed the collection; plan written. Nothing built yet — next up is P0. |

---

## Legal

Tools only. **No ROMs, no extracted scripts, no translated scripts, and no glyph tables are committed** — all per-game data lives under `work/`, which is gitignored. Bring your own legally obtained game. Game content is © its respective rightsholders.

## License

MIT.
