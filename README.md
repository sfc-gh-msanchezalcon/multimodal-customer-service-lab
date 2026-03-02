# Snowflake Cortex AI SQL: Multimodal Customer Service Analytics

Build a production-ready customer service analytics system that processes **audio**, **text**, and **PDF** data using Snowflake Cortex AI functions — all in pure SQL.

![Snowflake](https://img.shields.io/badge/Snowflake-29B5E8?style=for-the-badge&logo=snowflake&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=for-the-badge&logo=postgresql&logoColor=white)
![AI](https://img.shields.io/badge/Cortex_AI-FF6F00?style=for-the-badge&logo=openai&logoColor=white)

---

## What You'll Build

A complete analytics pipeline that:

| Capability | Description |
|------------|-------------|
| **Transcribe** | Convert customer service call recordings to searchable text |
| **Translate** | Automatically translate conversations to English |
| **Analyze Sentiment** | Detect customer emotions (positive/negative/neutral) |
| **Classify Issues** | Categorize calls into business-defined issue types |
| **Summarize** | Generate concise call summaries using LLMs |
| **Parse Documents** | Extract structured data from PDF documents |
| **Validate Data** | Cross-check chat logs against support tickets |

---

## AI Functions Covered

| Function | Purpose | Example Use Case |
|----------|---------|------------------|
| `AI_TRANSCRIBE` | Speech-to-text with speaker diarization | Call center recordings |
| `AI_TRANSLATE` | Multi-language translation | Global customer support |
| `AI_SENTIMENT` | Emotion detection | Customer satisfaction tracking |
| `AI_CLASSIFY` | Zero-shot categorization | Ticket routing |
| `AI_COMPLETE` | LLM text generation | Summarization, comparison |
| `AI_PARSE_DOCUMENT` | PDF/document extraction | Policy documents, invoices |
| `AI_EXTRACT` | Structured field extraction | Pull specific data points |

---

## Prerequisites

- [ ] Snowflake account in a [supported Cortex region](https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions#label-cortex-llm-availability)
- [ ] Role with permissions to create databases, stages, and tables
- [ ] Warehouse (MEDIUM or larger recommended)
- [ ] [Snowflake Notebooks](https://docs.snowflake.com/en/user-guide/ui-snowsight/notebooks) enabled

---

## Quick Start

### Step 1: Run the Setup Script (5 min)

1. Open a **SQL Worksheet** in Snowsight
2. Copy the contents of [`setup.sql`](./setup.sql)
3. Run the entire script
4. Verify you see: `✅ Setup complete!`

This creates:
- `MULTIMODAL_CUSTOMER_SERVICE` database
- Audio files stage (`@CUSTOMER_CALLS`)
- PDF documents stage (`@COMPANY_DOCUMENTS`)
- Sample tables (`CHAT_LOGS`, `SUPPORT_TICKETS`)

### Step 2: Import the Notebook (2 min)

1. Download [`notebook.ipynb`](./notebook.ipynb)
2. In Snowsight: **Projects** → **Notebooks** → **Import .ipynb file**
3. Select:
   - Database: `MULTIMODAL_CUSTOMER_SERVICE`
   - Schema: `DATA`
   - Warehouse: Your warehouse
4. Click **Create**

### Step 3: Run the Lab (30-45 min)

Work through the notebook sections in order:

| Part | Description | Key Functions | Time |
|------|-------------|---------------|------|
| **0** | Explore sample data | - | 2 min |
| **1** | Audio processing pipeline | `AI_TRANSCRIBE`, `AI_TRANSLATE`, `AI_SENTIMENT`, `AI_CLASSIFY`, `AI_COMPLETE` | 15 min |
| **2** | Document processing | `AI_PARSE_DOCUMENT` | 5 min |
| **3** | Chat log validation | `AI_CLASSIFY`, `AI_SENTIMENT`, `AI_EXTRACT` | 10 min |
| **4** | Ticket-chat alignment | `AI_COMPLETE` | 10 min |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA SOURCES                                  │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Audio Files    │  PDF Documents  │  Chat Logs & Tickets        │
│  (@CUSTOMER_    │  (@COMPANY_     │  (CHAT_LOGS,                │
│   CALLS)        │   DOCUMENTS)    │   SUPPORT_TICKETS)          │
└────────┬────────┴────────┬────────┴──────────────┬──────────────┘
         │                 │                       │
         ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                 CORTEX AI FUNCTIONS                              │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  AI_TRANSCRIBE  │ AI_PARSE_       │  AI_CLASSIFY                │
│  AI_TRANSLATE   │ DOCUMENT        │  AI_SENTIMENT               │
│  AI_SENTIMENT   │                 │  AI_EXTRACT                 │
│  AI_CLASSIFY    │                 │  AI_COMPLETE                │
│  AI_COMPLETE    │                 │                             │
└────────┬────────┴────────┬────────┴──────────────┬──────────────┘
         │                 │                       │
         ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    OUTPUT TABLES                                 │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ transcription_  │ parsed_         │  chat_validation_results    │
│ results         │ documents_raw   │  ticket_chat_alignment      │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

---

## Sample Data

| Dataset | Records | Description |
|---------|---------|-------------|
| Audio files | 5 | Customer service call recordings (MP3) |
| PDF documents | 3 | Company policy documents |
| Chat logs | 20 | Customer chat transcripts with agent classifications |
| Support tickets | 20 | Formal tickets linked to chat sessions |

---

## Troubleshooting

### "Function not found" errors
Ensure your account is in a [supported Cortex region](https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions#label-cortex-llm-availability) and cross-region is enabled:
```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```

### Slow transcription
Audio transcription is compute-intensive. Expected: ~30-60 seconds per audio file. Use a MEDIUM or larger warehouse.

### Stage files not found
Refresh the stage directories:
```sql
ALTER STAGE CUSTOMER_CALLS REFRESH;
ALTER STAGE COMPANY_DOCUMENTS REFRESH;
```

### Permission errors
Ensure your role has:
- `CREATE DATABASE` on account
- `USAGE` on warehouse
- `CREATE STAGE`, `CREATE TABLE` privileges

---

## Clean Up

To remove all lab resources:

```sql
DROP DATABASE IF EXISTS MULTIMODAL_CUSTOMER_SERVICE;
```

---

## Resources

- [Cortex AI Functions Documentation](https://docs.snowflake.com/en/sql-reference/functions-ai)
- [AI_TRANSCRIBE Reference](https://docs.snowflake.com/en/sql-reference/functions/ai_transcribe)
- [AI_COMPLETE Reference](https://docs.snowflake.com/en/sql-reference/functions/ai_complete)
- [Snowflake Notebooks Guide](https://docs.snowflake.com/en/user-guide/ui-snowsight/notebooks)

---

## License

This lab is provided as-is for educational purposes. Sample data is synthetic and does not represent real customer information.
