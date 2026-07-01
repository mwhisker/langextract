# Feasibility: Document Parsing / IDP inside Salesforce (langextract-style)

## Context

Customers routinely need to push documents (chiefly PDFs — invoices, POs, lab
reports) to an external system for IDP/parsing. The ask is to assess whether we
can do this **natively inside a Salesforce org**, either by:

- **(A)** letting customers plug in their **own LLM API keys** and using a
  langextract-style grounded-extraction approach on-platform, or
- **(B)** an **LLM-free deterministic OCR** pipeline that runs similarly in-org.

This plan delivers a **written feasibility analysis** comparing three compute
boundaries (pure Apex / Heroku sidecar / native SF Document AI) and both
approaches (BYO-LLM vs deterministic OCR), **plus a POC scaffold** for the
recommended path.

## Key findings from research (drive the whole analysis)

1. **langextract is pure Python, text-only, Apache-2.0.** It does **no** PDF
   parsing or OCR — it assumes you already have text (`Document.text` only; see
   `langextract/core/data.py`, `langextract/extraction.py`). Its real IP is
   *grounded structured extraction*: few-shot prompt construction
   (`prompting.py`), JSON/YAML schema-constrained output
   (`providers/schemas/*`, `core/format_handler.py`), token chunking
   (`chunking.py`), and char-offset alignment back to source
   (`resolver.py` WordAligner). ~11k lines; cannot run on Apex.
2. **Providers are thin wrappers over REST.** Gemini/OpenAI use vendor SDKs;
   Ollama uses raw HTTP (`providers/ollama.py`). The underlying Gemini/OpenAI/
   Anthropic REST endpoints are all reproducible from Apex `HttpRequest`.
3. **Apex is a hostile runtime for documents:** 6 MB sync / **12 MB async**
   heap, **120 s** max callout, 10 s sync / 60 s async CPU, 100 callouts/txn,
   **no native PDF/OCR/ML libraries**. A base64'd multi-MB invoice can exceed
   the heap on its own.
4. **The OCR problem can be offloaded to a multimodal model.** Claude, Gemini,
   and GPT REST APIs now accept **PDFs directly** (Claude: 32 MB inline / 100
   pages, or 500 MB via Files API). So the BYO-LLM path needs *no explicit OCR
   step at all* — the model does layout+OCR+extraction in one call.
5. **Salesforce already ships native document AI:** Data Cloud **Document AI**
   and Industries **Intelligent Document Reader** (OCR-based, invoice-aware,
   real-time + batch). This is the closest thing to an LLM-free deterministic
   in-platform option and should be positioned as "buy vs build."

## The analysis (what the document will conclude)

### Compute-boundary comparison

| Option | Can run real langextract? | PDF/OCR handling | Heap/CPU ceiling | Effort | Verdict |
|---|---|---|---|---|---|
| **Pure Apex (in-org)** | No (reimplement subset in Apex) | Offload to multimodal LLM via callout; no OCR in Apex | Bound by 12 MB async heap → practical inline PDF ~4–6 MB | Medium | **Feasible for BYO-LLM**; not feasible for in-Apex OCR |
| **Heroku sidecar** | Yes (run langextract unchanged in Python) | Real OCR/text-extract libs (pdfplumber, Tesseract) or LLM | Full Linux runtime, no governor limits | Medium-High | **Most capable**; still "Salesforce ecosystem"; adds an operated service |
| **Native SF Document AI** | N/A (product) | Salesforce OCR + ML/LLM, invoice models built-in | Handled by platform | Low (config) | **Best when it fits**; less control, licensing/edition-gated |

### Approach comparison (both weighted equally)

| Dimension | (A) BYO-LLM key (ideally **multimodal**, PDF→model) | (B) LLM-free deterministic OCR |
|---|---|---|
| PDF handling | Model does OCR+layout+extraction in one call | Needs a real OCR engine (Textract / Google Document AI / Azure DI / Tesseract-on-Heroku) — **cannot run in pure Apex** |
| Accuracy on messy invoices | High, tolerant of layout variance; few-shot tunable | High on structured forms; brittle on unseen layouts; needs templates/rules |
| Determinism / auditability | Lower (probabilistic); mitigate w/ schema constraints + grounding | High, repeatable, easier to govern |
| Cost | Per-token/per-page model cost; customer's own key | Per-page OCR cost; no LLM |
| Data governance | Customer controls key/endpoint; data leaves org to their chosen vendor | Same (OCR vendor) unless Tesseract self-hosted |
| Where it can live | Pure Apex ✔ / Heroku ✔ / Native ✔ | Heroku ✔ / Native ✔ / **Pure Apex �’** |

### Recommendation (to scaffold)

**Primary: Pure-Apex orchestrator + BYO-key multimodal LLM**, because it is the
only path that is genuinely "native, customer-plugs-in-their-own-key, PDFs
in" without operating extra infrastructure, and it reuses langextract's *ideas*
(few-shot + JSON-schema + grounding) rather than its Python. Design it behind a
provider interface so a **deterministic OCR implementation** (Textract/Document
AI) drops into the same pipeline, embodying both approaches. Position **Heroku
+ real langextract** and **native Document AI** as documented escalation paths
(large files, strict determinism, low-code).

## POC scaffold to build (recommended path)

Create an SFDX project under `salesforce/` in this repo (reference scaffold, not
auto-deployed). Pattern-based; representative files:

- `docs/salesforce/feasibility_analysis.md` — the full written analysis above,
  expanded with diagrams, limit citations, and the escalation matrix.
- `salesforce/README.md` — deploy + configure BYO key instructions.
- `salesforce/sfdx-project.json`.
- **External/Named Credential metadata** (`externalCredentials/`,
  `namedCredentials/`) — BYO API key stored in an **External Credential**
  (Custom auth), injected as the `x-api-key`/`Authorization` header via a Named
  Credential so the key never appears in Apex. Per-org or per-user principal.
- **Apex** (`force-app/main/default/classes/`):
  - `IDocumentParser.cls` — interface (`parse(ContentVersion) : ExtractionResult`).
    Two impls demonstrate both approaches:
    - `LlmDocumentParser.cls` (recommended default) — sends PDF straight to a
      multimodal model.
    - `OcrDocumentParser.cls` — stub calling AWS Textract / Google Document AI
      (the LLM-free path).
  - `DocumentParseQueueable.cls` — async worker (12 MB heap / 60 s CPU): reads
    `ContentVersion.VersionData`, base64-encodes, guards file-size ceiling,
    invokes the selected parser, persists results.
  - `ExtractionPromptBuilder.cls` — langextract-style prompt: task description +
    verbatim few-shot examples + JSON schema (mirrors `prompting.py` /
    `format_handler.py` concepts). Examples/schema stored as a static resource.
  - `ExtractionResultParser.cls` — parse model JSON → `Document_Extraction__c`
    + child line-item records; capture confidence + source text for grounding.
  - Test classes using `HttpCalloutMock` (`*Test.cls`) — no live keys needed.
  - `DocumentParseService.cls` — entry point (invocable for Flow + Apex API).
- **Custom object** (`objects/Document_Extraction__c/`) — header fields
  (vendor, invoice #, total, dates) + `Document_Line_Item__c` child, plus raw
  JSON + source-offset fields to preserve langextract-style grounding.
- **Invocable action** so the whole thing is callable from a Flow triggered on
  file upload (`ContentDocumentLink`).

### Honest limitations to document in the analysis

- **Heap ceiling:** inline base64 PDFs are practically capped ~4–6 MB even in
  async; larger files require the model's **Files API** (upload-by-reference) or
  the **Heroku sidecar**. Call this out explicitly.
- **120 s callout** vs slow large-doc inference → keep docs small or go async/
  sidecar with polling.
- Pure-Apex cannot do OCR itself; the deterministic-OCR approach *must* use an
  external OCR API or Heroku.

## Verification

- **Apex tests:** run the `*Test.cls` classes with `HttpCalloutMock` returning a
  canned invoice-extraction JSON; assert `Document_Extraction__c` + line items
  are created and grounding fields populated. (`sf apex run test` / Developer
  Console.) No real API key required.
- **Prompt fidelity:** unit-assert `ExtractionPromptBuilder` output contains the
  few-shot examples + JSON schema block (langextract structure).
- **Manual end-to-end (optional, needs a key):** configure the External
  Credential with a real Claude/Gemini key, upload a sample invoice PDF, run the
  invocable action, confirm extracted fields in the record.
- **Doc review:** the feasibility markdown renders with the two comparison
  tables, the escalation matrix, and cited platform limits.
