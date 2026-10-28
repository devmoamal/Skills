---
name: council
description: "Pressure-test a high-stakes decision with a council of 5 independent AI advisors (Contrarian, First Principles, Expansionist, Outsider, Executor), anonymous peer review, and a chairman verdict. Based on Karpathy's LLM Council. MANDATORY TRIGGERS: 'council this', 'run the council', '/council', 'war room this', 'pressure-test this', 'stress-test this', 'debate this'. STRONG TRIGGERS (only when there is a real decision with stakes and options): 'should I X or Y', 'which option', 'I can't decide', 'I'm torn between', 'review this plan before I commit'. Do NOT trigger on factual questions, coding tasks, simple yes/no, or casual 'what would you do' chatter."
---

# LLM Council

One AI answer is one perspective. The council runs a decision through 5 advisors with deliberately conflicting thinking styles, has their answers peer-reviewed anonymously, then the chairman (you, the parent agent) issues a verdict: where they agree, where they clash, what was missed, what to do, and what to do first.

Adapted from Andrej Karpathy's LLM Council. **Honest limitation:** Karpathy used *different models*. Here advisors are mostly the same model family wearing different lenses, so their errors correlate. The skill compensates with mixed model tiers, evidence tagging, and treating consensus as weak evidence.

---

## 0. Pick a mode

| Mode | When | Cost |
|---|---|---|
| **Full** (default for mandatory triggers) | Costly, hard-to-reverse decisions | 5 advisors + 2 reviewers = 7 subagents |
| **Lite** | User says "quick council"/"lite", or the decision is cheap and reversible | 3 advisors (Contrarian, First Principles, Executor), no review = 3 subagents |

If a STRONG (non-mandatory) trigger fired, run **Lite** and mention Full is available. Never council trivial questions — just answer them.

---

## 1. The five advisors

Paste the paragraph for each advisor **word for word** into its prompt.

- **The Contrarian** — Actively looks for what's wrong, missing, or will fail. Assumes the idea has a fatal flaw and hunts for it. If everything looks solid, digs deeper. Not a pessimist — the friend who saves you from a bad deal by asking the questions you're avoiding.
- **The First Principles Thinker** — Ignores the surface question and asks "what are we actually trying to solve?" Strips assumptions and rebuilds from the ground up. Willing to say "you're asking the wrong question."
- **The Expansionist** — Hunts for upside others miss: what could be bigger, what adjacent opportunity is hiding, what's undervalued. Ignores risk (the Contrarian covers it).
- **The Outsider** — Has zero context about the user, their field, or history. Reacts only to what's in front of them. Catches the curse of knowledge: jargon, unstated assumptions, things obvious to insiders but confusing to everyone else.
- **The Executor** — Only cares whether it can actually be done and the fastest path. "OK, but what happens Monday morning?" Flags any idea with no clear first step.

Tensions by design: Contrarian ↔ Expansionist (downside vs upside), First Principles ↔ Executor (rethink vs ship), Outsider keeps everyone honest.

---

## 2. Runbook

Follow these steps in order. Post a **one-line status to the user** at the start of each phase (e.g. "Council convened — 5 advisors deliberating…") so the run is never a silent freeze.

### Step 1 — Gather context and frame (no subagents yet)

1. **Read at most 4 files** for context: `GEMINI.md` / `AGENTS.md` / `README.md`, any files the user referenced, and the newest `council-*.md` transcript in the artifact dir if the same topic was counciled before.
2. **Run** `Get-Date -Format yyyyMMdd-HHmmss` → this is `<ts>`. Session folder: `<artifactDir>/scratch/council-<ts>/` (`<artifactDir>` = your conversation's artifact directory — never the user's repo).
3. **Write `question.md`** into the session folder with exactly these sections:
   - `## User's raw prompt` — verbatim, unedited.
   - `## Decision` — the core question in one or two sentences.
   - `## Underlying goal` — what the user actually wants to achieve.
   - `## Success looks like` — how we'd know it worked.
   - `## Constraints & facts` — budget, deadline, stack, audience, numbers (from user + files).
   - `## Reversibility & stakes` — how costly is being wrong; can it be undone?
   - `## User's current lean` — if stated; otherwise "not stated".
   - `## Files to read` — absolute paths of any artifacts under review (plans, code, copy).
4. Neutral framing only — no opinions. If you cannot state the underlying goal, **ask the user one clarifying question** and stop. Otherwise proceed without asking.

### Step 2 — Convene advisors (all in ONE turn)

Call `invoke_subagent` once per advisor, **all in the same response**, then end your turn with no further tool calls.

| Advisor | `TypeName` | `Model` |
|---|---|---|
| Contrarian | `research` | `pro` |
| First Principles | `research` | `pro` |
| Expansionist | `research` | `inherit` |
| Outsider | `research` | `flash` |
| Executor | `research` | `inherit` |

`Role`: `Council Advisor: <Name>`. Advisors are always `research` (read-only); they never need write access.

**Advisor prompt** (fill the brackets):
```text
You are [Advisor Name] on an LLM Council.

Your thinking style: [advisor paragraph, word for word]

Read this briefing in full, then every file listed under "Files to read":
[absolute path to question.md]
[OUTSIDER ONLY — replace the line above with: "Read ONLY the '## User's raw prompt' and '## Files to read' sections. Ignore all other sections; you have no background context."]

Rules:
- Answer strictly from your lens. Do not hedge or balance — other advisors cover other angles.
- Be specific to THIS situation. Generic advice is a failure.
- Ground claims in the briefing or files. Tag anything you are assuming or cannot verify as [ASSUMPTION].
- End with one line: "Bottom line: <your position in one sentence>".
- 150–300 words. No preamble.
- Reply with your analysis as your final message. Do not write files.
```

**Reply handling:** you are woken once per reply. Keep a running count. Below the expected number → end your turn without tool calls. If a spawn errors (e.g. rate limit), retry that advisor once, alone. If an advisor returns empty or off-task, re-run it once; if it fails again, proceed without it and note "N/5 advisors" in the verdict.

### Step 3 — Anonymized peer review (Full mode only)

1. **Run** `1..5 | Get-Random -Count 5`. The k-th number is the advisor (1=Contrarian, 2=First Principles, 3=Expansionist, 4=Outsider, 5=Executor) assigned letter A, B, C, D, E in order.
2. **Write `responses.md`** to the session folder: `### Response A` … `### Response E`, each containing the advisor's text **with its name and any self-identifying label removed**. Keep the key private (it goes in the transcript later).
3. Spawn **2 reviewers in one turn**: `TypeName: research`, models `pro` and `flash`, `Role: Council Peer Reviewer <n>`.

**Reviewer prompt:**
```text
You are reviewing an LLM Council. Read in full:
1. Briefing: [path to question.md] (and any files it lists)
2. Anonymized responses A–E: [path to responses.md]

Answer directly, referencing responses by letter:
1. Strongest response, and why?
2. Biggest blind spot (which response, what's missing)?
3. Which specific claim, if false, would most change the right decision?
4. What did ALL responses miss?

Under 200 words. Reply as your final message. Do not write files.
```

Same reply handling as Step 2.

### Step 4 — Chairman verdict (you)

De-anonymize and synthesize. You are an executive, not a vote counter: side with a lone dissenter if their reasoning is strongest. **Agreement among same-model advisors is weak evidence** — call out any consensus that rests on an unchecked `[ASSUMPTION]`.

```markdown
# Council Verdict: <short title>

## Where the Council Agrees
## Where the Council Clashes
## Blind Spots Caught in Review
## The Recommendation
<Decisive answer with reasoning. Not "it depends.">
**Confidence:** <low / medium / high> — **What would flip this:** <specific evidence> — **Revisit:** <date or trigger>
## The One Thing to Do First
<One concrete action, not a list.>
```

### Step 5 — Deliver

1. **Write the transcript** as an artifact: `<artifactDir>/council-<ts>.md` containing the raw prompt, framed briefing, every advisor response (named), the anonymization key, reviews, and the verdict.
2. **In chat:** show the full verdict, plus a link to the transcript (`file:///…/council-<ts>.md`). Keep the rest of the reply short.
3. **HTML report — only if the user asks** ("with report", "make it visual"). Use [resources/report-template.html](resources/report-template.html), save as `<artifactDir>/council-<ts>.html`, and follow the escaping rules in that file. Link it; don't ask whether to open it.

---

## Checklist

`mode → read ≤4 files → <ts> → question.md → status → 5 advisors (one turn) → count replies → Get-Random → responses.md → status → 2 reviewers (one turn) → count → verdict (confidence / flip / revisit) → transcript artifact → chat verdict + link → (HTML if asked)`

## Rules

- All advisors spawned in the same turn; never sequentially (later ones would anchor on earlier ones).
- Never reveal advisor names to reviewers.
- Never write council files into the user's repo unless asked.
- No-subagent fallback (only if `invoke_subagent` is unavailable): write each advisor's answer separately before reading the others, and label the verdict **"single-pass council — reduced independence"**.

---

Methodology by [Andrej Karpathy](https://x.com/karpathy). Original skill by [@tenfoldmarc](https://github.com/tenfoldmarc/llm-council-skill); adapted for Antigravity.
