# Step 14: Plain code vs AI

You measure loudness and motion in a practice video with FFmpeg and a little math, no AI, and learn when plain code beats a model.

## The idea

Some questions have exact, measurable answers: how loud is each second, how much does the picture change? **FFmpeg** reads almost any audio or video. Its **filters** measure **loudness** (in **decibels**) and a **scene score** (how different each frame is from the previous one, 0 to 1), a simple **motion** signal. Fast, free, exact, same answer every time. AI is for what code cannot measure.

```text
clip.mp4 ──► ffmpeg audio filters ──► loudness per second:  -41 -41 -21 -21 dB
         └─► ffmpeg video filters ──► motion per second:    0.01 0.02 0.35 0.08
                                               │
                    plain TypeScript: find the loud / busy moments
```

| Word | Simple meaning |
| --- | --- |
| FFmpeg | A free tool to read, convert and analyse audio/video. |
| Filter | An FFmpeg step that changes or measures audio/video. |
| Decibel (dB) | Loudness. For files, 0 dB is max; -60 is very quiet. |
| RMS level | Average loudness over a time window. |
| Scene score | Difference from the previous frame (0 = same, 1 = totally different). |
| Deterministic | Same input → same output. |

## The code

FFmpeg (Step 12) and a short clip `clip.mp4`.

**Quick test:** `ffmpeg -hide_banner -i clip.mp4 -af volumedetect -f null -`

Make `step14.ts`.

**Piece 1: a helper that runs FFmpeg.**

```ts
import { spawnSync } from "node:child_process";

const FILE = "clip.mp4";

function ffmpeg(args: string[]) {
  const r = spawnSync("ffmpeg", ["-hide_banner", "-nostats", "-i", FILE, ...args, "-f", "null", "-"], { encoding: "utf8" });
  return { out: r.stdout, err: r.stderr };
}
```

**Piece 2: overall loudness.**

```ts
const vol = ffmpeg(["-vn", "-af", "volumedetect"]).err;
const mean = vol.match(/mean_volume: (-?[\d.]+) dB/)?.[1];
const max = vol.match(/max_volume: (-?[\d.]+) dB/)?.[1];
console.log(`Loudness: average ${mean} dB, peak ${max} dB`);
```

**Piece 3: loudness for each second.**

```ts
const audio = ffmpeg([
  "-vn", "-af",
  "aresample=16000,asetnsamples=n=16000,astats=metadata=1:reset=1,ametadata=print:key=lavfi.astats.Overall.RMS_level:file=-",
]).out;

const loudness = [...audio.matchAll(/RMS_level=(-?[\d.]+|-inf)/g)].map((m) => (m[1] === "-inf" ? -100 : Number(m[1])));
loudness.forEach((db, s) => console.log(`second ${s}: ${db.toFixed(1)} dB ${"#".repeat(Math.max(0, Math.round((db + 60) / 3)))}`));
```

- Resample to 16,000/s, cut into 16,000-sample chunks (1 second), measure RMS, print. The last "second" may be a short leftover.

**Piece 4: motion for each second.**

```ts
const video = ffmpeg(["-an", "-vf", "select='gte(scene,0)',metadata=print:file=-"]).out;

const times = [...video.matchAll(/pts_time:([\d.]+)/g)].map((m) => Number(m[1]));
const scores = [...video.matchAll(/scene_score=([\d.]+)/g)].map((m) => Number(m[1]));

const motion: number[] = [];
scores.forEach((score, i) => {
  const second = Math.floor(times[i]);
  motion[second] = Math.max(motion[second] ?? 0, score);
});
motion.forEach((m, s) => console.log(`second ${s}: motion ${m.toFixed(3)} ${"#".repeat(Math.round(m * 50))}`));

const busiest = motion.indexOf(Math.max(...motion));
const loudest = loudness.indexOf(Math.max(...loudness));
console.log(`\nMost motion at second ${busiest}, loudest at second ${loudest}`);
```

## Run it and read the output

```text
Loudness: average -24.0 dB, peak -12.1 dB
second 0: -41.1 dB ######
second 3: -21.1 dB #############
...
second 0: motion 0.010
second 3: motion 0.350 #################
...
Most motion at second 3, loudest at second 3
```

## Try this

- Use a clip with a clear moment (a shot, a jump). Do the peaks match?
- Cut the best 5 seconds: `ffmpeg -ss 3 -t 5 -i clip.mp4 -c copy best.mp4`.
- Send only the busiest frames to the vision LLM (Step 13).

## When code beats AI

| Use plain code when... | Use AI when... |
| --- | --- |
| The answer is a measurement | The answer needs understanding |
| You need the same answer every time | Some variation is fine |
| It must be fast and free on every frame | It can be slower, on a few frames |
| Rules are clear | Rules are fuzzy |

## Quiz

1. Scene score near 0 means?
2. Two reasons to measure loudness with FFmpeg?
3. How can code and AI work together on a long video?

**Answers:** 1. Little motion. 2. Exact, free, fast, repeatable. 3. Code finds moments; AI looks only at those.

## Your practice-video app

FFmpeg finds the interesting moments before any AI runs.

## Learn more

- [FFmpeg filters](https://ffmpeg.org/ffmpeg-filters.html): `volumedetect`, `astats`, `ebur128`, `select`, `scdet`.
