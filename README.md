# Synq

**Partner briefs to launch-ready campaigns.**

Synq is a general-purpose tool for B2B partner and co-marketing campaigns, usable by any company in any industry. A short partner brief goes in; a review-ready campaign draft comes out. The assistant thinks like a senior B2B partner marketer and adapts to the partner's industry: it critiques the brief first, then drafts the campaign, proposes improvement options with trade-offs, and builds a test & measurement plan.

**Flow**

1. **Brief form**: partner, partner type (bank/fintech, retail/e-commerce, SaaS/tech, telco, travel/hospitality, food & beverage, affiliate/media, corporate, other), goal, audience/market, offer, key dates, channels (email, landing page, push, SMS, in-app, social), tone, constraints, KPI. Three fictional sample briefs are included: a bank × retailer cardholder cashback promo (Thailand), a SaaS × agency co-marketing webinar (APAC) and a marketplace × creator 12.12 sale (Indonesia).
2. **Critique first**: what's missing, vague or risky; assumptions to challenge; clarifying questions to send the partner (copy as email).
3. **Campaign draft**: 3 subject lines + preheaders (with character counts), email (hero, body, CTA, T&Cs), landing page hero block, and push / SMS / in-app / social variants for the chosen channels.
4. **Improvement options**: offer framing, segmentation, timing, personalisation, each with a trade-off and qualitative impact.
5. **Test & measurement plan**: A/B test with hypothesis, control, variant and no-send holdout; KPI formulas: CTR = clicks / delivered, conversion rate = conversions / clicks (purchases, sign-ups, registrations, etc.), ROAS = revenue / spend incl. promo cost, lift = test rate − control rate; pre-launch QA checklist (links, UTMs/tracking, dates/time zones, currency/pricing, opt-out, partner brand & legal rules).
6. **Builder.io prompt** (after the draft is ready, on demand): a copy-paste prompt for Builder.io's AI (Visual Editor > Generate tab) that builds the landing page: goal and audience, sections in order with the exact final copy, layout and visual direction, UTM-tagged button links, mobile layout and accessibility, with [placeholders] for anything unresolved. One prompt, or numbered step-by-step prompts for long pages, each with its own Copy button. Below it: a marketer checklist for after generating, risks with fixes, tracked links per channel and optional developer notes. No Builder.io API connection.
7. **Follow-up chat** with the full brief, draft and Builder.io prompt as context. Questions get answers; change requests ("make the subject lines more urgent") come back as a suggested edit tied to one section, with **Apply** (updates that tab) and **Undo**.
8. **Export**: copy the full draft as Markdown or download a `.md` file, including the Builder.io prompt, QA checklist ticks and Follow-up notes.

Results (all tabs, applied edits, the chat and QA ticks) are saved in the browser's localStorage, so they survive a refresh. **Clear** removes them.

## Stack

- Vite + React + TypeScript static frontend (`src/`)
- Netlify Functions (`netlify/functions/`) for all LLM calls, so API keys never reach the browser
  - `GET /api/status`: reports `demo` or `live` (provider + model, never the key)
  - `POST /api/generate`: `task` = `critique` | `draft` | `strategy` (structured JSON output) or `chat` (Markdown)
- Provider logic in `netlify/lib/llm.ts`. Gemini is used if `GEMINI_API_KEY` is set, otherwise OpenAI if `OPENAI_API_KEY` is set.
  - Gemini: `gemini-3.8-flash` (free tier available) via `generateContent` with `responseJsonSchema` and `thinkingLevel: "low"` for speed
  - OpenAI: `gpt-5.4-mini` via Chat Completions with strict `json_schema` Structured Outputs
- The three generation sections run as parallel calls, each with its own time budget (default 50s in production, under Netlify's 60s synchronous limit; 26s under `netlify dev`, which stops functions at 30s).
- **Resilience:** on 503 / 429 / UNAVAILABLE the primary model is retried once after a short backoff (only if it failed fast). Then the function falls back through other current Flash / Flash-Lite models (verified with ListModels). A hanging primary model is cut off after 12s (fallbacks get up to 20s each) and the next one is tried, and recently overloaded models are tried last. Responses report which model served (`model`, `attempts`), and the UI shows it. If every model is busy, the UI shows "AI is busy right now" with Retry, plus a one-click **Show demo output** for sample briefs.

## Demo mode

With no AI key configured, the app runs in **demo mode** (badge in the header, plus a "Demo mode — sample output" banner). The three sample briefs come with prewritten critique, draft, options, test plan and answers to the three suggested follow-ups ("Make it more urgent", "Adapt for Indonesia", "What would you test first?"). A custom brief in demo mode explains that a key is needed. That way the demo works end to end with no key.

## Environment variables

| Variable | Required | Notes |
| --- | --- | --- |
| `GEMINI_API_KEY` | one of the two | Preferred when both are set. Get one at https://aistudio.google.com/apikey |
| `OPENAI_API_KEY` | one of the two | |
| `GEMINI_MODEL` | no | Override the default `gemini-3.8-flash` |
| `OPENAI_MODEL` | no | Override the default `gpt-5.4-mini` |
| `GEMINI_FALLBACK_MODELS` | no | Comma-separated fallback models, default `gemini-3.6-flash,gemini-3.5-flash,gemini-3.1-flash-lite,gemini-3.5-flash-lite` (empty string disables fallback) |
| `LLM_DEADLINE_MS` | no | Total time budget per request across retries and fallbacks, default `50000` in production (`26000` under `netlify dev`, which stops functions at 30s) |
| `LLM_ATTEMPT_TIMEOUT_MS` | no | Max time for the primary model's attempt, default `12000` |
| `LLM_FALLBACK_ATTEMPT_TIMEOUT_MS` | no | Max time per fallback attempt, default `20000` |

Set them in Netlify: **Site configuration → Environment variables**, scope *Functions*, or with the CLI:

```bash
npx netlify-cli env:set GEMINI_API_KEY "<key>"
```

Environment variable changes take effect on the next deploy.

## Local development

```bash
npm install
npx netlify-cli dev          # http://localhost:8888 (frontend + functions)
# put GEMINI_API_KEY=... in a local .env (gitignored) to try live mode
```

`npm run dev` (plain Vite) also works; without functions the app falls back to demo mode.

## Tests

```bash
npm test                     # vitest: provider logic with mocked fetch (Gemini + OpenAI request shape, parsing, errors, timeouts), handler, Markdown export
npx netlify-cli dev &        # then:
npm run e2e                  # Playwright (playwright-core + local Chrome): demo flow end to end on desktop, mobile and dark mode, plus a mocked live-mode run; writes screenshots/
```

```bash
npm run build && npm run preview &   # then:
node scripts/e2e-demo.mjs    # demo-mode e2e for the Builder.io prompt, Follow-up Apply/Undo, refresh persistence and Clear; writes screenshots/v3-*.png (no AI calls)
```

`scripts/e2e.mjs` uses `/usr/bin/google-chrome`; set `CHROME=/path/to/chrome` to change it.

## Deploy to Netlify

```bash
npm install
npx netlify-cli login                       # or export NETLIFY_AUTH_TOKEN
npx netlify-cli link --name synq-campaigns  # existing site (id 0a8efc6c-e98a-44fb-9828-b1bbe72f5af8)
npx netlify-cli deploy --build --alias preview   # draft: https://preview--synq-campaigns.netlify.app
npx netlify-cli deploy --build --prod            # production: https://synq-campaigns.netlify.app
```

`netlify.toml` sets the build command (`npm run build`), publish dir (`dist`) and functions dir (`netlify/functions`). Without an AI key the deployed site runs in demo mode. Set `GEMINI_API_KEY` (or `OPENAI_API_KEY`) and redeploy to go live.

---

Built by Rocky Phanderson. All sample partners are fictional.
