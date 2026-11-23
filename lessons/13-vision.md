# Step 13: Vision

You try three kinds of vision AI: a vision LLM that describes a picture, an object detector that draws boxes, and a pose model that finds body joints.

## The idea

A **vision LLM** (**multimodal**) takes an image plus text and writes text: great for describing, slow and weak at exact details. **Object detection** outputs **bounding boxes** with labels and **confidence**: fast and exact, but only for object types it was trained on. A **pose model** finds **keypoints** (joints) on a person: ideal for counting reps.

```text
                     ┌─► vision LLM      ──► "A person juggling a football in a park."
photo (pixels) ──────┼─► object detector ──► person 0.99 [x 120..300, y 40..460]
                     │                      sports ball 0.94 [x 210..250, y 380..420]
                     └─► pose model      ──► 17 keypoints: nose (215,70), left knee (180,350)...
```

| Word | Simple meaning |
| --- | --- |
| Vision LLM / multimodal | Reads images and text, writes text. |
| Object detection | Finding objects with a box and label each. |
| Bounding box | A rectangle (xmin, ymin, xmax, ymax) in pixels. |
| Confidence | How sure the model is, 0 to 1. |
| Pose / keypoints | Points on the body (joints). |
| ONNX | A standard model format; ONNX Runtime runs it anywhere. |

## Part A: a vision LLM describes a photo

Ollama with `gemma3:4b`. Put a practice photo as `photo.jpg`. Make `step13a.ts`:

```ts
import fs from "node:fs";

const image = fs.readFileSync("photo.jpg").toString("base64");

const res = await fetch("http://localhost:11434/api/chat", {
  method: "POST",
  body: JSON.stringify({
    model: "gemma3:4b",
    messages: [{ role: "user", content: "Describe this practice photo in 2 sentences. What sport, and what is the person doing?", images: [image] }],
    stream: false,
  }),
});
console.log((await res.json()).message.content);
```

- **base64** writes binary data as text so it fits in JSON.

**Try:** "How many cones are in the picture?" Vision LLMs are often wrong at counting.

## Part B: object detection with boxes

Make `step13b.ts` (Transformers.js from Step 8; first run downloads the model):

```ts
import { pipeline } from "@huggingface/transformers";

const detector = await pipeline("object-detection", "Xenova/detr-resnet-50");
const results = await detector("photo.jpg", { threshold: 0.8 });

for (const r of results as any[]) {
  const b = r.box;
  console.log(`${r.label.padEnd(12)} ${r.score.toFixed(2)}  box x ${b.xmin}..${b.xmax}, y ${b.ymin}..${b.ymax}`);
}
```

```text
person       0.99  box x 118..302, y 41..458
sports ball  0.94  box x 209..251, y 379..421
```

- Labels come from the 80 everyday object types in the **COCO** dataset. "Guitar" is not one of them: detectors only find what they were trained on.

## Part C: pose on Google Colab

Open [Google Colab](https://colab.research.google.com), new notebook, Runtime → Change runtime type → T4 GPU (NVIDIA, via **CUDA**).

```python
!pip install ultralytics
```

```python
from ultralytics import YOLO

model = YOLO("yolo11n-pose.pt")
results = model("https://ultralytics.com/images/bus.jpg")
results[0].show()
print(results[0].keypoints.xy[0])
```

- `keypoints.xy` gives 17 (x, y) points per person. Upload your own photo with the folder icon.
- **Counting reps:** run on each video frame (`model("clip.mp4", stream=True)`), follow one keypoint (right wrist = index 10), count ups and downs.

## What each is good at

| | Vision LLM | Object detection | Pose |
| --- | --- | --- | --- |
| Output | Text | Boxes + labels | Body keypoints |
| Speed | Slow | Fast | Fast |
| Exact positions | No | Yes | Yes |
| Knows any object | Most things | Only trained types | People only |
| Best for | Describing, tagging | Finding/following objects | Reps, form |

## Quiz

1. Counting juggling touches in a 2-minute video: vision LLM or pose model?
2. Why can the detector not find a guitar?
3. What does confidence 0.94 mean?

**Answers:** 1. Pose (or detection): fast, exact, every frame. 2. Not a trained type. 3. 94% sure.

## Your practice-video app

Vision models for frames; pose and object detection (ONNX) for counting reps.

## Learn more

- [Ollama vision](https://docs.ollama.com/capabilities/vision)
- [Object detection](https://docs.ultralytics.com/tasks/detect/) (Ultralytics)
- [Pose landmarker](https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker) (MediaPipe, works in the browser)
