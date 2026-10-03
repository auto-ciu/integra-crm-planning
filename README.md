# Integra Scientific — CRM Feature Plan (A–E + extras + roadmap)

Prepared 3 Oct 2026, based on a read-only exploration of `/home/builder/integra-scientific` (crm/, integra-portal/, integra-web/, strategy/, legal/, .github/). **Revised 3 Oct 2026 with the founder decisions below.**

Effort scale (for one developer): **S** < 1 week · **M** 1–3 weeks · **L** 3–6 weeks · **XL** more than 6 weeks.

### Founder decisions (3 Oct 2026)

| # | Decision | Where it lands in this plan |
|---|---|---|
| D1 | **Apify** is the approved LinkedIn company-data source (e.g. actor `harvestapi/linkedin-company`). **Firecrawl Alexandria** is an optional enrichment provider. We still do not build our own scraper or browser extension. | Feature A (sources, data model, phases, guardrails) |
| D2 | **eIDAS-compliant signatures from an EU qualified trust service provider (QTSP)** replace DocuSign everywhere: AR mandates, service agreements, quotes. The portal's Phase 5 DocuSign work is redirected to the QTSP, and the CRM uses the same provider. | New section **S**; Feature C; extra #3; roadmap |
| D3 | **Pricing display:** if `strategyType === ADD_ON` or the strategy has optional extras, show "from €X". Otherwise show exact totals. TIERED and BUNDLE show exact totals per tier or bundle option. `fromPrefix` is set per strategy. | Feature C (rule, data model, `PublicPricingV1`, components, tests) |
| D4 | The CRM is self-hosted and internal-only. **The SAML licence item is closed** and removed from the risks. | §0, risks |
| D5 | **Portal spec amendments approved** for the C3 Stripe lookup-key contract, X2 portal↔CRM sync, and X3 CPQ + eIDAS signing, together with the database migrations they need. | §0, Feature C, extras #2 and #3, roadmap |
| D6 | **Cloudflare project name resolved.** The live Pages project is **`integra-scientific`**. | §0, Feature E, risks |

---

## 0. Read this first: where the brief and the repo disagree

| Brief says | Repo actually has | Consequence for this plan |
|---|---|---|
| Clerk auth | **Keycloak** (`auth.integrascientific.com`, realm `integra`). The portal uses next-auth v4 OIDC and the CRM uses SAML. `users.clerkUserId` stores the Keycloak `sub`. The portal docs and `sst-env.d.ts` still say Clerk. | All "staff identity" integration goes through Keycloak group `/integra-staff` (role `crm-access`) plus Twenty workspace members, which are matched by email. |
| Next.js 15 portal | Portal runs **Next 16.2.4**, TS 6, zod 4. integra-web runs Next 15. | Minor. Watch the zod 4 API in shared code. |
| Stripe integrated | **Stripe is unbuilt.** It exists only as SST secrets plus the `organisations.tier`, `subscriptionType` and `stripeCustomerId` columns. Spec Phase 6 is unchecked. | Feature C's Stripe half *is* portal Phase 6. Design it as one effort. |
| Website contact form | `<form action="mailto:info@...">` with no backend and no spam protection. The site is `output: "export"` on Cloudflare Pages. | Enquiries currently depend on the visitor's desktop mail client, which usually fails in WeChat's in-app browser and on phones. Feature E fixes a real leak. |
| CRM app with objects/views | `crm/app` **has never been `npm install`ed, typechecked or synced** (README). It has no logic functions. `ops/nightly-status.mjs` is **not scheduled by anything**. The worker runs with `DISABLE_CRON_JOBS_REGISTRATION=true`. | Everything depends on a **Phase 0** that makes the app real and proves Twenty logic functions on v2.41. |
| — | Portal `CONSTITUTION.md`: the spec is supreme. Article 10 requires human approval for new deps, infra, migrations and workflow changes. `MISSION.md` lists "AI chatbot" and "Training portal" as non-goals. | Keep A, B, D and E (and most of C) **in the CRM and website**. The portal-touching parts (C3, extras #2 and #3) need spec amendments. **These and their database migrations were approved by the founder on 3 Oct 2026 (D5).** Any new dependencies, `sst.config.ts`/infra changes and workflow changes still go through Article 10 approval one by one. |
| DocuSign (in the brief's stack) | The portal has an SST secret `DocusignIntegrationKey`, an SQS `DocusignQueue` with no subscriber, and the column `ar_mandates.docusignEnvelopeId`. The CRM's `ArMandate` has a `docusignEnvelopeId` field. Nothing calls DocuSign yet. | **Replaced by eIDAS QTSP signing (D2).** See section S for the renames and migration. Because nothing is built yet, the switch costs almost nothing. |
| Pricing | Canonical pricing is in Digest §3.1, which matches spec §8.1 and legal Schedule 2: Beginner/Boost/Builder/Boss, AR €250/€1,200/€3,000, DPP €950/€2,500/€6,000, bundle −15%, plus DPP and sector add-ons. Older price models in `Service_Offering.md` and the GTM doc are superseded. | Seed data for B and C comes from the canonical table. **No prices are published on the website today.** |
| Calendar | Today is 3 Oct. **Canton Fair is 15–19 Oct 2026** (12 days away), which GTM rates as the HIGH channel and the launch event. Battery DPP becomes mandatory on **18 Feb 2027**. | The roadmap front-loads E1 and fair lead capture, and keeps the portal team on the DPP critical path. |

One more thing to know:
- **Cloudflare project name: resolved (D6).** CI (`.github/workflows/deploy.yml`) is the only deploy path, and it runs `pages deploy out --project-name=integra-scientific --branch=main` from `workingDirectory: integra-web` (account `c716…0a74`). **`integra-scientific` is the live project**, so put all Pages secrets (`TURNSTILE_SECRET`, the Twenty intake key, SES credentials) there. The local `npm run deploy` script and the `name` in `wrangler.jsonc` still say `integra-web`. Change both to `integra-scientific` (a one-line hygiene fix in E1) so a manual deploy can't create a second, orphaned project.

### Architecture principle used throughout

```
                ┌───────────────────────── Twenty CRM (crm.integrascientific.com) ─────────────────────────┐
 staff (Keycloak│  System of record for COMMERCIAL data: streams, offerings, pricing strategies, tickets,  │
 SAML) ───────▶ │  leads, research. Logic functions = glue (httpRoute / cron / databaseEvent).            │
                └──────┬───────────────┬────────────────────┬────────────────────┬──────────────────────┘
                       │ git commit    │ Stripe API         │ Claude API         │ SES
                       ▼ (pricing.json)▼ (catalogue only)   ▼ (Messages +        ▼ (notifications,
              integra-web (static)   Stripe ◀── webhooks ── portal (Phase 6)    Managed Agents)   replies)
              + Pages Function       Products/Prices with      sets organisations.tier
              /api/contact ────────▶ lookup_key + metadata     = system of record for ENTITLEMENTS
              (Turnstile) → CRM
```

- **The CRM owns commercial configuration.** Staff already work there. The portal has no admin area (spec Phase 5), and Twenty provides tables, kanbans and record pages for free.
- **The portal owns entitlements and billing state.** It reads Stripe webhooks and needs no copy of the catalogue, because **Stripe Price `lookup_key` + `metadata` are the contract** between the two.
- **The website consumes a whitelisted JSON snapshot committed to git.** It is a static export, so there is no runtime dependency on the CRM, and git gives an audit trail of public prices.

---

## Phase 0 — Foundations (prerequisite for A–E) · Effort **M**

| # | Task | Deliverable |
|---|---|---|
| F0.1 | **Make `crm/app` real.** Run `npm install`, commit the lockfile, and pin `twenty-sdk` / `twenty-client-sdk` to exactly `2.41.x`. Run `npm run typecheck`, then `twenty-sdk dev` against the e2e docker Twenty. Resolve the four README unknowns: export names, generated-client signatures, relation keys, and the FILES widget config. | Green `verify` + `typecheck`. App synced to a dev workspace. |
| F0.2 | **CI/CD for the app.** Add `.github/workflows/ci-crm-app.yml` (verify + typecheck on PR) and `deploy-crm-app.yml` (`twenty-sdk deploy` with `TWENTY_API_KEY` from Secrets Manager). | The app ships to production repeatably. |
| F0.3 | **Logic-function spike on v2.41.** Check whether `defineLogicFunction` (from `twenty-sdk/define`) exists in 2.41, and test each trigger and capability: (a) `httpRouteTriggerSettings`, with and without `isAuthRequired`; (b) `cronTriggerSettings`, which needs testing given `DISABLE_CRON_JOBS_REGISTRATION=true` on the worker and may need `cron:register:all` or an env change; (c) `databaseEventTriggerSettings` with `updatedFields`, including whether the payload carries the acting workspace member; (d) app secret variables; (e) bundling npm deps (`@anthropic-ai/sdk`, `stripe`, `@aws-sdk/client-sesv2`); (f) max timeout and the serverless driver on ECS; (g) `enqueueJobs`. | **Exit test:** port `ops/nightly-status.mjs` to a cron logic function. That also fixes today's unscheduled urgency job. |
| F0.3b | **Fallback if the spike fails:** build a "CRM sidecar" as CloudFormation `phase7-crm-functions.yaml` with API Gateway, Lambda (Node 22) and EventBridge Scheduler, calling Twenty REST via the existing `ops/lib/twenty-api.mjs` pattern. Every design below still works; only the host changes. | Decision recorded. |
| F0.4 | **Field helpers** in `src/lib/fields.ts`: `boolean`, `multiSelect`, `dateTime`, `rawJson`, `emails`, `links`, `phones`, `rating`, plus a `manyToOne` to WorkspaceMember (add `STANDARD.workspaceMember` to `standard-ids.ts`, and verify custom relations to workspace members are allowed in 2.41; fallback is a TEXT `assigneeEmail`). Extend `verify-model.mjs`. | Helpers + tests. |
| F0.5 | **Cross-cutting field.** Add `leadSource` SELECT to Person, Company and Opportunity: `WEB_FORM, CANTON_FAIR, TRADE_FAIR_OTHER, LINKEDIN, REFERRAL, WEBINAR, OUTBOUND, PARTNER, AI_PROSPECTING, IMPORT_LIST, OTHER`. No lead-source field exists anywhere today. | Field + backfill to `OTHER`. |
| F0.6 | **SES (eu-central-1)** for the CRM AWS account: domain identity `integrascientific.com` (DKIM), custom MAIL FROM `mail.integrascientific.com`, a configuration set with bounce/complaint → SNS, and a **production-access request** (lead time is about a day). Shared later with portal Phase 8. | Verified sender `notifications@` / `support@`. |
| F0.7 | **Anthropic workspace "integra-crm"** with its own API key in Secrets Manager (`integra-crm/app`), separate from the portal's `ClaudeApiKey` so spend is attributable, plus a workspace spend limit. | Key + limit. |

Conventions for every new CRM file:
- Labels `EN · 中文`, except where e2e matchers require English only.
- Universal IDs added to `src/ids.ts` (v4 UUIDs).
- Select options in `src/options.ts` as `[value, label, color]`.
- Logic functions under `crm/app/src/logic-functions/<name>.ts`.
- Shared pure logic in `crm/app/shared/*.mjs` so `verify-model.mjs` and unit tests can import it.

---

## Feature A — AI Lead Identification

### 1. Summary
An AI-scored lead discovery pipeline. It pulls candidate manufacturers from several sources:
- trade-fair exhibitor lists;
- EU registers and alerts;
- import data;
- **LinkedIn company data via Apify**, the approved third-party API (D1);
- LinkedIn Lead Gen Forms and staff quick-add.

Candidates are enriched from the company's own website and, optionally, from **Firecrawl Alexandria**. Claude scores each against Integra's ICP, and results land in a review queue. Accepted candidates become Company/Person records that feed the existing **Fair leads** kanban.

> **We still do not build our own LinkedIn scraper or profile-copying browser extension.** LinkedIn's User Agreement §8.2 prohibits scraping and "browser plugins and add-ons" that copy profile data. Running either would put the founder's own LinkedIn account at risk, and that account *is* the LinkedIn channel. **LinkedIn company data comes from Apify instead (D1).** We call Apify's API; Apify runs the actors.
>
> **Residual risk, stated plainly so the guardrails below make sense:**
> - Apify Technologies s.r.o. (Prague) acts as our processor under its Data Processing Addendum.
> - However, its terms (§5.8) leave the customer "solely responsible for the legality … and use of all Customer Data". Using Apify does **not** shift LinkedIn-terms or GDPR responsibility to Apify.
> - LinkedIn has sued third-party data APIs before; Proxycurl shut down in 2025 after a LinkedIn lawsuit.
> - The guardrails therefore keep exposure low: company pages only, no-cookie actors only, no person profiles, and provenance on every record.
>
> GDPR still applies to any *people* we source indirectly (Art. 14 notice, legitimate-interest assessment). Also, GTM §4 rates LinkedIn as only a **MEDIUM** channel for this ICP. Chinese SME battery makers are found more reliably via Canton Fair / CIBF exhibitor lists, Alibaba, customs data and EU registers. So the pipeline stays multi-source, with LinkedIn/Apify as the main **firmographic enrichment** layer rather than the only source.

### 2. Technical approach
- **Where it lives:** the CRM (objects, views, front components, logic functions) plus the Claude API. Nothing in the portal.
- **Sources, ranked by ICP fit × compliance:**
  1. **Trade-fair exhibitor lists** (Canton Fair Phase 1 new-energy/electronics, CIBF, The Battery Show Europe): CSV upload. Company-level, public data.
  2. **Fair business cards** (extra feature #1): Claude vision extraction.
  3. **Website enquiries** (Feature E): the same scorer runs on every enquiry.
  4. **EU Safety Gate** weekly alerts filtered to origin = China and batteries/electronics: a pain signal.
  5. **Import/shipment data** (Panjiva / ImportGenius / Volza): HS 8507.60 (Li-ion) shipments to EU consignees. Gives a hard "exports to EU" signal and a volume band. Paid, Phase A3.
  6. **LinkedIn company data via Apify (primary LinkedIn source, D1).** We use actor **`harvestapi/linkedin-company`** ("LinkedIn Company Details Scraper" by HarvestAPI, a community publisher on Apify).
     - **Input:** LinkedIn company URLs, *or* company names that it resolves through LinkedIn search. The name lookup matters most, because exhibitor lists and business cards rarely carry LinkedIn URLs.
     - **Output:** name, LinkedIn URL, website, employee count, locations/HQ, industries, company type, specialties, tagline, founded year, follower count.
     - **Access:** "No cookies or account required", so no Integra LinkedIn account is ever involved.
     - **Price:** pay per event, "from $3.00 / 1,000 companies" (the README says $4 per 1k). Treat it as about **$3–4 per 1,000 companies**.
     - **How we call it:**
       - Use the official **`apify-client`** npm package with an Apify API token stored as an app secret.
       - The logic function starts an **async run** (`client.actor('harvestapi/linkedin-company').start(input)`) and pins the actor **build/version** so its output schema can't drift silently.
       - The existing 10-minute poll cron (or an Apify run webhook to `/s/leads/apify-webhook`, protected by a shared secret) reads the run's default dataset and merges it into `LeadCandidate`.
       - Batches of up to a few hundred names go in one run.
     - **Field mapping:**
       - website → `website` / dedupe domain
       - employee count → `employeeBand`
       - HQ/locations → `province` (city → province lookup in `shared/icp.mjs`, e.g. 深圳/Shenzhen → GUANGDONG)
       - industries + specialties → scorer input (product-category hints)
       - the full item → `rawPayload.linkedin`
     - **Guardrails:**
       - (a) **Company-page actors only.** No person/profile actors, so no personal data enters from LinkedIn.
       - (b) **No-cookie actors only.** Never pass an Integra LinkedIn session.
       - (c) Provenance on every record: `source=LINKEDIN_APIFY`, actor + build, `linkedinFetchedAt`.
       - (d) Sign Apify's DPA.
       - (e) Refresh a company at most quarterly.
       - (f) A kill switch (app variable `APIFY_ENABLED`) so the source can be turned off without a deploy.
       - (g) A monthly spend cap in the Apify console.
  7. **LinkedIn Lead Gen Forms via the Marketing API (Lead Sync)** for paid campaigns aimed at 外贸经理 / compliance titles. Requires an ad account and API access approval. This is LinkedIn's own API, used in A3.
  8. **Staff quick-add:** staff paste a LinkedIn company URL, company name and/or website into the CRM. That triggers an Apify lookup (D1) plus website enrichment. Sales Navigator stays a manual research tool.
  9. **Firecrawl Alexandria (optional enrichment, evaluated in A2, D1).**
     - **What it is:** a "knowledge library" giving AI agents one access point to **93 providers / 640 capabilities across 20 categories**, including finance & company data, government records & public data, news and people data.
     - **Access:** via **MCP**, a CLI (`firecrawl-cli`), or API/SDKs.
     - **Status:** **limited access** (explore / waitlist) and **pricing is not published**, so it cannot be on the critical path.
     - **Evaluation:** in A2, check whether it has providers that help *this* ICP (Chinese company registries, trade/customs data, certification databases). If it does, attach it to the enrichment call as an MCP connector, the same way as other Claude tools.
     - **Restriction:** exclude its "people data" providers by default (GDPR data minimisation). Company-level enrichment only.
- **Pipeline:** `ingest → normalise → dedupe → enrich → score → review → accept`.
  - **Dedupe:** `dedupeKey` = registrable domain, else `normalise(nameEn|nameZh)+province`. Check against `LeadCandidate` and `Company.domainName` / `nameZh`. Re-run dedupe after Apify enrichment, because it often supplies the website a list was missing.
  - **Enrich, in this order:**
    1. **Apify LinkedIn company data** (firmographics: size, HQ, industry, website).
    2. A Claude call with the `web_fetch_20260209` server tool, restricted with `allowed_domains: [candidateDomain]`. It extracts product lines, certifications (CE, UN38.3, IEC 62133), stated export markets, EU distributors, staff count and "factory vs trader". Where Apify and the website disagree on staff count, keep both and let the scorer weigh them.
    3. *(Optional, if A2 evaluation passes)* Alexandria providers via MCP.
  - **Score:** a Claude Messages API call with **structured outputs** (`output_config.format` JSON schema) against the stream's ICP rubric (Feature B `ProductStream.icpRubric`). Use model `claude-opus-5-5` at `effort: "low"`. For imports over 20 rows use the **Message Batches API** (50% cheaper, async): submit from a logic function and poll with cron. Handle `stop_reason: "refusal"` and use server-side `fallbacks`. Moving to a cheaper model is a choice to make after an eval, not by default.
  - **ICP rubric v1** (from `strategy/`): Chinese *manufacturer* (not trader) of Li-ion/LMT batteries, later textiles; 50–500 staff; €0.5–10M EU exports; in Guangdong/Zhejiang/Jiangsu/Fujian; sells into the EU without its own EU entity or AR. Hard disqualifiers: EU-established, no EU sales, already a client.
  - **Score output schema:**
    ```ts
    { isManufacturer: boolean; productCategories: ('BATTERY_LI_ION'|'BATTERY_LMT'|'TEXTILES'|'ELECTRONICS'|'OTHER')[];
      euExportEvidence: { level: 'NONE'|'WEAK'|'STRONG'; evidence: string[] };
      employeeBand: 'LT_50'|'B_50_500'|'GT_500'|'UNKNOWN';
      exportRevenueBand: 'LT_500K'|'B_500K_1M'|'B_1M_5M'|'B_5M_10M'|'GT_10M'|'UNKNOWN';   // = existing EXPORT_REVENUE_BAND
      province: 'GUANGDONG'|'ZHEJIANG'|'JIANGSU'|'FUJIAN'|'SHANDONG'|'OTHER'|'UNKNOWN';    // = existing PROVINCE
      hasEuAr: 'YES'|'NO'|'UNKNOWN'; recommendedStream: 'DPP'|'AR'|'TRAINING'|'BUNDLE';
      icpScore: number /*0-100*/; icpTier: 'A'|'B'|'C'|'REJECT'; rationale: string /*≤80 words*/; citations: string[] }
    ```
  - **Human in the loop:** nothing becomes a Company until a staff member clicks **Accept**.
- **Files:**
  - Objects: `src/objects/lead-candidate.object.ts`, `lead-import.object.ts`
  - Logic functions: `src/logic-functions/{lead-quick-add,lead-import-process,lead-apify-enrich,lead-score-batch-poll,lead-accept,safety-gate-feed}.ts`. `lead-score-batch-poll` also collects finished Apify runs.
  - Shared logic: `shared/icp.mjs` (dedupe key, freemail list, tier thresholds, city → province map, Apify item → LeadCandidate mapper)

### 3. Data model (Twenty)
```
LeadCandidate (leadCandidates) "Lead candidate · 潜在客户"
  name               TEXT        label identifier (EN company name)
  nameZh             TEXT
  website            LINKS
  linkedinUrl        LINKS       LinkedIn company page (from list, staff, or Apify lookup)
  source             SELECT      CANTON_FAIR_LIST, CIBF_LIST, OTHER_FAIR_LIST, BUSINESS_CARD, WEB_ENQUIRY,
                                 SAFETY_GATE, IMPORT_DATA, LINKEDIN_APIFY, LINKEDIN_LEADGEN, STAFF_QUICK_ADD,
                                 REFERRAL, OTHER
  sourceRef          TEXT        row id / alert id / URL
  leadImport         → LeadImport
  rawPayload         RAW_JSON    original row as received; enrichment results under .linkedin / .alexandria
  linkedinEmployeeCount NUMBER   from Apify
  linkedinIndustry   TEXT        from Apify
  linkedinSpecialties TEXT       from Apify (comma-joined)
  linkedinHq         TEXT        from Apify (city, country)
  linkedinFoundedYear NUMBER     from Apify
  linkedinFetchedAt  DATE_TIME   provenance + quarterly refresh rule
  apifyRunId         TEXT        provenance (actor build recorded in rawPayload.linkedin._meta)
  enrichmentSources  MULTI_SELECT LINKEDIN_APIFY, COMPANY_WEBSITE, ALEXANDRIA, IMPORT_DATA
  contactName        TEXT        business contact only (optional)
  contactTitle       TEXT
  contactEmail       EMAILS
  province           SELECT      PROVINCE
  productCategory    SELECT      PRODUCT_CATEGORY
  employeeBand       SELECT      LT_50, B_50_500, GT_500, UNKNOWN
  exportRevenueBand  SELECT      EXPORT_REVENUE_BAND (+UNKNOWN)
  euExportEvidence   SELECT      NONE, WEAK, STRONG
  hasEuAr            SELECT      YES, NO, UNKNOWN
  recommendedStream  → ProductStream                     (Feature B)
  icpScore           NUMBER      0–100
  icpTier            SELECT      A, B, C, REJECT
  scoreRationale     RICH_TEXT
  scoreCitations     RAW_JSON    string[]
  scoreModel         TEXT        e.g. claude-opus-5-5
  scoreRubricVersion NUMBER
  scoredAt           DATE_TIME
  dedupeKey          TEXT
  duplicateOfCompany → Company   set when dedupe hits an existing Company
  reviewStatus       SELECT      PENDING_SCORE, PENDING_REVIEW, ACCEPTED, REJECTED, DUPLICATE
  rejectReason       SELECT      NOT_MANUFACTURER, NO_EU_EXPORTS, TOO_SMALL, TOO_LARGE, EXISTING_CLIENT, OTHER
  reviewedBy         → WorkspaceMember
  promotedCompany    → Company
  promotedPerson     → Person

LeadImport (leadImports) "Lead import · 线索导入"
  name, source (SELECT as above), file FILES (CSV/XLSX), columnMap RAW_JSON,
  status SELECT (UPLOADED, PARSING, ENRICHING, SCORING, DONE, FAILED), rowCount, acceptedCount NUMBER,
  apifyRunId TEXT, anthropicBatchId TEXT, error TEXT, candidates ← LeadCandidate (one-to-many)

Company  (+)  icpScore NUMBER, icpTier SELECT, euExportEvidence SELECT, hasEuAr SELECT,
              leadSource SELECT (F0.5), sourceRefs RAW_JSON           (standard linkedinLink, domainName reused)
Person   (+)  leadSource SELECT, dataSource TEXT, lawfulBasis SELECT (LEGITIMATE_INTEREST, CONSENT, CONTRACT),
              art14NoticeSentAt DATE                                   (GDPR, for indirectly sourced contacts)
```

### 4. UI/UX
- **Views:**
  - `Lead review · 线索审核`: TABLE on LeadCandidate, filter `reviewStatus = PENDING_REVIEW`, sorted by `icpScore` descending.
  - `Leads by tier`: KANBAN grouped by `icpTier`.
  - `Lead imports`: TABLE.
- **LeadCandidate record page layout** (`lead-candidate-record.page-layout.ts`):
  - **Overview:** a `LeadScoreCard` front component (score dial, tier pill, evidence bullets, citations as links, duplicate warning linking the existing Company, **Accept / Reject** buttons that call `/s/leads/accept` and `/s/leads/reject`), then FIELDS.
  - **Raw data:** FIELDS.
  - **Activity:** TIMELINE.
- **LeadImport record:** a `LeadImportPanel` front component showing the uploaded file's first 5 rows, a column-mapping dropdown (name, nameZh, website, booth, product, contact…), an **Enrich from LinkedIn (Apify)** checkbox (on by default), the estimated Apify cost (rows × ~$0.004), and a **Run scoring** button.
- **LeadScoreCard** also shows a "LinkedIn (via Apify)" block: employees, HQ, industry, specialties, fetched date and a link to the company page.
- **Today dashboard:**
  - `LeadQuickAdd` front component: a text box for a LinkedIn company URL, a company name or a website, with a "Score it" button. It runs Apify lookup + website enrichment + scoring.
  - `NewLeadsWidget`: count of A-tier leads in the last 7 days, linking to the review view.
- **Accept behaviour:**
  - Create or merge a Company (name, nameZh, domainName, linkedinLink, province, productCategory, exportRevenueBand, icp*, leadSource).
  - Create a Person if a contact exists, with `leadStatus = NEW`. It appears on the existing `Fair leads · 展会线索` kanban.
  - Optionally create an Opportunity (stage LEAD, `productStream` = recommended).

### 5. Integration points
- **Twenty logic functions:**
  - `httpRoute` for quick-add, accept and reject.
  - `databaseEvent` `leadImport.created` → parse.
  - `cron` every 10 minutes → poll Anthropic batches and Apify runs.
  - `cron` weekly → Safety Gate feed.
  - Optional `httpRoute` `/s/leads/apify-webhook` (Apify run-succeeded webhook, protected by a shared secret).
- **Apify API** (`apify-client`, actor `harvestapi/linkedin-company`, pinned build): LinkedIn company firmographics (D1).
- **Firecrawl Alexandria** (optional, A2 evaluation): MCP connector on the enrichment call; company-level providers only.
- **Claude API:** Messages, structured outputs, Batches, `web_fetch` server tool.
- **Feature B:** ICP rubric per stream; `recommendedStream`.
- **Feature E:** enquiries are scored with the same function, and the score is shown on the Ticket.
- **Extra #1 (fair capture):** card captures create LeadCandidates.
- **LinkedIn Marketing API** (A3): Lead Sync webhook → `/s/leads/linkedin-leadgen`.
- **Keycloak / Twenty:** the reviewer is a workspace member.
- **Portal:** not involved.

### 6. Phases
| Phase | Scope | Deliverable |
|---|---|---|
| **A1** (M) | LeadCandidate + LeadImport; CSV/XLSX import with column mapping; **Apify LinkedIn company enrichment** (name → company lookup; guardrails a–g; DPA signed); dedupe; Claude scoring (sync if 20 rows or fewer, batch above); review views; Accept → Company/Person/Opportunity. | Post-fair: the Canton Fair / CIBF exhibitor lists and captured cards are enriched with LinkedIn firmographics, scored and triaged within a day. |
| **A2** (M) | Website enrichment via `web_fetch`; `LeadQuickAdd` (LinkedIn URL / name / website → Apify + website + score); Safety Gate weekly feed; GDPR Art. 14 notice template + `art14NoticeSentAt` tracking; enquiry scoring hook; **Firecrawl Alexandria evaluation** (request access; test company-data providers on 50 known candidates; adopt as MCP enrichment only if it adds signal Apify + website don't). | Day-to-day prospecting flow for staff; go/no-go on Alexandria. |
| **A3** (L) | LinkedIn Lead Gen Forms sync (needs ad spend + API approval); import-data provider integration; calibration report (tier vs actual win rate) → rubric v2. | Paid-channel leads arrive automatically; scoring is tuned on real outcomes. |

### 7. Dependencies
- Required:
  - Claude API (F0.7) and Phase 0 logic functions.
  - **Apify account + API token** (pay-per-event, about $3–4 per 1,000 companies, with a monthly spend cap) and **Apify's DPA signed** (D1).
- Optional:
  - **Firecrawl Alexandria** access (limited access; pricing not public).
  - LinkedIn Marketing Developer Platform access + Campaign Manager.
  - An import-data subscription.
- **Explicitly not used:**
  - a scraper or crawler we build or run ourselves;
  - a profile-reading browser extension;
  - Apify actors that need cookies or a LinkedIn login;
  - Apify actors that scrape *person* profiles.
  - If any of these is ever wanted, get legal review first.

### 8. Effort: **L** overall (A1 M, A2 M, A3 L).

---

## Feature B — Product Streams

### 1. Summary
Make business lines first-class CRM records: a `ProductStream` with its own `Offering` catalogue and lifecycle `StreamStage`s. This replaces the hard-coded `productLine` select (DPP/AR/TRAINING/BUNDLE) on Opportunity. Each stream has its description, target market, ICP rubric, regulations, documents, pipeline and KPIs in one place. It is the foundation for C (pricing attaches to offerings), D (research briefs are per stream) and A (ICP rubric per stream).

### 2. Technical approach
- **Where it lives:** CRM only. Custom objects, a record page layout, views, two front components, and a seed script `crm/app/ops/seed-catalogue.mjs`. The seed is idempotent and upserts by `code`, from the canonical Digest §3.1 / legal Schedule 2 pricing and the `ServiceCatalog.tsx` "Four Pillars".
- **Naming:** use **"Offering"**, not "Product", for what Integra sells. "Product" already means the *client's* goods in both the portal (`products` table) and the CRM (`MandateProduct`).
- **Kanban constraint:** Twenty kanban groups only by SELECT fields, so data-driven per-stream stages can't drive kanban columns. The decision:
  - Keep the global `stage` (LEAD → … → LOST) for the pipeline kanban.
  - Add per-stream kanban views filtered by `productStream`.
  - Use `StreamStage` as the stream-specific delivery lifecycle, shown in a stepper widget on each Opportunity.
  - If a stream later needs its own kanban columns, add a dedicated SELECT (e.g. `arStage`) deliberately, as a code change.
- **Migration:** add `Opportunity.productStream` and backfill from `productLine` (DPP→DPP, AR→AR, TRAINING→TRAINING, BUNDLE→DPP plus OpportunityLines for both). Keep `productLine` read-only for one release, then remove it from views.

### 3. Data model (Twenty)
```
ProductStream (productStreams) "Product stream · 产品线"
  name / nameZh      TEXT
  code               TEXT        immutable key: DPP, AR, TRAINING, AUTHORITY_ADVISORY (used in Stripe metadata,
                                 pricing.json, research briefs); validated unique by logic function
  status             SELECT      IDEA, PILOT, ACTIVE, PAUSED, RETIRED
  tagline / taglineZh TEXT
  description        RICH_TEXT
  targetMarket       RICH_TEXT   segments, geographies, buyer roles (外贸经理, 合规官, GM)
  icpRubric          RICH_TEXT   consumed by Feature A scorer
  icpRubricVersion   NUMBER
  regulations        MULTI_SELECT ESPR_2024_1781, BATTERY_REG_2023_1542, MSR_2019_1020, GPSR_2023_988,
                                  CE_NLF, EPR, OTHER
  keyDeadline        DATE        e.g. 2027-02-18 for DPP
  keyDeadlineLabel   TEXT
  owner              → WorkspaceMember
  documents          FILES       decks, one-pagers EN/ZH, contract templates
  websiteSection     SELECT      DPP, AR, TRAINING, AUTHORITIES (anchor on integra-web)
  position           NUMBER
  offerings ← Offering · stages ← StreamStage · opportunities ← Opportunity
  pricingStrategies ← PricingStrategy (C) · researchBriefs ← ResearchBrief (D)

StreamStage (streamStages)
  name / nameZh, code TEXT, stream → ProductStream, position NUMBER,
  kind SELECT (OPEN, WON, DELIVERY, LOST), probability NUMBER (0–100), exitCriteria TEXT

Offering (offerings) "Offering · 服务项目"
  name / nameZh, code TEXT (DPP_SUBSCRIPTION, DPP_ADDON_BATTERY, DPP_ADDON_TEXTILES, DPP_ADDON_CPR,
                             DPP_EPREL_REGISTRATION, DPP_AI_CHATBOT, DPP_PORTAL_CUSTOMISATION, DPP_READINESS_ASSESSMENT,
                             AR_SUBSCRIPTION, AR_SECTOR_MACHINERY|RED|EMC|LVD|TOYS|PPE|CPR|PED|ATEX,
                             TRN_BATTERY_PASSPORT, TRN_ESPR, TRN_CARBON_FOOTPRINT, TRN_APPOINTING_AR,
                             TRN_MARKET_SURVEILLANCE, TRN_TECHNICAL_FILE, TRN_COMPANY_LIBRARY, TRN_ONSITE_DAY,
                             ADV_DAY_RATE, ADV_DELIVERABLE, BUNDLE_AR_DPP)
  stream → ProductStream
  kind SELECT (SUBSCRIPTION, ADD_ON, ONE_OFF_SERVICE, COURSE, DAY_RATE, BUNDLE)
  unit SELECT (PER_YEAR, PER_MONTH, PER_SEAT, PER_DAY, ONE_OFF)
  description / descriptionZh RICH_TEXT
  sectors MULTI_SELECT (PRODUCT_CATEGORY values)
  portalEntitlement SELECT (NONE, DPP, AR, BUNDLE)    ↔ portal enum subscription_type
  status SELECT (DRAFT, ACTIVE, RETIRED)
  deliverableChecklist RICH_TEXT
  stripeProductId TEXT                                 (C3)

OpportunityLine (opportunityLines)        — shared with Feature C
  name TEXT, opportunity → Opportunity, offering → Offering, pricePoint → PricePoint (C),
  quantity NUMBER, unitAmount CURRENCY, discountPercent NUMBER, lineTotal CURRENCY (computed)

Opportunity (+)  productStream → ProductStream, streamStage → StreamStage, lines ← OpportunityLine
ArMandate   (+)  offering → Offering
TrainingEvent(+) offering → Offering
```

Seeded lifecycles:
- **DPP:** Lead → Readiness assessment → Proposal → Onboarding / data collection → Passport live → Renewal due | Lost
- **AR:** Lead → Mandate sent → Signed → Active → Expiring → Lapsed | Lost. This mirrors `MANDATE_STATUS`.
- **Training:** Interest → Registered → Attended → Certified → Upsold
- **Authority advisory:** Contact → Scoping → Proposal → Engagement → Delivered

### 4. UI/UX
- **Navigation:** "Product streams · 产品线" (OBJECT).
- **ProductStream record page** (`product-stream-record.page-layout.ts`), with these tabs:
  - **Overview:** a `StreamKpiWidget` front component (open pipeline €, weighted pipeline from `StreamStage.probability`, won ARR, active clients, renewals within 90 days, open enquiries in this category), then FIELDS.
  - **Offerings:** VIEW widget.
  - **Pricing:** VIEW widget (C).
  - **Pipeline:** VIEW widget of Opportunities.
  - **Lifecycle:** VIEW widget of StreamStages.
  - **Research:** VIEW widget (D).
  - **Documents:** FILES.
  - **Activity:** TIMELINE.
- **Views:**
  - `Streams` table.
  - `Offerings` table, sorted by stream.
  - `Pipeline · DPP`, `Pipeline · AR`, `Pipeline · Training`: kanban by `stage`, filtered by `productStream`, summing `amount`, cloned from `pipeline-kanban.view.ts`.
- **Opportunity record:** a `StreamStageStepper` front component (clickable stages of the opportunity's stream) and an `OpportunityLine` VIEW widget.
- **Today dashboard:** `StreamSummaryWidget`, one row per ACTIVE stream.

### 5. Integration points
- **A:** `icpRubric`.
- **C:** offerings carry price points; `code` goes into Stripe Product metadata.
- **D:** briefs per stream.
- **E:** ticket category = stream code; the contact form's "service interest" values = stream codes.
- **Website:** optionally, stream tagline and description are published with the pricing snapshot (C).
- **Portal:** `Offering.portalEntitlement` ↔ `subscription_type` via Stripe price metadata.

### 6. Phases
| Phase | Scope | Deliverable |
|---|---|---|
| **B1** (S) | ProductStream / StreamStage / Offering objects; seed script; `Opportunity.productStream` + backfill; per-stream pipeline views; nav. | The four streams and about 40 offerings exist in the CRM, and every opportunity has a stream. |
| **B2** (M) | Record page layout; `StreamKpiWidget`; `StreamStageStepper`; OpportunityLine; ArMandate/TrainingEvent → Offering; `productLine` deprecated. | A full stream workspace for each business line. |
| **B3** (S) | Stream copy publishing via the C pipeline; stream-level reports. | Website service copy can be driven from the CRM (optional). |

### 7. Dependencies: Phase 0 only. No external services.

### 8. Effort: **M**.

---

## Feature C — Pricing Strategies Engine

### 1. Summary
Versioned pricing strategies in the CRM, typed as **flat, tiered, bundle, per-seat, add-on, quote-only or custom**, attached to offerings and gated by an approval workflow. **Publishing** commits a whitelisted JSON snapshot to `integra-web`, where it renders with the component that suits its type (tier table, bundle comparison, price card…). Staff control visibility per strategy. Approved prices sync to **Stripe** as Products/Prices with stable lookup keys. Per-client **price agreements** flow into opportunities, mandates and Stripe Quotes, Coupons and Invoices.

### 2. Technical approach
Three parts:

**(a) Authoring and approval in Twenty**
- Objects are described below.
- A validation logic function runs on `pricingStrategy.updated` / `pricePoint.*` events, with `updatedFields: ['status', ...]`. Rules:
  - TIERED needs at least 2 points with `tierCode`.
  - BUNDLE points need BundleItems, and the computed savings must match.
  - `amount > 0` unless `isOnRequest`.
  - CUSTOM may only be `INTERNAL`.
  - Only one PUBLISHED strategy per (offering, websiteSection); publishing a new version archives the previous one.
  - **`fromPrefix` is recomputed on every save and publish**, overwriting any manual edit: `fromPrefix = strategyType === 'ADD_ON' || hasOptionalExtras` (D3; see "Price display rule" below).
  - `appliesTo` may only be set on ADD_ON strategies.
  - **Warning, not an error:** if an APPROVED/PUBLISHED ADD_ON strategy `appliesTo` this strategy but `hasOptionalExtras` is false, `PricingPreview` asks "add-ons extend this strategy — should prices read 'from'?" Staff decide.
- Transitions to `APPROVED` / `PUBLISHED` are allowed only for approvers (Twenty Admin role = Keycloak group `business-dev-chief`). If database events don't carry the actor (F0.3c), transitions go through front-component buttons calling routes that check the caller.

**(b) Publishing to the website (static export, so publish = git commit)**
- The **Publish** button calls the `/s/pricing/publish` logic function, which:
  1. builds a `PublicPricingV1` snapshot from **only** `status=PUBLISHED` and `visibility = PUBLIC`, with an explicit field whitelist (never `floorAmount`, Stripe IDs, internal notes or CUSTOM);
  2. commits `integra-web/data/pricing.json` through the GitHub Contents API, using a GitHub App or fine-grained token with `contents:write` on this repo only;
  3. lets the existing `deploy.yml` (push to main) build and deploy.
- **Preview** commits to branch `pricing-preview`, which yields a preview URL on the live Pages project `integra-scientific` (D6). This needs `deploy.yml` to also deploy non-main branches with `--branch=${{ github.ref_name }}`, which is a workflow change.
- Why git rather than a runtime fetch: the site has no server, git is an audit log of public prices with one-click revert, and builds never depend on the CRM being up.
- `integra-web/scripts/validate-pricing.mjs` runs in `prebuild` and validates the JSON against a zod schema (zod as a devDependency). A failed validation fails the build, so the live site keeps the last good snapshot.

**(c) Stripe**
- **Catalogue sync, from the CRM** (logic function on PricePoint `APPROVED`; uses a **restricted** Stripe key with write access to Products, Prices, Coupons, Promotion Codes and Quotes only):
  - Product = Offering, `metadata: {offeringCode, streamCode}`.
  - Price = PricePoint: `currency: eur`, `recurring.interval: year` (or one-off), `tax_behavior: exclusive`, `lookup_key: "{offeringCode}_{tierCode}_{interval}"` (e.g. `DPP_SUBSCRIPTION_BOOST_year`), `metadata: {tier, subscriptionType, limits, twentyPricePointId, strategyVersion}`.
  - Stripe Prices are immutable, so an amount change creates a new Price with `transfer_lookup_key: true` and deactivates the old one. Existing subscribers stay on their old price (grandfathering) until they are migrated deliberately.
- **Billing, in the portal (spec Phase 6, already planned):**
  - Checkout resolves `prices.list({ lookup_keys })`.
  - A webhook reads `price.metadata.tier` / `subscriptionType` and updates `organisations`.
  - Tier limits are enforced in `dpps/create.ts`.
  - The **spec amendment** recording the lookup-key/metadata contract is **approved (D5)**. Write it into the spec before Phase 6 starts.
- **Per-client agreements:**
  - Discount only → Stripe **Coupon** + customer-restricted **Promotion Code**, sent with the checkout link.
  - Custom amount or multi-line → Stripe **Quote** with `price_data` line items. The Quote is the billing object; **acceptance happens through eIDAS signing (D2, section S), not Stripe's hosted "accept" page**:
    1. The finalised quote PDF (bilingual, from extra #3) goes to the client through the QTSP. Integra applies a qualified electronic seal and the client signs at the level section S sets for quotes.
    2. On the QTSP's "signed" event, the signing service calls `stripe.quotes.accept(quoteId)`, which creates the subscription.
    3. The agreement moves to ACCEPTED.
  - **Boss** tier → Stripe **Invoice** with `collection_method: send_invoice`, net 30, matching the legal Schedule.
  - Payment methods per spec §8.2: card, SEPA Debit, Alipay. **Verify Alipay with recurring subscriptions.** Annual `send_invoice` lets clients pay each invoice by Alipay on the hosted invoice page.

**Strategy type → display mapping**

| strategyType | Example (canonical pricing) | displayFormat (default, overridable) | Price display (D3) | Website component |
|---|---|---|---|---|
| `FLAT` | EPREL registration €500; DPP Readiness Assessment | `PRICE_CARD` | **Exact total**: "€500" | `PriceCard`: price, unit, bullets, CTA |
| `TIERED` | AR €250 / €1,200 / €3,000 / on request; DPP €950 / €2,500 / €6,000 | `TIER_TABLE` | **Exact total per tier**: "€250", "€1,200", "€3,000", "On request". "from" per tier only if `hasOptionalExtras` | `TierTable`: column per tier, limits row ("≤10 products"), feature rows, highlighted tier |
| `BUNDLE` | AR+DPP €1,020 / €3,145 / €7,650 (−15%) | `BUNDLE_COMPARISON` | **Exact total per bundle option**: "€1,020" (was €1,200, save 15%). "from" only if `hasOptionalExtras` | `BundleComparison`: components, struck-through list total (€1,200), bundle price, "Save 15%" computed |
| `PER_SEAT` | Training €150 live / €89 recorded; −20% for 5+ seats | `SEAT_PRICING` | **Exact per seat**: "€150 / seat". The volume discount is shown as a separate rule. "from" only if `hasOptionalExtras` | `SeatPricing`: per-seat prices, volume rule, company library |
| `ADD_ON` | Battery Passport €1,500/yr; sector add-ons €600–2,000 | `ADD_ON_LIST` | **Always "from €X"**: "from €1,500 / yr" | `AddOnList`: grouped compact table |
| `QUOTE_ONLY` | Boss tier; authority advisory | `CONTACT_CTA` | "On request / 价格面议" | `ContactCta`: "On request" plus a link to `/contact?interest=AR&tier=BOSS`, which prefills the Feature E form |
| `CUSTOM` | Client-specific | — never public | — | — |

**Price display rule (founder decision D3)**
- One flag per **strategy**: `fromPrefix`. It is computed, never typed in.
  ```js
  // crm/app/shared/public-pricing.mjs — imported by the validator, the publish function and verify-model tests
  export const computeFromPrefix = ({ strategyType, hasOptionalExtras }) =>
    strategyType === 'ADD_ON' || hasOptionalExtras === true;
  ```
- **`fromPrefix = true`** means every amount in that strategy renders as an indicative minimum: "from €X" / "€X 起".
- **`fromPrefix = false`** means every amount renders as the exact total: "€X". This covers FLAT, TIERED, BUNDLE and PER_SEAT by default, i.e. strategies whose price is fully determined up front.
- **`hasOptionalExtras`** is a staff-set BOOLEAN on the strategy. Tick it only when the client can add priced extras *within* that strategy, so the shown price is a floor rather than a total.
- **Seeding choice:** `hasOptionalExtras = false` on the AR tiers, DPP tiers and AR+DPP bundle, so they show exact totals per tier as D3 requires. DPP and sector add-ons are published as their own ADD_ON strategies (always "from") and linked with `appliesTo`, so the tier table can show an "Optional add-ons ↓" link. **Open question for the founder:** should DPP tiers count as "with optional extras" (→ "from €950") because add-ons exist? The rule supports either; it's one checkbox.
- **On-request** points (`amount: null`) always render "On request / 价格面议", whatever `fromPrefix` is.

Visibility options (D3 replaces the old `PUBLIC_INDICATIVE` option: "from" is now decided by the rule above, not by visibility):
- `INTERNAL`
- `PORTAL` (future portal billing page)
- `PUBLIC`: published to the website, rendered per the rule.

### 3. Data model
```
PricingStrategy (pricingStrategies) "Pricing strategy · 定价策略"
  name TEXT · stream → ProductStream · offering → Offering (null for multi-offering bundles)
  strategyType SELECT (FLAT, TIERED, BUNDLE, PER_SEAT, ADD_ON, QUOTE_ONLY, CUSTOM)
  displayFormat SELECT (PRICE_CARD, TIER_TABLE, BUNDLE_COMPARISON, SEAT_PRICING, ADD_ON_LIST, CONTACT_CTA)
  visibility SELECT (INTERNAL, PORTAL, PUBLIC)
  hasOptionalExtras BOOLEAN (staff-set; client can add priced extras within this strategy)
  fromPrefix BOOLEAN (computed: strategyType = ADD_ON or hasOptionalExtras; overwritten on save — do not edit)
  appliesTo → PricingStrategy (ADD_ON only: the strategy these add-ons extend) · optionalExtras ← PricingStrategy
  status SELECT (DRAFT, PENDING_APPROVAL, APPROVED, PUBLISHED, ARCHIVED)
  version NUMBER · supersedes → PricingStrategy
  currency SELECT (EUR) · billingInterval SELECT (YEAR, MONTH, ONE_OFF)
  effectiveFrom / effectiveTo DATE
  publicHeadline / publicHeadlineZh TEXT · publicFootnote / publicFootnoteZh TEXT ("EUR excl. VAT · net 30")
  websiteSection SELECT (DPP, AR, TRAINING, AUTHORITIES, PRICING_PAGE) · displayOrder NUMBER
  approvedBy → WorkspaceMember · approvedAt DATE_TIME · publishedAt DATE_TIME
  rationale RICH_TEXT (why this price; links to D benchmarks)
  pricePoints ← PricePoint · publications ← PricingPublication

PricePoint (pricePoints)
  name TEXT · strategy → PricingStrategy · offering → Offering
  tierCode SELECT (BEGINNER, BOOST, BUILDER, BOSS, NONE)       = existing TIER options
  label / labelZh TEXT · amount CURRENCY · isOnRequest BOOLEAN
  floorAmount CURRENCY (internal minimum for discounting)        (no fromPrefix here — it is per strategy, D3)
  limits RAW_JSON  {"maxProducts":10} / {"maxPassports":50} / {"minSeats":5}
  features / featuresZh RICH_TEXT (one bullet per line) · highlight BOOLEAN · position NUMBER
  stripeProductId · stripePriceId · stripeLookupKey TEXT
  stripeSyncStatus SELECT (NOT_SYNCED, SYNCED, ERROR) · stripeSyncError TEXT

BundleItem (bundleItems)
  bundlePricePoint → PricePoint · componentPricePoint → PricePoint · quantity NUMBER

ClientPriceAgreement (clientPriceAgreements) "Price agreement · 价格协议"
  name · company → Company · opportunity → Opportunity · pricePoint → PricePoint (base)
  type SELECT (DISCOUNT, CUSTOM_AMOUNT, CUSTOM_BUNDLE)
  discountPercent NUMBER · customAmount CURRENCY · discountReason SELECT (FOUNDER_CLIENT, MULTI_YEAR,
     VOLUME, COMPETITIVE_MATCH, PARTNER, OTHER) · notes RICH_TEXT
  validFrom / validTo DATE
  status SELECT (DRAFT, PENDING_APPROVAL, APPROVED, SENT, ACCEPTED, DECLINED, EXPIRED)
  approvalRequired BOOLEAN (computed: discount > 15% or amount < floorAmount) · approvedBy → WorkspaceMember
  stripeCouponId · stripePromotionCode · stripeQuoteId · stripeInvoiceId · stripeSubscriptionId TEXT
  signatureRequestId TEXT · signatureStatus SELECT (NOT_SENT, SENT, SIGNED, DECLINED, EXPIRED)   (eIDAS, section S)

PricingPublication (pricingPublications)     — audit log
  name · target SELECT (PREVIEW, PRODUCTION) · snapshot RAW_JSON · commitSha TEXT · commitUrl LINKS
  publishedBy → WorkspaceMember · publishedAt DATE_TIME · status SELECT (OK, FAILED) · error TEXT

Company (+)     stripeCustomerId TEXT · priceAgreements ← ClientPriceAgreement
Opportunity (+) priceAgreement → ClientPriceAgreement
ArMandate (+)   pricePoint → PricePoint · priceAgreement → ClientPriceAgreement   (annualFee auto-filled)
TrainingEvent(+) pricePoint → PricePoint
```

**Portal (spec amendment and migration approved, D5):**
```ts
// packages/core/src/db/schema/billing.ts
stripe_events:  id varchar PK (Stripe evt_ id), type varchar, received_at, processed_at, error text   // idempotency
subscriptions:  id uuid PK, organisation_id FK → organisations, stripe_subscription_id unique,
                stripe_price_lookup_key, tier tierEnum, subscription_type subscriptionTypeEnum,
                status varchar (active|past_due|canceled|incomplete|trialing), current_period_end timestamptz,
                cancel_at_period_end boolean, created_at, updated_at
```

**Public snapshot contract** (`integra-web/lib/pricing/schema.ts`, mirrored by `crm/app/shared/public-pricing.mjs`; a contract test runs a CRM fixture through the website validator):
```ts
type L = { en: string; zh: string };
type PublicPricingV1 = {
  schemaVersion: 1; generatedAt: string; currency: 'EUR'; vatNote: L;
  streams: Array<{ code: string; name: L; tagline?: L;
    strategies: Array<{ id: string;
      strategyType: 'FLAT'|'TIERED'|'BUNDLE'|'PER_SEAT'|'ADD_ON'|'QUOTE_ONLY';          // CUSTOM can never appear
      displayFormat: 'PRICE_CARD'|'TIER_TABLE'|'BUNDLE_COMPARISON'|'SEAT_PRICING'|'ADD_ON_LIST'|'CONTACT_CTA';
      hasOptionalExtras: boolean;
      fromPrefix: boolean;                 // per STRATEGY (D3): must equal strategyType === 'ADD_ON' || hasOptionalExtras
      optionalExtrasStrategyIds?: string[]; // ADD_ON strategies that extend this one → "Optional add-ons ↓" anchor link
      headline: L; footnote?: L; billingInterval: 'year'|'month'|'one_off';
      points: Array<{ tierCode?: 'BEGINNER'|'BOOST'|'BUILDER'|'BOSS'; label: L; amount: number | null; // null = on request
        limits?: Record<string, number>; features: { en: string[]; zh: string[] }; highlight: boolean;
        bundle?: { components: Array<{ label: L; amount: number }>; listTotal: number; savingsPercent: number } }>;
      cta: { kind: 'contact' | 'portal'; href: string } }> }> };
```
- The website validator (`scripts/validate-pricing.mjs`, zod) **rejects the snapshot** if `fromPrefix !== (strategyType === 'ADD_ON' || hasOptionalExtras)` for any strategy. That way a hand-edited or stale `pricing.json` can't show "from" on a fixed price, or an exact total on an add-on.
- Compared with the first draft, this drops the per-point `fromPrefix` and the strategy-level `indicativeOnly`.

### 4. UI/UX
**CRM:**
- **PricingStrategy record page:**
  - **Overview:** FIELDS, plus a `PricingPreview` front component. It renders an approximation of the public card, lists validation errors live, and has **Approve / Preview / Publish** buttons.
  - **Price points:** VIEW widget.
  - **Bundle items:** VIEW widget.
  - **Publications:** VIEW widget.
  - **Benchmarks:** VIEW of D's CompetitorPriceObservation filtered to the stream.
  - **Activity:** TIMELINE.
- **Views:**
  - `Pricing · 定价`: table grouped by stream.
  - `Pending approval`: table.
  - `Price agreements`: kanban by status.
- **Opportunity record:** a `QuoteBuilder` front component. Pick offering → price point → quantity → discount; it shows approval-required state and creates OpportunityLines + ClientPriceAgreement. `Opportunity.amount` = sum of lines.

**Website (`integra-web`):**
- New `app/pricing/page.tsx`, bilingual via `Bi`.
- `components/pricing/{PricingSection,PriceCard,TierTable,BundleComparison,SeatPricing,AddOnList,ContactCta}.tsx`. `PricingSection` switches on `displayFormat`.
- **Every amount goes through one formatter.** No component decides "from" on its own; each passes the *strategy's* `fromPrefix`:
  ```ts
  // integra-web/lib/pricing/format.ts
  export function formatPrice(amount: number | null, fromPrefix: boolean, lang: 'en' | 'zh'): string {
    if (amount === null) return lang === 'en' ? 'On request' : '价格面议';
    const eur = new Intl.NumberFormat(lang === 'en' ? 'en-IE' : 'zh-CN',
      { style: 'currency', currency: 'EUR', maximumFractionDigits: 0 }).format(amount);
    if (!fromPrefix) return eur;                        // fixed price → exact total
    return lang === 'en' ? `from ${eur}` : `${eur} 起`;  // add-ons / optional extras → indicative minimum
  }
  ```
  - `TierTable`, `BundleComparison`, `PriceCard` and `SeatPricing` show exact totals unless the strategy's `fromPrefix` is true.
  - `AddOnList` always shows "from", because ADD_ON makes `fromPrefix` true by construction.
  - `TierTable` and `BundleComparison` render an "Optional add-ons ↓" link when `optionalExtrasStrategyIds` is present. That link doesn't change their own prices.
- **`ServiceCatalog` pillar price chips** follow the same rule:
  - FLAT (exact) → "€500".
  - TIERED or BUNDLE (exact) → the range of exact tier totals, e.g. "€250 – €3,000 / yr", plus "Boss: on request".
  - Any strategy with `fromPrefix` → "from €min".
  - This replaces the first draft's blanket "From €X" chips.
- A NavBar "Pricing" link.
- Tests:
  - **Vitest/unit** on `formatPrice` (EN/ZH, null, from/exact) and on `computeFromPrefix` (ADD_ON → true; FLAT/TIERED/BUNDLE/PER_SEAT → false; any type with `hasOptionalExtras` → true).
  - **Playwright:**
    - Render a fixture at the 8 configured viewports.
    - Assert no INTERNAL or CUSTOM strings leak.
    - Assert bundle savings maths (1,200 → 1,020 = 15%).
    - Assert "On request" renders a contact CTA.
    - **Assert "from"/"起" never appears inside any exact-price strategy's section, and appears on every amount of an ADD_ON section.**
  - **Contract test:** a CRM fixture with an inconsistent `fromPrefix` fails `validate-pricing.mjs`.

### 5. Integration points
- **Feature B** (offerings) and **Feature D** (benchmarks shown during pricing decisions).
- **GitHub** Contents API + Actions; **Cloudflare Pages** (production + preview).
- **Stripe** (catalogue from CRM; billing in portal Phase 6).
- **Portal:** `organisations.tier/subscriptionType/stripeCustomerId`, new `subscriptions` / `stripe_events`, webhook route `apps/portal/src/app/api/webhooks/stripe/route.ts`. This route must be **added to `lib/auth/public-routes.ts`**, and `StripeWebhookSecret` must be **linked to the Portal in `infra/web.ts`** (today only `StripeSecretKey` is linked).
- **Feature E:** quote-only CTAs prefill the enquiry form.
- **Extra #3 + section S:** quote and mandate generation, signed through the eIDAS QTSP. A signed quote triggers `stripe.quotes.accept`.

### 6. Phases
| Phase | Scope | Deliverable |
|---|---|---|
| **C1** (M) | PricingStrategy / PricePoint / BundleItem / PricingPublication; seed canonical pricing (AR tiered, DPP tiered, AR+DPP bundle, DPP add-ons, sector add-ons, training per-seat, advisory quote-only); validation; `PricingPreview`; publish via git commit; `/pricing` page + components + tests. | Canonical prices live on the website, controlled and audited from the CRM, shown per the D3 rule: exact totals for fixed / tiered / bundle, "from" for add-ons and strategies with optional extras. |
| **C2** (M) | ClientPriceAgreement; OpportunityLine; `QuoteBuilder`; discount approval rules; mandate/training price linkage; preview-branch deployments. | Every deal and mandate carries a traceable price; discounts are approved. |
| **C3** (L, joint with portal Phase 6) | Stripe catalogue sync; Coupons / Quotes / Invoices for agreements; portal Checkout + webhook + `subscriptions` / `stripe_events` + tier enforcement; the approved spec amendment (D5) for the lookup-key contract, written into the spec. Signed quotes (section S) → `stripe.quotes.accept`. | Customers can pay; tier is set automatically; CRM and Stripe prices cannot drift. |

### 7. Dependencies
- Stripe account (restricted key; decide on Stripe Tax).
- GitHub App or fine-grained PAT.
- Cloudflare Pages.
- Phase 0 and Feature B1.
- Confirm VAT wording with the accountant. B2B services to non-EU clients are generally outside EU VAT scope, but the "excl. VAT" footnote is the safe default.

### 8. Effort: **L** (C1+C2); **XL** including C3 and portal billing.

---

## Feature D — AI Market Research on Product Streams

### 1. Summary
Each product stream gets a staff-editable **research brief**: focus areas, competitor watch-list, regulatory watch-list, cadence and recipients. On schedule, a Claude research agent with web search and fetch produces a **cited report** plus structured **findings**: competitor moves, regulatory updates, pricing benchmarks and demand signals. These are stored in the CRM, turned into tasks for the right people, and emailed as a digest to subscribed staff.

### 2. Technical approach
- **The CRM is the configuration and results store. Claude Managed Agents (beta) runs the research.** Anthropic hosts the agent loop and sandbox, so there is no new AWS compute and no fight with logic-function timeouts. A multi-minute run with dozens of searches and `pause_turn` continuations doesn't fit a short serverless function.
- **Flow:**
  1. `research-scheduler` cron logic function (hourly). It finds `ResearchBrief`s with `nextRunAt ≤ now` (the **Run now** button does the same on demand) and creates a Managed Agents session, passing the agent ID (created once by a setup script and stored as an app secret) and an `initial_events` entry of type `user.define_outcome`. That event carries:
     - **description:** built from the brief (stream, focus areas, competitor list, regulatory list, period since `lastRunAt`, titles and URLs of the last 90 days of findings for dedupe, and the output contract);
     - **rubric:** below;
     - **budget:** `{type: "limit", max_list_cost: {amount: "1000", currency: "USD"}}`, i.e. $10 per run.

     It stores `sessionId` on a new `ResearchReport` with status RUNNING.
  2. **Agent:** `claude-opus-5-5` at `effort: high`, with the agent toolset's `web_search` / `web_fetch` (`blocked_domains: ["linkedin.com"]`). The system prompt is versioned in `crm/app/research/agent.md` and synced with `ant apply`. It writes `/mnt/session/outputs/report.md` and `/mnt/session/outputs/findings.json`. It has **no credentials and no write access to the CRM**, which neutralises prompt injection from fetched pages.
  3. `research-ingest` cron (every 15 minutes). It checks RUNNING reports' sessions; when they are idle or terminated it:
     - lists the output files (`files.list({scope_id: sessionId})`);
     - validates `findings.json` with zod and creates `ResearchFinding`s;
     - upserts `Competitor` and `CompetitorPriceObservation`;
     - creates Twenty **Tasks** for HIGH-importance findings, assigned to the stream owner or the finding's suggested owner;
     - sets the report to READY and sends the **SES digest** to subscribers.

     An optional upgrade is a Managed Agents webhook (`session.status_idled`) to a Twenty httpRoute, verified with the SDK's `webhooks.unwrap()`, instead of polling.
- **Alternative considered:** Managed Agents *scheduled deployments* (Anthropic-side cron). They are simpler to schedule, but every brief edit in the CRM would have to be pushed into the deployment's `initial_events`. CRM-driven scheduling keeps all configuration in one place.
- **Starter rubric** (tune after the first runs):
  1. Every finding has ≥1 source URL that was actually fetched during this run.
  2. Regulatory findings cite an official source (eur-lex.europa.eu, ec.europa.eu, single-market-economy.ec.europa.eu, cencenelec.eu, Official Journal).
  3. Every finding has a publication date; anything older than `lastRunAt` is flagged `background: true`.
  4. No finding duplicates a supplied previous finding (by URL or substance).
  5. Pricing benchmarks state competitor, offering, amount, currency, unit/term and observation URL.
  6. The executive summary is ≤200 words and ends with 3–5 concrete "so what for Integra" actions.
  7. `findings.json` validates against the supplied schema.
- **Seed watch-lists:**
  - **Competitors** (from `strategy/Integra_Scientific_Competitor_Analysis_April2026.docx`, the Service Offering doc and `.firecrawl/`):
    - AR and training: ARC, EU Compliance Partner, Cert-Rep, Compliance Gate, EaseCert, Instrktiv, Authorized.eu
    - DPP: Circularise, Spherity, Scantrust, SupplyOn, Arianee, Kezzler, iPoint, Circulor, TrusTrace, Avery Dennison atma.io
    - Training: TÜV SÜD Academy, SGS Academy
  - **Regulatory, DPP:** ESPR delegated acts and Working Plan; Battery Reg 2023/1542 battery passport provisions and implementing acts; DPP registry; JTC 24 standards status (prEN 18216, 18219–18223, 18239, 18246); CIRPASS-2.
  - **Regulatory, AR:** MSR 2019/1020; GPSR 2023/988 guidance; Safety Gate trends for Chinese-origin products.

### 3. Data model
```
ResearchBrief (researchBriefs) "Research brief · 调研简报"
  name · stream → ProductStream · isActive BOOLEAN
  cadence SELECT (WEEKLY, FORTNIGHTLY, MONTHLY, QUARTERLY, ON_DEMAND) · nextRunAt / lastRunAt DATE_TIME
  focusAreas MULTI_SELECT (MARKET_LANDSCAPE, COMPETITORS, REGULATORY, PRICING_BENCHMARKS, DEMAND_SIGNALS, CHINA_MARKET)
  competitors ← (via Competitor.streamCodes match) · regulatoryWatch RICH_TEXT · customInstructions RICH_TEXT
  preferredDomains RAW_JSON · budgetUsd NUMBER (default 10)
  subscriptions ← ResearchSubscription · reports ← ResearchReport

ResearchSubscription (researchSubscriptions)
  brief → ResearchBrief · member → WorkspaceMember
  channel SELECT (EMAIL_DIGEST, IN_APP_ONLY) · minImportance SELECT (HIGH, MEDIUM, LOW) · language SELECT (EN, ZH)

ResearchReport (researchReports)
  name ("DPP · market brief · 2026-W43") · brief → ResearchBrief · stream → ProductStream
  periodStart / periodEnd DATE · status SELECT (QUEUED, RUNNING, READY, FAILED, ARCHIVED)
  executiveSummary RICH_TEXT · body RICH_TEXT (report.md) · findingsRaw RAW_JSON
  sessionId TEXT · model TEXT · listCostUsd NUMBER · webSearches NUMBER · outcomeResult TEXT
  rating RATING · reviewerNotes TEXT · findings ← ResearchFinding

ResearchFinding (researchFindings)
  title · report → ResearchReport · stream → ProductStream
  category SELECT (COMPETITOR, REGULATORY, PRICING, MARKET, DEMAND_SIGNAL, STANDARDS)
  importance SELECT (HIGH, MEDIUM, LOW) · summary RICH_TEXT · soWhat TEXT
  sourceUrls LINKS · sourcePublishedAt DATE · isOfficialSource BOOLEAN · background BOOLEAN
  competitor → Competitor · suggestedOwner → WorkspaceMember
  status SELECT (NEW, ACKNOWLEDGED, ACTIONED, DISMISSED) · dedupeHash TEXT

Competitor (competitors)
  name · website LINKS · hqCountry TEXT · streamCodes MULTI_SELECT (DPP, AR, TRAINING, AUTHORITY_ADVISORY)
  positioning RICH_TEXT · targetSegment TEXT · threatLevel SELECT (LOW, MEDIUM, HIGH)
  priceObservations ← CompetitorPriceObservation · findings ← ResearchFinding

CompetitorPriceObservation (competitorPriceObservations)
  name · competitor → Competitor · stream → ProductStream · offeringLabel TEXT
  amount CURRENCY · unit SELECT (PER_YEAR, PER_MONTH, ONE_OFF, PER_SEAT, PER_DAY) · conditions TEXT
  observedAt DATE · sourceUrl LINKS · finding → ResearchFinding
```

### 4. UI/UX
- **Navigation:** "Research · 市场调研" (ResearchReport).
- **Views:**
  - `Reports`: table, newest first.
  - `Findings inbox`: kanban by status, filtered to HIGH and MEDIUM.
  - `Competitors`: table.
  - `Price benchmarks`: table grouped by stream.
- **ResearchReport record page:**
  - **Report:** a `ResearchReportView` front component rendering markdown with citation links, plus rating stars.
  - **Findings:** VIEW widget.
  - **Run details:** FIELDS (cost, searches, outcome).
  - **Activity:** TIMELINE.
- **ResearchBrief record:** FIELDS, a `RunResearchNowButton` front component, Subscriptions, and a Reports VIEW widget.
- **ProductStream:** a Research tab (B). **PricingStrategy:** a Benchmarks tab (C).
- **Today dashboard:** a `ResearchInboxWidget` showing new HIGH findings since the member's last visit.
- **Email digest** (SES HTML, EN or ZH per subscription), e.g. subject `[DPP] Weekly market brief — 3 high-priority findings`. It contains the summary, findings grouped by category, an "Open in CRM" link per finding, and a one-click "Dismiss / Acknowledge" link to an httpRoute protected by a signed token.

### 5. Integration points
- Claude Managed Agents (beta) and the Files API.
- SES (F0.6).
- Feature B (streams), Feature C (benchmarks feed pricing rationale), extra #4 (regulatory findings → milestones).
- **integra-web `public/tools/dpp-compass/data.js`:** regulatory findings can propose changes as a **PR** for human review. Never auto-merged.

### 6. Phases
| Phase | Scope | Deliverable |
|---|---|---|
| **D1** (M) | Objects; agent + environment setup (`ant apply`); DPP brief on demand via "Run now"; ingest poller; report page. | The first cited DPP market report lives in the CRM. Quality is reviewed by a human and the rubric tuned. |
| **D2** (M) | Scheduler cron; subscriptions + SES digest; findings → Tasks; Competitor and price observations; briefs for all 4 streams. | Weekly or monthly briefs arrive in staff inboxes automatically. |
| **D3** (S) | Feedback loop (ratings and dismissals fed into the next run's description); ZH digests; Benchmarks on pricing pages; DPP Compass PR proposals. | Research improves with use and feeds pricing and public tools. |

### 7. Dependencies and costs
- **Dependencies:** Anthropic Managed Agents (**beta**, so pin the beta header and expect changes), SES, Phase 0, Feature B1.
- **Cost:** list cost per run = model tokens (Opus 5.5 at $4 / $20 per MTok) + web searches ($10 per 1,000) + session runtime ($0.08/hour). Expect roughly **$2–8 per stream-run**, so about **$35–140/month** for 4 weekly briefs. This is an estimate to verify in D1; it is capped per session by the budget.

### 8. Effort: **M**.

---

## Feature E — Contact Form → CRM Ticketing

### 1. Summary
Replace the `mailto:` form with a real bilingual form. It posts to a Cloudflare Pages Function that verifies Turnstile and creates an **Enquiry** ticket in Twenty: the requester is matched or created, AI triage runs, the ticket is routed, and both staff and requester get emails. Staff then work tickets like Zendesk: assign, change status, add internal notes, reply, track SLA. **Promote to lead** converts a ticket into Company + Person + Opportunity in one click.

### 2. Technical approach
**Website (`integra-web`, static export unchanged):**
- `components/ContactForm.tsx` (`"use client"`) replaces the form in `app/contact/page.tsx`.
- Fields:
  - **Required:** name, work email, company, service interest (DPP / EU Representative / Training / For Authorities / Other; values = stream codes), message, privacy consent (checkbox linking `/privacy`).
  - **Optional:** country (China first), product category (Li-ion / LMT / Textiles / Electronics / Other = `PRODUCT_CATEGORY`), phone or WeChat ID, preferred language (EN / 中文, defaulting from the site toggle), marketing opt-in (unchecked).
  - **Hidden:** a honeypot `website` field, the Turnstile widget, `intakeId` (a UUID generated client-side for idempotency), source page and UTM parameters.
- Prefill from the query string (`/contact?interest=AR&tier=BOSS`, used by C's CTAs).
- Success state shows a bilingual reference (e.g. `ENQ-261003-7K2Q`). On error it shows the old mailto link as a fallback.
- Turnstile loads from `challenges.cloudflare.com`. **Test from a mainland-China vantage point.** If the widget fails to load, accept the submission flagged `spamCheck: UNVERIFIED` behind a stricter rate limit rather than locking out the core market.

**Edge endpoint `integra-web/functions/api/contact.ts`** (a Cloudflare Pages Function on the live project **`integra-scientific`** (D6). It works alongside `output: "export"`. `deploy.yml` already runs `wrangler pages deploy out` with `workingDirectory: integra-web`, so `./functions` is picked up with no workflow change. Set its secrets on `integra-scientific`):
1. Validate the payload, check the honeypot, call Turnstile `siteverify` with secret `TURNSTILE_SECRET`.
2. POST to `https://crm.integrascientific.com/s/enquiries/intake` with a Bearer token (a Twenty API key scoped to a minimal role, stored as a Pages secret), 8-second timeout.
3. Return `{ reference }`.
4. **If the CRM is down or returns 5xx,** send the full submission via SES (signed with `aws4fetch`) to info@ with subject `[CRM intake failed] …`, and return 202. No enquiry is ever lost.
- A Cloudflare WAF rate-limit rule allows 5 requests per minute per IP on `/api/contact`.

**CRM logic functions:**
- `enquiry-intake` (httpRoute POST `/enquiries/intake`, auth required):
  1. Idempotency check on `intakeId`.
  2. Upsert the Person by `emails.primaryEmail`. Set language and `preferredChannel=EMAIL`; for a new person also set `leadSource=WEB_FORM` and `leadStatus=NEW`.
  3. Match a Company by email domain. Skip freemail domains: qq.com, 163.com, 126.com, sina.com, foxmail.com, yeah.net, aliyun.com, gmail.com, outlook.com, hotmail.com, yahoo.com (list in `shared/icp.mjs`).
  4. Create the Ticket and an INBOUND TicketMessage. Raise priority to HIGH if the company has an ACTIVE/EXPIRING ArMandate.
  5. Run `enqueueJobs` → `enquiry-triage`.
  6. Route the ticket with `TicketRoutingRule` (category + language → assignee), falling back to round-robin over the routing pool.
  7. Set `firstResponseDueAt` to +1 business day, skipping weekends.
  8. Send an SES email to the assignee (with CRM deep link) and cc info@.
  9. Send an SES auto-acknowledgement to the requester in their language, with the reference and expected response time.
- `enquiry-triage` (job): Claude `claude-opus-5-5`, `effort: low`, structured output `{category, language, summary (≤40 words), spamLikelihood, urgency, suggestedReply: {en, zh}}`, plus the Feature A ICP scorer. If `spamLikelihood > 0.9`, set status SPAM and suppress notifications.
- `ticket-promote` (httpRoute, called by a button). It is idempotent and:
  - matches or creates the Company (name, domain from email, country, productCategory, nameZh);
  - links Person → Company and sets `leadStatus=QUALIFIED`;
  - creates an Opportunity `{name: "{Company} · {Stream}", stage: LEAD, productStream, pointOfContact, leadSource: WEB_FORM}`;
  - sets `ticket.promotedOpportunity` / `promotedAt` and adds an INTERNAL_NOTE "Promoted by …";
  - optionally sets the ticket to RESOLVED.
- `ticket-sla` (cron, every 15 minutes): recomputes `slaStatus`, notifies the assignee when AT_RISK and business-dev-chief when BREACHED.
- `ticket-retention` (cron, daily): deletes SPAM after 30 days, and anonymises CLOSED, never-promoted tickets after 24 months.

**Replies:**
- **E1:** a "Reply by email" button opens the staff mail client with `Subject: [ENQ-…] Re: …`. If staff mailboxes are connected to Twenty's Google/Microsoft sync, the thread also lands on the Person timeline. Staff update the status manually.
- **E2:** an in-CRM composer sends via SES from `support@integrascientific.com` with `Reply-To: reply+{ticketToken}@reply.integrascientific.com`. Inbound handling:
  - Point the MX for the subdomain `reply.integrascientific.com` (never the root domain's MX) at SES receiving → S3 → Lambda. This is a new CloudFormation `phase7`, using `mailparser` to strip quoted text.
  - The Lambda calls `/s/tickets/inbound-email`, which appends an INBOUND message, sets status to OPEN and notifies the assignee.
  - **SES receiving is only available in some regions.** Check eu-central-1, otherwise use eu-west-1 for the receipt rule set.
  - Bounces and complaints come back via the SES configuration set → `deliveryStatus`.

### 3. Data model
```
Ticket (tickets) "Enquiry · 咨询"
  reference        TEXT        ENQ-YYMMDD-XXXX (label identifier)
  subject          TEXT        (AI summary title if none)
  status           SELECT      NEW, OPEN, PENDING_CUSTOMER, ON_HOLD, RESOLVED, CLOSED, SPAM
  priority         SELECT      LOW, NORMAL, HIGH, URGENT
  category         → ProductStream   (+ SELECT fallback OTHER, PARTNERSHIP, SUPPORT, PRESS)
  channel          SELECT      WEB_FORM, EMAIL, WECHAT, PHONE, FAIR, PORTAL
  language         SELECT      EN, ZH, EN_ZH (= existing LANGUAGE)
  requesterName    TEXT        · requesterEmail EMAILS · requesterPhone PHONES · requesterWechat TEXT
  requesterCompany TEXT        · requesterCountry TEXT · productCategory SELECT (PRODUCT_CATEGORY)
  person           → Person    · company → Company
  assignee         → WorkspaceMember
  firstResponseDueAt / firstRespondedAt / resolvedAt  DATE_TIME
  slaStatus        SELECT      ON_TRACK, AT_RISK, BREACHED, MET
  aiSummary        TEXT        · aiSuggestedReply RICH_TEXT · spamLikelihood NUMBER · spamCheck SELECT (PASSED, UNVERIFIED, FAILED)
  icpScore         NUMBER      · icpTier SELECT (from Feature A)
  consentPrivacy   BOOLEAN     · consentMarketing BOOLEAN
  sourcePage       TEXT        · utm RAW_JSON · intakeId TEXT (unique) · replyToken TEXT
  promotedOpportunity → Opportunity · promotedAt DATE_TIME
  messages ← TicketMessage

TicketMessage (ticketMessages)
  name TEXT · ticket → Ticket · direction SELECT (INBOUND, OUTBOUND, INTERNAL_NOTE)
  body RICH_TEXT · bodyText TEXT · fromEmail TEXT · toEmails TEXT · author → WorkspaceMember
  attachments FILES · sentAt DATE_TIME · sesMessageId TEXT · inReplyTo TEXT
  deliveryStatus SELECT (QUEUED, SENT, DELIVERED, BOUNCED, COMPLAINED, FAILED)

CannedResponse (cannedResponses)
  name · category → ProductStream · body RICH_TEXT · bodyZh RICH_TEXT · isActive BOOLEAN

TicketRoutingRule (ticketRoutingRules)
  name · category → ProductStream · language SELECT · assignee → WorkspaceMember · position NUMBER · isActive BOOLEAN

Person / Company (+)  tickets ← Ticket
```

### 4. UI/UX
- **Navigation:** "Enquiries · 咨询".
- **Views:**
  - `Inbox`: kanban by status (NEW, OPEN, PENDING_CUSTOMER, ON_HOLD, RESOLVED).
  - `My enquiries`: table filtered to assignee = current member. Verify how app-defined views express "is me"; the fallback is a per-member saved view.
  - `Unassigned`, `SLA at risk`, `Spam review`.
- **Ticket record page** (`ticket-record.page-layout.ts`):
  - **Conversation:** a `TicketThread` front component. It shows the AI summary banner, then messages in chronological order with inbound, outbound and internal notes styled differently. The composer supports internal notes in E1 and email replies in E2, with a canned-response picker, an "Insert AI draft" (EN/ZH) action, and a **status-after-send** dropdown.
  - **Details:** a `PromoteToLeadButton` front component, then FIELDS.
  - **Requester:** relation FIELDS and an "Other enquiries from this person/company" VIEW widget.
  - **Activity:** TIMELINE.
- **Person and Company records:** an "Enquiries" VIEW widget.
- **Today dashboard:** a `TicketQueueWidget` (new today, unassigned, SLA at risk, my open).

### 5. Integration points
- **integra-web:** form + Pages Function.
- **Cloudflare:** Turnstile, WAF, Pages secrets.
- **Twenty:** logic functions + front components.
- **SES** (F0.6; reused by portal Phase 8).
- **Claude:** triage and drafts.
- **Feature A:** ICP score on every enquiry.
- **Feature B:** category = stream.
- **Feature C:** CTA prefill.
- **Keycloak:** staff are workspace members.
- **Future:** a portal "Support" page creating `channel=PORTAL` tickets via the same intake route. This needs its own spec amendment; it is not covered by the D5 approval.
- **Privacy:** update `/privacy` to list Cloudflare (Turnstile), AWS SES and Anthropic (AI triage) as processors.

### 6. Phases
| Phase | Scope | Deliverable |
|---|---|---|
| **E1** (M) — **target before Canton Fair** | New form + Pages Function + Turnstile; Ticket / TicketMessage / RoutingRule; intake + triage; staff notification + auto-ack; inbox views; internal notes; Promote to lead. **Fallback if the Phase 0 spike slips:** ship the form + Function in *email-only mode* (structured SES email to info@), then switch the Function to CRM intake later with no frontend change. | No more lost enquiries. Every enquiry is tracked, owned and convertible. |
| **E2** (M–L) | In-CRM composer; SES outbound; inbound reply loop (subdomain MX → SES → Lambda); canned responses; SLA cron; AI reply drafts; retention cron. | Zendesk-like two-way handling without leaving the CRM. |
| **E3** (S–M) | Extra channels (fair capture, portal support, WeChat Official Account messages, which need a separate WeChat API assessment); reports (time to first response, enquiry → opportunity conversion by source and stream). | Unified inbound across channels with metrics. |

### 7. Dependencies
- Cloudflare Turnstile (free) and a WAF rule.
- SES with production access.
- Phase 0 (or the email-only fallback).
- Lambda + `mailparser` for E2 inbound.
- Feature B1 for categories (with a SELECT fallback if B1 isn't ready).

### 8. Effort: **L** overall (E1 **M**).

---

## S — E-signatures: eIDAS QTSP (replaces DocuSign) · decision D2

### 1. Summary
Every signature workflow (AR mandates, AR service agreements, quotes and order forms, and later any other contract) uses **one EU qualified trust service provider (QTSP)** under the eIDAS framework: Regulation (EU) 910/2014 as amended by (EU) 2024/1183. The integration is built **once, in portal core**: portal Phase 5 is redirected from DocuSign to this provider. The CRM requests signatures through that same service, so there is one provider contract, one evidence store and one audit trail.

### 2. Signature levels per document
eIDAS defines three levels:
- **SES** (simple);
- **AdES** (advanced: uniquely linked to and identifies the signer);
- **QES** (qualified: an AdES made with a qualified signature-creation device and a qualified certificate). A QES has the legal effect of a handwritten signature in every member state (Art. 25(2)).

A legal person can apply a **qualified electronic seal**. **Qualified timestamps** and long-term-validation PDFs (PAdES B-LTA) keep signatures verifiable for years, which matters for a mandate that market-surveillance authorities may inspect.

| Document | Integra side | Client side | Notes |
|---|---|---|---|
| **AR mandate** (Reg. 2019/1020 Art. 4: a written mandate) | QES by the authorised Integra signatory **plus** Integra Scientific Ltd's qualified seal | **QES** where the client's signatory can complete the QTSP's remote identity check; otherwise **AdES + qualified timestamp** | Art. 4 requires a written mandate but does not itself prescribe an e-signature level. **Confirm the minimum level with Maltese counsel** before go-live. |
| **AR service agreement** (`legal/Integra_AR_Service_Agreement_v1.2.docx`, Malta law) | Same as the mandate | Same as the mandate | Signed in the same transaction as the mandate. |
| **Quote / order form** (C2 / extra #3) | Qualified seal on the PDF (proves origin and integrity) | AdES (or SES where the counsel review allows) | Signing triggers `stripe.quotes.accept` (Feature C). |
| Renewals / amendments | Qualified seal | Same level as the original contract | — |

### 3. Provider selection (short PoC, then one contract)
- **Hard criteria:**
  1. On the **EU Trusted List** (eidas.ec.europa.eu trust-services browser) for qualified certificates for e-signatures, qualified e-seals and qualified timestamps.
  2. **Remote identity verification that accepts PRC passports**, with a Chinese-language signer UI. Most counterparties are Chinese signatories, so this is the deciding criterion.
  3. REST API + webhooks + sandbox.
  4. EU data residency.
  5. PAdES B-LTA output + downloadable evidence / audit trail.
  6. Per-transaction pricing.
- **Candidates to test** (each publicly states QTSP status; verify each service on the Trusted List):
  - **DocuSign EU Qualified.** DocuSign France SAS is a QTSP supervised by ANSSI and listed on the French trusted list. This answers the founder's question: DocuSign *does* offer eIDAS QES through an EU QTSP, so it remains an option *as a QTSP*. Its EU qualified flow can also use Evrotrust as the identifying QTSP.
  - **Evrotrust** (Bulgaria): QTSP, remote identification marketed for 58 countries, SES/AdES/QES.
  - **Namirial** (Italy): QTSP, eSignAnyWhere workflow API.
  - **InfoCert** (Italy, Tinexta).
  - **Universign** (France): QTSP, SES/AdES/QES.
- **PoC (S, about 1 week, founder + one developer):** take two shortlisted providers end-to-end. Sign a real mandate template with an Integra QES + seal, and a test signatory holding a Chinese passport on a phone in mainland China. Score identity-check pass rate and time, signer UX in Chinese, API/webhook quality, evidence package and price. Then pick one.

### 4. Architecture
```
CRM (Twenty)                         portal core (single signing service)                 QTSP
QuoteBuilder / ArMandate  ──POST /internal/v1/signature-requests──▶  packages/core/src/signing/
  logic fn `signature-request`        (API Gateway route, shared-secret / IAM auth)        ├─ SignatureProvider interface
portal Phase 5 AR flow ─────────────────────────────────────────────▶ ├─ providers/<qtsp>.ts  ──▶ create request (docs, signers, level)
                                                                        └─ signature_requests table
QTSP webhook ──▶ POST /v1/webhooks/signing (verify provider HMAC) ──▶ SignatureQueue (was DocusignQueue)
                                                                   ──▶ consumer: fetch signed PDF + evidence → S3 Documents (versioned)
                                                                        → update ar_mandates / quote status → stripe.quotes.accept (quotes)
                                                                        → CRM: GET poll (until X2) / CrmSyncQueue event (after X2)
```
- **Provider-agnostic interface:**
  ```ts
  interface SignatureProvider {
    createRequest(i: { documents: { name: string; pdfS3Key: string }[];
      signers: { name: string; email: string; phone?: string; role: 'integra'|'client'; order: number;
                 level: 'SES'|'AES'|'QES' }[];
      sealWithIntegraQualifiedSeal: boolean; locale: 'en'|'zh'; expiresAt: Date }): Promise<{ providerRequestId: string }>;
    getStatus(id: string): Promise<SignatureStatus>;
    downloadSigned(id: string): Promise<Uint8Array>;      // PAdES B-LTA
    downloadEvidence(id: string): Promise<Uint8Array>;    // audit trail / validation report
  }
  ```
  If the provider is ever changed, only one adapter file changes.
- **The CRM never holds QTSP credentials.** Its `signature-request` logic function calls the portal's internal route. Status flows back through a 15-minute GET poll until extra #2 (portal↔CRM sync) lands, then through `CrmSyncQueue` events.

### 5. Data model and renames (migrations approved, D5)
```ts
// portal: packages/core/src/db/schema/signing.ts  (new)
signature_requests: id uuid PK, organisation_id FK → organisations NULL (CRM-originated prospects may have no portal org),
  subject_type enum (ar_mandate, service_agreement, quote, other), subject_ref varchar (portal id or "twenty:<object>:<id>"),
  provider varchar, provider_request_id varchar unique, level enum (ses, aes, qes),
  status enum (draft, sent, viewed, signed, declined, expired, failed, cancelled),
  signers jsonb, document_s3_key, signed_document_s3_key, evidence_s3_key, integra_seal_applied boolean,
  created_by varchar, sent_at, completed_at, expires_at, created_at, updated_at
// portal: ar_mandates — drop docusign_envelope_id, add signature_request_id FK → signature_requests (keep signed_at)
```
- **Portal infra** (each still a per-change Article 10 approval; D5 covered the spec amendment and migrations only):
  - secret `DocusignIntegrationKey` → `SigningProviderApiKey` + `SigningWebhookSecret`;
  - queue `DocusignQueue` → `SignatureQueue` with a subscriber;
  - API Gateway route `POST /v1/webhooks/signing` and internal route `POST|GET /internal/v1/signature-requests`;
  - the provider SDK, if one is used, is a new dependency.
- **Spec amendment (approved, D5):** replace DocuSign with "eIDAS QTSP (provider per PoC)" in the locked stack (`CLAUDE.md` / spec), Phase 5 and the AR flow.
- **CRM** (the app has never been synced, so renaming is free):
  - `ArMandate.docusignEnvelopeId` → `signatureRequestId`. Keep the universal ID in `ids.ts` and update the label.
  - Add to ArMandate `signatureStatus` SELECT (NOT_SENT, SENT, SIGNED, DECLINED, EXPIRED), `signatureLevel` SELECT (SES, AES, QES) and `signedAt` DATE_TIME. Signed PDFs and evidence go into the existing `documents` FILES field.
  - `ClientPriceAgreement` and the extra #3 `Quote` get `signatureRequestId` + `signatureStatus`.
  - `MANDATE_STATUS` SENT/SIGNED transitions are driven by `signatureStatus`.

### 6. Phases and effort
| Phase | Scope | When |
|---|---|---|
| **S1** (S) | Counsel confirms signature levels; QTSP PoC with 2 providers; contract signed. | Wave 1 (founder-led, parallel to development) |
| **S2** (M) | `SignatureProvider` + adapter, `signature_requests`, webhook → `SignatureQueue` consumer, renames/migration, AR mandate + service agreement flow in portal Phase 5. | Wave 2, inside portal Phase 5 (replaces the DocuSign work already planned there; no added effort) |
| **S3** (S) | CRM `signature-request` logic function + status poll; quote signing → `stripe.quotes.accept`. | Wave 3, with C3 / extra #3 |

Net effort is roughly neutral against the original DocuSign plan, plus the S1 PoC. Extra #3 stays **L**.

---

## Additional feature recommendations

| # | Feature | Technical approach | Data model | Why it fits | Effort |
|---|---|---|---|---|---|
| 1 | **Fair & business-card capture** | A "Fair capture" STANDALONE_PAGE layout with a `CardCapture` front component (mobile camera upload; works in Twenty's mobile web UI). An httpRoute stores the image, then Claude vision (`claude-opus-5-5`, structured output) extracts nameEn/nameZh, company, title, phone, email, WeChat and products. The result becomes a LeadCandidate (A) or, with "fast accept", a Person with `leadStatus=NEW` on the existing Fair-leads kanban. Batch mode handles a folder of photos after the fair. | `Event` (name, nameZh, startDate, endDate, city, stream →, boothRef); `CaptureItem` (image FILES, extracted RAW_JSON, status, event →, capturedBy →, leadCandidate →); `Person.event` → Event | Canton Fair (15–19 Oct) is the #1 launch channel and is 12 days away. The `fair-leads-kanban` view already anticipates "Requirement 6 (Canton Fair intake)". | **S–M** |
| 2 | **Portal ↔ CRM account sync (Customer 360)** | The portal publishes domain events (org created, subscription changed, DPP published, tier limit at 80%, AR mandate signed, last login) to a new SQS `CrmSyncQueue`. A Lambda consumer upserts the Company in Twenty REST by `portalOrganisationId`. The CRM creates upsell Opportunities on tier-limit events. **Spec amendment and migrations approved (D5).** The new queue and Lambda are still a per-change infra approval. It also carries signature status events (section S) back to the CRM. | Company (+) `portalOrganisationId`, `portalTier`, `subscriptionStatus`, `dppCount`, `publishedDppCount`, `tierLimitUsagePct`, `lastPortalActivityAt`. Decide the source of truth for AR mandates: the portal `ar_mandates` (legal record) vs CRM `ArMandate` (mirror). | It closes the C3 loop, drives renewals and upgrades from real usage, and removes double entry of mandates. | **M–L** |
| 3 | **Quote & mandate generation with eIDAS signing** (D2) | OpportunityLines + ClientPriceAgreement → bilingual quote PDF (rendered in a Lambda, stored in S3 / Twenty FILES). The AR Mandate + Service Agreement (`legal/Integra_AR_Mandate_v1.2.docx`, Schedule 2) are filled from CRM/portal data into PDFs. They are sent through the **section S signing service** (EU QTSP) at the levels section S defines (Integra QES + qualified seal; client QES or AdES + qualified timestamp). The QTSP webhook → `SignatureQueue` → status SIGNED/ACTIVE and `renewalDate = start + 12 months`; for quotes, `stripe.quotes.accept`. It is built once in portal core: portal Phase 5 is redirected from DocuSign to the QTSP, and the unused `DocusignQueue` becomes `SignatureQueue`. The CRM triggers it through the internal route. **Spec amendment and migrations approved (D5).** | `Quote` (opportunity →, number, pdf FILES, validUntil, status, signatureRequestId, signatureStatus); ArMandate (+) `signatureRequestId` (renamed from `docusignEnvelopeId`), `signatureStatus`, `signatureLevel`, `signedAt`; portal `signature_requests` (section S) | It turns C's prices into signed revenue with no retyping; AR mandates are the core recurring product; EU-qualified signatures suit a Malta-based, EU-regulated AR. | **L** |
| 4 | **Renewal & regulatory-deadline automation** | Port `nightly-status.mjs` to cron (Phase 0), then add renewal sequences at 90/60/30 days: a client email in ZH/EN, a task for the account manager, and a Stripe invoice draft for `send_invoice` customers. A `RegulationMilestone` object, kept current with D's regulatory findings, segments Companies by `productCategory` to drive deadline-based nurture emails (e.g. the Battery DPP 18 Feb 2027 countdown) and keeps `KEY_DATES` / DPP Compass in sync. | `RegulationMilestone` (regulation, productCategory MULTI_SELECT, milestoneDate, status PROPOSED/ADOPTED/IN_FORCE, sourceUrl, finding →); `NurtureSend` (company →, person →, milestone →, template, sentAt, sesMessageId) | Renewal is the business model (the CRM "is built to make the renewal clock visible"). Deadline-driven urgency is Integra's strongest sales argument. | **M** |
| 5 | **Training registration & paid enrolment** | Publish `TrainingEvent`s (date, language, price point from C `PER_SEAT`) to `integra-web/data/training.json` via the same git-commit pipeline. A website `/training` calendar uses Stripe Payment Links / Checkout per seat, with registrations via the Feature E Pages Function. Attendees become Persons (leads). The free quarterly "Appointing an EU AR" webinar is the lead magnet named in the strategy docs. This stays out of the portal (MISSION non-goal: training portal). | `TrainingRegistration` (trainingEvent →, person →, company →, seats, paid BOOLEAN, stripeCheckoutSessionId, attended BOOLEAN, certificate FILES); TrainingEvent (+) `language`, `capacity`, `publicListing BOOLEAN`, `pricePoint →` | It turns the Training stream into a self-serve funnel feeding DPP/AR, reusing C and E. | **M** |

---

## Overall roadmap

### Assumptions
- **Two tracks run in parallel:**
  - **Track P (portal team)** stays on the DPP critical path to **18 Feb 2027**: Phase 3 viewer → 4 REST API → 5 AR/admin → 6 Stripe → 8 email → 9 hardening → 10 launch.
  - **Track C (one CRM/web developer)** does everything here except C3 and extras #2/#3, which are joint.
- The portal spec amendments for C3, X2 and X3 (eIDAS) and their database migrations are **approved (D5, 3 Oct 2026)**. They are written into the spec at the start of the wave that needs them. New dependencies, infra and workflow changes still get per-change Article 10 sign-off.

### Dependency graph
```
Phase 0 (app real + logic-fn spike + SES + Anthropic) ─┬─▶ E1 ──▶ E2 ──▶ E3
                                                       ├─▶ X1 Fair capture ──┐
                                                       ├─▶ B1 ─┬─▶ A1 ◀─────┘ ──▶ A2 ──▶ A3
                                                       │       ├─▶ C1 ──▶ C2 ──▶ C3 ◀── portal Phase 6 (Stripe)
                                                       │       ├─▶ D1 ──▶ D2 ──▶ D3        │
                                                       │       └─▶ B2 ──▶ B3               ▼
                                                       └─▶ X4 renewals (cron proven)   X2 Portal↔CRM sync ──▶ X3 CPQ + eIDAS
                                                       S1 QTSP PoC ──▶ S2 signing service (portal Phase 5) ──▶ S3 CRM/quote signing ──▶ X3
 E1 ─▶ A (enquiries scored)     D2 ─▶ C (benchmarks)     C1/C2 ─▶ X5 training enrolment     D2 ─▶ X4 milestones
```

### Sequence

| Window | Track C (CRM / web) | Track P (portal) | Why now |
|---|---|---|---|
| **Wave 0** · 5–14 Oct | Phase 0 (F0.1–F0.7); **E1** (or email-only fallback by 12 Oct); **B1** (streams needed for enquiry categories); **X1** card capture, batch mode at minimum | Continue Phase 3. Write the approved spec amendments (D5) into the spec: Stripe lookup-key contract (C3), DocuSign → eIDAS QTSP (Phase 5 / X3), portal↔CRM sync (X2) | Canton Fair traffic and contacts start 15 Oct; mailto is leaking enquiries today |
| *15–19 Oct* | *Fair support: capture, triage* | — | — |
| **Wave 1** · 20 Oct – 30 Nov | **A1** (Apify-enrich + score fair exhibitor lists + cards); **E2**; **B2**; **C1** (prices on the website, D3 display rule); X4 renewal sequences | Phases 3–4. **S1** QTSP PoC + counsel review (founder-led) | Convert fair leads while warm; publishing prices removes sales friction (the competitor analysis recommends it) |
| **Wave 2** · Dec – mid-Jan | **C2**; **D1 → D2**; **A2**; X5 training enrolment (optional) | Phase 5 (AR + admin, with **S2** eIDAS signing service in place of DocuSign); start Phase 6 against the C3 contract | Deal desk before first paid contracts; research once the catalogue is stable |
| **Wave 3** · mid-Jan – Mar 2027 | **C3** CRM side (Stripe catalogue sync, agreements → Coupons/Quotes/Invoices); X2 sync consumer; **S3** CRM signature requests + quote signing; D3; E3; A3 | Phase 6 Stripe billing, Phase 8 email (reuse F0.6 SES), X2 event producer, X3 quote/mandate generation on the eIDAS signing service | Billing must work for customers onboarding against the Feb 2027 deadline |

### What can run in parallel
- After Phase 0, **E** and **B** are independent; with a second developer, run them side by side.
- After B1, **A1**, **C1** and **D1** are independent of each other.
- All of Track C is independent of Track P **except** C3, X2 and X3.

### ROI ranking (Integra's business: DPP/AR for Chinese manufacturers)
1. **E1:** cheapest fix for a live revenue leak; every inbound lead is captured and owned.
2. **X1 + A1:** monetises the Canton Fair, the single most important channel this quarter.
3. **C1 + C3:** published, consistent pricing plus the ability to actually charge (C3 shares work with Phase 6).
4. **B:** low cost, and it unblocks C, D and A.
5. **X4 renewals:** protects recurring AR revenue (mandates are annual).
6. **E2, C2, X3:** sales efficiency.
7. **D:** strategic insight with lower urgency; a competitor analysis already exists.
8. **A3:** depends on paid LinkedIn ads and data subscriptions; validate A1/A2 conversion first.

### Effort summary
| Item | Effort | Item | Effort |
|---|---|---|---|
| Phase 0 | M | D (D1–D3) | M |
| A (A1–A3) | L | E (E1–E3) | L (E1 M) |
| B (B1–B3) | M | X1 Fair capture | S–M |
| C (C1–C2) | L | X2 Portal↔CRM sync | M–L |
| C incl. C3 + portal billing | XL | X3 CPQ + eIDAS signing | L |
| S eIDAS signing (S1 PoC S; S2 replaces Phase 5 DocuSign work; S3 S) | S + M | X4 Renewals + milestones | M |
| | | X5 Training enrolment | M |

### Risks and decisions needed (owner: founder / business-dev-chief)
1. **Twenty logic functions on v2.41** are unproven here. Phase 0 spike, with the sidecar-Lambda fallback (F0.3b).
2. **LinkedIn data via Apify (decided, D1).** Residual risk: Apify's terms (§5.8) leave the legality of the data with Integra, and LinkedIn has sued third-party data APIs before. This is mitigated by company-page, no-cookie actors only, no person data, provenance on every record, a signed DPA, a kill switch and a spend cap (Feature A guardrails a–g). Alexandria is limited access with unpublished pricing, so it stays optional.
3. **Public pricing display (decided, D3).** One open question remains: do DPP tiers count as "with optional extras" (→ "from €950")? The seed assumes no. Still to do: VAT wording, and confirming Alipay works for recurring billing.
4. **eIDAS signing (decided, D2).** Still needed:
   - Maltese counsel confirms the minimum signature level for AR mandates.
   - The S1 PoC proves that Chinese signatories can pass remote identity verification. If they can't, client-side AdES + qualified timestamp is the fallback.
5. **Portal spec amendments (approved, D5)** for C3, X2 and X3/eIDAS, plus their migrations. New dependencies (stripe, signing SDK), `sst.config.ts`/infra and workflow changes still need per-change Article 10 sign-off.
6. **Managed Agents is beta:** pin the beta header; D's ingest code tolerates API changes; budget cap on every session.
7. **GDPR:** update the privacy policy to list Cloudflare, SES, Anthropic, **Apify** and the **QTSP** as processors; Art. 14 notices for indirectly sourced leads; retention crons.
8. **Docs drift:** fix the Clerk → Keycloak and DocuSign → eIDAS references in portal `CLAUDE.md`, `ARCHITECTURE.md`, spec and `sst-env.d.ts`. The DocuSign change is covered by D5; the Clerk wording still needs approval per the constitution.
9. **Cloudflare project name (resolved, D6):** the live project is `integra-scientific`. Align `package.json`'s `deploy` script and `wrangler.jsonc` `name` with it in E1.
