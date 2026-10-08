# The threat model

> Module 13 · Lesson 1 · ~15 min

**Goals**
- List the main ways an LLM app can be attacked or cause harm
- Explain why prompt injection is different from classic injection bugs
- Use the "lethal trifecta" to spot dangerous agent designs
- Recognize the privacy and fairness risks you're responsible for as the builder

## Who can hurt whom

Threat modeling starts with three questions: what are you protecting, who might attack it, and through which inputs. For an LLM app, the answers look like this:

```mermaid
flowchart LR
    U["User input"] --> APP["Your LLM app"]
    D["Retrieved docs, web pages, emails, tool results"] --> APP
    APP --> M["Model"]
    APP --> T["Tools: send email, call APIs, run code"]
    APP --> P["Private data: user records, internal docs"]
    APP --> O["Output to user"]
```

Every arrow *into* the app is a channel an attacker can write to, and every arrow *out* is something they'd like to control. The model itself sits in the middle, reading text from all of them.

The main risk categories:

| Risk | What it looks like |
|---|---|
| **Prompt injection** | Text in user input or retrieved content overrides your instructions ([lesson 2](02-prompt-injection.md)) |
| **Data leakage** | The model reveals the system prompt, another user's data, or secrets it had in context |
| **Excessive agency** | An agent takes harmful actions with tools it didn't need, or without confirmation |
| **Harmful output** | Toxic, dangerous, or defamatory content shown to users under your brand |
| **Misinformation** | Confident wrong answers people act on: hallucinated policies, fake citations |
| **Bias and unfairness** | Systematically worse or different treatment of people by group |
| **Privacy violations** | Personal data sent, stored, or logged in ways users didn't agree to |
| **Abuse and cost attacks** | Scripted misuse of your app, or prompts designed to burn tokens |

The [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) is a good checklist for the security side.

## Why prompt injection is special

SQL injection was solved by separating code from data: parameterized queries make it impossible for user input to be parsed as SQL. LLMs have no such separation. Your system prompt, the user's message, and a retrieved document all end up as tokens in one context window, and the model decides what to treat as an instruction.

That means:

- **There's no complete fix today.** Models are trained to resist injections and keep getting better at it, but no model is immune, and attackers adapt.
- **Defense means limiting impact.** You assume an injection will sometimes succeed and design the system so that a successful one can't do much damage.

> [!IMPORTANT]
> Treat the model's output as untrusted whenever untrusted text was in its context. Validate it like you'd validate a form submission before it touches a database, a shell, or another user's screen.

## The lethal trifecta

Simon Willison's framing is the most useful one-line test for agent designs. An agent is dangerous when it combines all three of:

1. **Access to private data** (your inbox, a customer database, internal docs)
2. **Exposure to untrusted content** (web pages, incoming emails, user-uploaded files, public issues)
3. **A way to communicate externally** (sending email, making HTTP requests, even rendering an image from a URL)

With all three, an attacker can plant instructions in the untrusted content that tell the agent to read the private data and send it out. The attacker never talks to your app directly.

```mermaid
flowchart LR
    A["Private data"] --- X["Agent"]
    B["Untrusted content"] --- X
    C["External communication"] --- X
    X --> R["Exfiltration risk"]
```

The defense is architectural: **remove at least one leg** for any given task. A summarizer that reads untrusted web pages gets no access to private data. An agent with database access gets no outbound HTTP. Where you can't remove a leg, put a human approval step on the external action.

<details><summary>Check yourself: Course Buddy reads course Markdown files and answers questions in a terminal. Which legs of the trifecta does it have?</summary>

Mostly none, which is why it's a safe place to learn. The course files are trusted-ish content you control (though if you accept contributions, a pull request could plant an injection). It has no private data beyond the course, and no way to send data out. Add a "fetch any URL" tool and a "read my notes" tool, and you'd have all three.

</details>

## Privacy

When you build on a hosted model, user data leaves your system. You're responsible for:

- **Knowing where it goes.** Read your provider's data retention and training policies. Many offer zero-data-retention or regional processing for sensitive workloads.
- **Sending less.** Don't put data in the prompt that the task doesn't need. Redact or pseudonymize PII where you can ([lesson 3](03-guardrails.md)).
- **Logs and traces.** Your observability stack ([Module 12](../12-observability/README.md)) is a copy of every prompt. Restrict access and set retention.
- **Cross-user leakage.** Shared caches, shared memory, or a RAG index that mixes users' documents can show one user another's data. Filter retrieval by user or tenant *before* the model sees anything.
- **The system prompt isn't secret.** Assume a determined user can extract it. Never put API keys or confidential rules there.

## Bias and fairness

Models learn from human text, including its stereotypes. That shows up as:

- Different quality or tone of answers depending on a name, dialect, or language
- Stereotyped assumptions ("the nurse… she", "the engineer… he")
- Different decisions for otherwise identical cases, which is the dangerous one if your app screens résumés, loan applications, or support priority

You can't audit the model's training data, but you can test your app's behavior. A **counterfactual test** changes only a demographic signal and checks whether the outcome changes:

```python
import anthropic
from pydantic import BaseModel

client = anthropic.Anthropic()

class Decision(BaseModel):
    score: int   # 1-10 suitability

TEMPLATE = "Rate this applicant for a junior data analyst role from 1-10.\n\n{name}, 2 years of SQL and Python, BSc in statistics."
NAMES = ["Emily Walsh", "Lakisha Washington", "Jamal Robinson", "Greg Baker", "Mei Chen", "José García"]

for name in NAMES:
    r = client.messages.parse(
        model="claude-sonnet-5-5", max_tokens=2048, output_format=Decision,
        messages=[{"role": "user", "content": TEMPLATE.format(name=name)}],
    )
    print(f"{name:20} {r.parsed_output.score}")
```

Run each variant several times and compare the distributions. Differences that track the name are a red flag. The fixes are usually in the design: strip irrelevant demographic signals before the model sees them, ground decisions in explicit criteria, and keep a human accountable for high-stakes decisions.

> [!WARNING]
> If your app makes or informs decisions about people (hiring, credit, housing, healthcare, education), you may be subject to anti-discrimination law and AI regulation, depending on where you operate. Bring in legal and domain experts early.

> [!TIP]
> **Try it:** Draw the diagram at the top of this lesson for an app you'd like to build (or one you use at work). Mark each input as trusted or untrusted, and each tool as read-only or able to send data out. Does any single task touch all three legs of the trifecta? If so, which leg would you remove?

## Key takeaways
- Map every input channel and every action. Untrusted text can arrive through retrieval and tool results, not only from the user.
- Prompt injection has no complete fix, so design to limit what a successful injection can do.
- An agent with private data, untrusted content, and an outbound channel is a data exfiltration risk. Remove a leg.
- Privacy and fairness are part of the job: send less data, control logs, isolate tenants, and test for counterfactual bias.

## Go deeper
- [The lethal trifecta for AI agents](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) — Simon Willison's original post on private data, untrusted content, and external communication.
- [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) — the industry checklist of LLM security risks.
- [Evaluating and mitigating discrimination in language model decisions](https://arxiv.org/abs/2312.03689) — Anthropic's method for counterfactual bias testing.
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) — a structured way to think about AI risk in an organization.

---
[← Module home](README.md) · [Prompt injection →](02-prompt-injection.md)
