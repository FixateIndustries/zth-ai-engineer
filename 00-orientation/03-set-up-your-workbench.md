# Set up your workbench

> Module 0 · Lesson 3 · ~20 min

**Goals**
- Create a Python project with `uv`
- Get an API key and keep it out of your code with a `.env` file
- Make your first call to Claude and read the response
- (Optional) Run a model locally with Ollama

## 1. Install uv

[`uv`](https://docs.astral.sh/uv/) is a fast Python package and project manager. It installs Python for you, creates virtual environments automatically, and keeps a lockfile. The whole course uses it.

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
# or: brew install uv

uv --version
```

On Windows, see the [installation page](https://docs.astral.sh/uv/getting-started/installation/).

## 2. Create the Course Buddy project

This is the capstone project you'll grow through the whole course.

```bash
uv init course-buddy --python 3.12
cd course-buddy
uv add anthropic python-dotenv
```

`uv init` gives you a `pyproject.toml`, a `main.py`, and a git repo. `uv add` installs packages into a project-local virtual environment and records them in `pyproject.toml` and `uv.lock`. You never activate the environment by hand. You run things with `uv run`.

## 3. Get an API key and store it safely

Create a key in the [Claude Console](https://platform.claude.com/docs/en/get-started) (the quickstart walks you through it). You'll need to add a small amount of credit. The exercises in this course cost cents, not dollars.

Put the key in a file called `.env` in the project root:

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-...
```

Then make sure git never sees it:

```bash
echo ".env" >> .gitignore
git status   # .env should NOT appear in the list
```

> [!WARNING]
> Never paste an API key into source code, a notebook, or a chat message. Keys committed to a public repo get scraped within minutes. If one leaks, revoke it in the console immediately and make a new one.

Later modules also use `OPENAI_API_KEY` and `GEMINI_API_KEY`. Add them to the same `.env` when you get there. You don't need them yet.

## 4. Hello, model

Replace the contents of `main.py`:

```python
from dotenv import load_dotenv
import anthropic

load_dotenv()                 # reads .env into environment variables
client = anthropic.Anthropic()  # picks up ANTHROPIC_API_KEY automatically

response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "In one sentence, what is an AI engineer?"}
    ],
)

# The response holds a list of content blocks. Print the text ones.
for block in response.content:
    if block.type == "text":
        print(block.text)

print("---")
print("stop reason:", response.stop_reason)
print("input tokens:", response.usage.input_tokens)
print("output tokens:", response.usage.output_tokens)
```

Run it:

```bash
uv run main.py
```

You should see a one-sentence answer, followed by the stop reason (`end_turn` means the model finished on its own) and the token counts for the request. Those token counts are what you're billed for. [Module 1](../01-how-llms-work/README.md) explains what a token is.

<details><summary>Getting an error?</summary>

- `AuthenticationError` (401): the key is wrong or wasn't loaded. Check that `.env` is in the folder you ran the command from and that the variable name is exactly `ANTHROPIC_API_KEY`.
- An error mentioning your credit balance: add credit in the console.
- `NotFoundError` (404) mentioning the model: check the model ID for typos.
- `ModuleNotFoundError`: you ran `python main.py` instead of `uv run main.py`, so you're outside the project's environment.

</details>

## 5. (Optional) Run a model locally with Ollama

[Ollama](https://ollama.com) runs open-weight models on your own machine. It's free, private, and works offline, and you'll use it in [Module 3](../03-choosing-models/README.md). Install it from the website, then:

```bash
ollama run llama3.2 "In one sentence, what is an AI engineer?"
```

The first run downloads a few gigabytes. Small local models are noticeably weaker than Claude, which is a useful thing to feel for yourself early.

> [!TIP]
> **Try it:** Make three changes to `main.py`, one at a time, and run after each:
> 1. Ask the same question three times in a row. Is the wording identical each time?
> 2. Change `max_tokens` to `10`. What's the `stop_reason` now, and what happened to the answer?
> 3. Swap the model to `claude-haiku-5-5`. Compare speed and answer quality.
>
> <details><summary>What you should notice</summary>
>
> 1. The wording usually varies. Models sample their output, which [Lesson 1.4](../01-how-llms-work/04-sampling-knobs.md) explains.
> 2. The answer gets cut off mid-sentence and `stop_reason` is `max_tokens`. Always check the stop reason in real code.
> 3. Haiku is faster and cheaper. For a simple question you may not see much quality difference, and that's exactly the trade-off [Module 3](../03-choosing-models/README.md) is about.
>
> </details>

## Key takeaways
- `uv init`, `uv add`, and `uv run` are all you need to manage a Python project in this course.
- API keys live in `.env`, which is listed in `.gitignore`. Code reads them from the environment.
- A Messages API response contains content blocks, a `stop_reason`, and `usage` token counts. Check all three.
- Ollama lets you run open models locally for free, with lower quality than frontier models.

## Go deeper
- [uv documentation](https://docs.astral.sh/uv/) — projects, dependencies, and running scripts.
- [Claude API quickstart](https://platform.claude.com/docs/en/get-started) — account setup, keys, and a first request in several languages.
- [python-dotenv](https://pypi.org/project/python-dotenv/) — how `.env` loading works and its options.
- [Ollama quickstart](https://docs.ollama.com/quickstart) — installing Ollama and running your first local model.

---
[← The vocabulary](02-the-vocabulary.md) · [Module home](README.md) · [Module review →](review.md)
