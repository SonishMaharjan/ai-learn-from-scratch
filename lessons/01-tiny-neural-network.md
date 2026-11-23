# Step 1: Tiny neural network

You build a neural network with no libraries and watch it learn. Part A is one neuron. Part B is a few neurons that learn a pattern one neuron cannot.

## The idea

A **neuron** is a tiny formula: `guess = w × input + b`. The numbers `w` (**weight**) and `b` (**bias**) are what the model "knows". They start random, so guesses are bad. **Training** repeats one loop: guess, measure how wrong (the **error**), nudge the weights a little to make the error smaller. The size of each nudge is the **learning rate**.

```text
   input x ──► [ w·x + b ] ──► guess
                   ▲             │
                   │     compare with the right answer
                   │             │
                   └── nudge w,b ◄── error
            (repeat many times = training)
```

Everyday example: tuning a guitar string. Pluck, hear it is too high, turn the peg a little, repeat. You never calculate the perfect position; you just get closer each time.

| Word | Simple meaning |
| --- | --- |
| Neuron | A small formula: multiply inputs by weights, add them up. |
| Weight, bias | The numbers a neuron learns. |
| Error (loss) | How far the guess is from the right answer. |
| Learning rate | How big each nudge is. |
| Epoch | One pass through all training examples. |
| Gradient descent | The rule for nudging: move each weight in the direction that lowers the error. |

## Part A: one neuron learns a rule

Make `step01.ts` and type each piece.

**Piece 1: the data.** Practice minutes and a score. The hidden rule is `score = 2 × minutes + 1`. The model does not know it.

```ts
const data = [
  { x: 1, y: 3 },
  { x: 2, y: 5 },
  { x: 3, y: 7 },
  { x: 4, y: 9 },
];
```

**Piece 2: the model.**

```ts
let w = Math.random();
let b = Math.random();
const predict = (x: number) => w * x + b;
```

- `w` and `b` start random. These two numbers are the whole "brain".
- `predict` is the code part of the model; `w` and `b` are the weights part.

**Piece 3: training.**

```ts
const learningRate = 0.01;

for (let epoch = 0; epoch <= 200; epoch++) {
  let totalError = 0;
  for (const { x, y } of data) {
    const guess = predict(x);
    const error = guess - y;
    w -= learningRate * error * x;
    b -= learningRate * error;
    totalError += error ** 2;
  }
  if (epoch % 20 === 0) {
    console.log(`epoch ${epoch}  w=${w.toFixed(3)}  b=${b.toFixed(3)}  error=${totalError.toFixed(4)}`);
  }
}

console.log("Prediction for 10 minutes:", predict(10).toFixed(2));
```

- `error = guess - y`: positive means the guess was too high.
- `w -= learningRate * error * x`: too high → make `w` smaller. Times `x` because `w` mattered more when the input was big. This is gradient descent.
- `error ** 2` squares the error so negatives do not cancel positives.
- The last line asks about 10 minutes, which was never in the data.

**Run:** `npx tsx step01.ts`

```text
epoch 0  w=0.948  b=0.656  error=67.7820
epoch 20  w=1.993  b=1.017  error=0.0002
...
epoch 200  w=1.998  b=1.006  error=0.0000
Prediction for 10 minutes: 20.99
```

By epoch 20, `w ≈ 2` and `b ≈ 1`: it found the hidden rule. 20.99 is close to the true 21 for an input it never saw.

**Try this:** set `learningRate` to `0.001` (does it finish by epoch 200?) and to `0.5` (watch `w` explode, like turning the peg far too hard).

## Part B: a few neurons learn XOR

**XOR** means "one or the other, but not both": output 1 when exactly one input is 1. A single neuron cannot learn it. A small **network** can: a **hidden layer** of 4 neurons feeds one output neuron.

```text
  input A ─┬─► (h1) ─┐
           ├─► (h2) ─┼─► (out) ──► 0 or 1
  input B ─┼─► (h3) ─┤
           └─► (h4) ─┘
      hidden layer = 4 neurons
```

Make `step01b.ts`.

**Piece 1: data and helpers.**

```ts
const data = [
  { x: [0, 0], y: 0 },
  { x: [0, 1], y: 1 },
  { x: [1, 0], y: 1 },
  { x: [1, 1], y: 0 },
];

const sigmoid = (z: number) => 1 / (1 + Math.exp(-z));
const rand = () => Math.random() * 2 - 1;
```

- `sigmoid` squashes any number into 0..1. It is an **activation function**: it lets the network learn curves, not only straight lines.

**Piece 2: the network's weights.**

```ts
const HIDDEN = 4;
let w1 = Array.from({ length: HIDDEN }, () => [rand(), rand()]);
let b1 = Array.from({ length: HIDDEN }, rand);
let w2 = Array.from({ length: HIDDEN }, rand);
let b2 = rand();
```

- Count: 4×2 + 4 + 4 + 1 = 17 **parameters**.

**Piece 3: forward pass** (inputs → guess).

```ts
function forward(x: number[]) {
  const h = w1.map((w, i) => sigmoid(w[0] * x[0] + w[1] * x[1] + b1[i]));
  const out = sigmoid(h.reduce((sum, hi, i) => sum + hi * w2[i], b2));
  return { h, out };
}
```

**Piece 4: training with backpropagation.**

```ts
const learningRate = 0.5;

for (let epoch = 0; epoch <= 10000; epoch++) {
  let totalError = 0;
  for (const { x, y } of data) {
    const { h, out } = forward(x);
    const error = out - y;
    totalError += error ** 2;

    const dOut = error * out * (1 - out);
    for (let i = 0; i < HIDDEN; i++) {
      const dH = dOut * w2[i] * h[i] * (1 - h[i]);
      w2[i] -= learningRate * dOut * h[i];
      w1[i][0] -= learningRate * dH * x[0];
      w1[i][1] -= learningRate * dH * x[1];
      b1[i] -= learningRate * dH;
    }
    b2 -= learningRate * dOut;
  }
  if (epoch % 2000 === 0) console.log(`epoch ${epoch}  error=${totalError.toFixed(4)}`);
}

for (const { x, y } of data) {
  console.log(`${x[0]} XOR ${x[1]} = ${forward(x).out.toFixed(3)}  (correct: ${y})`);
}
```

- `dOut`: how much the output neuron is to blame. `out * (1 - out)` is the slope of sigmoid.
- `dH`: the blame passed back to each hidden neuron through `w2[i]`. Passing blame backwards is **backpropagation**. Every LLM is trained with this idea.

**Run:** `npx tsx step01b.ts`

```text
epoch 0  error=1.0738
...
epoch 10000  error=0.0010
0 XOR 0 = 0.016  (correct: 0)
0 XOR 1 = 0.984  (correct: 1)
1 XOR 0 = 0.982  (correct: 1)
1 XOR 1 = 0.014  (correct: 0)
```

Sometimes the error gets stuck near 0.5 and two answers sit near 0.48. That is a **local minimum**. Run it again; a new random start usually works.

**Try this:** set `HIDDEN = 1`. Can it ever learn XOR?

## Why GPUs help

Part B does a few dozen multiplications per example. An 8B model does about 8 billion for every token it writes. A **GPU** does thousands of multiplications at once; a CPU does a few.

## Quiz

1. In Part A, what are the weights, and how many are there?
2. What does the learning rate control?
3. Why can a network with a hidden layer learn XOR, but one neuron cannot?

**Answers:** 1. `w` and `b`, two numbers. 2. How big each nudge is. 3. Hidden neurons plus sigmoid let the network bend its answer into a shape that is not a straight line.

## Your practice-video app

Every model in your app is this same idea with millions or billions of weights. Someone already trained them, so your app only runs the forward pass (**inference**).

## Learn more

- [Neural networks](https://3blue1brown.com/neural-networks) (3Blue1Brown), chapters 1 to 4.
- [Zero to Hero, lecture 1](https://karpathy.ai/zero-to-hero.html) (Karpathy).
