# Step 6: Your notes + hallucination

You give the LLM your own notes, watch it make something up, then add a code check (every quote must really be in your notes) and one fix round.

## The idea

A model only knows its training plus what is in the prompt. To answer about *your* practice, paste your notes into the prompt: **grounding**. Models still sometimes invent things: a **hallucination**. You can **check** in code: ask for exact **quotes** from your notes, verify each quote really appears, and if not, send it back for one fix round.

```text
notes + question ─► LLM ─► { answer, quotes: [...] }
                                   │
                    code: is every quote really in the notes?
                        │ yes                    │ no
                        ▼                        ▼
                     accept        "These quotes are not in the notes...
                                    Answer only from the notes." ─► LLM (once)
```

| Word | Simple meaning |
| --- | --- |
| Grounding | Giving the model the facts to use, inside the prompt. |
| Hallucination | A confident answer not supported by the facts. |
| Quote check | Code that confirms each quote exists in the source. |
| Fix round | One extra request saying exactly what was wrong. |

## The code

Make `step06.ts`.

**Piece 1: notes and a helper.**

```ts
import { z } from "zod";

const MODEL = "gemma3:4b";

const notes = [
  "2026-11-01 Guitar: G to C chord change, 20 min, still slow.",
  "2026-11-03 Guitar: barre chords with a metronome, buzzing on the B string.",
  "2026-11-04 Football: 50 left-foot passes against the wall.",
  "2026-11-06 MTB: rode the rocky descent twice, braked too late.",
  "2026-11-08 Guitar: G to C change felt smoother, practiced 15 min.",
];

type Message = { role: "system" | "user" | "assistant"; content: string };

async function chat(messages: Message[], format?: object): Promise<string> {
  const res = await fetch("http://localhost:11434/api/chat", {
    method: "POST",
    body: JSON.stringify({ model: MODEL, messages, format, stream: false, options: { temperature: 0 } }),
  });
  return (await res.json()).message.content;
}
```

**Piece 2: answer shape and quote check.**

```ts
const Answer = z.object({
  answer: z.string(),
  quotes: z.array(z.string()).min(1),
});

const clean = (s: string) => s.toLowerCase().replace(/\s+/g, " ").trim();
const notesText = clean(notes.join(" "));

function missingQuotes(quotes: string[]) {
  return quotes.filter((q) => !notesText.includes(clean(q)));
}
```

**Piece 3: ask, check, fix once.**

```ts
async function ask(question: string) {
  const messages: Message[] = [
    {
      role: "system",
      content:
        "Answer using ONLY the practice notes below. Support the answer with exact quotes copied from the notes.\n\nNOTES:\n" +
        notes.join("\n"),
    },
    { role: "user", content: question },
  ];

  for (let round = 1; round <= 2; round++) {
    const result = Answer.parse(JSON.parse(await chat(messages, z.toJSONSchema(Answer))));
    const missing = missingQuotes(result.quotes);

    console.log(`Round ${round}:`, result);
    if (missing.length === 0) return result;

    console.log("  Not found in notes:", missing);
    messages.push({ role: "assistant", content: JSON.stringify(result) });
    messages.push({
      role: "user",
      content: `These quotes are not in the notes: ${JSON.stringify(missing)}. Use only exact text from the notes. If the notes do not say, answer "The notes don't say."`,
    });
  }
  return null;
}
```

**Piece 4: two questions.**

```ts
await ask("How is my G to C chord change going?");
await ask("What tempo in bpm did I reach with the metronome?");
```

The second is a trap: the notes mention a metronome but no bpm.

## Run it and read the output

```text
Round 1: { answer: 'It is improving: slow at first, smoother later.', quotes: [ 'G to C chord change, 20 min, still slow.', 'G to C change felt smoother, practiced 15 min.' ] }
Round 1: { answer: 'You reached 60 bpm.', quotes: [ 'barre chords with a metronome at 60 bpm' ] }
  Not found in notes: [ 'barre chords with a metronome at 60 bpm' ]
Round 2: { answer: "The notes don't say.", quotes: [ 'barre chords with a metronome, buzzing on the B string.' ] }
```

Your output will differ; run the trap a few times.

## Try this

- Your own traps: "How many goals did I score?"
- Remove "ONLY" and the quotes rule. More invention?
- Check numbers too: every number in `answer` must appear in the notes.

## Quiz

1. What is a hallucination?
2. Why ask for exact quotes?
3. Why offer "The notes don't say"?

**Answers:** 1. A confident answer not supported by the facts. 2. Quotes can be checked by simple code. 3. Without a safe way out, models tend to invent.

## Your practice-video app

Your app's "code checks" stage before a story draft is accepted.

## Learn more

- [Reduce hallucinations](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations) (Anthropic)
