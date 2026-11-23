# Step 2: Tokens

You split text into tokens with a real tokenizer, count them, and see why every model has a size limit and why long answers are slow.

## The idea

Models do not read letters or words. They read **tokens**: small pieces of text, often part of a word. A **tokenizer** turns text into a list of numbers (token IDs) and back. A model can only look at a fixed number of tokens at once: its **context window**. Speed and cost are counted in tokens too: a model writes roughly the same number of tokens per second, so a longer answer takes longer.

```text
"Practiced barre chords"
        │  tokenizer
        ▼
["Pract", "iced", " bar", "re", " chords"]   ← pieces (yours may split differently)
        │
        ▼
[ 89512, 4087, 3240, 265, 99183 ]               ← token IDs the model sees
```

Everyday example: a suitcase. The context window is the suitcase size. Your notes, the question and the answer must all fit inside.

| Word | Simple meaning |
| --- | --- |
| Token | A small piece of text. In English, 1 token ≈ ¾ of a word. |
| Tokenizer | The tool that splits text into tokens and joins them back. |
| Vocabulary | The full list of tokens a tokenizer knows (often 100,000 to 200,000). |
| Context window | The most tokens a model can handle at once: input + output. |
| Prompt | The text you send to the model. |

## The code

Install a tokenizer (about 5 MB): `npm install js-tiktoken`. It is OpenAI's tokenizer in plain JavaScript. Other models use their own tokenizers, but they work the same way.

Make `step02.ts`.

**Piece 1: load the tokenizer.**

```ts
import { getEncoding } from "js-tiktoken";

const enc = getEncoding("o200k_base");
```

**Piece 2: split a note and look at the pieces.**

```ts
const note = "Practiced barre chords with a metronome at 60 bpm.";
const ids = enc.encode(note);

console.log("Characters:", note.length);
console.log("Tokens:", ids.length);
console.log("IDs:", ids);
console.log("Pieces:", ids.map((id) => JSON.stringify(enc.decode([id]))));
```

- `encode` turns text into token IDs; `decode` turns IDs back into text. `JSON.stringify` makes spaces visible.

**Piece 3: how many notes fit in a context window?**

```ts
const notes = [
  "Guitar: G to C chord change, 20 min, still slow.",
  "Football: 50 left-foot passes against the wall.",
  "MTB: rode the rocky descent twice, braked too late.",
];

const contextWindow = 8192;
const answerRoom = 500;
let used = 0;
let count = 0;

while (used + answerRoom < contextWindow) {
  const next = notes[count % notes.length];
  used += enc.encode(next + "\n").length;
  count++;
}

console.log(`About ${count} notes fit in ${contextWindow} tokens (leaving ${answerRoom} for the answer).`);
```

**Piece 4: tokens and time.**

```ts
const tokensPerSecond = 30;
for (const answerTokens of [50, 300, 2000]) {
  console.log(`${answerTokens} tokens at ${tokensPerSecond}/s ≈ ${(answerTokens / tokensPerSecond).toFixed(1)} s`);
}
```

## Run it and read the output

`npx tsx step02.ts` (exact numbers depend on the tokenizer):

```text
Characters: 50
Tokens: 12
Pieces: [ '"Pract"', '"iced"', '" bar"', '"re"', '" chords"', ... ]
About 590 notes fit in 8192 tokens (leaving 500 for the answer).
50 tokens at 30/s ≈ 1.7 s
300 tokens at 30/s ≈ 10.0 s
2000 tokens at 30/s ≈ 66.7 s
```

- Common words are one token; rare words split into pieces.
- A few hundred short notes fill an 8K window. Years of notes will not fit. That is why Steps 7 to 9 add search.

## Try this

- Tokenize a sentence in another language (Nepali, Hindi, Japanese). Does the same meaning take more tokens?
- Try a number like `1234567` or a made-up word.
- Compare tokenizers in [Tiktokenizer](https://tiktokenizer.vercel.app/).

## Quiz

1. Why does a model need a tokenizer?
2. Your notes use 7,900 tokens and the window is 8,192. What problem will you hit?
3. Why is a 2,000-token answer slower than a 50-token answer?

**Answers:** 1. Models work with numbers; the tokenizer turns text into token IDs and back. 2. Only about 290 tokens are left for the answer. 3. The model writes one token at a time, at a roughly fixed speed.

## Your practice-video app

Years of logs do not fit in one prompt. That is why your app searches first and writes summaries in layers.

## Learn more

- [Tiktokenizer](https://tiktokenizer.vercel.app/)
- [Let's build the GPT Tokenizer](https://karpathy.ai/zero-to-hero.html) (Karpathy)
