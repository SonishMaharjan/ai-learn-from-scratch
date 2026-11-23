# Step 3: Next-token guessing

You simulate the last step of an LLM: turning scores into chances and picking the next token. Then you change temperature and top-p and watch the picks change.

## The idea

An LLM writes one token at a time. For each step it gives every token in its vocabulary a score (a **logit**). **Softmax** turns scores into probabilities that add up to 100%. The program picks one token at random, weighted by those chances (**sampling**), adds it to the text, and repeats. **Temperature** stretches or flattens the chances. **Top-p** throws away the unlikely tail before picking.

```text
"Today I practiced" ──► model ──► scores for every token
                                      │ softmax (with temperature)
                                      ▼
            guitar 54%  football 24%  bike 16%  piano 4%  banana 1%
                                      │ top-p: keep the top until 80%
                                      ▼
                         pick one at random → " guitar"
                                      │
"Today I practiced guitar" ──► model ──► ... (again and again)
```

Everyday example: autocomplete on your phone, but it can pick the second or third suggestion instead of always the first.

| Word | Simple meaning |
| --- | --- |
| Logit | The raw score a model gives each possible next token. |
| Softmax | Turns scores into probabilities that add up to 1. |
| Sampling | Picking a token at random, weighted by its probability. |
| Temperature | Low (0.2) = safe, repetitive. High (1.5+) = varied, odd. 0 = always the top token. |
| Top-p | Keep only the most likely tokens whose chances add up to p, then pick. |
| Transformer | The network design inside every modern LLM; it produces the scores. |

## The code

The five scores are made up. A real model gives about 100,000 scores per step, but the math after that is exactly this. Make `step03.ts`.

**Piece 1: the model's scores.**

```ts
const candidates = [
  { token: " guitar", score: 3.0 },
  { token: " football", score: 2.2 },
  { token: " bike", score: 1.8 },
  { token: " piano", score: 0.5 },
  { token: " banana", score: -1.0 },
];
```

**Piece 2: softmax with temperature.**

```ts
function softmax(scores: number[], temperature: number) {
  const scaled = scores.map((s) => s / temperature);
  const max = Math.max(...scaled);
  const exps = scaled.map((s) => Math.exp(s - max));
  const sum = exps.reduce((a, b) => a + b, 0);
  return exps.map((e) => e / sum);
}
```

- Small temperature makes big gaps bigger. `Math.exp` makes every number positive. Subtracting `max` avoids overflow. Dividing by `sum` makes them add up to 1.

**Piece 3: top-p and picking.**

```ts
function pick(temperature: number, topP: number) {
  const probs = softmax(candidates.map((c) => c.score), temperature);
  const sorted = candidates
    .map((c, i) => ({ token: c.token, p: probs[i] }))
    .sort((a, b) => b.p - a.p);

  const kept: typeof sorted = [];
  let total = 0;
  for (const item of sorted) {
    kept.push(item);
    total += item.p;
    if (total >= topP) break;
  }

  let r = Math.random() * total;
  for (const item of kept) {
    r -= item.p;
    if (r <= 0) return item.token;
  }
  return kept[kept.length - 1].token;
}
```

- `r` is a random point on a line of length `total`; each token owns a piece as long as its chance.

**Piece 4: show the chances and pick 1,000 times.**

```ts
const temperature = 1.0;
const topP = 1.0;

const probs = softmax(candidates.map((c) => c.score), temperature);
candidates.forEach((c, i) => console.log(`${c.token.padEnd(10)} ${(probs[i] * 100).toFixed(1)}%`));

const counts: Record<string, number> = {};
for (let i = 0; i < 1000; i++) {
  const t = pick(temperature, topP);
  counts[t] = (counts[t] ?? 0) + 1;
}
console.log("Picked in 1000 tries:", counts);
```

## Run it and read the output

`npx tsx step03.ts`

```text
 guitar    54.0%
 football  24.3%
 bike      16.3%
 piano     4.4%
 banana    1.0%
Picked in 1000 tries: { ' guitar': 547, ' football': 232, ' bike': 173, ' piano': 40, ' banana': 8 }
```

Even " banana" gets picked sometimes. This is why the same question can get different answers.

## Try this

| Change | What you should see |
| --- | --- |
| `temperature = 0.2` | guitar about 98%: safe and repetitive. |
| `temperature = 2.0` | guitar about 38%, banana about 5%: more variety, more nonsense. |
| `topP = 0.8` | piano and banana are never picked. |

Then open [Transformer Explainer](https://poloclub.github.io/transformer-explainer/) and move its temperature slider: a real GPT-2 in your browser.

## Quiz

1. What does an LLM produce at each step, before a token is picked?
2. You want the same answer every time. Low or high temperature?
3. What does top-p = 0.9 do?

**Answers:** 1. A score for every token, turned into probabilities. 2. Low. 3. Keeps only the most likely tokens that together make 90% of the chance, then picks among them.

## Your practice-video app

Low temperature keeps JSON and option picks stable; higher temperature gives more varied story drafts.

## Learn more

- [Transformer Explainer](https://poloclub.github.io/transformer-explainer/)
- [Transformers, the tech behind LLMs](https://www.3blue1brown.com/lessons/gpt) (3Blue1Brown)
- [Large Language Models explained briefly](https://3blue1brown.com/lessons/mini-llm)
