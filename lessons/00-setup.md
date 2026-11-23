# Step 0: Set up your computer

You check your computer, install two tools, make the project folder and run a first script. No AI code yet.

## How this course works

- **One step at a time.** Each step has its own file: `step00.ts`, `step01.ts`, and so on. You can always go back.
- **You type the code.** Code comes in small pieces with every line explained. Typing it yourself is how it sticks.
- **Each step has the same parts:** the idea, the smallest code, run it and read the output, one thing to try, a short quiz, and how your practice-video app uses it.
- **Nothing paid.** Models run on your computer. Anything that needs an NVIDIA GPU runs on Google Colab (free).
- **Before any download,** the lesson tells you its size.

## The idea: what is a model?

A model is two things: a big file of numbers, and a small program that does math with those numbers. The numbers are called **weights**. You give the program an input (text, a photo, sound). It multiplies and adds your input with the weights. Out comes an answer.

```text
  input            the model                     output
  "Hello"  --->  [ code  +  weights file ]  --->  "Hi! How can I help?"
                  (small)    (big: 1-10 GB)
```

Think of a recipe book (the code) and a jar of secret spice amounts (the weights). The recipe is short and the same for every cook. The spice amounts make the taste. Training is the slow work of finding good spice amounts. Using the model later is fast: you just follow the recipe.

| Word | Simple meaning |
| --- | --- |
| Model | Weights file + code that runs it. |
| Weights | The numbers inside a model. They hold what it learned. |
| Parameters | Another name for weights. "8B" means 8 billion of them. |
| Training | Changing the weights, little by little, until answers get good. Slow. Needs lots of compute. |
| Inference | Using a trained model to get an answer. Weights do not change. This is what we do on your computer. |
| GPU | A chip with thousands of tiny cores. It does many multiplications at the same time, so models run much faster than on the CPU. |

## Check your computer: chip and memory

Your chip and memory decide which models you can run.

**Mac** (open Terminal: Cmd + Space, type "Terminal"):

```bash
sysctl -n machdep.cpu.brand_string
sysctl -n hw.memsize | awk '{print $1/1073741824 " GB"}'
sw_vers -productVersion
```

- The first line prints the chip: `Apple M1`…`M5` = **Apple Silicon**; `Intel(R) Core(TM)…` = **Intel**.
- The second prints memory (RAM) in GB. `hw.memsize` gives bytes; `awk` divides by 1,073,741,824.
- The third prints your macOS version.

**Windows / Linux:** see "On Windows or Linux" below.

**Why memory matters so much.** To run a model, all its weights must sit in memory. On Apple Silicon, CPU and GPU share one pool of memory (**unified memory**), so your RAM is also your GPU memory. On a PC with an NVIDIA card, the model should fit in the card's own memory (VRAM). Plan on using about two-thirds of it for a model.

**Rule of thumb:** a compressed (4-bit) model needs about 0.6 GB per billion parameters, plus about 1 GB extra. You learn what "4-bit" means in Step 4.

| Your computer | Biggest chat model that runs well | Example download |
| --- | --- | --- |
| 8 GB (Apple Silicon or GPU) | 1B to 4B parameters | 1 to 3 GB |
| 16 GB | up to about 8B | about 5 GB |
| 24 to 36 GB | up to about 14B | about 9 GB |
| 48 GB or more | up to about 32B | about 20 GB |
| CPU only (Intel Mac, no GPU) | 1B to 3B, slowly | 1 to 2 GB |

## Metal and MLX vs CUDA

A GPU needs special software so programs can send it math work. NVIDIA's is **CUDA**. Apple's is **Metal**. They do the same job, but code written for one does not run on the other.

```text
  AI tool (Ollama, whisper.cpp, PyTorch...)
        |                          |
     CUDA                        Metal / MLX
        |                          |
  NVIDIA GPU                  Apple Silicon GPU
  (Windows/Linux PCs, Colab)  (M1-M5 Macs)
```

| Word | Simple meaning |
| --- | --- |
| CUDA | NVIDIA's way for programs to use its GPUs. Most AI research code is written for it first. |
| Metal | Apple's way for programs to use the Apple GPU. Ollama and whisper.cpp use it automatically on Apple Silicon. |
| MLX | Apple's own machine-learning library, built on Metal, made for unified memory. |
| Google Colab | A free website that runs Python notebooks on Google's computers, with a free NVIDIA GPU for a few hours at a time. |

Most steps run on your computer with no GPU setup. A few tools need CUDA (training in Step 15). For those we use Colab in the browser.

## Install the tools: Homebrew and Node

**Homebrew** is an app store for the Mac Terminal: `brew install x` installs a program. **Node** runs JavaScript and TypeScript outside the browser.

**1. Check what you have:**

```bash
brew --version
node --version
npm --version
```

A version number means it is installed; `command not found` means install it. Node 20 or newer is fine; 22+ is best.

**2. Install Homebrew (Mac, skip if you have it).** Homebrew is small, but it may first install Apple's Command Line Tools (about 1 to 2 GB), which also gives you `git`.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

- It asks for your Mac password (typing shows nothing; that is normal).
- At the end it prints **Next steps** with lines to copy. On Apple Silicon, run them; they add `/opt/homebrew/bin` to your PATH. Then reopen Terminal.

**3. Install Node (skip if you have 20+).** About 100 MB: `brew install node`

**4. Check again:** `node --version` and `npm --version`.

We do not install TypeScript globally. `npx tsx` runs `.ts` files directly, with no build step.

## On Windows or Linux

| Task | Windows | Linux |
| --- | --- | --- |
| Check chip and memory | Settings → System → About ("Processor", "Installed RAM"). Task Manager → Performance shows your GPU. | `lscpu` and `free -h`; `nvidia-smi` for an NVIDIA GPU |
| Install Node | Installer from [nodejs.org](https://nodejs.org), or `winget install OpenJS.NodeJS.LTS` | Package manager, or nodejs.org |
| Package manager (instead of Homebrew) | `winget` (built in) | `apt`, `dnf` or similar |
| Home folder (`~`) | `C:\Users\<you>`; use PowerShell or the VS Code terminal | `/home/<you>` |
| GPU for local models | NVIDIA GPU → CUDA (Ollama uses it automatically). Otherwise CPU: choose smaller models. | Same as Windows |

No strong GPU? Everything still runs, just slower: pick the smallest model in Step 4, and use Google Colab for Steps 15 and 15b.

## Make the folder and run step00.ts

**1. Make the project** (one line at a time; skip `mkdir`/`cd` if you cloned this repo):

```bash
mkdir ~/ai-learn
cd ~/ai-learn
npm init -y
npm pkg set type=module
npm install -D tsx typescript @types/node
```

- `npm init -y` makes `package.json`.
- `npm pkg set type=module` lets us use `import` and top-level `await`.
- `npm install -D ...` installs the TypeScript runner (`tsx`), the type checker and Node's types. About 30 MB.

**2. Type your first file,** `step00.ts`:

```ts
import os from "node:os";

const chip = os.cpus()[0].model;
const arch = os.arch();
const memoryGB = os.totalmem() / 1024 ** 3;

console.log("Chip:  ", chip);
console.log("Arch:  ", arch);
console.log("Memory:", memoryGB.toFixed(0), "GB");

const notes: string[] = [
  "Guitar: G to C chord change, 20 min, still slow",
  "Football: 50 left-foot passes against the wall",
  "MTB: rode the rocky descent twice, braked too late",
];

console.log(`I have ${notes.length} practice notes.`);
```

- `import os from "node:os"` loads Node's built-in tool for asking about the computer.
- `os.cpus()[0].model` is the CPU name, for example "Apple M2".
- `os.arch()`: `arm64` = Apple Silicon (or ARM), `x64` = Intel/AMD.
- `os.totalmem()` is memory in bytes; dividing by `1024 ** 3` gives GB.
- `notes` is the list of practice notes every later step adds AI to.

**3. Run it:** `npx tsx step00.ts`

```text
Chip:   Apple M2
Arch:   arm64
Memory: 16 GB
I have 3 practice notes.
```

On a Mac, if Chip says "Apple" but Arch says `x64`, your Node is the Intel version running through Rosetta. It works but is slower; reinstall Node.

## Try this, quiz, and your app

**Try:** add this line and run again. Does it match the table above?

```ts
console.log("Biggest 4-bit model (billions):", Math.floor((memoryGB * 0.66 - 1) / 0.6));
```

**Quiz**

1. A model is made of two parts. What are they?
2. A friend has a Windows PC with an NVIDIA card; you have an M-series Mac. Which GPU software does each use?
3. Why does a 16 GB machine run bigger models than an 8 GB one?

**Answers:** 1. A file of numbers (weights) and code that runs math with them. 2. Your friend: CUDA; you: Metal (or MLX). 3. All the weights must fit in memory at once.

**Your practice-video app:** every AI feature in it (speech to text, embeddings, summaries, vision, pose) is a different weights file plus the code that runs it, and each one has to fit in memory.
