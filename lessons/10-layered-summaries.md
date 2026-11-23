# Step 10: Layered summaries

You summarize notes in layers: notes → weekly → monthly → one "journey" summary. Each layer reads only the layer below, so years of notes never need to fit in one prompt.

## The idea

One prompt cannot hold years of notes (Step 2). So you summarize in a **hierarchy**: each week from its notes, each month from its weekly summaries, the journey from the monthly ones. This is simple **prompt chaining**: one LLM output becomes the next input. The risk is **drift** (errors growing at each layer), so every prompt says "only facts from the text" and "keep numbers exact".

```text
notes (365 per year)
   │ group by week, summarize each
   ▼
weekly summaries (52)
   │ group by month, summarize each
   ▼
monthly summaries (12)
   │ summarize all
   ▼
journey summary (1)
```

| Word | Simple meaning |
| --- | --- |
| Summary | A short version that keeps the important facts. |
| Hierarchy / layers | Summaries of summaries. |
| Prompt chaining | Using one LLM answer as input to the next call. |
| Drift | Small errors or additions that grow through layers. |

## The code

Uses `lib.ts` (Step 9). Make `step10.ts`.

**Piece 1: notes and grouping helpers.**

```ts
import { chat } from "./lib.ts";

const notes = [
  "2026-10-02 Guitar: started G to C changes, very slow, 10 per minute.",
  "2026-10-05 Football: juggling, best was 12 in a row.",
  "2026-10-09 MTB: first time on the rocky descent, walked the steep part.",
  "2026-10-14 Guitar: G to C at 18 changes per minute.",
  "2026-10-20 Football: juggling record 21.",
  "2026-10-25 MTB: rode the whole descent, braked too late at the bottom.",
  "2026-11-01 Guitar: barre chords buzzing on the B string.",
  "2026-11-04 Football: 50 left-foot passes against the wall.",
  "2026-11-08 Guitar: G to C at 30 changes per minute, felt easy.",
  "2026-11-12 MTB: braked earlier, much smoother on the descent.",
];

function weekStart(day: string): string {
  const d = new Date(day + "T00:00:00Z");
  const daysSinceMonday = (d.getUTCDay() + 6) % 7;
  d.setUTCDate(d.getUTCDate() - daysSinceMonday);
  return d.toISOString().slice(0, 10);
}

function groupBy<T>(items: T[], key: (item: T) => string): Map<string, T[]> {
  const groups = new Map<string, T[]>();
  for (const item of items) {
    const k = key(item);
    groups.set(k, [...(groups.get(k) ?? []), item]);
  }
  return groups;
}
```

- `weekStart` finds the Monday of a note's week (`getUTCDay()` is 0 for Sunday).

**Piece 2: one summarize function for every layer.**

```ts
async function summarize(level: string, lines: string[]): Promise<string> {
  return chat([
    {
      role: "system",
      content: `Summarize this ${level} of practice in at most 2 short sentences. Use only facts from the text. Keep numbers exact. No advice.`,
    },
    { role: "user", content: lines.join("\n") },
  ]);
}
```

**Piece 3: build the layers.**

```ts
const weekly: string[] = [];
for (const [week, items] of groupBy(notes, (n) => weekStart(n.slice(0, 10)))) {
  const s = await summarize("week", items);
  weekly.push(`${week}: ${s}`);
  console.log(`WEEK ${week}: ${s}`);
}

const monthly: string[] = [];
for (const [month, items] of groupBy(weekly, (w) => w.slice(0, 7))) {
  const s = await summarize("month", items);
  monthly.push(`${month}: ${s}`);
  console.log(`\nMONTH ${month}: ${s}`);
}

console.log("\nJOURNEY:", await summarize("whole practice journey", monthly));
```

## Run it and read the output

```text
WEEK 2026-09-28: Started G to C changes at 10 per minute.
WEEK 2026-10-05: Juggling best was 12; first rocky descent, walked the steep part.
...
MONTH 2026-09: Started G to C changes at 10 per minute.

MONTH 2026-10: Guitar changes rose from 10 to 18 per minute, juggling from 12 to 21, and the full descent was ridden.
MONTH 2026-11: G to C reached 30 per minute; braking on the descent improved.

JOURNEY: Over two months, G to C changes tripled from 10 to 30 per minute, juggling rose to 21, and descents went from walking to smooth riding.
```

- Check every number against the notes. Did anything new appear (drift)?
- "MONTH 2026-09": the week of Oct 2 starts on Monday Sep 28, and our simple rule puts a week in the month it starts in. Real apps pick a rule and stick to it.

## Try this

- Remove "Keep numbers exact. No advice." Does drift appear?
- Add a check: every number in a summary must appear in the layer below.
- Store summaries in SQLite; only redo a week when its notes change.

## Quiz

1. Why summarize in layers?
2. What is drift, and how does this code reduce it?
3. A new note arrives this week. Which summaries must be redone?

**Answers:** 1. Small calls always fit the context window. 2. Errors growing through layers; strict prompts. 3. This week's, this month's, and the journey summary.

## Your practice-video app

Your app's "session → week → month → whole journey" summaries.

## Learn more

- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) (Anthropic)
