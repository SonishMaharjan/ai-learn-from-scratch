# Step 5: Data, not text

You make the LLM return data your code can use: JSON that matches a schema, checked by Zod, with one automatic retry when it is wrong.

## The idea

Apps need data, not chatty text. So we ask for **JSON** and describe the exact shape with a **schema**. Ollama can force the output to follow a JSON schema (**structured output**). Still **validate** every answer in code, because values can still be wrong. **Zod** lets you write the schema once in TypeScript, get the JSON schema for the model, and check the answer. If the check fails, send the errors back and ask once more (**retry**).

```text
note ─► LLM (format = JSON schema) ─► text ─► JSON.parse ─► Zod check
                                                              │
                          ok ◄────────────────────────────────┤
                                                              │ failed
             "Your JSON was wrong: minutes must be ≤ 300" ◄───┘
                    └─► LLM again (once) ─► check ─► ok, or give up
```

| Word | Simple meaning |
| --- | --- |
| JSON | Text format for data: `{ "minutes": 20 }`. |
| Schema | The exact shape: fields, types, allowed values. |
| Structured output | The model is forced to match a JSON schema. |
| Validation | Checking data in code before using it. |
| Zod | A TypeScript library for schemas and validation. |
| Retry | Asking again, with the error, when the check fails. |

## The code

`npm install zod` (about 2 MB). Ollama running with `gemma3:4b`. Make `step05.ts`.

**Piece 1: the schema.**

```ts
import { z } from "zod";

const MODEL = "gemma3:4b";

const NoteInfo = z.object({
  sport: z.enum(["guitar", "football", "mtb"]),
  minutes: z.number().int().min(1).max(300),
  mood: z.enum(["good", "ok", "bad"]),
  tags: z.array(z.string()).max(3),
});
type NoteInfo = z.infer<typeof NoteInfo>;
```

**Piece 2: a helper that calls the model.**

```ts
type Message = { role: "system" | "user" | "assistant"; content: string };

async function chat(messages: Message[], format?: object): Promise<string> {
  const res = await fetch("http://localhost:11434/api/chat", {
    method: "POST",
    body: JSON.stringify({ model: MODEL, messages, format, stream: false, options: { temperature: 0 } }),
  });
  const data = await res.json();
  return data.message.content;
}
```

**Piece 3: extract, check, retry once.**

```ts
async function extract(note: string): Promise<NoteInfo> {
  const messages: Message[] = [
    { role: "system", content: "Extract practice info from the note. Reply with JSON only." },
    { role: "user", content: note },
  ];
  const schema = z.toJSONSchema(NoteInfo);

  for (let attempt = 1; attempt <= 2; attempt++) {
    const text = await chat(messages, schema);

    let json: unknown = null;
    try {
      json = JSON.parse(text);
    } catch {}

    const result = NoteInfo.safeParse(json);
    if (result.success) return result.data;

    const problems = result.error.issues.map((i) => `${i.path.join(".")}: ${i.message}`).join("; ");
    console.log(`Attempt ${attempt} failed: ${problems}`);
    messages.push({ role: "assistant", content: text });
    messages.push({ role: "user", content: `Your JSON was invalid: ${problems}. Fix it. Reply with JSON only.` });
  }
  throw new Error("Model gave invalid JSON twice");
}
```

**Piece 4: try it.**

```ts
const notes = [
  "Guitar, about 20 minutes on G to C changes. Still slow, a bit annoyed.",
  "Wall passes with my left foot for half an hour. Felt great!",
  "MTB: two runs down the rocky descent, braked too late both times.",
];

for (const note of notes) {
  const info = await extract(note);
  console.log(info);
}
```

## Run it and read the output

```text
{ sport: 'guitar', minutes: 20, mood: 'bad', tags: [ 'chord changes', 'G to C' ] }
{ sport: 'football', minutes: 30, mood: 'good', tags: [ 'left foot', 'passing' ] }
{ sport: 'mtb', minutes: 15, mood: 'ok', tags: [ 'descent', 'braking' ] }
```

- "half an hour" became `30`.
- The MTB note never said how long: the schema forced a number, so the model **guessed**. Keep this in mind for Step 6.

## Try this

- Force a retry: add `.refine((n) => n.tags.length >= 2, { message: "need at least 2 tags" })` to the schema.
- Remove `format` from `chat`. How often is the JSON broken?
- Allow unknowns: `minutes: z.number().int().min(1).max(300).nullable()` + "use null if the note does not say".

## Quiz

1. Why validate if Ollama already forces the shape?
2. What does the retry message contain?
3. Why did the MTB note get minutes?

**Answers:** 1. Values can be wrong, and some rules cannot be expressed in the format. 2. The bad answer plus the exact errors. 3. The schema required a number; allow `null`.

## Your practice-video app

Your app's "every AI answer as JSON, checked with Zod, retried once" rule.

## Learn more

- [Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs)
- [Zod docs](https://zod.dev/api)
