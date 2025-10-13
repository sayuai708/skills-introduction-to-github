# Kruncher-Style Dossier Platform – Technical Plan

## Overview
This document outlines an implementation strategy for a web application that ingests a company URL and pitch deck, performs structured extraction and research, and renders a Kruncher-style investment dossier. The goal is to guide development from zero to a minimal viable product that meets the acceptance criteria described by the client.

## End-to-End Flow
1. **User submission** (Frontend)
   - Upload pitch deck (PDF/PPTX) and provide company URL.
   - Optional metadata (company name, contact email) captured for notifications.
   - Submission persisted immediately to Postgres with status `pending`.

2. **Processing pipeline** (Backend worker)
   - **Storage**: Deck stored in S3-compatible bucket; reference URL stored.
   - **Parsing**: Deck converted to text and slide images using `unstructured`/`pdfplumber` + slide segmentation.
   - **Crawling**: Queue URLs for focused crawl (home, product, pricing, blogs, press) using Playwright headless or `requests-html` with heuristics to avoid deep crawl.
   - **LLM extraction**: Structured prompt to model (e.g., GPT-4.1-mini) with deck text + crawl snippets, generating JSON object for all dossier sections. Prompts require provenance tags (slide number, URL) and source type (Company/Public/Analyst/Model).
   - **Scoring**: Deterministic scoring module (Python) combining heuristics and LLM outputs to produce 5-dimension radar scores and aggregate deal score out of 100.
   - **Competitor/comps research**: Use embeddings + similarity search against in-house startup index (vector DB) to locate competitors; fallback to simple web search API. Generate comps table with revenue multiple bands (seed, series A/B).
   - **PDF generation**: HTML templates rendered with dossier data. Use `WeasyPrint`/`Puppeteer` to export one-pager and full memo PDFs, saved to storage.
   - **Vector index**: Store processed deck text, crawl text, and dossier JSON embeddings in `pgvector` or external service (Pinecone, Qdrant) to support retrieval and future updates.
   - **Finalization**: Persist dossier JSON, metadata, scoring, storage keys, and status `completed` in Postgres.

3. **Frontend review**
   - Polls job status; once ready, presents dossier sections with provenance badges.
   - Provide download buttons for PDFs and raw JSON export.

## System Architecture
- **Frontend**: Next.js 14 (React Server Components) with Tailwind CSS. File uploads via signed URLs. Authentication optional (Magic link or Clerk) for MVP.
- **Backend API**: FastAPI or NestJS. Handles submissions, job status, dossier retrieval, signed URL issuance.
- **Worker**: Celery/RQ (Python) or Temporal workflow orchestrating ingestion, LLM calls, scoring, PDF export. Runs inside container with GPU access optional.
- **Database**: Postgres with pgvector extension. Tables for `submissions`, `documents`, `dossiers`, `scores`, `competitors`, `audit_logs`.
- **Object Storage**: S3-compatible (AWS S3, MinIO). Buckets for uploads, processed assets, PDF exports.
- **Vector Index**: pgvector tables for document chunks and competitor embeddings. Alternatively external service with similarity search API.
- **LLM Provider**: OpenAI GPT-4.1 and 4.1-mini for extraction; fallback smaller model for scoring or summarization.

## Data Model Sketch
```sql
CREATE TABLE submissions (
    id UUID PRIMARY KEY,
    company_name TEXT,
    company_url TEXT NOT NULL,
    contact_email TEXT,
    deck_storage_key TEXT,
    status TEXT CHECK (status IN ('pending','processing','completed','failed')),
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE dossiers (
    submission_id UUID PRIMARY KEY REFERENCES submissions(id) ON DELETE CASCADE,
    dossier_json JSONB NOT NULL,
    deal_score INTEGER NOT NULL,
    radar_scores JSONB NOT NULL,
    insights JSONB,
    competitor_grid JSONB,
    comps_table JSONB,
    summary TEXT,
    product_notes TEXT,
    market_notes TEXT,
    business_model_notes TEXT,
    provenance JSONB,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE documents (
    id UUID PRIMARY KEY,
    submission_id UUID REFERENCES submissions(id) ON DELETE CASCADE,
    source_type TEXT CHECK (source_type IN ('deck','crawl','generated','pdf')),
    label TEXT,
    storage_key TEXT,
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

## Structured Output Contract
Define a JSON schema enforced via LLM function-calling:
```json
{
  "summary": {"text": "", "provenance": []},
  "product_or_tech": {"text": "", "provenance": []},
  "market": {...},
  "business_model": {...},
  "gtm": {...},
  "competitive_landscape": {...},
  "metrics": {...},
  "round": {...},
  "insights": [{"insight": "", "type": "strength|risk|question", "provenance": []}],
  "score": {
    "overall": 0,
    "radar": {"team": 0, "market": 0, "business_model": 0, "traction": 0, "portfolio_fit": 0},
    "rationale": ""
  },
  "competitors": [
    {"name": "", "description": "", "stage": "", "provenance": [], "strengths": [], "weaknesses": []}
  ],
  "comps_table": [
    {"name": "", "stage": "", "revenue": "", "revenue_multiple": "", "source": ""}
  ],
  "provenance_registry": [
    {"id": "slide-3", "type": "Company", "source": "pitch-deck", "details": {"page": 3}},
    {"id": "url-1", "type": "Public", "source": "https://..."}
  ],
  "artifacts": {
    "one_pager_pdf": "s3://...",
    "full_memo_pdf": "s3://..."
  }
}
```
Each `provenance` array contains objects referencing `provenance_registry` IDs, allowing traceability.

## Deterministic Scoring Strategy
- Create baseline weights: `team` 0.25, `market` 0.25, `business_model` 0.2, `traction` 0.2, `portfolio_fit` 0.1.
- For each dimension, compute score from 0-100 using rule-based heuristics derived from LLM-extracted facts:
  - **Team**: founder experience level, prior exits, team size; degrade if unknown.
  - **Market**: TAM size, growth rate; use model to categorize (large, medium, niche).
  - **Business model**: revenue model clarity, pricing, margins.
  - **Traction**: metrics (ARR, users, growth); penalize missing data.
  - **Portfolio fit**: optional config per investor thesis.
- Combine into radar and overall average. Include rationales referencing provenance.

## Research Bundle
- Build `research_bundle(submission_id)` pipeline returning:
  - `deck_chunks`: text + slide metadata.
  - `web_snippets`: list of {url, title, snippet, content, crawl_depth}.
  - `news_mentions`: optional integration with news API.
  - `competitor_candidates`: generated via embedding similarity to knowledge base, plus heuristics from LLM.

## PDF Rendering
- Use server-side React components with `@react-pdf/renderer` or HTML templates rendered through headless Chromium.
- One-pager layout: summary, radar chart, top insights, competitor grid, key metrics.
- Full memo layout: dedicated section for each dossier field, appended comps table, methodology, provenance appendix.

## Frontend UI Outline
1. **Submission Page**
   - Form with file upload (drag-and-drop) and URL input.
   - Displays upload progress, validation (PDF/PPTX ≤ 50MB).
   - On submit, call API `POST /api/submissions`.
   - Show job status view with skeleton while processing.

2. **Dossier View**
   - Header with company name, submission metadata, CTA buttons for exports.
   - Tabs/accordion for Summary, Product, Market, Business Model, GTM, Competitive Landscape, Metrics, Round, Insights.
   - Provenance badges (pill with type + reference). Tooltip reveals slide/page/URL.
   - Radar chart (Recharts) for score.
   - Competitor grid (cards) + comps table (table with revenue multiples).
   - JSON viewer for raw dossier.

## Deployment Considerations
- Containerize services with Docker Compose.
- Use managed Postgres (Supabase/Neon) and S3 (AWS/Backblaze B2).
- Background workers on separate queue (Redis). Use `rq-scheduler` for retries.
- Observability: structured logs, tracing (OpenTelemetry), metrics (Prometheus) for pipeline runtimes.

## Next Steps
1. Scaffold repo with Turbo monorepo (apps/web, apps/api, packages/workers).
2. Implement submission API with signed upload and job enqueue.
3. Implement worker skeleton with placeholders for parsing, crawling, LLM, scoring.
4. Add LLM structured prompts + schema validation with `pydantic`.
5. Build frontend pages and integrate status polling.
6. Implement PDF rendering with templates.
7. Add end-to-end tests for dossier JSON completeness.
8. Harden provenance tracking and auditing.

