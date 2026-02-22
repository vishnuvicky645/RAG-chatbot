# RAG-based AI Chat Assistant using n8n, Pinecone & Groq

A Retrieval-Augmented Generation (RAG) chatbot that answers domain-specific queries by ingesting documents into a vector store and retrieving relevant context at query time.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Problem Statement](#2-problem-statement)
3. [Architecture](#3-architecture)
4. [Tech Stack](#4-tech-stack)
5. [Workflow Explanation](#5-workflow-explanation)
6. [How It Works](#6-how-it-works)
7. [Setup Instructions](#7-setup-instructions)
8. [Screenshots](#8-screenshots)
9. [Sample Queries](#9-sample-queries)
10. [Limitations](#10-limitations)
11. [Future Improvements](#11-future-improvements)

---

## 1. Project Overview

This project implements a fully automated RAG pipeline using **n8n** (a low-code workflow automation platform). Documents uploaded to Google Drive are automatically ingested, chunked, embedded, and stored in a **Pinecone** vector database. A chat interface (exposed via webhook) accepts user queries, retrieves relevant document chunks using similarity search, and generates contextual answers using the **Groq** LLM API.

The system is demonstrated with Union Budget documents as the knowledge domain but is domain-agnostic by design.

---

## 2. Problem Statement

Large Language Models (LLMs) have a fixed knowledge cutoff and no access to proprietary or domain-specific documents. Fine-tuning an LLM for every new document set is expensive and time-consuming.

**RAG** solves this by keeping the LLM general-purpose and supplying relevant document context at inference time through vector similarity search. This project demonstrates how to build such a pipeline without writing a traditional backend, using n8n as the orchestration layer.

---

## 3. Architecture

The system is split into two independent n8n workflows.

### 3.1 Document Ingestion Flow

```
Google Drive (fileCreated trigger)
        │
        ▼
  Download File
        │
        ▼
  Default Data Loader   ←── parses PDF / text content
        │
        ▼
  Recursive Text Splitter  ←── splits content into overlapping chunks
        │
        ▼
  Google Embeddings (768-dim)  ←── converts each chunk to a dense vector
        │
        ▼
  Pinecone Vector Store (upsert)
  [Serverless index · Dense · AWS us-east-1]
```

### 3.2 Chat Flow

```
Webhook (Chat Message Received)
        │
        ▼
    AI Agent node
     ┌──────────────────────────────────┐
     │  Groq Chat Model (LLM)           │
     │  Vector Store QA Tool            │
     │    └── Pinecone similarity search│
     └──────────────────────────────────┘
        │
        ▼
  Structured Answer → HTTP Response
```

**RAG Pipeline summary:** At query time the AI Agent embeds the user question, retrieves the top-k most similar document chunks from Pinecone, injects them as context into the prompt, and asks Groq to generate an answer grounded in those chunks.

---

## 4. Tech Stack

| Component | Role |
|---|---|
| **n8n** | Workflow orchestration (no-code/low-code) |
| **Google Drive** | Document source and ingestion trigger |
| **Google Embeddings** | Text → 768-dimensional dense vectors |
| **Pinecone** | Serverless vector database (similarity search) |
| **Groq** | Hosted LLM inference (fast, low-latency) |
| **Webhook** | Chat interface endpoint |

---

## 5. Workflow Explanation

### Ingestion Workflow

| Step | Node | Description |
|---|---|---|
| 1 | Google Drive Trigger | Fires when a new file is created in a monitored folder |
| 2 | Download File | Fetches the binary content of the file |
| 3 | Default Data Loader | Extracts plain text from the document |
| 4 | Recursive Text Splitter | Splits text into overlapping chunks (e.g. 500 tokens, 50 overlap) |
| 5 | Google Embeddings | Generates a 768-dim embedding vector for each chunk |
| 6 | Pinecone Vector Store | Upserts chunk text + embedding + metadata into the index |

### Chat Workflow

| Step | Node | Description |
|---|---|---|
| 1 | Webhook | Receives `{ "chatInput": "..." }` via HTTP POST |
| 2 | AI Agent | Orchestrates tool use and prompt construction |
| 3 | Groq Chat Model | LLM used for answer generation |
| 4 | Vector Store QA Tool | Performs similarity search on Pinecone for top-k chunks |
| 5 | Response | Returns the generated answer to the caller |

---

## 6. How It Works

1. **Upload a document** — Place a PDF or text file in the configured Google Drive folder.
2. **Automatic ingestion** — The Google Drive trigger fires, the file is downloaded and parsed.
3. **Chunking** — The text is split into manageable, overlapping chunks to preserve context across boundaries.
4. **Embedding** — Each chunk is converted to a 768-dimensional vector using Google's embedding model.
5. **Indexing** — Vectors and their source text are upserted into a Pinecone serverless index.
6. **Query** — A user sends a question to the webhook endpoint.
7. **Retrieval** — The AI Agent embeds the question and queries Pinecone for the most semantically similar chunks.
8. **Generation** — Retrieved chunks are injected into the Groq prompt as context; Groq returns a grounded answer.
9. **Response** — The answer is returned to the user via the webhook response.

---

## 7. Setup Instructions

### Prerequisites

- **n8n** instance (cloud or self-hosted)
- **Google Cloud** project with the Embeddings API enabled and a service account key
- **Google Drive** folder to monitor
- **Pinecone** account with a serverless index (Dense, 768 dimensions, AWS us-east-1)
- **Groq** API key

### Steps

1. **Import workflows**

   In n8n, go to **Workflows → Import** and import the two workflow JSON files (ingestion and chat) from this repository.

2. **Configure credentials**

   | Credential | Used by |
   |---|---|
   | Google OAuth2 / Service Account | Google Drive Trigger, Download File |
   | Google PaLM API | Google Embeddings node |
   | Pinecone API Key | Pinecone Vector Store node |
   | Groq API Key | Groq Chat Model node |

3. **Set Pinecone index details**

   In both the ingestion and chat workflows, open the Pinecone node and set:
   - **Index name** — your Pinecone index name
   - **Namespace** — optional, e.g. `union-budget`

4. **Configure the Google Drive Trigger**

   Set the folder ID to the Drive folder you want to monitor for new files.

5. **Activate workflows**

   Enable both workflows in n8n. The ingestion flow activates on file upload; the chat flow activates on POST requests to the webhook URL.

6. **Test ingestion**

   Upload a PDF to the monitored Google Drive folder and confirm that entries appear in the Pinecone index (visible in the Pinecone console).

7. **Test the chat endpoint**

   ```bash
   curl -X POST <your-n8n-webhook-url> \
     -H "Content-Type: application/json" \
     -d '{"chatInput": "What are the key highlights of the Union Budget?"}'
   ```

---

## 8. Screenshots


**Ingestion Workflow (n8n canvas)**

![Ingestion workflow screenshot](screenshots/ingestion-workflow.png)

**Chat Workflow (n8n canvas)**

![Chat workflow screenshot](screenshots/chat-workflow.png)

**Sample Chat Interaction**

![Chat response screenshot](screenshots/sample-chat-response.png)

---

## 9. Sample Queries

The following queries were tested against Union Budget documents:

| Query | Expected Response Type |
|---|---|
| "What is the total fiscal deficit target for the current budget?" | Specific numerical fact with context |
| "What allocations were made for the education sector?" | Enumerated list of schemes and amounts |
| "Explain the changes to income tax slabs." | Structured explanation from document |
| "What infrastructure projects were announced?" | Summary of key announcements |
| "How does this budget address rural employment?" | Context-grounded paragraph response |

---

## 10. Limitations

- **n8n cloud trial expired** — The deployed n8n instance was running on a 14-day free trial, which has now expired. The workflow cannot currently be demonstrated live. To replicate it, follow the [Setup Instructions](#7-setup-instructions) and deploy n8n on a self-hosted instance or a new cloud trial.
- **Embedding dimension lock-in** — The Pinecone index is configured for 768-dimensional vectors (Google Embeddings). Switching to a different embedding model would require re-indexing all documents.
- **Single-turn context** — The current AI Agent does not maintain multi-turn conversation history across separate webhook calls.
- **File type support** — The Default Data Loader handles common formats (PDF, DOCX, TXT). Scanned PDFs without OCR pre-processing will not yield useful text.
- **Rate limits** — Both the Groq API and Google Embeddings API have free-tier rate limits that may throttle high-volume ingestion or query loads.

---

## 11. Future Improvements

- **Persistent chat memory** — Add a session-aware memory node (e.g., Redis-backed buffer) to support multi-turn conversations.
- **Multi-domain support** — Use Pinecone namespaces to segregate document domains and route queries to the appropriate namespace automatically.
- **OCR pre-processing** — Integrate a document OCR step before the Data Loader to support scanned PDFs.
- **Front-end UI** — Build a lightweight React or Streamlit chat interface instead of relying on raw webhook calls.
- **Re-ranking** — Add a cross-encoder re-ranking step after Pinecone retrieval to improve answer relevance.
- **Metadata filtering** — Store document metadata (date, source, section) in Pinecone and support filtered retrieval.
- **Self-hosted deployment** — Migrate from n8n cloud to a self-hosted n8n instance (Docker/Kubernetes) for production-grade availability.
