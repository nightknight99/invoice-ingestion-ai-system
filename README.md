# Production AI System — Invoice Ingestion at Scale

**OCR + LLM-driven structured extraction + a custom DIAN UBL 2.1 parser, orchestrated on n8n and persisted to Postgres.** Ran in production throughout 2025 on a self-hosted instance, processing real supplier-invoice traffic for a Colombian construction company (HCG Construcciones) with no daily intervention.

> This repository contains a **sanitized export** of the core ingestion workflow (`workflow.json`) plus the case study below. Credential IDs, folder IDs, instance and workflow IDs have all been redacted. Orchestration logic and the XML parser remain intact and reviewable.

This is the ingestion stage of a broader operational AI system I architect and run. The OCR sub-workflow (`AgenteFacturasRecibidas_V3.01`, referenced here as `Agente_Facturas1`), the monthly close jobs, and the human-review surface on top of Supabase are separate workflows in the same system, not in this export.

## Why this is an AI system, not just an automation

This is the question a serious reviewer asks before reading further. Three components make this an AI system rather than a glorified Zapier flow:

1. **LLM-driven structured extraction (OCR path).** Invoices arrive in two formats: structured XML (the DIAN-mandated electronic invoice) and unstructured PDF. The PDF path is handed to a sibling workflow that runs LlamaCloud OCR with a target JSON schema I defined; an LLM reads the PDF and emits the same fields the XML parser produces. That is the canonical "ML for unstructured-to-structured" use case, in production, against real Colombian invoices.
2. **Schema-driven unification.** The system has one target schema (the Supabase tables) and two extractors writing into it — a deterministic parser for XML, a probabilistic LLM extractor for PDF. Both flow into the same downstream graph (deduplication, supplier upsert, line-item flattening, archival). That convergence is the architecture pattern of every serious structured-extraction system in 2026.
3. **AI-augmented construction.** The dense code in this workflow — most clearly the DIAN UBL parser, a pure-JavaScript regex implementation — was built using Claude as a code generator from input/output specs I wrote. I describe this openly in the next section because it is the actual working method that produced the result, and it is how I will build the next system.

## How this was built — the honest version

I am a Construction Project Director by trade, not a software engineer. The way I work with software is:

- **I design the system.** Data flow, dispatchers, switch points, dedup strategy, Drive folder layout, Supabase schemas, the trigger / catch-up pair on the Gmail side, how the OCR path rejoins the XML path. Those decisions are mine, made against the constraints of the domain (DIAN regulation, Colombian tax regimes, the accountant's downstream workflow).
- **I choose the stack and operate it.** I picked n8n (self-hosted on my own VPS, not SaaS) over Zapier and over writing this as a script. I picked Supabase over hosted Postgres for the storage layer with RLS on top. I picked LlamaCloud for OCR + schema extraction over a generic vision model. I ran this system through twelve months of real traffic and fixed it when it broke.
- **I specify the code; an LLM writes it.** The dense logic — the DIAN XML parser is the clearest example — was implemented using Claude as a code generator. I supplied input fixtures, the exact output shape I needed, and the edge cases (nested CDATA wrappers, credit-note vs invoice tag switches, missing-field tolerance). Claude wrote the JavaScript. I can read and reason about every node, and I have iterated the system live when something broke; I do not pretend I would have written every line of the regex parser from a blank file.

I describe this openly because it is the working method that produced the result, and because it predicts how I would build the next system: by being precise about the problem, picking the right pieces, integrating them well, and using LLMs to compress the implementation step that is no longer the bottleneck.

## The problem

Every Colombian supplier sends invoices by email. Most send the DIAN-mandated XML (structured but verbose UBL 2.1) plus a "human" PDF, sometimes inside a ZIP. The accountant used to download each attachment, type the relevant fields into Excel, and file the originals by year and type. Each invoice took roughly five to ten minutes of attention; the monthly close was bottlenecked by this work.

This pipeline removes that work for the XML path entirely and structures the PDF-only path through OCR + LLM extraction.

## What it does, end to end

1. **Watch the `FACTURAS` Gmail label** — both via a real-time trigger and a manual catch-up runner that processes anything previously missed.
2. **Mark messages as read** and **download all attachments**.
3. **Detect the attachment type** (XML, PDF, or ZIP). ZIPs are decompressed and their contents re-enter type detection. Deduplication by `fileName` runs at every entry point.
4. **For XML attachments** (deterministic parser):
   - Extract the inner `Invoice` payload (DIAN nests the actual document as CDATA inside `cbc:Description` of an outer `AttachedDocument`; the parser unwraps it).
   - Parse all header fields: invoice number, CUFE, issue and due dates, supplier NIT and legal name, totals, VAT, consumption tax, document type (invoice / credit note / debit note), tax regime, operation type (domestic / international), QR code.
   - Parse every line item: description, code, quantity, unit price, line discount (rate and total), per-line VAT and consumption tax, totals with and without taxes.
   - **Upsert the supplier** in `proveedores`.
   - **Insert the invoice header** into `facturas`, with accounting fields (`cuenta_contable`, `tipo_gasto`, `centro_costo_id`, `estado_pago`) left as `pendiente` so the accountant completes them downstream.
   - **Insert each line item** into `compras_detalle`, linked to the invoice by supplier NIT + invoice number.
5. **For PDF-only attachments** (LLM extractor path): the workflow hands the file to a sibling sub-workflow that runs LlamaCloud OCR with the same target schema. Once those fields exist, the same Supabase write logic runs — both paths converge.
6. **Archive originals on Google Drive**, in folders organized by year and document type. Files are renamed to a canonical key — `HCG_REC_{issue_date}_{supplier_nit}_{invoice_number}` — so a human can find any invoice by glance and so duplicate uploads collide on filename.
7. **Write the Drive URL back to the `facturas` row** and flip `registro_items` to `registrado`, closing the loop.

## Design decisions worth naming

- **Pure-JavaScript XML parser, no library.** The DIAN format wraps the real `Invoice` element inside `cbc:Description` as CDATA in an outer `AttachedDocument` envelope. Most generic UBL parsers don't unwrap that. The hosted n8n task runner doesn't allow `npm install` either. So the parser is a pure-JS code node with regex helpers, tolerant by design — every field returns `null` when missing, so a malformed invoice never crashes the pipeline; it lands in Supabase with partial data and is flagged for manual review.
- **Two extractors, one target schema.** XML path is deterministic (regex parser); PDF path is probabilistic (LLM + JSON schema). They write the same shape into the same tables. Downstream code never knows which extractor produced a row.
- **Both real-time and catch-up triggers.** A pure trigger is fragile: if the n8n instance is down or the workflow is paused, messages slip through. A catch-up runner on the unread filter ensures eventual consistency. Both feed the same downstream graph.
- **Filename as the canonical key.** Renaming Drive files to `HCG_REC_{date}_{nit}_{invoice_number}` means the human-readable archive and the Supabase row are joined by a string a human can also recognize, and Drive itself becomes a backstop deduplication layer.
- **`pendiente` as a first-class state.** The accountant's downstream classification work (cost center, account, payment status) is modeled by leaving those columns as `pendiente` rather than NULL. This makes the "still needs review" query a literal `WHERE estado_pago = 'pendiente'` instead of three OR-NULLs.

## AI / Data stack

- **n8n** — self-hosted on a VPS I administer. Workflow orchestration, code nodes, triggers, scheduled jobs.
- **LlamaCloud** — OCR + JSON-schema extraction, the LLM path for unstructured PDFs.
- **Supabase / PostgreSQL** — three tables: `proveedores`, `facturas`, `compras_detalle`. Target schema for both extractors.
- **Google Drive API** — original-document archival (OAuth).
- **Gmail API** — trigger plus catch-up reader (OAuth).
- **JavaScript (Node 18+)** — the DIAN UBL parser, file dispatcher, detail flattener. Pure-JS regex; no external dependencies. Implemented from specs using Claude.
- **Claude (Anthropic)** — used as a code generator for dense logic during construction, via direct conversation and (more recently) via n8n's MCP integration for in-tool iteration.

## Operational learnings (from running this in production)

- The flaky layer is rarely the parser. It is the credential layer — OAuth tokens that expire silently on the Drive or Gmail side and produce surprise empty results. An explicit health-check workflow before scaling is worth more than another round of parser polish.
- Deduplication by filename alone is necessary but not sufficient. CUFE (the DIAN-issued unique identifier) is the real key for the XML path; the workflow uses filename for early discard and CUFE-aware lookups downstream.
- Suppliers send malformed XML more often than expected. Building tolerance into the parser (every getter returns `null` instead of throwing) eliminated a category of pager fires.
- Once the system was stable, the bottleneck moved from ingestion to the human classification step (`cuenta_contable`, `centro_costo_id`). The accountant still had to assign those by hand. The next workflow on the roadmap was a suggestion engine for those fields using historical patterns; it was scoped but not built before the company shut down.

## Limitations of what's in this repo

- **One workflow of several.** The full operational AI system includes the LlamaCloud OCR sub-workflow, the monthly close jobs, and the Supabase-backed review surface. Not exported here.
- **The Supabase schema is implied, not included.** Column names are visible inside each `n8n-nodes-base.supabase` node; a future commit will add `schema.sql`.
- **No production payloads.** Real invoice traffic includes supplier NITs and contract metadata that aren't mine to publish.

## Files

- [`workflow.json`](./workflow.json) — full sanitized n8n workflow export. Drop into any n8n instance after wiring credentials (Supabase, Gmail OAuth, Google Drive OAuth) and replacing the redacted folder IDs.

---

**Author.** Felipe Castro — Civil Engineer (Universidad de los Andes), Construction Project Director by trade, building operational AI systems where domain knowledge and LLMs-as-code-generators multiply each other.

- LinkedIn: [linkedin.com/in/felipecastro99](https://linkedin.com/in/felipecastro99)
- Contact: <castrof500@gmail.com>
