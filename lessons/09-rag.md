# Step 9: RAG

You combine Steps 4, 6 and 8: search your notes first, then give only the best matches to the LLM, so it can answer about years of notes without them all fitting in the prompt.

## The idea

**RAG** = **Retrieval-Augmented Generation**: **retrieve** the notes that matter, **augment** the prompt with them, let the model **generate** the answer. It solves two problems: the context window is too small for all notes (Step 2), and the model does not know your notes (Step 6). If the right note is not retrieved, the model cannot use it.

```text
question ──► embed ──► compare with all note vectors ──► top 3 notes
                                                           │
            system prompt: "Answer only from these notes:" + top 3 notes
                                                           ▼
                                                   LLM ──► answer + quotes ──► quote check
```

Everyday example: an open-book exam. Look up the right pages, then write from them.

| Word | Simple meaning |
| --- | --- |
| RAG | Search first, then answer using only what was found. |
| Retrieval | The search part (keyword, embeddings, or both). |
| Top-k | The k best matches. |
| Chunk | A piece of a longer document, embedded separately. |

## The code

From now on, steps share helpers. Make `lib.ts` once:

```ts
import { pipeline } from "@huggingface/transformers";

export const MODEL = "gemma3:4b";
export type Message = { role: "system" | "user" | "assistant"; content: string };

export async function chat(messages: Message[], format?: object, temperature = 0): Promise<string> {
  const res = await fetch("http://localhost:11434/api/chat", {
    method: "POST",
    body: JSON.stringify({ model: MODEL, messages, format, stream: false, options: { temperature } }),
  });
  return (await res.json()).message.content;
}

const embedder = await pipeline("feature-extraction", "Xenova/all-MiniLM-L6-v2");
export async function embed(text: string): Promise<number[]> {
  const out = await embedder(text, { pooling: "mean", normalize: true });
  return Array.from(out.data as Float32Array);
}

export function cosine(a: number[], b: number[]): number {
  let dot = 0, la = 0, lb = 0;
  for (let i = 0; i < a.length; i++) { dot += a[i] * b[i]; la += a[i] ** 2; lb += b[i] ** 2; }
  return dot / (Math.sqrt(la) * Math.sqrt(lb));
}
```

Now make `step09.ts`.

**Piece 1: more notes, embedded once.**

```ts
import { z } from "zod";
import { chat, embed, cosine } from "./lib.ts";

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

const vectors = await Promise.all(notes.map(embed));
```

**Piece 2: retrieve the top 3.**

```ts
async function retrieve(question: string, k = 3) {
  const q = await embed(question);
  return notes
    .map((text, i) => ({ text, score: cosine(q, vectors[i]) }))
    .sort((a, b) => b.score - a.score)
    .slice(0, k)
    .map((r) => r.text);
}
```

**Piece 3: answer only from what was found.**

```ts
const Answer = z.object({ answer: z.string(), quotes: z.array(z.string()) });

async function rag(question: string) {
  const found = await retrieve(question);
  console.log(`\nQ: ${question}\nRetrieved:`, found);

  const reply = await chat(
    [
      { role: "system", content: "Answer ONLY from these notes, with exact quotes. If they don't say, answer \"The notes don't say.\"\n\n" + found.join("\n") },
      { role: "user", content: question },
    ],
    z.toJSONSchema(Answer),
  );

  const result = Answer.parse(JSON.parse(reply));
  const allText = found.join(" ").toLowerCase();
  const fake = result.quotes.filter((q) => !allText.includes(q.toLowerCase()));
  console.log("Answer:", result.answer, fake.length ? `\n  ⚠ quotes not found: ${fake}` : "");
}

await rag("How has my G to C chord change speed improved?");
await rag("Am I getting better at stopping on the bike?");
await rag("What is my juggling record?");
```

## Run it and read the output

```text
Q: How has my G to C chord change speed improved?
Retrieved: [ '...G to C at 18...', '...G to C at 30...', '...started G to C changes, very slow, 10 per minute.' ]
Answer: From 10 changes per minute in early October to 30 per minute by November 8.

Q: Am I getting better at stopping on the bike?
Retrieved: [ '...braked earlier, much smoother...', '...braked too late at the bottom.', '...walked the steep part.' ]
Answer: Yes. On Oct 25 you braked too late; on Nov 12 you braked earlier and were smoother.
```

"stopping" never appears in the notes, but meaning search found the braking notes.

## Try this

- `k = 1`: what happens to the chord answer?
- Ask about swimming: "The notes don't say"?
- Hybrid search: merge Step 7 FTS5 results with these.

## Quiz

1. What do R, A, G stand for?
2. Why not put all notes in the prompt?
3. The answer is wrong but followed the notes. Where do you look first?

**Answers:** 1. Retrieval, Augmented, Generation. 2. They don't fit, and more text is slower and costlier. 3. Retrieval.

## Your practice-video app

Search first (keyword + meaning), send only the best logs to the LLM: RAG.

## Learn more

- [Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval) (Anthropic)
