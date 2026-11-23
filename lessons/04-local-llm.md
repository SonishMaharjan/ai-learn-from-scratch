# Step 4: Local LLM

You run a real LLM on your own computer with Ollama, talk to it from TypeScript over HTTP, and measure its speed in tokens per second.

## The idea

**Ollama** is a free app that downloads models and runs them on your computer. It starts a small web server at `http://localhost:11434`. Your code sends a JSON request and gets the answer back, like any REST API. Models are **quantized**: each weight is stored with fewer bits (4 instead of 16), about 4× smaller with little quality loss. They come as **GGUF** files: weights, tokenizer and settings in one file.

```text
step04.ts ── POST /api/chat (JSON) ──► Ollama server (localhost:11434)
                                         │ loads model file (GGUF) into memory
                                         │ runs it on GPU (Metal / CUDA) or CPU
          ◄── answer + token counts ──────┘
```

| Word | Simple meaning |
| --- | --- |
| Inference | Using a trained model to get an answer. |
| Quantization | Storing weights with fewer bits so the model is smaller and faster. |
| GGUF | The file format for quantized models used by Ollama and llama.cpp. |
| Parameters ("4B") | Number of weights. More = smarter, bigger, slower. |
| Tokens per second | How many tokens the model writes each second. |
| System prompt | Hidden instructions that set the model's role and rules. |

## Install and download a model

1. Install Ollama from [ollama.com](https://ollama.com) (Mac: also `brew install ollama`).
2. Pick a model that fits your memory (check [ollama.com/library](https://ollama.com/library) for newer ones):

| Your RAM | Model | Download | Notes |
| --- | --- | --- | --- |
| 8 GB | `gemma3:1b` | about 0.8 GB | Fast, simple answers |
| 16 GB or more | `gemma3:4b` | about 3.3 GB | **Course default.** Also understands images (Step 13) |
| 24 GB or more | `gemma3:12b` | about 8 GB | For the speed comparison |

3. Download and chat:

```bash
ollama pull gemma3:4b
ollama run gemma3:4b "Give me one tip for faster chord changes."
```

## The code

Make `step04.ts`. No library needed: Node has `fetch` built in.

**Piece 1: one question, full answer.**

```ts
const MODEL = "gemma3:4b";

const res = await fetch("http://localhost:11434/api/chat", {
  method: "POST",
  body: JSON.stringify({
    model: MODEL,
    messages: [
      { role: "system", content: "You are a short, practical practice coach." },
      { role: "user", content: "I keep braking too late on rocky descents. One tip?" },
    ],
    stream: false,
    options: { temperature: 0.7 },
  }),
});

const data = await res.json();
console.log(data.message.content);
```

**Piece 2: measure speed.**

```ts
const inputTokens = data.prompt_eval_count;
const outputTokens = data.eval_count;
const seconds = data.eval_duration / 1e9;

console.log(`\nInput tokens: ${inputTokens}`);
console.log(`Output tokens: ${outputTokens}`);
console.log(`Speed: ${(outputTokens / seconds).toFixed(1)} tokens/second`);
```

- `eval_duration` is in nanoseconds; dividing by 1e9 gives seconds.

**Piece 3: streaming, token by token.**

```ts
const streamRes = await fetch("http://localhost:11434/api/chat", {
  method: "POST",
  body: JSON.stringify({
    model: MODEL,
    messages: [{ role: "user", content: "Write a 3-line pep talk for football practice." }],
    stream: true,
  }),
});

const decoder = new TextDecoder();
let buffer = "";
for await (const chunk of streamRes.body!) {
  buffer += decoder.decode(chunk, { stream: true });
  const lines = buffer.split("\n");
  buffer = lines.pop()!;
  for (const line of lines) {
    if (!line.trim()) continue;
    const part = JSON.parse(line);
    process.stdout.write(part.message?.content ?? "");
  }
}
console.log();
```

- With `stream: true`, Ollama sends one small JSON object per line as each token is ready.

## Run it and read the output

Make sure Ollama is running, then `npx tsx step04.ts`:

```text
Start braking earlier, on the smooth part before the rocks, then let the bike roll through...

Input tokens: 41
Output tokens: 58
Speed: 35.2 tokens/second
You've got this! ...
```

Speed depends on your computer: tens of tokens per second for a 4B model on Apple Silicon or an NVIDIA GPU; a few on CPU only.

## Try this

- Compare `gemma3:1b`, `gemma3:4b`, `gemma3:12b`: speed vs quality.
- `temperature: 0` twice: same answer?
- Remove the system message.
- Run `ollama ps` while it answers: memory use and GPU.

## Quiz

1. Why are local models quantized?
2. What does `stream: true` change?
3. Why is a 12B model slower than a 4B one?

**Answers:** 1. To fit in memory and run fast, with little quality loss. 2. You get the answer piece by piece. 3. 3× more weights to read for every token.

## Your practice-video app

Every AI text feature is an HTTP call like this to a local model.

## Learn more

- [Ollama chat API](https://docs.ollama.com/api/chat)
- [GGUF](https://www.huggingface.co/docs/hub/en/gguf)
- [Intro to Large Language Models](https://www.youtube.com/watch?v=zjkBMFhNj_g)
