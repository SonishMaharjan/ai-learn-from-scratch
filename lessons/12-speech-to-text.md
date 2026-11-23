# Step 12: Speech to text

You record a voice note, turn it into text on your own computer with whisper.cpp, and read the time of every word.

## The idea

**Speech to text** (**ASR**) turns audio into words. **Whisper** is a free speech model from OpenAI; **whisper.cpp** runs it locally, using the GPU when it can (Metal on Apple Silicon, CUDA on NVIDIA, or CPU). Audio becomes a **spectrogram** (a picture of sound over time); Whisper's **encoder** reads it and its **decoder** writes text tokens one at a time, like an LLM. It can also give **timestamps** for each word.

```text
voice note (.m4a/.wav) ──► ffmpeg: 16 kHz mono WAV ──► whisper.cpp
                                                       │ spectrogram → encoder → decoder
                                                       ▼
                         [00:00.32 → 00:00.61] "Rode"
                         [00:00.61 → 00:00.80] "the"
                         [00:00.80 → 00:01.30] "descent" ...
```

| Word | Simple meaning |
| --- | --- |
| ASR | Turning spoken audio into text. |
| Sample rate | Sound measurements per second. Whisper expects 16,000. |
| Mono | One audio channel. |
| Spectrogram | A picture of which frequencies are loud at each moment. |
| Encoder / decoder | One part reads the input; the other writes the output. |
| Timestamp | Start and end time of a word in the audio. |

## Setup (one time)

1. **ffmpeg:** Mac `brew install ffmpeg`, Windows `winget install ffmpeg`, Linux `sudo apt install ffmpeg`.
2. **whisper.cpp:** Mac `brew install whisper-cpp` (command `whisper-cli`). Windows: prebuilt zip from the [releases page](https://github.com/ggml-org/whisper.cpp/releases). Linux: build with the [README](https://github.com/ggml-org/whisper.cpp) commands.
3. **Model** (about 142 MB, English):

```bash
mkdir -p models
curl -L -o models/ggml-base.en.bin https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin
```

4. **Record a 10–20 second voice note** with any recorder, then convert:

```bash
ffmpeg -i recording.m4a -ar 16000 -ac 1 -c:a pcm_s16le note.wav
```

On a Mac you can record directly: `ffmpeg -f avfoundation -i ":0" -t 15 -ar 16000 -ac 1 note.wav`.

**Quick test:** `whisper-cli -m models/ggml-base.en.bin -f note.wav`

## The code

Make `step12.ts`.

**Piece 1: run whisper.cpp with word-level timestamps.**

```ts
import { execFileSync } from "node:child_process";
import fs from "node:fs";

execFileSync("whisper-cli", [
  "-m", "models/ggml-base.en.bin",
  "-f", "note.wav",
  "-ml", "1",
  "-oj",
  "-of", "note",
], { stdio: "ignore" });
```

- `-ml 1`: one word per segment. `-oj`: JSON output. `-of note`: file name `note.json`. (On Windows, use the full path to `whisper-cli.exe`.)

**Piece 2: read the words and times.**

```ts
type Segment = { offsets: { from: number; to: number }; text: string };

const result = JSON.parse(fs.readFileSync("note.json", "utf8"));
const words: Segment[] = result.transcription.filter((s: Segment) => s.text.trim() !== "");

const fmt = (ms: number) => (ms / 1000).toFixed(2).padStart(6);
for (const w of words) {
  console.log(`[${fmt(w.offsets.from)}s → ${fmt(w.offsets.to)}s] ${w.text.trim()}`);
}

const fullText = words.map((w) => w.text.trim()).join(" ");
console.log("\nFull text:", fullText);
```

**Piece 3: find a moment.**

```ts
const target = "braked";
const hit = words.find((w) => w.text.toLowerCase().includes(target));
console.log(hit ? `\n"${target}" was said at ${fmt(hit.offsets.from)}s` : `\n"${target}" not found`);
```

## Run it and read the output

```text
[  0.32s →   0.61s] Rode
[  0.61s →   0.80s] the
[  0.80s →   1.30s] descent
...
Full text: Rode the descent twice and I braked too late at the bottom.

"braked" was said at   2.41s
```

## Try this

- Compare `tiny` and `small` models: speed vs accuracy.
- Record in a noisy place.
- Send `fullText` to Step 5's `extract()`.

## Quiz

1. Why convert to 16 kHz mono?
2. What does `-ml 1` give you?
3. How is Whisper's decoder like an LLM?

**Answers:** 1. Whisper expects it. 2. A timestamp for each word. 3. It writes one token at a time based on the previous ones.

## Your practice-video app

Logging videos by talking while you watch: speech to text with word times.

## Learn more

- [whisper.cpp](https://github.com/ggml-org/whisper.cpp)
