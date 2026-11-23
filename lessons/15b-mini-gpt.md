# Step 15b: Mini GPT on Colab (optional)

You train a tiny transformer (a mini GPT) from zero on a free Colab GPU and watch it go from random letters to text that looks like Shakespeare.

## The idea

Step 3b counted which letter follows which. A **transformer** does the same job but uses **attention**: each position looks back at earlier positions and decides which ones matter, so it can use long context. Training is the Step 1 loop at a bigger scale: about 10 million weights, many thousands of nudges, on a GPU. Andrej Karpathy wrote a short, readable script for this.

```text
text ─► characters → token IDs ─► embeddings ─► [attention + MLP] × layers ─► scores for next char
                                                         ▲                          │
                                                         └── training: compare, nudge ◄┘

step    0: loss 4.2   output: "xQ;k!Bz.."       (random)
step 2000: loss 1.8   output: "KING: what thou art..." (Shakespeare-ish)
```

| Word | Simple meaning |
| --- | --- |
| Transformer | The network design inside GPT, Claude, Gemma, Llama. |
| Attention | Each position weighs which earlier positions matter. |
| Layer | One block of attention + a small neural network (MLP). |
| Train / validation loss | Error on training text / on unseen text. |
| PyTorch | The Python library most AI training uses. |

## Run it on Colab

1. [Google Colab](https://colab.research.google.com) → new notebook → Runtime → Change runtime type → **T4 GPU**.
2. Get the code from Karpathy's "Let's build GPT" lecture:

```python
!git clone https://github.com/karpathy/ng-video-lecture
%cd ng-video-lecture
!ls
```

You should see `bigram.py`, `gpt.py` and `input.txt` (Shakespeare, about 1 MB).

3. Run the simple one, then the transformer:

```python
!python bigram.py
```

```python
!python gpt.py
```

On a T4 `gpt.py` can take tens of minutes. To go faster, open `gpt.py` and lower the settings at the top (`max_iters`, `n_layer`, `n_embd`).

## Read the output

```text
step 0: train loss 4.2849, val loss 4.2823
step 500: train loss 1.9...
...
step 4500: train loss 1.1..., val loss 1.4...

KING RICHARD:
What say you, my lord? the duke is come...
```

- Loss starts near 4.2 (random among 65 characters) and falls.
- Train loss falling while val loss rises = overfitting.

## Try this

- Replace `input.txt` with your own text (you need a lot of it).
- Watch the lecture while reading `gpt.py`: find attention, and the training loop (forward, loss, backward, step).

## Quiz

1. What does attention allow that the bigram model cannot do?
2. Why does loss start near 4.2?
3. Train loss 0.8, val loss 2.5. What is happening?

**Answers:** 1. Use many earlier characters and decide which matter. 2. ln(65) ≈ 4.17 for random guessing. 3. Overfitting.

## Learn more

- [Let's build GPT](https://karpathy.ai/zero-to-hero.html) (Karpathy)
- [Attention in transformers](https://www.3blue1brown.com/lessons/attention) (3Blue1Brown)
- [Transformer Explainer](https://poloclub.github.io/transformer-explainer/)
