# Step 11: Learning from feedback

You save your edits to AI output, turn repeated edits into short "lessons", and add them to the next prompt, so the AI behaves better without any training. Then you compare this with fine-tuning.

## The idea

The model's weights never change when you use it. So an app "learns" by changing the **prompt**. Save each correction as a before/after pair. Ask the LLM to describe each as a general **rule**. When the same kind of rule shows up several times, it becomes a **lesson** added to the system prompt. **Fine-tuning** (Step 15) changes the weights instead: more powerful, but slower and harder to undo.

```text
AI answer ──► you edit it ──► save {before, after}
                                   │ LLM: "what rule does this edit show?"
                                   ▼
                              rule: "Remove praise and cheering."
                                   │ group similar rules (embeddings, Step 8)
                                   ▼
                     seen 2+ times? ──► lesson ──► added to next system prompt
```

| Word | Simple meaning |
| --- | --- |
| Feedback | Your corrections to AI output. |
| Lesson | A rule learned from repeated corrections, added to the prompt. |
| Prompting | Changing behaviour by changing instructions. |
| Fine-tuning | Changing behaviour by training the weights. |
| RLHF | Training with human ratings; part of how ChatGPT and Claude were made. |

## The code

Uses `lib.ts`. Make `step11.ts`.

**Piece 1: save and load corrections.**

```ts
import fs from "node:fs";
import { z } from "zod";
import { chat, embed, cosine } from "./lib.ts";

type Correction = { before: string; after: string; rule?: string };
const FILE = "corrections.json";

const load = (): Correction[] => (fs.existsSync(FILE) ? JSON.parse(fs.readFileSync(FILE, "utf8")) : []);
const save = (list: Correction[]) => fs.writeFileSync(FILE, JSON.stringify(list, null, 2));

const corrections = load();
if (corrections.length === 0) {
  corrections.push(
    { before: "Great job! You practiced guitar for 20 minutes. Keep it up!", after: "Practiced guitar for 20 minutes." },
    { before: "Amazing session! Juggling record is now 21. Keep pushing!", after: "Juggling record is now 21." },
    { before: "You rode the descent in 12 mins.", after: "Rode the descent in 12 min." },
    { before: "Fantastic! 50 left-foot passes. You're doing awesome.", after: "50 left-foot passes." },
  );
}
```

**Piece 2: turn each edit into a rule.**

```ts
const Rule = z.object({ rule: z.string() });

async function toRule(c: Correction): Promise<string> {
  const reply = await chat(
    [
      { role: "system", content: "A user edited an AI-written practice summary. State, in one short sentence starting with a verb, the general rule the user wants followed in future." },
      { role: "user", content: `BEFORE: ${c.before}\nAFTER: ${c.after}` },
    ],
    z.toJSONSchema(Rule),
  );
  return Rule.parse(JSON.parse(reply)).rule;
}

for (const c of corrections) c.rule ??= await toRule(c);
save(corrections);
corrections.forEach((c) => console.log("rule:", c.rule));
```

**Piece 3: repeated rules become lessons.**

```ts
async function findLessons(minCount = 2, threshold = 0.7) {
  const vectors = await Promise.all(corrections.map((c) => embed(c.rule!)));
  const groups: number[][] = [];
  vectors.forEach((v, i) => {
    const group = groups.find((g) => cosine(vectors[g[0]], v) >= threshold);
    if (group) group.push(i);
    else groups.push([i]);
  });
  return groups.filter((g) => g.length >= minCount).map((g) => corrections[g[0]].rule!);
}

const lessons = await findLessons();
console.log("\nLessons:", lessons);
```

**Piece 4: before and after lessons.**

```ts
const note = "2026-11-14 Guitar: G to C at 32 changes per minute.";
const base = "Write a one-sentence summary of this practice note.";
const withLessons = base + "\nFollow these lessons from the user's past edits:\n" + lessons.map((l) => "- " + l).join("\n");

console.log("\nWithout lessons:", await chat([{ role: "system", content: base }, { role: "user", content: note }], undefined, 0.7));
console.log("With lessons:   ", await chat([{ role: "system", content: withLessons }, { role: "user", content: note }], undefined, 0.7));
```

## Run it and read the output

```text
rule: Remove praise and encouragement; state only the facts.
rule: Remove enthusiastic praise and motivational phrases.
rule: Use "min" as the abbreviation for minutes.
rule: Remove compliments and keep only the practice facts.

Lessons: [ 'Remove praise and encouragement; state only the facts.' ]

Without lessons: Great progress! You reached 32 G to C changes per minute.
With lessons:    Reached 32 G to C changes per minute.
```

If no lesson appears, lower `threshold` to 0.6; if unrelated rules merge, raise it.

## Prompting vs fine-tuning

| | Lessons in the prompt | Fine-tuning (Step 15) |
| --- | --- | --- |
| What changes | The instructions | The weights |
| Examples needed | A few | Hundreds to thousands |
| Speed to update | Instant | Minutes to hours |
| Undo | Delete the lesson | Retrain or keep the old model |
| Cost | Some context tokens per call | A GPU to train |
| Best for | Rules, preferences, changing facts | Consistent style/format, a narrow skill |

Start with prompting. Fine-tune only when prompting is not enough.

## Quiz

1. Does this step change the weights?
2. Why group rules with embeddings?
3. One reason to prefer lessons over fine-tuning?

**Answers:** 1. No. 2. The same rule comes in different words. 3. Instant, undoable, few examples, no GPU.

## Your practice-video app

Your app's "learns from my edits" feature.

## Learn more

- [Deep Dive into LLMs like ChatGPT](https://www.youtube.com/watch?v=7xTGNNLPyMI) (post-training and RLHF sections)
- [What is fine-tuning](https://unsloth.ai/docs/get-started/fine-tuning-llms-guide) (Unsloth)
