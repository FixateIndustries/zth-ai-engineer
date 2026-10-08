# Module 2 review: Calling Models

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. Your chatbot forgets the user's name between turns. The code calls `client.messages.create` with only the latest user message. Why does it forget?

<details><summary>Answer</summary>

The API is stateless. The model only sees what's in the current request, so you have to resend the whole conversation (user and assistant turns) every time. See [Your first API call](01-your-first-api-call.md#multi-turn-you-own-the-history).

</details>

### 2. What's wrong with this code?

```python
response = client.messages.create(model="claude-sonnet-5-5", max_tokens=16000, messages=msgs)
print(response.content[0].text)
```

<details><summary>Answer</summary>

`content` is a list of typed blocks, and the first one isn't guaranteed to be text. Current Claude models can return a thinking block first, and tool calls come back as `tool_use` blocks. Filter with `b.type == "text"`. It also ignores `stop_reason`, so a truncated or refused answer would look normal. See [lesson 1](01-your-first-api-call.md#claude-the-messages-api).

</details>

### 3. Which parameter holds the system prompt in each SDK?
a) Claude  b) OpenAI Responses  c) Gemini `generate_content`  d) OpenAI Chat Completions

<details><summary>Answer</summary>

a) `system=`, b) `instructions=`, c) `config.system_instruction`, d) a message with `"role": "system"`. Same idea, four spellings. See the [side-by-side table](01-your-first-api-call.md#side-by-side).

</details>

### 4. You want to try a Llama model on your laptop and a DeepSeek model in the cloud without writing new client code. What two settings change between them?

<details><summary>Answer</summary>

The `base_url` (for example `http://localhost:11434/v1` for Ollama, `https://openrouter.ai/api/v1` for OpenRouter) and the API key, plus the model ID string. That's the point of [OpenAI-compatible APIs](02-openai-compatible-apis.md).

</details>

### 5. True or false: if a compatible endpoint doesn't support a parameter, it returns an error.

<details><summary>Answer</summary>

False, often. Many compatibility layers silently ignore unsupported fields. Anthropic's ignores `response_format` and tool `strict`, for example. That's why you validate outputs instead of trusting the request. See [What "compatible" leaves out](02-openai-compatible-apis.md#what-compatible-leaves-out).

</details>

### 6. After you add streaming, users say the app feels much faster, but total request time is unchanged. Explain.

<details><summary>Answer</summary>

Streaming cuts time to first token: users see words almost immediately instead of waiting for the full answer. Generation speed is the same. See [Streaming responses](03-streaming-responses.md#why-stream).

</details>

### 7. Your streaming chat prints answers in big clumps instead of word by word. What's the most likely fix?

<details><summary>Answer</summary>

Add `flush=True` to `print(text, end="")`. Python buffers stdout, so without flushing the chunks pile up before they appear. See [lesson 3](03-streaming-responses.md#claude).

</details>

### 8. Match each to its guarantee: (1) "respond in JSON" in the prompt, (2) JSON mode, (3) structured output with a schema.
a) Valid JSON of any shape  b) No guarantee  c) Valid JSON matching your schema

<details><summary>Answer</summary>

1-b, 2-a, 3-c. Structured output uses constrained decoding so the model can only produce tokens that fit your schema. See [Structured output](04-structured-output.md#why-just-ask-for-json-isnt-enough).

</details>

### 9. You use Claude's `messages.parse()` with a Pydantic model, yet one response in a thousand fails. Name two likely causes.

<details><summary>Answer</summary>

The response hit `max_tokens` and was truncated, or the model refused (`stop_reason == "refusal"`). Both produce output that doesn't match your schema. Check the stop reason. See [What can still go wrong](04-structured-output.md#what-can-still-go-wrong).

</details>

### 10. In function calling, who executes the function?

<details><summary>Answer</summary>

Your code. The model only emits a request (a tool name plus arguments). You decide whether to run it, run it, and send back a result with the matching ID. See [Function calling](05-function-calling.md#the-model-asks-your-code-acts).

</details>

### 11. A model calls `delete_user` with `{"user_id": "42"}`. The tool has `strict: true`. Is it safe to run the deletion immediately?

<details><summary>Answer</summary>

No. `strict` guarantees the arguments match the schema, not that the action is wanted or safe. Treat arguments as untrusted input, check permissions, and require human confirmation for destructive actions. See [Keep it safe](05-function-calling.md#keep-it-safe).

</details>

### 12. Your tool lookup throws an exception for a bad ID. What should you send back to the model?

<details><summary>Answer</summary>

A `tool_result` (or `function_call_output`) carrying a helpful error message, with `is_error: True` for Claude. Don't drop the result or crash: the model can often correct its input or explain the problem. See [lesson 5](05-function-calling.md#one-round-trip-with-claude).

</details>

## Hands-on challenge

Build a tiny "unit converter" assistant. Give Claude one tool, `convert(value: float, from_unit: str, to_unit: str)`, that supports a handful of length units, and stream the final answer to the terminal. Ask it "How many feet is 3.5 meters, and how many inches is that?" It should call the tool twice (in parallel or in sequence). Print each tool call as it happens.

<details><summary>Hint</summary>

Use `Literal` enums for the units in the schema so the model can't invent `"metres"`. Since you don't know how many rounds the model needs, wrap the tool round trip from [lesson 5](05-function-calling.md) in a `while response.stop_reason == "tool_use":` loop. That's your first taste of an agent loop. For the final answer, swap `messages.create` for `messages.stream` and print `text_stream`. Return unsupported unit pairs as `is_error` results.

</details>

## Capstone checkpoint
- [ ] `course-buddy chat` runs a terminal chat loop that keeps conversation history across turns
- [ ] Answers stream to the terminal as they're generated, and history stores the final message
- [ ] A `--json` flag returns a validated `{answer, confidence}` object (Pydantic model, printed as JSON) instead of prose
- [ ] Failures are handled: a bad key gives a clear message, and a `max_tokens` or `refusal` stop reason is reported instead of crashing the JSON mode

---
[← Last lesson](05-function-calling.md) · [Course home](../README.md) · [Next module →](../03-choosing-models/README.md)
