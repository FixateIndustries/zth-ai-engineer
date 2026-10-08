# Module 4 review: Prompt Engineering

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. Which of these belongs in the system prompt rather than the user message?
a) The document the user uploaded
b) "Always answer in British English and never give legal advice."
c) The user's question
d) Search results retrieved for this request

<details><summary>Answer</summary>

**b.** Standing rules and behavior that apply to every request go in the system prompt. Per-request data (documents, questions, retrieved results) goes in the user message. See [Anatomy of a prompt](01-anatomy-of-a-prompt.md).

</details>

### 2. What's wrong with this prompt?

```text
Here is a contract. IMPORTANT!!! DO NOT MISS ANY RISKS!!! Summarize it.
Party A agrees to deliver... [3 pages of contract text] ...
```

<details><summary>Answer</summary>

Several things: no role or audience, no definition of "risk" or what the summary should contain, no length or output format, the contract isn't delimited from the instructions, and the instruction comes before a long document instead of after it. The all-caps shouting also tends to cause overcorrection on current models. A better version wraps the text in `<contract>` tags, says who the summary is for, defines what counts as a risk, specifies a format, and puts the request after the document. See [Anatomy of a prompt](01-anatomy-of-a-prompt.md).

</details>

### 3. Why wrap input data in tags like `<ticket>…</ticket>`?

<details><summary>Answer</summary>

So the model can tell instructions apart from content. Without delimiters, text inside the data ("ignore the above and...") or simply its formatting can blur into your instructions. Tags also let you refer to the data precisely ("the ticket in the `<ticket>` tags"). See [Anatomy of a prompt](01-anatomy-of-a-prompt.md).

</details>

### 4. Your few-shot examples are all one-sentence reviews, and now the model writes one-sentence answers even for long, complex reviews. What happened and how do you fix it?

<details><summary>Answer</summary>

The model copied an unintended pattern from the examples (their length). Vary the examples, including longer inputs with appropriately longer outputs, so length isn't something it learns to copy. See [Zero-shot and few-shot prompting](02-zero-shot-and-few-shot.md).

</details>

### 5. True or false: adding few-shot examples is a good way to teach the model facts about your company's products.

<details><summary>Answer</summary>

**False.** Examples teach format, style, and judgment. Facts should be provided as context in the prompt, or retrieved with RAG. See [Zero-shot and few-shot prompting](02-zero-shot-and-few-shot.md).

</details>

### 6. In a chain-of-thought prompt, why should the reasoning come *before* the answer?

<details><summary>Answer</summary>

The model generates tokens in order, and each token builds on what came before. Reasoning written first becomes context the answer can use. Reasoning written after the answer can only justify a conclusion that's already been committed to. See [Reasoning patterns](03-reasoning-patterns.md).

</details>

### 7. You're using a current model with built-in reasoning for a high-volume intent classifier. Responses are slow and expensive. What's the first knob to try?

a) Add "think step by step" to the prompt
b) Lower the effort level
c) Add more few-shot examples
d) Switch to ReAct prompting

<details><summary>Answer</summary>

**b.** Classification rarely needs deep reasoning. Lowering effort cuts thinking tokens, latency, and cost. Adding "think step by step" does the opposite. See [Reasoning patterns](03-reasoning-patterns.md).

</details>

### 8. What problem did native tool calling solve compared to text-based ReAct prompting?

<details><summary>Answer</summary>

Text-based ReAct requires parsing the model's free-text `Action:` lines, which is brittle: the format drifts, and the model can even invent its own observations. Native tool calling returns structured, schema-checked tool requests, so your code doesn't parse prose. See [Reasoning patterns](03-reasoning-patterns.md).

</details>

### 9. Why might you choose `string.Template` over `str.format()` for prompt templates?

<details><summary>Answer</summary>

Prompts often contain literal curly braces (JSON examples), which `str.format()` treats as placeholders and forces you to escape. `string.Template` uses `$name` placeholders, and `substitute()` raises an error on a missing variable. See [Prompts as code](04-prompts-as-code.md).

</details>

### 10. A teammate edits the system prompt to fix one user's complaint, runs it on that one question, and ships. A week later, refusals for off-topic questions have stopped working. What process would have caught this?

<details><summary>Answer</summary>

A versioned prompt plus a test set that includes off-topic cases, run on every change. The fix should also have been preceded by adding the complaint as a new test case. Change one thing, run the full set, compare against the previous version. See [Prompts as code](04-prompts-as-code.md).

</details>

## Hands-on challenge

Build a "commit message linter" that takes a git diff summary and a proposed commit message, and returns JSON: `{"ok": bool, "problems": [str], "suggestion": str}`. Follow your team's convention (for example, Conventional Commits with a 50-character subject).

1. Write v1 zero-shot, with the prompt in `prompts/lint.v1.md`.
2. Create 10 test cases (good messages, too-long subjects, wrong type prefix, vague descriptions like "fix stuff").
3. Write v2 with a clear role, explicit rules with reasons, and three few-shot examples aimed at v1's failures.
4. Report the pass rate for both versions.

<details><summary>Hint</summary>

Some checks (subject length, type prefix) are deterministic, so check them in code in your test script rather than trusting the model's judgment. The model's job is the fuzzy part: is the description vague? Is the type right for this diff? Splitting the work this way is a pattern you'll use constantly.

</details>

## Capstone checkpoint
- [ ] Course Buddy's system prompt lives in a versioned file (e.g. `prompts/system.v1.md`), loaded with a template
- [ ] The prompt defines Course Buddy's role, audience, rules (stay on topic, don't invent lesson names), and answer format
- [ ] Three to five few-shot examples demonstrate its voice and answer format
- [ ] The prompt version is logged or printed with each answer
- [ ] A small test set (10+ questions) runs against the prompt, and you've compared at least two versions

---
[← Last lesson](04-prompts-as-code.md) · [Course home](../README.md) · [Next module →](../05-context-engineering/README.md)
