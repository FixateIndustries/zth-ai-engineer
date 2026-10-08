# Module 10 review: Multimodal AI

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. Which of these can Claude do through its API?
a) Read a screenshot and explain it
b) Generate a PNG logo
c) Transcribe an MP3 file directly
d) Read a PDF

<details><summary>Answer</summary>

**a and d.** Claude understands images and PDFs but doesn't generate images or take audio input. For b use an image model like GPT Image or Nano Banana. For c, transcribe first. See [Beyond text](01-beyond-text.md).

</details>

### 2. Why does sending a 4K screenshot cost more than sending the same error as text?

<details><summary>Answer</summary>

Images are converted into visual tokens (patches), roughly in proportion to resolution up to a cap. A big screenshot can be thousands of tokens, while the error text might be fifty. See [Vision](02-vision.md#controlling-cost).

</details>

### 3. What's wrong with this request?

```python
content=[
    {"type": "text", "text": "Compare these two dashboards."},
    {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": a}},
    {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": b}},
]
```

<details><summary>Answer</summary>

It works, but it's not ideal. The images come after the question, and they aren't labelled, so the model (and your follow-up questions) can't say "dashboard 1" unambiguously. Put `"Image 1:"` / `"Image 2:"` text blocks before each image, and the question last. See [Vision](02-vision.md#images-are-just-another-content-block).

</details>

### 4. You need to find the moment in a 45-minute recorded lecture where the speaker first mentions "embeddings". Name two approaches.

<details><summary>Answer</summary>

(1) Transcribe with timestamps (`whisper-1` with `verbose_json` and segment granularity) and search the transcript. (2) Send the video to a native video model such as Gemini and ask for the timestamp. The first is cheaper and easier to verify. See [Audio](04-audio.md) and [Vision](02-vision.md#video-frames-or-a-model-that-watches).

</details>

### 5. True or false: for a voice assistant, a speech-to-text → LLM → text-to-speech pipeline is always worse than a native speech-to-speech model.

<details><summary>Answer</summary>

**False.** The pipeline is slower and loses tone, but every step is inspectable, testable and swappable, and you can put guardrails on the transcript. Many teams start there. See [Beyond text](01-beyond-text.md#two-ways-to-build-native-or-pipeline).

</details>

### 6. Your transcripts keep turning "Chroma" into "chroma key" and "MCP" into "MSP". What's the cheapest fix?

<details><summary>Answer</summary>

Pass a `prompt` to the transcription call that names the domain and lists the unusual terms. Check the audio quality next. See [Audio](04-audio.md#speech-to-text-with-openai).

</details>

### 7. A designer wants 50 banner variations featuring the same mascot. How do you keep it consistent?

<details><summary>Answer</summary>

Use a reference image of the mascot with the edit endpoints (`images.edit` for GPT Image, or image input with Nano Banana) instead of 50 independent text-only generations. Change one thing per edit. See [Image generation](03-image-generation.md).

</details>

### 8. Your extraction prompt says "return the total as a number". On a blurry receipt the model returns `42.0`, but the real total is `47.50`. What two changes reduce this risk?

<details><summary>Answer</summary>

Tell the model to return `null` for unreadable fields instead of guessing, and validate the output with a schema, routing nulls or low-confidence results to a human. Better input (resolution, cropping) helps too. See [Vision](02-vision.md#from-pixels-to-json).

</details>

### 9. In multimodal RAG with the "describe, then embed" strategy, what do you send to the LLM at answer time: the description or the image?

<details><summary>Answer</summary>

The original image (plus the question). The description exists so you can *find* the image. The image holds details the description may have dropped. See [Multimodal pipelines](05-multimodal-pipelines.md#multimodal-rag).

</details>

### 10. You need to tag 2 million product photos into 12 fixed categories every night. Frontier vision LLM, or something else?

<details><summary>Answer</summary>

Something else first: a local zero-shot classifier like CLIP (or a small fine-tuned model) through Hugging Face pipelines. Send only low-confidence cases to a frontier model. That's the cascade pattern from [lesson 5](05-multimodal-pipelines.md#specialists-still-matter).

</details>

### 11. Name one new security risk that appears when your app accepts images.

<details><summary>Answer</summary>

Prompt injection hidden in the image: text in a screenshot or photo saying "ignore previous instructions…". The model reads it just like retrieved text. Treat image content as untrusted input. See [Module 13](../13-safety-and-ethics/02-prompt-injection.md).

</details>

## Hands-on challenge

Build a tiny "explain this screenshot" CLI: `uv run explain.py path/to/image.png "optional question"`. It should:

1. Resize the image so its long edge is at most 1568 px before sending.
2. Send it to Claude with the question (default: "What is this, and what should I do about it?").
3. Print the answer and the input token count.
4. Bonus: accept `--voice question.m4a` and transcribe the question instead of reading it from the command line.

<details><summary>Hint</summary>

Use Pillow: `img.thumbnail((1568, 1568))` keeps the aspect ratio. Save it to a `BytesIO` as PNG, base64 it, and reuse `image_block()` logic from [lesson 2](02-vision.md). For the bonus, call `client.audio.transcriptions.create(...)` from [lesson 4](04-audio.md) and use `transcript.text` as the question.

</details>

## Capstone checkpoint
- [ ] Course Buddy accepts an image path (e.g. `--image shot.png`) alongside or instead of a text question
- [ ] Images are resized before sending, and token usage for image questions is logged
- [ ] It can explain a screenshot of a mermaid diagram from the course and point to the lesson it likely came from (using your M7/M8 retrieval with the model's description of the image as the query)
- [ ] It handles a screenshot of an error from its own CLI and suggests a fix
- [ ] Stretch: `--voice question.m4a` transcribes the question with speech-to-text (with a `prompt` listing course terms) before answering

---
[← Last lesson](05-multimodal-pipelines.md) · [Course home](../README.md) · [Next module →](../11-evals/README.md)
