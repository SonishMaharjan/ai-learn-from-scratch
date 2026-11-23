# Step 15: Fine-tuning on Colab

You fine-tune a small real LLM on your own examples on Google Colab's free NVIDIA GPU, and compare it with Step 1. This step uses Python, because training tools are written in Python.

## The idea

**Fine-tuning** trains an existing model a little more on your examples, so its weights change. With **LoRA** you freeze the original weights and train a small **adapter** on top. You give input → ideal output pairs. The loop is Step 1's: guess, measure **loss**, nudge, repeat. Afterwards you can export to GGUF and run it in Ollama.

```text
your 30–100 examples          base model (frozen, 1B–4B params, loaded in 4-bit)
  note → ideal summary  ──►       +  LoRA adapter (small, trainable)
                                        │ training loop on the T4 GPU (CUDA)
                                        │ loss: 2.1 → 1.4 → 0.9 ...
                                        ▼
                         fine-tuned model ──► export GGUF ──► ollama run
```

| Word | Simple meaning |
| --- | --- |
| Fine-tuning | Training an existing model more, on your examples. |
| LoRA | Train a small add-on instead of all weights. |
| Dataset | Your training examples. |
| Loss | The error during training. Down = learning. |
| Epoch | One pass over all examples. |
| Overfitting | Memorizing examples, worse on new ones. |
| VRAM | GPU memory. Colab's free T4 has about 15 GB. |
| CUDA | NVIDIA's GPU software; most training tools need it. |

## Part 1: write your dataset

Make `step15-data.ts` (writes **JSONL**: one JSON object per line):

```ts
import fs from "node:fs";

const examples = [
  { note: "Guitar 20 min, G to C changes, still slow, annoyed.", summary: "Guitar · 20 min · G→C changes · slow" },
  { note: "Wall passes with left foot for half an hour, felt great.", summary: "Football · 30 min · left-foot passes · good" },
  { note: "Rode the rocky descent twice, braked too late.", summary: "MTB · descent x2 · late braking" },
];

const lines = examples.map((e) =>
  JSON.stringify({
    conversations: [
      { role: "user", content: `Summarize: ${e.note}` },
      { role: "assistant", content: e.summary },
    ],
  }),
);
fs.writeFileSync("train.jsonl", lines.join("\n"));
console.log(`Wrote ${lines.length} examples`);
```

Add more until you have **30 to 100 examples** in one consistent style.

## Part 2: train on Colab

1. Open the [Unsloth notebooks](https://unsloth.ai/docs/get-started/unsloth-notebooks) page and open a **conversational** notebook for a small model (1B–4B, e.g. Gemma 3 or Llama 3.2).
2. Runtime → Change runtime type → **T4 GPU**.
3. Upload `train.jsonl` (folder icon).
4. Run the cells top to bottom (follow the current notebook; exact code may differ):

| Notebook cell | What it does |
| --- | --- |
| `pip install unsloth` | Installs the training library. |
| `FastLanguageModel.from_pretrained(..., load_in_4bit=True)` | Loads the base model in 4-bit to fit the T4. |
| `FastLanguageModel.get_peft_model(model, r=16, ...)` | Adds the LoRA adapter (`r` = its size). |
| Load dataset | Use yours: `load_dataset("json", data_files="train.jsonl", split="train")`. |
| Chat template / formatting | Turns conversations into the model's token format. |
| `SFTTrainer(...)`, `trainer.train()` | The training loop. Watch the loss. |
| Inference cell | Ask a new note. |
| Save / export to GGUF | Download for Ollama. |

5. Try a note **not** in your dataset. Does it answer in your style?

## Read the output

```text
Step   Training Loss
  10        2.10
  20        1.45
  30        0.92
  60        0.41
```

Same pattern as Step 1. If loss goes almost to 0 with few examples, watch for overfitting.

## Part 3 (optional): run it in Ollama

Download the `.gguf`. Make `Modelfile` with `FROM ./your-model.gguf`, then `ollama create my-coach -f Modelfile` and `ollama run my-coach`.

## Try this

- Train with 10 examples, then 60. Compare.
- Compare with Step 11: 5 examples in a prompt with the normal model. Is prompting already enough?

## Quiz

1. What does LoRA change, and what stays frozen?
2. Why Colab instead of a laptop?
3. What is overfitting, and how would you spot it?

**Answers:** 1. Trains small adapter weights; the original weights stay frozen. 2. Training needs a strong GPU, and most tools need CUDA. 3. Memorizing; very low training loss but poor answers on new notes.

## Your practice-video app

Your app uses lessons in the prompt (Step 11), not fine-tuning: instant, cheap, undoable.

## Learn more

- [What is fine-tuning](https://unsloth.ai/docs/get-started/fine-tuning-llms-guide) (Unsloth)
- [Unsloth notebooks](https://unsloth.ai/docs/get-started/unsloth-notebooks)
- [Hugging Face LLM Course](https://huggingface.co/learn/llm-course/chapter1/1)
