# ⚖️ DV Act 2005 — RAG Legal AI Agent

> An AI-powered Legal Support Agent built with n8n, Supabase pgvector, OpenAI Embeddings, and Claude — helping women understand their rights under the **Protection of Women from Domestic Violence Act, 2005 (India)**.

---

## 🎯 What This Does

This system answers legal questions about the DV Act 2005 by:
1. **Searching** a vector knowledge base built from the actual Act text
2. **Retrieving** the most relevant legal sections
3. **Generating** clear, cited, actionable legal guidance
4. **Remembering** conversation context per session

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    PIPELINE 2 (One-time)                 │
│                   Knowledge Base Builder                  │
│                                                           │
│  Manual Trigger → Read PDF → Extract Text                 │
│       → Chunk Text → Embed → Store in Supabase           │
└──────────────────────────┬────────────────────────────────┘
                           │
                    Supabase pgvector
                    documents table
                           │
┌──────────────────────────┴────────────────────────────────┐
│                    PIPELINE 1 (Always Live)                │
│                      AI Q&A Agent                          │
│                                                            │
│  Webhook → Extract Input → AI Agent (Tools Agent)          │
│       → Vector Search → Claude → Format → Respond         │
└────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| Automation & Orchestration | [n8n](https://n8n.io) (self-hosted) |
| Vector Database | [Supabase](https://supabase.com) + pgvector |
| Embeddings | OpenAI `text-embedding-3-small` (1536 dims) |
| LLM | Anthropic Claude 3.5 Sonnet |
| Similarity Search | PostgreSQL `match_documents()` function |
| Memory | n8n Window Buffer Memory (per session) |
| Document Processing | n8n PDF extraction + Recursive Text Splitter |

---

## 📁 Project Structure

```
dv-act-rag-agent/
├── README.md
├── DV_Act_2005_-_Knowledge_Base_Builder.json   ← n8n workflow export
├── docs/
│   └── DV_Act_2005.pdf                         ← Source legal document
└── sql/
    ├── create_table.sql                         ← Supabase table setup
    └── match_documents.sql                      ← Vector search function
```

---

## ⚙️ Prerequisites

- [n8n](https://docs.n8n.io/hosting/) self-hosted (v2.7.5+)
- [Supabase](https://supabase.com) account (free tier works)
- [OpenAI](https://platform.openai.com) API key
- [Anthropic](https://console.anthropic.com) API key
- DV Act 2005 PDF

---

## 🚀 Setup Guide

### Step 1 — Supabase Database Setup

Run this SQL in your Supabase SQL Editor:

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create documents table
CREATE TABLE IF NOT EXISTS documents (
  id bigserial PRIMARY KEY,
  content text,
  metadata jsonb,
  embedding vector(1536)
);

-- Create vector search function
CREATE OR REPLACE FUNCTION match_documents(
  query_embedding vector(1536),
  match_count int DEFAULT 4,
  filter jsonb DEFAULT '{}'
)
RETURNS TABLE(
  id bigint,
  content text,
  metadata jsonb,
  embedding vector(1536),
  similarity float
)
LANGUAGE sql STABLE
AS $$
  SELECT
    documents.id,
    documents.content,
    documents.metadata,
    documents.embedding,
    1 - (documents.embedding <=> query_embedding) AS similarity
  FROM documents
  WHERE documents.metadata @> filter
  ORDER BY documents.embedding <=> query_embedding
  LIMIT match_count;
$$;

-- Create index for faster search
CREATE INDEX IF NOT EXISTS documents_embedding_idx
ON documents USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

### Step 2 — n8n Setup

1. Import `DV_Act_2005_-_Knowledge_Base_Builder.json` into n8n
2. Configure credentials:

| Credential | Where to get |
|---|---|
| Anthropic API | [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys) |
| OpenAI API | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) |
| Supabase API | Supabase Dashboard → Settings → API |

3. Update file path in `Read/Write Files from Disk` node:
```
C:\path\to\your\DV_Act_2005.pdf
```

### Step 3 — Build Knowledge Base (Pipeline 2)

1. Open the workflow in n8n
2. Click **Manual Trigger** node
3. Click **Execute Workflow**
4. Wait for all chunks to be embedded and stored (~56 documents)

Verify in Supabase:
```sql
SELECT COUNT(*), COUNT(embedding) FROM documents;
-- Should return: 56, 56
```

### Step 4 — Activate Pipeline 1

1. Click the **toggle** in top-right of n8n → set to **Active**
2. Click **Publish**

### Step 5 — Test

```bash
curl -X POST http://localhost:5678/webhook/dv-agent \
  -H "Content-Type: application/json" \
  -d '{"question": "What is domestic violence under Section 3?", "session_id": "test_001"}'
```

---

## 📡 API Reference

### Endpoint

```
POST /webhook/dv-agent
```

### Request Body

```json
{
  "question": "What is domestic violence under Section 3 of DV Act 2005?",
  "session_id": "user_001"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `question` | string | ✅ | Legal question to ask |
| `session_id` | string | ✅ | Unique ID for conversation memory |

### Response

```json
{
  "success": true,
  "question": "What is domestic violence under Section 3 of DV Act 2005?",
  "answer": "Under Section 3 of the DV Act 2005, domestic violence includes...",
  "session_id": "user_001",
  "timestamp": "2026-02-26T09:38:26.914Z",
  "source_document": "Protection of Women from Domestic Violence Act, 2005"
}
```

---

## 💬 Example Conversations

### Question 1 — Definition
```bash
curl -X POST http://localhost:5678/webhook/dv-agent \
  -H "Content-Type: application/json" \
  -d '{"question": "What is domestic violence under Section 3?", "session_id": "s001"}'
```

**Response covers:**
- All 4 categories (physical, sexual, verbal, economic abuse)
- Dowry-related violence
- Key interpretive principles
- Who qualifies as aggrieved person

### Question 2 — Follow-up (memory in action)
```bash
curl -X POST http://localhost:5678/webhook/dv-agent \
  -H "Content-Type: application/json" \
  -d '{"question": "What evidence do I need to collect?", "session_id": "s001"}'
```

**Response covers:**
- Medical documentation
- Affidavits and written statements
- Communication records
- Financial records for economic abuse
- Practical do's and don'ts

### Question 3 — Remedies
```bash
curl -X POST http://localhost:5678/webhook/dv-agent \
  -H "Content-Type: application/json" \
  -d '{"question": "What protection orders can I get?", "session_id": "s001"}'
```

---

## 🔄 Pipeline Details

### Pipeline 2 — Knowledge Base Builder

```
Manual Trigger
      ↓
Read/Write Files from Disk
(loads DV_Act_2005.pdf)
      ↓
Extract from File
(extracts raw text via PDF parser)
      ↓
Code in JavaScript
(passes full text with metadata)
      ↓
Supabase Vector Store (Insert Documents)
      ├── Default Data Loader + Recursive Text Splitter
      │   (chunks: 1000 chars, 200 overlap → ~56 chunks)
      └── OpenAI Embeddings
          (text-embedding-3-small → 1536 dimensions)
      ↓
Supabase documents table
(56 rows with content + metadata + vector)
```

### Pipeline 1 — AI Q&A Agent

```
Webhook POST /dv-agent
      ↓
Extract Input
(question + session_id)
      ↓
AI Agent (Tools Agent - Claude 3.5 Sonnet)
      ├── Window Buffer Memory (per session_id)
      └── Supabase Vector Store (Retrieve as Tool)
              ├── Converts question → embedding (OpenAI)
              ├── Calls match_documents() in Supabase
              └── Returns top 4 similar chunks
      ↓
Format Response
(success, question, answer, session_id, timestamp)
      ↓
Respond to Webhook
```

---

## 🧠 AI Agent System Prompt

The agent is configured to:
- ✅ Search knowledge base before answering
- ✅ Cite specific sections (Section 3, 18, 19, etc.)
- ✅ Explain in plain language
- ✅ Suggest evidence types and remedies
- ❌ Never hallucinate legal provisions
- ❌ Never draft FIRs or court petitions
- ❌ Never predict case outcomes
- ❌ Always end with legal disclaimer

---

## 🗄️ Database Schema

```sql
CREATE TABLE documents (
  id        bigserial PRIMARY KEY,
  content   text,           -- DV Act chunk text
  metadata  jsonb,          -- source, type, jurisdiction, year
  embedding vector(1536)    -- OpenAI embedding
);
```

Sample row:
```json
{
  "id": 2044,
  "content": "Section 3 defines domestic violence as any act...",
  "metadata": {
    "source": "DV_Act_2005",
    "document_type": "legal_act",
    "jurisdiction": "India",
    "year": 2005
  },
  "embedding": [0.039, 0.034, ...]
}
```

---

## 🔧 Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `webhook not registered` | Workflow not active | Toggle Active in n8n |
| `embedding is null` | Wrong Supabase node type | Use Vector Store node, not Create Row |
| `match_documents ambiguous` | Column/variable clash | Use `documents.metadata` in SQL |
| `model not found` | Wrong Claude model string | Select from dropdown in node |
| `A Document sub-node must be connected` | Missing Data Loader | Add Default Data Loader to Vector Store |
| `knowledge base not returning results` | match_documents missing embedding in RETURNS | Add `embedding vector(1536)` to function return |

---

## 🚀 Extending This Project

### Add More Legal Documents
1. Add new PDF to the file path
2. Update metadata in `Code in JavaScript`:
```javascript
metadata: {
  source: "IPC_498A",
  document_type: "legal_act",
  jurisdiction: "India",
  year: 1983
}
```
3. Re-run Pipeline 2

### Connect to WhatsApp
Replace Webhook trigger with Twilio WhatsApp node in n8n

### Connect to Telegram
Replace Webhook trigger with Telegram Trigger node in n8n

### Add Multilingual Support
Add a translation node before the AI Agent to support Tamil, Hindi, Telugu

---

## ⚠️ Disclaimer

This system provides **informational guidance only** based on the DV Act 2005. It is **not legal advice**. Users should consult a qualified lawyer for case-specific counsel.

---

## 📄 License

MIT License — feel free to use, modify, and distribute.

---

## 🙏 Acknowledgements

- [n8n](https://n8n.io) — powerful open-source automation
- [Supabase](https://supabase.com) — open-source Firebase alternative with pgvector
- [Anthropic](https://anthropic.com) — Claude AI
- [OpenAI](https://openai.com) — text embeddings
- Ministry of Law & Justice, India — DV Act 2005

---

## 📬 Contact

Built with ❤️ for legal awareness and women's safety in India.

If you're an NGO, legal aid center, or developer wanting to deploy this — reach out!

⚖️ *Because every woman deserves to know her rights.*
