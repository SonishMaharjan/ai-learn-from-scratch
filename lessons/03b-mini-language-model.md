# Step 3b: Mini language model (optional)

You build the smallest possible language model: it learns from your practice notes which letter usually comes next, then writes new "notes" one letter at a time.

## The idea

A **language model** predicts the next piece of text. The simplest one is a **bigram model**: it only looks at the current character and asks "what usually came next in the training text?". Training here is just counting. **Generation** is the Step 3 loop: get chances for the next character, pick one, add it, repeat. A real LLM does the same loop, but looks at thousands of previous tokens and learns its chances with a neural network.

```text
Training text: "chord change ..."

Count table (after 'c' came...):   h: 6   e: 6   t: 3   k: 1   l: 1

Generate:  ^ → 'p' → 'r' → 'a' → 'c' → 'h' → ... → $
```

Everyday example: a child who has heard a few songs and hums new tunes from bits they remember.

| Word | Simple meaning |
| --- | --- |
| Language model | A model that predicts the next piece of text. |
| Bigram | A pair of neighbours: current character + next character. |
| Generation | Writing new text by predicting the next piece again and again. |
| Start / end markers | `^` = a note starts, `$` = the note ends. |

## The code

Make `step03b.ts`.

**Piece 1: the training text.**

```ts
const notes = [
  "practiced g to c chord change for twenty minutes",
  "played fifty left foot passes against the wall",
  "rode the rocky descent twice and braked too late",
  "practiced barre chords slowly with a metronome",
  "worked on first touch and passing with both feet",
  "rode the climb in a low gear and kept a steady pace",
  "played the chord change faster today",
  "practiced dribbling with the left foot",
];

const text = notes.map((n) => "^" + n + "$").join("");
```

**Piece 2: training = counting pairs.**

```ts
const counts: Record<string, Record<string, number>> = {};

for (let i = 0; i < text.length - 1; i++) {
  const a = text[i];
  const b = text[i + 1];
  if (a === "$") continue;
  counts[a] ??= {};
  counts[a][b] = (counts[a][b] ?? 0) + 1;
}

console.log("After 'c' comes:", counts["c"]);
```

**Piece 3: pick the next character.**

```ts
function nextChar(current: string, temperature: number) {
  const options = Object.entries(counts[current]);
  const weights = options.map(([, n]) => Math.pow(n, 1 / temperature));
  const total = weights.reduce((a, b) => a + b, 0);
  let r = Math.random() * total;
  for (let i = 0; i < options.length; i++) {
    r -= weights[i];
    if (r <= 0) return options[i][0];
  }
  return options[options.length - 1][0];
}
```

**Piece 4: generate new notes.**

```ts
function generate(temperature: number) {
  let out = "";
  let current = "^";
  while (out.length < 80) {
    const c = nextChar(current, temperature);
    if (c === "$") break;
    out += c;
    current = c;
  }
  return out;
}

for (let i = 0; i < 5; i++) console.log("-", generate(1.0));
```

## Run it and read the output

`npx tsx step03b.ts`

```text
After 'c' comes: { t: 3, e: 6, ' ': 1, h: 6, k: 1, l: 1 }
- raye
- wast s
- ploot and aito weth ced o a choraretoo d fach todsceressstr gan oth tot parken o
- pt t ftrd towicllin che
- we
```

It looks a bit like English but means nothing: it only knows one letter of context.

## Try this

- `generate(0.3)`: more repetitive.
- Add 20 of your own notes.
- Harder: a **trigram** model that uses the last 2 characters as the key.

## Quiz

1. What is "training" in this model?
2. Why does the output make no sense?
3. Name two things a real LLM does differently.

**Answers:** 1. Counting which character follows which. 2. It only sees one character back. 3. Thousands of tokens of context; tokens instead of characters; a neural network (transformer) instead of counts; trillions of training tokens.

## Learn more

- [Zero to Hero](https://karpathy.ai/zero-to-hero.html), "makemore" lectures.
- [Attention in transformers](https://www.3blue1brown.com/lessons/attention) (3Blue1Brown).
