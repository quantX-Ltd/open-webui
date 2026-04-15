# open-webui (Freiheit Media test task fork)

Fork of [open-webui/open-webui](https://github.com/open-webui/open-webui) with two small UI
changes and a short write-up, submitted as the developer test task.

- Fork: https://github.com/quantX-Ltd/open-webui
- Video: _to be added after recording_
- Time spent: roughly 2 hours

---

## Part A: local setup and UI changes

### Setup

Standard Docker Compose, built from local source so the UI changes end up in the image:

```bash
git clone https://github.com/quantX-Ltd/open-webui.git
cd open-webui
docker compose up --build -d
```

Then open <http://localhost:3000>. The first account you create becomes the admin.

The `--build` flag matters. The bundled `docker-compose.yaml` also references
`ghcr.io/open-webui/open-webui:main`, so without `--build` Docker would just pull the upstream
image and the local changes would not be visible.

### UI change 1: branded body color

File: `src/app.css`, body rules around line 696.

The default backgrounds were a plain white (light) and near-black (dark). Both were shifted
to a warmer palette so the whole app has a subtly different surface from vanilla Open WebUI,
while staying readable.

- light: `#fff` → `#faf5f0`
- dark: `#171717` → `#1a1612`

### UI change 2: top deployment banner

Files: `src/app.css` (new `.fm-banner` rule), `src/routes/+layout.svelte` (banner element
and flex-col wrapper).

A 32px banner at the top of the root layout reading `Freiheit Media - Internal LLM`. The
original layout used a single `flex flex-row h-screen` wrapper for the sidebar and chat
area. That wrapper was moved one level deeper and the root became a `flex flex-col h-screen`
so the banner sits above and the remaining area still fills the viewport cleanly.

Banner CSS:

```css
.fm-banner {
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(90deg, #c9443a 0%, #e07a28 100%);
    color: #fff;
    font-family: 'Archivo', sans-serif;
    font-weight: 600;
    font-size: 13px;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    border-bottom: 1px solid rgba(0, 0, 0, 0.2);
    flex-shrink: 0;
}
```

Layout change (simplified):

```svelte
<div class="flex flex-col h-screen overflow-hidden">
    <div class="fm-banner">Freiheit Media - Internal LLM</div>
    <div class="flex-1 min-h-0">
        <!-- original sidebar + chat area goes here -->
    </div>
</div>
```

---

## Part B: RAG answer quality and customer safety

### System prompt

```
You are a knowledge assistant for a specific customer. You answer questions using ONLY the
active Knowledge Base provided to you. Follow these rules:

1. Use only information from the Knowledge Base. Do not use general knowledge, training
   data, or information from any other source.

2. For every answer, classify the information availability:
   - FULLY AVAILABLE: the Knowledge Base contains a clear, complete answer.
   - PARTIALLY AVAILABLE: the Knowledge Base contains related information but not a
     complete answer. State what is available and what is missing.
   - NOT AVAILABLE: the Knowledge Base does not contain relevant information. Say so
     directly. Do not guess.

3. Never speculate, infer, or fill gaps with plausible-sounding information. If the answer
   is not in the Knowledge Base, say "This information is not available in the current
   Knowledge Base."

4. When referencing source material, keep it minimal. Use the document title and year
   only, for example: "Based on the Onboarding Guide (2024)." Do not quote long passages.

5. Never mix information across different customers or knowledge bases. You have access to
   exactly one customer's data at a time.

6. If a question is ambiguous, ask for clarification rather than assuming.
```

### Three example questions

Assumed Knowledge Base: Onboarding Guide (2024), Product FAQ, Service Level Agreement.

1. **Complete answer available.** "What are the support hours for enterprise customers?"
2. **Partial answer available.** "What is the escalation process for critical production
   incidents?"
3. **No information available.** "What are the pricing terms for the upcoming Q3 contract
   renewal?"

### Expected model answers

**1. Complete**

> Enterprise support is available Monday through Friday, 08:00 to 20:00 CET, with
> emergency coverage on weekends for Severity 1 issues. Based on the Service Level
> Agreement (2024).

**2. Partial**

> The Service Level Agreement defines severity levels and response times for production
> incidents. Severity 1 issues require a response within 30 minutes. However, the specific
> escalation path, who gets notified at each stage and through which channel, is not
> documented in the current Knowledge Base. I would recommend checking with your account
> manager for the detailed escalation matrix.

**3. Not available**

> This information is not available in the current Knowledge Base. The existing documents
> cover onboarding, product functionality, and service levels, but do not include
> commercial terms or renewal pricing. Your account manager would be the right contact for
> this.

### Short explanation

**Why this prompt improves reliability.** The prompt forces a classification step before
the model generates a user-facing answer. Instead of producing a plausible response and
hoping it is correct, the model first decides whether the information actually exists in
the KB. This prevents the most common failure in customer-facing RAG: confidently
delivering a wrong answer that sounds right.

**Typical failure modes it prevents.**

- **Cross-customer contamination.** The prompt restricts the model to one active Knowledge
  Base. Without this, a model serving multiple tenants could leak information from
  Customer A into Customer B's answers.
- **Hallucination on partial data.** When the KB has related but incomplete information,
  unguarded models tend to fill the gap with plausible fabrication. The three-tier
  classification forces the model to admit what is missing instead of inventing the rest.
- **False confidence.** By requiring the model to say "not available" rather than
  attempting an answer, you avoid the scenario where a customer acts on fabricated
  information.

**Improvements for later.**

- **Structured output.** JSON with fields for `answer`, `classification`, `sources`,
  `confidence` would enable downstream automation: routing unanswered questions to human
  agents, tracking KB coverage gaps, and measuring answer quality programmatically.
- **Per-answer confidence scoring** would enable automatic escalation when the model is
  uncertain, even within the "fully available" bucket.
- **Feedback loop.** A simple mechanism where agents flag wrong answers would continuously
  improve the retrieval step, not just the generation step.

---

For upstream Open WebUI documentation, see the
[source repository](https://github.com/open-webui/open-webui).
