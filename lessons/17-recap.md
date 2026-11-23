# Step 17: Recap

Every AI part of your practice-video app is now something you have built a small version of.

## Your app, feature by feature

| App feature | What is really happening | Steps |
| --- | --- | --- |
| Log videos by talking while you watch | Speech to text (Whisper) with word timestamps | 12, 2 |
| Store logs, tags, notes; keyword search | SQLite + FTS5, ranked by BM25 | 7 |
| Search by meaning | Embedding vectors + cosine similarity | 8 |
| Summaries: session → week → month → journey | Layered prompt chaining, "only facts from the text" | 10, 2, 4 |
| Draft video stories | Plan → code searches → AI picks numbered options → code checks → one fix round | 16, 9, 6 |
| Learns from your edits | Corrections → rules → repeated rules become prompt lessons | 11 |
| Every AI answer as JSON, Zod, retry once | Structured output + validation + one retry | 5 |
| Looks at frames | Vision LLM describes and tags images | 13 |
| Counts reps | Pose and object detection (ONNX) per frame; code counts | 13 |
| Measures motion and sound | FFmpeg filters; plain code | 14 |
| Long AI work in the background | Jobs table + worker | 16 |
| Runs locally, private | Quantized GGUF models via Ollama, on Metal or CUDA | 4, 0 |

## The ideas underneath everything

| Idea | One line | Step |
| --- | --- | --- |
| Model = weights + code | A file of numbers plus a program that does math with them | 0, 1 |
| Training | Guess, measure error, nudge weights, repeat | 1, 15 |
| Tokens and context window | Limited tokens at once; speed in tokens per second | 2 |
| Next-token prediction | One token at a time from a probability list | 3, 3b |
| Transformer and attention | Lets models use long context | 3b, 15b |
| Inference and quantization | Running a trained model; fewer bits per weight | 4 |
| Grounding and hallucination | Give facts in the prompt; check claims in code | 6 |
| RAG | Search first, answer only from what was found | 9 |
| Prompting vs fine-tuning | Change instructions vs change weights | 11, 15 |
| Code vs AI | Measure with code; use AI for understanding | 14 |
| Workflows | Small AI steps with code checks between | 16 |

## How the pieces fit together

```text
 video ──► FFmpeg (motion, sound) ──┐
   │                                ├──► interesting moments ──► vision + pose models ──┐
   └──► your voice ──► Whisper ──► logs with word times ──► SQLite (FTS5 + vectors) ◄─┘
                                                               │
                                    search (keyword + meaning) │
                                                               ▼
                    background worker: summaries │ story pipeline │ lessons from edits
                                                               │
                                    local LLM (Ollama) → JSON → Zod check → code checks
```

## Check yourself

1. Why can a model not read all your logs at once? (Step 2)
2. What exactly does temperature change? (Step 3)
3. Why ask for JSON and check it with Zod? (Step 5)
4. How does your app catch a made-up detail? (Steps 6, 16)
5. When does keyword search beat meaning search? (Steps 7, 8)
6. Does "learns from my edits" change the weights or the prompt? (Step 11)
7. Why count reps with a pose model, not a vision LLM? (Step 13)
8. Why does long AI work run in a worker? (Step 16)

## Where to go next

- Rewatch [Intro to Large Language Models](https://www.youtube.com/watch?v=zjkBMFhNj_g) and [Deep Dive into LLMs like ChatGPT](https://www.youtube.com/watch?v=7xTGNNLPyMI).
- Open your real app's code and find each idea from the table above.
- Rebuild one feature yourself, starting from your step files.
- Go deeper: [Neural Networks: Zero to Hero](https://karpathy.ai/zero-to-hero.html) and the [Hugging Face LLM Course](https://huggingface.co/learn/llm-course/chapter1/1).
