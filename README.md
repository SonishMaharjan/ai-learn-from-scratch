# Learn AI from Scratch

You build a tiny practice-notes app and add one AI idea per step, from a hand-made neuron to a full AI pipeline. It works on Mac, Windows or Linux; the GPU-heavy parts run free on Google Colab.

## How to use this course

Every step has its own lesson file in [`lessons/`](lessons/), so you can follow the whole course on your own.

1. Read the step's row in the map below. Watch the linked video first if you like to see the idea before the code.
2. Open the lesson. Each lesson has the same parts: the idea with a small diagram, new words, the code in small pieces (type it yourself), the output to expect, things to try, a quiz with answers, and how your practice-video app uses the idea.
3. Keep each step in its own file (`step01.ts`, `step02.ts`...) in this folder, so you can go back.

Steps marked *optional* can be skipped. From Step 4 on you need Ollama running; from Step 9 on, the shared `lib.ts` file.

## Watch first: the big picture

These explain how ChatGPT and Claude work, with no code. Start with the short one; watch the others over time.

| Order | Resource | Length | What you get |
| --- | --- | --- | --- |
| 1 | [Large Language Models explained briefly](https://3blue1brown.com/lessons/mini-llm) (3Blue1Brown) | 8 min | What an LLM is: a next-word guesser with billions of weights |
| 2 | [Intro to Large Language Models](https://www.youtube.com/watch?v=zjkBMFhNj_g) (Andrej Karpathy) | 1 hour | "A model is two files", training vs using, how a chatbot is made |
| 3 | [Neural Networks series](https://3blue1brown.com/neural-networks) (3Blue1Brown) | about 1 hour | Neurons, weights, gradient descent, drawn as animations |
| 4 | [Deep Dive into LLMs like ChatGPT](https://www.youtube.com/watch?v=7xTGNNLPyMI) (Andrej Karpathy) | 3.5 hours | The full ChatGPT story: data, tokens, training, fine-tuning, feedback training |

**Go deeper later (needs Python):** [Neural Networks: Zero to Hero](https://karpathy.ai/zero-to-hero.html) builds a GPT from scratch in code, and the free [Hugging Face LLM Course](https://huggingface.co/learn/llm-course/chapter1/1) covers transformers and fine-tuning.

## Course map

Twenty steps. Steps 3b and 15b are optional and show how an LLM itself is built.

| Step | What you build | Learn the idea |
| --- | --- | --- |
| [0. Setup](lessons/00-setup.md) | Check your computer, make `~/ai-learn`, run `step00.ts` | [Intro to LLMs](https://www.youtube.com/watch?v=zjkBMFhNj_g), first 5 minutes ("two files") |
| [1. Tiny neural network](lessons/01-tiny-neural-network.md) | A: one neuron learns a rule. B: a few neurons learn a pattern one neuron can't | [Neural networks](https://3blue1brown.com/neural-networks) chapters 1-2 |
| [2. Tokens](lessons/02-tokens.md) | Split text with a real tokenizer, count tokens, hit the context limit | [Tiktokenizer](https://tiktokenizer.vercel.app/) |
| [3. Next-token guessing](lessons/03-next-token.md) | Watch a model pick the next token; change temperature and top-p | [Transformer Explainer](https://poloclub.github.io/transformer-explainer/), [Transformers](https://www.3blue1brown.com/lessons/gpt) |
| [3b. Mini language model](lessons/03b-mini-language-model.md) *(optional)* | A next-character guesser in plain TypeScript, trained on your notes | [Attention](https://www.3blue1brown.com/lessons/attention), [Zero to Hero](https://karpathy.ai/zero-to-hero.html) "makemore" |
| [4. Local LLM](lessons/04-local-llm.md) | Run a model with Ollama, call it from TypeScript, measure tokens per second | [Ollama chat API](https://docs.ollama.com/api/chat), [GGUF](https://www.huggingface.co/docs/hub/en/gguf) |
| [5. Data, not text](lessons/05-json-and-zod.md) | Ask for JSON, check it with Zod, retry once if wrong | [Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs), [Zod](https://zod.dev/api) |
| [6. Your notes + hallucination](lessons/06-hallucination.md) | Put notes in the prompt, catch a made-up quote in code, one fix round | [Reduce hallucinations](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations) |
| [7. Keyword search](lessons/07-keyword-search.md) | Notes in SQLite, search with FTS5 | [SQLite FTS5](https://sqlite.org/fts5.html) |
| [8. Embeddings](lessons/08-embeddings.md) | A local embedding model, cosine similarity by hand, meaning vs keyword search | [Embeddings explained](https://simonwillison.net/2023/Oct/23/embeddings/), [EmbeddingGemma](https://huggingface.co/blog/embeddinggemma) |
| [9. RAG](lessons/09-rag.md) | Search first, then give only the found notes to the LLM | [Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval) |
| [10. Layered summaries](lessons/10-layered-summaries.md) | Daily → weekly → monthly summaries, only from the notes | [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) |
| [11. Learning from feedback](lessons/11-learning-from-feedback.md) | Save corrections; repeated ones become "lessons" in the prompt | [Deep Dive into LLMs](https://www.youtube.com/watch?v=7xTGNNLPyMI), [What is fine-tuning](https://unsloth.ai/docs/get-started/fine-tuning-llms-guide) |
| [12. Speech to text](lessons/12-speech-to-text.md) | Transcribe a voice note with whisper.cpp, read word timestamps | [whisper.cpp](https://github.com/ggml-org/whisper.cpp) |
| [13. Vision](lessons/13-vision.md) | A vision model describes a photo; object detection (boxes); pose | [Ollama vision](https://docs.ollama.com/capabilities/vision), [Object detection](https://docs.ultralytics.com/tasks/detect/), [Pose landmarker](https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker) |
| [14. Plain code vs AI](lessons/14-code-vs-ai.md) | Measure loudness and motion with FFmpeg; see when code beats AI | [FFmpeg filters](https://ffmpeg.org/ffmpeg-filters.html) |
| [15. Fine-tuning on Colab](lessons/15-fine-tuning.md) | Train a small real model on a few examples, on a free GPU | [Unsloth notebooks](https://unsloth.ai/docs/get-started/unsloth-notebooks) |
| [15b. Mini GPT on Colab](lessons/15b-mini-gpt.md) *(optional)* | Train a tiny transformer from scratch on Colab | [Let's build GPT](https://karpathy.ai/zero-to-hero.html) |
| [16. Pipelines and jobs](lessons/16-pipelines-and-jobs.md) | Plan → code search → AI picks numbered options → check → fix; run as a background job | [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents), [Node worker threads](https://nodejs.org/api/worker_threads.html) |
| [17. Recap](lessons/17-recap.md) | Every idea mapped to your practice-video app | Rewatch [Intro to LLMs](https://www.youtube.com/watch?v=zjkBMFhNj_g) |

Links as of November 2026. Videos are optional: the code you type is the main lesson.
