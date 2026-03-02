# Speaker Guide: Multimodal Customer Service Analytics Lab

> **For**: Lab facilitator (you)
> **Audience**: Postcode Loterij inspiration day participants
> **Duration**: ~2 hours
> **Format**: Guided hands-on lab with live narration

---

## Pre-Lab Checklist

- [ ] Verify `setup.sql` runs cleanly on a fresh account (takes ~2-3 min)
- [ ] Confirm `CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` is set (setup.sql does this)
- [ ] Test notebook import in Snowsight — select DB `MULTIMODAL_CUSTOMER_SERVICE`, schema `DATA`
- [ ] Ensure participants have a MEDIUM (or larger) warehouse — audio processing is compute-heavy
- [ ] Have the Streamlit app code (`streamlit_app.py`) ready to paste
- [ ] Pre-load the notebook yourself so you can screen-share results if someone gets stuck

---

## Timing Overview

| Block | Module | Duration | Cumulative |
|-------|--------|----------|------------|
| 1 | Welcome + Setup | 10 min | 0:10 |
| 2 | Audio Pipeline | 30 min | 0:40 |
| 3 | Document Processing | 15 min | 0:55 |
| — | *Break* | *5 min* | *1:00* |
| 4 | Chat & Ticket Validation | 25 min | 1:25 |
| 5 | Streamlit Dashboard | 25 min | 1:50 |
| 6 | Explore + Wrap-up | 10 min | 2:00 |

> **Buffer**: The timing is tight. If audio processing runs long (~3 min for 5 files), use that wait time to explain concepts. If you're running behind, Module 5 (Explore) can be shortened or skipped.

---

## Module 0: Welcome + Setup (10 min)

### What to say

- "Today we're building a complete AI-powered customer service analytics system — processing audio calls, PDFs, chat logs, and support tickets — all inside Snowflake, all with SQL."
- "No Python needed. No external APIs. No data leaves Snowflake."
- "The data is synthetic but the functions and patterns are production-ready."

### Key talking points

- **Why Cortex AI?** It's built into Snowflake. Your data stays in place — no copying to external services, no API keys, no extra infrastructure. Governance and access control come for free.
- **Cross-region setting**: `CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` lets you use AI functions even if your Snowflake account is in a region that doesn't natively host the models. The data is processed in the nearest available region. This is in setup.sql already.
- **COPY FILES**: We copy from an external S3 stage to an internal stage because `AI_TRANSCRIBE` and `AI_PARSE_DOCUMENT` need internal stages. External stages won't work for these functions.

### Pacing

- Have everyone open Snowsight and create a SQL worksheet
- Paste and run `setup.sql` — it takes ~2-3 minutes
- While it runs, walk through what it creates (stages, tables, file formats)
- Verification queries at the bottom should return: 5 audio files, 20 chats, 20 tickets

### Watch out for

- Participants without `ACCOUNTADMIN` or equivalent role can't run `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION` — handle this before the lab or have an admin pre-set it
- Some accounts may need `USAGE` grants on the warehouse

---

## Module 1: Audio Processing Pipeline (30 min)

### What to say

- "We're going to take 5 audio recordings of customer service calls and extract everything useful from them — transcription, translation, sentiment, category, and a summary — using 6 AI functions chained together."

### Two paths — how to guide participants

The notebook has **two approaches** to the audio pipeline:
1. **Complex Pipeline** (one big query) — chains all 6 functions in CTEs
2. **Step-by-Step** (individual cells) — one function per cell with temp tables

**Recommended flow for the lab:**
- "First, let's run the complex pipeline to see the end result. It takes about 2-3 minutes."
- While it runs, walk through each AI function conceptually
- After results appear, say: "Now let's break that apart and understand each piece."
- Walk through the step-by-step cells
- The "Combine All Results" cell starts with a `TRUNCATE` so running both paths is safe — no duplicate data

### Function-by-function talking points

**AI_TRANSCRIBE**
- Input: audio file reference via `TO_FILE()` pointing to an internal stage
- `timestamp_granularity: 'speaker'` gives speaker diarization (who said what)
- Output is VARIANT (JSON) with `audio_duration` and `segments` array
- Each segment has `speaker`, `text`, `start`, `end`
- "This is the foundation — everything else builds on having text"

**FLATTEN + ARRAY_AGG**
- Not an AI function — standard Snowflake SQL
- Segments come as an array; we need a single text string for downstream functions
- `FLATTEN` explodes the array, `ARRAY_AGG` with `WITHIN GROUP (ORDER BY)` reassembles it
- "This is a common pattern when working with semi-structured data in Snowflake"

**AI_TRANSLATE**
- Auto-detects source language when you pass `''` as the source
- "The sample data includes calls in Dutch, Spanish, and English — AI_TRANSLATE handles all of them"
- Useful for global teams: analyze everything in one language regardless of where the call originated

**AI_SENTIMENT**
- Returns a JSON object with scored categories: `positive`, `negative`, `neutral`, `mixed`
- We extract the top category: `:categories[0]:sentiment`
- "Notice this returns structured JSON, not just a label — you get confidence scores too"

**AI_CLASSIFY**
- Zero-shot classification — no model training needed
- You define categories with labels AND descriptions (descriptions improve accuracy significantly)
- The 5 categories used here are common customer service buckets but they're fully customizable
- "The `task_description` parameter gives the model context about what it's classifying"
- Uses `OBJECT_CONSTRUCT` for each category — point this out as a Snowflake pattern

**AI_COMPLETE**
- The most flexible function — any LLM prompt
- We use `claude-sonnet-4-5` here; other options include `llama3.1-70b`, `mistral-large2`
- "The prompt engineering matters. We say '50 words' to keep summaries concise. You could ask for bullet points, key issues, action items — whatever you need."

### Timing tip

- The complex pipeline cell takes ~2-3 min. Use that time to explain the CTE structure on screen.
- Step-by-step cells: `AI_TRANSCRIBE` is the slowest (~30-60 sec per file). Other functions are fast (<5 sec each).

---

## Module 2: Document Processing (15 min)

### What to say

- "Now let's process a different data type — PDF documents. Same idea: SQL in, structured data out."

### Key talking points

**AI_PARSE_DOCUMENT**
- Takes a file reference and returns structured content
- `mode: 'LAYOUT'` preserves document structure (headers, paragraphs, tables)
- `page_split: TRUE` returns content page-by-page — useful for large documents
- "Think invoices, contracts, policy documents — anything your business stores as PDFs"
- Uses `DIRECTORY()` table function to list all files in a stage — another useful Snowflake pattern

**Why internal stages?**
- `AI_PARSE_DOCUMENT` (like `AI_TRANSCRIBE`) requires files on internal stages
- That's why `setup.sql` copies from external S3 to internal stages via `COPY FILES`

### Pacing

- This is the shortest module — one main cell to run
- The cell creates a table `parsed_documents_raw` with the parsed output
- Have participants explore the `parsed_result` column — it's a VARIANT with rich structure
- If time permits, show how you could combine this with `AI_EXTRACT` or `AI_COMPLETE` to answer questions about the documents

---

## Break (5 min)

Good place for a break. Modules 3 and 4 are more complex.

---

## Module 3: Chat & Ticket Validation (25 min)

### What to say

- "So far we've processed raw data. Now let's use AI to validate human work — checking if customer service agents classified chats correctly."

### Part 3: Chat Validation — talking points

**The business problem**
- Agents manually tag chats with category and sentiment during or after the conversation
- Humans make mistakes, especially under time pressure
- AI can re-classify at scale and flag discrepancies

**What the query does**
1. Flattens chat messages (same `FLATTEN` + `ARRAY_AGG` pattern as audio)
2. Runs `AI_CLASSIFY` with different categories than Module 1 — show how flexible it is
3. Runs `AI_SENTIMENT` on chat text
4. Runs `AI_EXTRACT` — **new function** — pulls structured fields from unstructured text
5. Compares AI results vs. agent labels and flags mismatches

**AI_EXTRACT deep dive**
- Define a schema with field names and descriptions
- The model extracts matching values from the text
- Output is structured JSON you can query with Snowflake's semi-structured syntax
- "This is like having an army of interns reading every chat and filling out a form — except it takes seconds"
- The `responseFormat` uses `OBJECT_CONSTRUCT` — each key is a field name, each value is a description/question telling the model what to extract

**Flagging logic**
- Category mismatch: agent said "Billing" but AI says "Technical Support"
- Sentiment mismatch: agent said "Positive" but AI detects "negative"
- Mixed sentiment: always flagged because it needs human review
- "This is a QA system, not a replacement for humans. The flags tell you where to look."

### Part 4: Ticket-Chat Alignment — talking points

**The business problem**
- Support tickets are often created after chats, sometimes by different people
- Information can drift: the ticket might describe a different issue than what was actually discussed
- Misalignment = bad data = bad decisions

**How it works**
- Joins tickets to their linked chats
- Uses `AI_COMPLETE` with a detailed prompt for semantic comparison
- The prompt asks the model to return structured JSON with alignment status, confidence, and reasoning
- `temperature: 0.1` makes output more deterministic (less creative, more consistent)
- `REGEXP_REPLACE` strips markdown formatting from the LLM response before parsing

**Point out**
- This is a more advanced `AI_COMPLETE` usage — not just summarization but structured reasoning
- The prompt is long and specific — prompt engineering matters for complex tasks
- We parse the JSON response and create actionable flags (category mismatch, product mismatch, severity)

### Timing tip

- The chat validation cell takes ~1-2 min (20 chats x 3 AI functions each)
- The alignment cell also takes ~1-2 min
- Use wait times to walk through the SQL on screen

---

## Module 4: Streamlit Dashboard (25 min)

### What to say

- "We've built all the data. Now let's visualize it with a Streamlit app — inside Snowflake, no deployment needed."

### Setup steps to guide

1. **Projects > Streamlit > + Streamlit App**
2. Name it `Customer Service Analytics`
3. Set DB: `MULTIMODAL_CUSTOMER_SERVICE`, Schema: `DATA`
4. Select their warehouse
5. **Replace all default code** with contents of `streamlit_app.py`
6. Click Run

### Key talking points

**Streamlit in Snowflake (SiS)**
- Runs inside Snowsight — no external hosting, no deployment pipeline
- Data never leaves Snowflake — queries run via `get_active_session()`
- "Notice there's no connection string, no credentials — the session is already authenticated"

**Important SiS differences from open-source Streamlit**
- `st.set_page_config()` is **not supported** — that's why we don't have it
- `get_active_session()` replaces `snowflake.connector` or `snowpark` connection setup
- Some Streamlit features behave differently — check docs if customizing

**Dashboard walkthrough**
- **Overview tab**: KPI metrics at the top, sentiment and category bar charts below
- **Call Analytics tab**: Expandable cards for each processed call with AI summary
- **Chat Validation tab**: Filterable table — toggle "flagged only" checkbox
- **Alignment Issues tab**: Severity-sorted list (red/yellow/green) with expandable detail

**Code patterns to highlight**
- `session.sql("...").collect()` for single values (KPIs)
- `session.sql("...").to_pandas()` for dataframes (charts, tables)
- `st.expander` for progressive disclosure — keeps the UI clean
- `st.column_config` for custom column rendering (checkboxes, lists)

### Watch out for

- If participants haven't run all notebook cells, some tables won't exist and the dashboard will error
- The dashboard queries all three result tables: `transcription_results`, `chat_validation_results`, `ticket_chat_alignment`

---

## Module 5: Explore + Wrap-up (10 min)

### If you have time

Suggest participants try:
- Changing `AI_CLASSIFY` categories to something relevant to Postcode Loterij
- Swapping `claude-sonnet-4-5` for `llama3.1-70b` and comparing summary quality
- Looking at the raw `segments` column to see speaker diarization detail
- Examining `extracted_features` JSON from chat validation

### Wrap-up talking points

- "You built a complete multimodal analytics system in under 2 hours — audio, PDFs, chat, tickets — all in SQL"
- "Every function we used today is production-ready and runs on your existing Snowflake infrastructure"
- "The patterns here scale: swap sample data for your real customer service data and you have an operational system"

### Next steps to mention

- **Snowflake Tasks**: Schedule the pipeline to run automatically on new data
- **Cortex Agents**: Build a natural language interface over these results
- **Custom categories**: Tailor `AI_CLASSIFY` to your specific business taxonomy
- Connect to real data sources via stages or Snowpipe

---

## Troubleshooting Quick Reference

| Issue | Fix |
|-------|-----|
| `AI_TRANSCRIBE does not exist` | Run `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION'` — needs ACCOUNTADMIN |
| Audio processing is very slow | Use MEDIUM or larger warehouse. ~30-60 sec per file is normal. |
| Dashboard shows errors | Ensure all notebook cells have been run first (all 3 result tables must exist) |
| `st.set_page_config` error | This was removed — if someone has an old copy, delete lines with `set_page_config` |
| Permission errors | Need USAGE on database, schema, and warehouse; SELECT on tables |
| Duplicate rows in transcription_results | The step-by-step "Combine Results" cell runs TRUNCATE first, so just re-run it |
| LLM returns markdown-wrapped JSON | The alignment query has `REGEXP_REPLACE` to strip ` ```json ``` ` wrappers |

---

## Architecture Recap (for Q&A)

```
                        Snowflake Account
                    ┌──────────────────────────┐
  S3 Sample Data    │  Internal Stages          │
  ──────────────►   │  @CUSTOMER_CALLS (.mp3)   │
  (COPY FILES)      │  @COMPANY_DOCUMENTS (.pdf) │
                    │                            │
                    │  Cortex AI Functions        │
                    │  ┌──────────────────────┐  │
                    │  │ AI_TRANSCRIBE        │  │
                    │  │ AI_TRANSLATE         │  │
                    │  │ AI_SENTIMENT         │  │
                    │  │ AI_CLASSIFY          │  │
                    │  │ AI_COMPLETE          │  │
                    │  │ AI_PARSE_DOCUMENT    │  │
                    │  │ AI_EXTRACT           │  │
                    │  └──────────────────────┘  │
                    │                            │
                    │  Result Tables              │
                    │  ├─ transcription_results   │
                    │  ├─ parsed_documents_raw    │
                    │  ├─ chat_validation_results │
                    │  └─ ticket_chat_alignment   │
                    │                            │
                    │  Streamlit Dashboard        │
                    │  (reads from result tables) │
                    └──────────────────────────┘
```

---

## AI Functions Cheat Sheet

| Function | Input | Output | Use Case |
|----------|-------|--------|----------|
| `AI_TRANSCRIBE` | Audio file (internal stage) | JSON with segments, speakers, timestamps | Speech-to-text with diarization |
| `AI_TRANSLATE` | Text + target language | Translated text | Multi-language support |
| `AI_SENTIMENT` | Text | JSON with scored sentiment categories | Customer satisfaction monitoring |
| `AI_CLASSIFY` | Text + category definitions | JSON with ranked labels + scores | Zero-shot categorization |
| `AI_COMPLETE` | Model name + prompt | Generated text | Summarization, reasoning, any LLM task |
| `AI_PARSE_DOCUMENT` | PDF file (internal stage) | JSON with structured content | Document digitization |
| `AI_EXTRACT` | Text + response schema | JSON with extracted fields | Structured data extraction from text |
