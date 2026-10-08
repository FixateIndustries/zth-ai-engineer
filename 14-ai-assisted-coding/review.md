# Module 14 review: AI-Assisted Coding

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. Match each tool to its primary category: Cursor, Gemini CLI, Devin, Replit Agent.
Categories: a) IDE assistant b) terminal agent c) autonomous/cloud agent d) app builder

<details><summary>Answer</summary>

Cursor → a, Gemini CLI → b, Devin → c, Replit Agent → d. Many tools span several modes now, so classify by how you're using them. See [The AI coding landscape](01-the-ai-coding-landscape.md#four-categories).

</details>

### 2. What four questions let you compare any coding tool, even one released next month?

<details><summary>Answer</summary>

Where it runs, how much it does before checking in (autonomy), what context it gets (files, instruction files, MCP), and which models it uses. See [lesson 1](01-the-ai-coding-landscape.md#theyre-agents-and-you-know-how-agents-work).

</details>

### 3. Your team uses Claude Code, Codex and Cursor. Where do you put the shared project instructions?

<details><summary>Answer</summary>

In `AGENTS.md`, which is the cross-tool convention, read by Codex and Cursor and also by Claude Code. Keep any tool-specific files (`CLAUDE.md`, `.cursor/rules`) thin. See [Working with AI pair programmers](02-working-with-ai-pair-programmers.md#context-files-onboarding-in-a-markdown-file).

</details>

### 4. What's wrong with this context file?

```markdown
# Project
This is a Python project. Write good, clean code. Follow best practices.
Be careful. Don't make mistakes. Use descriptive variable names.
```

<details><summary>Answer</summary>

Nothing in it is project-specific or actionable. It has no commands (install, test, lint), no layout, no conventions the agent would otherwise get wrong. It just costs tokens. Replace it with concrete facts: how to run tests, where things live, and the rules that are unusual in this repo.

</details>

### 5. You ask an agent to "add caching to the app". It returns a 1,200-line diff touching 30 files. What should you have done differently?

<details><summary>Answer</summary>

Explore and plan first, then implement in small steps. Specify what to cache and where, ask for a plan, and approve one small diff at a time with tests. Large diffs can't be reviewed properly. See [the loop](02-working-with-ai-pair-programmers.md#the-loop-explore-plan-implement-verify).

</details>

### 6. Why write (or approve) the test *before* letting the agent implement a feature?

<details><summary>Answer</summary>

The test becomes an executable spec you've checked yourself. The agent can then verify its own work in the loop, and you aren't relying on tests written to fit whatever code it produced.

</details>

### 7. CI is green on an agent's PR, but a test now uses `pytest.mark.skip`. What happened, and what do you do?

<details><summary>Answer</summary>

The agent likely sidestepped a failing test instead of fixing the code. Remove the skip, restore the test, and ask it to make the real test pass. Always diff tests as carefully as code. See [Tests as guardrails](02-working-with-ai-pair-programmers.md#tests-as-guardrails).

</details>

### 8. The agent adds `uv add fastjsonx-utils` to fix a bug. Name two things to check before accepting.

<details><summary>Answer</summary>

(1) Does the package actually exist, and is it the legitimate, maintained one? Hallucinated or typo-squatted names are a supply-chain risk. (2) Is a new dependency needed at all, or does the standard library or an existing dependency already do it?

</details>

### 9. Rank these by how much scrutiny the AI-written change needs, lowest to highest: a) a database migration that drops a column b) test fixtures for an existing function c) a change to the permission check on an admin endpoint.

<details><summary>Answer</summary>

b (verifiable, low risk) → c (security-critical) → a (security-adjacent *and* hard to reverse, since data loss is permanent). You could argue c and a either way. The point is that both deserve careful human review. See [When to trust it](02-working-with-ai-pair-programmers.md#when-to-trust-it-and-how-much).

</details>

### 10. You clone an unfamiliar open-source repo and start a terminal agent in it with auto-approve on. Why is that risky?

<details><summary>Answer</summary>

The repo's `AGENTS.md`, READMEs, issues or code comments could contain instructions that the agent follows, which is indirect prompt injection. With auto-approve, those instructions could run shell commands on your machine. Keep approvals on, or use a sandbox. See [Module 13](../13-safety-and-ethics/02-prompt-injection.md).

</details>

## Hands-on challenge

Pair with a coding agent to add a **"quiz me"** command to Course Buddy: `course-buddy quiz 07-rag` reads a module's lessons and asks you three questions, then grades your answers.

Do it the disciplined way:
1. Write or update `AGENTS.md` (or `CLAUDE.md`) first.
2. Ask the agent to explore the code and propose a plan. Edit the plan.
3. Have it write failing tests (mock the model call). Review them.
4. Implement in two or three small commits, running tests and lint each time.
5. Review the final diff with the checklist from lesson 2, then ask a fresh agent session to review it too.

<details><summary>Hint</summary>

Reuse what you already have. `read_lesson` and `list_modules` from M8 give you the content, and your M2 `--json` structured output can return `{questions: [...]}`. Keep the grader prompt separate and versioned in `prompts/`, as in M4. Unit tests can assert the command reads the right files and handles an unknown module name, without calling a real model.

</details>

## Capstone checkpoint
- [ ] `course-buddy` has an `AGENTS.md` and/or `CLAUDE.md` with install, run, test and lint commands, a layout map and key conventions
- [ ] You used an AI coding assistant to refactor one module (for example, tidying `providers.py`) in small, reviewed commits
- [ ] You shipped one new feature (such as quiz mode) with the agent, with tests written or approved before implementation
- [ ] Fast unit tests run offline (model calls mocked), separate from the paid eval suite from M11
- [ ] You reviewed every AI-written diff yourself and can explain each change

---
[← Last lesson](02-working-with-ai-pair-programmers.md) · [Course home](../README.md) · [Capstone →](../capstone/README.md)
