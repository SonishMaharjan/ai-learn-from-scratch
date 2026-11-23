# Step 8: Embeddings

You turn each note into a list of numbers that captures its meaning, compare them with cosine similarity written by hand, and see meaning search beat keyword search.

## The idea

An **embedding model** turns text into a fixed-length list of numbers: a **vector** (here 384 numbers). Similar meanings point in similar directions, even with no shared words. **Cosine similarity** compares two vectors: 1 = same direction, 0 = unrelated. **Semantic search** = embed every note once, embed the question, return the closest notes.

```text
"braked too late"    ──► embedding model ──► [0.12, -0.03, 0.41, ...]
"stopping on a bike" ──► embedding model ──► [0.10, -0.01, 0.38, ...]
                                               │
                              cosine similarity = high → similar meaning
```

Everyday example: a city map. Notes about the same thing end up in the same neighbourhood.

| Word | Simple meaning |
| --- | --- |
| Embedding | A list of numbers representing the meaning of a text. |
| Vector | A list of numbers; an arrow pointing somewhere. |
| Dimensions | How many numbers in the vector. |
| Cosine similarity | How closely two arrows point the same way (-1 to 1). |
| Semantic search | Search by meaning instead of exact words. |
| ONNX | A standard model file format, run by ONNX Runtime on any computer. |

## The code

`npm install @huggingface/transformers` (a bigger install: it includes the ONNX runtime). The first run downloads `all-MiniLM-L6-v2` (about 23 MB). Make `step08.ts`.

**Piece 1: load the embedding model.**

```ts
import { pipeline } from "@huggingface/transformers";

const embedder = await pipeline("feature-extraction", "Xenova/all-MiniLM-L6-v2");

async function embed(text: string): Promise<number[]> {
  const output = await embedder(text, { pooling: "mean", normalize: true });
  return Array.from(output.data as Float32Array);
}
```

- `pooling: "mean"` averages the per-token vectors into one. `normalize: true` makes every vector length 1.

**Piece 2: cosine similarity by hand.**

```ts
function cosine(a: number[], b: number[]): number {
  let dot = 0;
  let lenA = 0;
  let lenB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    lenA += a[i] * a[i];
    lenB += b[i] * b[i];
  }
  return dot / (Math.sqrt(lenA) * Math.sqrt(lenB));
}
```

**Piece 3: embed the notes once.**

```ts
const notes = [
  "Guitar: G to C chord change, 20 min, still slow",
  "Football: 50 left-foot passes against the wall",
  "MTB: rode the rocky descent twice, braked too late",
  "Guitar: barre chords with a metronome",
  "Football: first touch drills, both feet",
];

const noteVectors = await Promise.all(notes.map(embed));
console.log("Each vector has", noteVectors[0].length, "numbers. First 5:", noteVectors[0].slice(0, 5));
```

**Piece 4: meaning search vs keyword search.**

```ts
async function semanticSearch(query: string) {
  const q = await embed(query);
  return notes
    .map((text, i) => ({ score: cosine(q, noteVectors[i]).toFixed(2), text }))
    .sort((a, b) => Number(b.score) - Number(a.score))
    .slice(0, 2);
}

const keywordSearch = (query: string) =>
  notes.filter((n) => query.toLowerCase().split(" ").some((w) => n.toLowerCase().includes(w)));

for (const query of ["stopping", "string instrument", "kicking a ball"]) {
  console.log(`\n"${query}"`);
  console.log("  keyword :", keywordSearch(query));
  console.log("  semantic:", await semanticSearch(query));
}
```

## Run it and read the output

Something like (scores differ a little):

```text
"stopping"
  keyword : []
  semantic: [ { score: '0.31', text: 'MTB: rode the rocky descent twice, braked too late' }, ... ]

"string instrument"
  keyword : []
  semantic: [ { score: '0.38', text: 'Guitar: barre chords with a metronome' }, ... ]

"kicking a ball"
  keyword : [ ...notes containing "a" ... ]
  semantic: [ { score: '0.45', text: 'Football: first touch drills, both feet' }, ... ]
```

Scores are not percentages; the order matters.

## Try this

- Search "metronome": keyword search is exact here. Real systems use both (**hybrid search**).
- `cosine(noteVectors[0], noteVectors[3])` (two guitar notes) vs `cosine(noteVectors[0], noteVectors[1])`.
- Upgrade to `onnx-community/embeddinggemma-300m-ONNX`; see the [EmbeddingGemma guide](https://huggingface.co/blog/embeddinggemma) for its prefixes.

## Quiz

1. What does an embedding model output?
2. What does cosine near 1 mean?
3. When is keyword search better?

**Answers:** 1. A fixed-length vector representing meaning. 2. Very similar meaning. 3. Exact words, names, numbers, rare terms.

## Your practice-video app

"Search by meaning with a local embedding model" is exactly this.

## Learn more

- [Embeddings: what they are and why they matter](https://simonwillison.net/2023/Oct/23/embeddings/)
- [EmbeddingGemma](https://huggingface.co/blog/embeddinggemma)
- [Transformers, chapter 5](https://www.3blue1brown.com/lessons/gpt) (3Blue1Brown)
