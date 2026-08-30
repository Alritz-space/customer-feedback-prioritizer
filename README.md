# Customer Feedback → Prioritized Backlog

An agentic workflow (built in [Make.com](https://make.com)) that turns raw,
scattered customer feedback into a ranked, actionable backlog — no manual
theme-tagging or spreadsheet triage required.

> 📺 **Walkthrough:** [link to Loom recording]
> *(60–90s walkthrough of the pipeline and output — add link here once recorded)*

## The Problem

Customer feedback piles up across App Store reviews, support tickets, and
surveys — each channel siloed, each item read in isolation. Prioritizing it
usually means a PM manually skimming dozens of entries, mentally clustering
themes, and eyeballing what feels most urgent. That doesn't scale, and it's
inherently inconsistent from one review session to the next.

## The Approach

Rather than treating this as "call an LLM once," the pipeline separates
**deterministic steps** (data shaping, array handling) from the **AI
reasoning step** (theme extraction, clustering, and impact scoring) —
mirroring how I'd design this as a production feature, not just a demo
script.

**Pipeline:**

```
Sample feedback (24 items: App Store reviews, support tickets, surveys)
        │
        ▼
Set Variables  →  wraps the feedback array so Make treats it as one bundle
        │
        ▼
Parse JSON  →  structures the raw feedback into source/text fields
        │
        ▼
Create JSON  →  serializes the array into a clean string (avoids Make's
                 implicit per-item iteration when passing arrays into
                 scalar fields — a real gotcha worth documenting)
        │
        ▼
Gemini 3.6 Flash (HTTP)  →  one combined call: extracts theme/sentiment per
                             item, clusters into themes, scores impact (1–3)
                             with a grounded rationale — single request,
                             not three
        │
        ▼
Parse JSON  →  structures Gemini's response into a themes array
        │
        ▼
Iterator → Text Aggregator  →  formats into a readable ranked table
```

## Sample Output

Run against [feedback_sample_data.csv](./feedback_sample_data.csv) (24
synthetic items across pricing, performance, onboarding, mobile stability,
export requests, and support quality):

| Theme | Frequency | Impact | Rationale |
| --- | --- | --- | --- |
| Data Export Functionality | 5 | 3 | Customers express severe frustration over manual workarounds like copy-pasting or taking screenshots, explicitly calling the lack of native CSV/Excel export a dealbreaker for their teams. |
| User Onboarding | 4 | 2 | The absence of a guided walkthrough or setup checklist leaves new users feeling lost and frustrated, with some almost abandoning the product on day one. |
| Performance & Load Times | 4 | 3 | Slow dashboard loading times and poor performance with larger datasets force users to give up on tasks or wait excessively, severely interrupting core daily workflows. |
| Pricing Clarity | 4 | 2 | Unclear plan comparisons and unclarified usage limit policies cause confusion, wrong upgrades, and surprise charges that require support intervention. |
| Mobile App Stability | 3 | 3 | Frequent app crashes when uploading images or attaching files prevent field workers from completing reports during client visits, directly blocking core mobile usage. |
| Customer Support Quality | 4 | 1 | Users consistently praise the support team for fast response times and clear, helpful interactions, creating a positive experience that mitigates customer friction. |

Note the last row: the model correctly distinguished a *frequently
mentioned but positive* theme (support quality) from an actionable
complaint, scoring it low-impact despite high frequency — the kind of
judgment call that separates a useful signal from noise.

## What's in This Repo

| File | What it is |
|---|---|
| [`feedback_sample_data.csv`](./feedback_sample_data.csv) | 24 synthetic feedback items (reviews, tickets, surveys) used as pipeline input |
| [`feedback-prioritizer-blueprint.json`](./feedback-prioritizer-blueprint.json) | Exported Make.com scenario — import directly to inspect or run it yourself (add your own `GEMINI_API_KEY` where the placeholder appears) |
| [`canvas-screenshot.png`](./canvas-screenshot.png) | Full pipeline view on the Make canvas |

## A Few Build Notes

Getting this working end-to-end surfaced some real lessons worth
documenting rather than hiding:

- **Combining extraction + scoring into a single prompt** cut API calls in
  half — both for cost and for staying inside free-tier rate limits, an
  active constraint when iterating on a live demo, not just a production
  nice-to-have.
- **Array-into-scalar-field mapping in Make silently iterates** rather than
  erroring — passing a raw array where a text field is expected causes the
  whole downstream chain to fire once *per item* instead of once total.
  Fixed by explicitly serializing the array to a string (`Create JSON`)
  before it hits any text field.
- **Nested API response paths matter** — Gemini wraps its actual output
  several levels deep (`candidates[0].content.parts[0].text`); parsing the
  outer envelope instead of the inner content is an easy, silent mistake.

## What I'd Build Next

- Replace the sample CSV with a live source (App Store Connect API, a
  support-ticket export, or a survey tool webhook).
- Add a persistence layer (Google Sheets or Airtable) so the ranked backlog
  accumulates over time instead of resetting each run.
- Surface a "confidence" flag per theme, similar to the tuning approach in
  my [AI Financial Insights Copilot artifacts](https://github.com/Alritz-space/pm-artifacts-ai-financial-insights),
  so low-confidence clusters get flagged for human review before they hit
  the backlog.

## About Me

Principal Product Manager, 17+ years across Cloud ERP, Fintech, and B2B
platforms — currently at Oracle NetSuite working on ML and GenAI-driven
financial product capabilities. [LinkedIn](https://in.linkedin.com/in/riteshjainpm)
