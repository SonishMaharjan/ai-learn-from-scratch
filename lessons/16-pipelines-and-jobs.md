# Step 16: Pipelines and jobs

You build a small version of your app's story drafter: plan → code searches → AI picks from numbered options → code checks → one fix round. Then you run it as a background job with a worker.

## The idea

One big prompt is hard to control. A **pipeline** splits work into small steps and lets code do what code is good at. The AI **plans** what to search for; **code searches**; code shows **numbered options** and the AI only **picks numbers** and writes, so it cannot invent notes; **code checks** the picks and allows **one fix round**. Because this takes many slow LLM calls, it runs as a **background job**: the app adds the job to a **queue**, and a **worker** process runs it and saves the result.

```text
goal: "A short story about my guitar progress"
  1. PLAN (AI)        → searches: ["G to C changes", "barre chords"]
  2. SEARCH (code)    → options: [1] ...  [2] ...  [3] ...  [4] ...
  3. PICK (AI)        → picks: [1, 3, 4], story: "..."
  4. CHECK (code)     → numbers exist? 2–4 picks? ≤ 60 words?
  5. FIX (AI, once)   → only if a check failed

app ── add job ──► jobs table (queued) ◄── worker picks it up ──► runs pipeline ──► saves result (done)
```

| Word | Simple meaning |
| --- | --- |
| Pipeline | A chain of small steps, some AI, some code. |
| Numbered options | Code lists real items; the AI answers with numbers only. |
| Check / guard | Code that rejects bad AI output. |
| Background job | Work that runs separately so the app stays fast. |
| Queue | Jobs waiting to be done (here, a SQLite table). |
| Worker | A process that takes jobs from the queue and runs them. |

## Part 1: the pipeline

Make `pipeline.ts`.

**Piece 1: notes and search.**

```ts
import { z } from "zod";
import { chat, embed, cosine, type Message } from "./lib.ts";

const notes = [
  "2026-10-02 Guitar: started G to C changes, very slow, 10 per minute.",
  "2026-10-14 Guitar: G to C at 18 changes per minute.",
  "2026-11-01 Guitar: barre chords buzzing on the B string.",
  "2026-11-08 Guitar: G to C at 30 changes per minute, felt easy.",
  "2026-10-20 Football: juggling record 21.",
  "2026-11-12 MTB: braked earlier, much smoother on the descent.",
];
const vectors = await Promise.all(notes.map(embed));

async function search(query: string, k = 2): Promise<string[]> {
  const q = await embed(query);
  return notes
    .map((text, i) => ({ text, score: cosine(q, vectors[i]) }))
    .sort((a, b) => b.score - a.score)
    .slice(0, k)
    .map((r) => r.text);
}
```

**Piece 2: plan, search, number the options.**

```ts
const Plan = z.object({ searches: z.array(z.string()).min(1).max(4) });
const Pick = z.object({ picks: z.array(z.number().int()), story: z.string() });

export async function runStory(goal: string) {
  const plan = Plan.parse(JSON.parse(await chat(
    [
      { role: "system", content: "Plan searches over a person's practice notes. Return 2 to 4 short search queries." },
      { role: "user", content: goal },
    ],
    z.toJSONSchema(Plan),
  )));

  const found = new Set<string>();
  for (const q of plan.searches) for (const t of await search(q)) found.add(t);
  const options = [...found];
  const numbered = options.map((t, i) => `[${i + 1}] ${t}`).join("\n");
```

**Piece 3: pick, check, fix once.**

```ts
  const messages: Message[] = [
    { role: "system", content: `Pick 2 to 4 notes by their numbers, in story order, and write a story of at most 60 words using only those notes.\n\nOPTIONS:\n${numbered}` },
    { role: "user", content: goal },
  ];

  for (let round = 1; round <= 2; round++) {
    const reply = await chat(messages, z.toJSONSchema(Pick));
    const result = Pick.parse(JSON.parse(reply));

    const problems: string[] = [];
    if (result.picks.length < 2 || result.picks.length > 4) problems.push("pick 2 to 4 notes");
    const bad = result.picks.filter((n) => n < 1 || n > options.length);
    if (bad.length) problems.push(`these numbers do not exist: ${bad.join(", ")}`);
    if (new Set(result.picks).size !== result.picks.length) problems.push("do not pick the same note twice");
    const words = result.story.split(/\s+/).length;
    if (words > 60) problems.push(`the story has ${words} words; the maximum is 60`);

    if (problems.length === 0) {
      return { searches: plan.searches, picked: result.picks.map((n) => options[n - 1]), story: result.story };
    }
    console.log(`Round ${round} failed checks: ${problems.join("; ")}`);
    messages.push({ role: "assistant", content: reply }, { role: "user", content: `Fix these problems: ${problems.join("; ")}.` });
  }
  throw new Error("Story failed the checks twice");
}
```

**Try it directly** with `step16.ts`:

```ts
import { runStory } from "./pipeline.ts";
console.log(await runStory("A short story about my guitar progress"));
```

```text
{
  searches: [ 'G to C chord changes', 'guitar progress' ],
  picked: [ '2026-10-02 Guitar: started G to C changes...', '2026-10-14 ...18 per minute.', '2026-11-08 ...30 per minute, felt easy.' ],
  story: 'In early October, G to C changes crawled at 10 per minute. Two weeks later: 18. By November, 30 per minute, and it felt easy.'
}
```

## Part 2: run it as a background job

**Piece 4: add a job.** `step16-enqueue.ts`:

```ts
import Database from "better-sqlite3";

const db = new Database("jobs.db");
db.exec(`CREATE TABLE IF NOT EXISTS jobs (
  id INTEGER PRIMARY KEY,
  goal TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'queued',
  result TEXT,
  error TEXT
)`);

const goal = process.argv[2] ?? "A short story about my guitar progress";
const info = db.prepare("INSERT INTO jobs (goal) VALUES (?)").run(goal);
console.log(`Queued job ${info.lastInsertRowid}: ${goal}`);
```

**Piece 5: the worker.** `step16-worker.ts`:

```ts
import Database from "better-sqlite3";
import { runStory } from "./pipeline.ts";

const db = new Database("jobs.db");
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));
console.log("Worker started. Waiting for jobs...");

while (true) {
  const job = db.prepare("SELECT * FROM jobs WHERE status = 'queued' ORDER BY id LIMIT 1").get() as
    | { id: number; goal: string }
    | undefined;
  if (!job) {
    await sleep(1000);
    continue;
  }

  db.prepare("UPDATE jobs SET status = 'running' WHERE id = ?").run(job.id);
  console.log(`Job ${job.id} running: ${job.goal}`);
  try {
    const result = await runStory(job.goal);
    db.prepare("UPDATE jobs SET status = 'done', result = ? WHERE id = ?").run(JSON.stringify(result), job.id);
    console.log(`Job ${job.id} done: ${result.story}`);
  } catch (err) {
    db.prepare("UPDATE jobs SET status = 'failed', error = ? WHERE id = ?").run(String(err), job.id);
    console.log(`Job ${job.id} failed: ${err}`);
  }
}
```

**Run with two terminals** (run the enqueue script once first so the table exists):

```bash
# Terminal 1
npx tsx step16-worker.ts

# Terminal 2
npx tsx step16-enqueue.ts "A short story about my guitar progress"
npx tsx step16-enqueue.ts "A short story about my bike riding"
```

Terminal 2 returns instantly; Terminal 1 works through the jobs.

## Try this

- Stricter check (max 30 words): watch the fix round.
- Stop the worker mid-job; the job is stuck as `running`. Fix: on start, set `running` jobs back to `queued`.
- `step16-status.ts`: `console.table(db.prepare("SELECT id, goal, status FROM jobs").all())`.

## Quiz

1. Why does the AI answer with numbers?
2. Name two checks.
3. Why a background job?

**Answers:** 1. Numbers can be checked; the AI cannot invent notes. 2. Picks exist, 2–4 picks, no duplicates, story length. 3. Many slow LLM calls; the app stays responsive, survives restarts, keeps results.

## Your practice-video app

Your story drafter, almost exactly, run as background jobs in a worker.

## Learn more

- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) (Anthropic)
- [Node.js worker threads](https://nodejs.org/api/worker_threads.html)
