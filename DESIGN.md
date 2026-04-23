# PaperPal — Design Document

## 1. Overview

PaperPal is an AI research companion that helps research students — thesis writers, graduate students, and PhD researchers — deeply engage with academic papers through citation-grounded Q&A, targeted concept explanations, and spaced-repetition flashcards. Unlike using ChatGPT directly, PaperPal provides a dedicated workspace per paper that grounds every answer in the source text to reduce hallucination, preserves full conversation context, and organizes key concepts into a retention system designed for long-term learning.

## 2. Problem Statement

Research students face daily exhaustion trying to understand dense academic papers. They struggle with dense concepts, forget much of what they read, and lose time on sections that turn out to be unimportant. A single technical paper takes 2-4 hours to read carefully, yet retention after a week is low — meaning much of that time is effectively lost. ChatGPT summaries help in the moment, but they're often too generic, miss connections between sections, and don't aid retention. Reading alone isn't much better — researchers finish papers feeling lost and without the ability to reference what they read later. Since progress in a field depends on absorbing many papers efficiently, this bottleneck compounds over a semester or a thesis.

## 3. Target User

PaperPal's primary user is a student researcher — whether in college (undergrad thesis writers) or grad school (Master's or PhD) — deep in thesis research, reading for seminars, or literature review. They're most often in technical fields where papers are dense with jargon and notation. They open PaperPal when sitting down to read a paper with limited time before an advisor meeting or seminar, and need quick but efficient understanding. They're comfortable using AI tools for intellectual curiosity and efficiency, and accept rough edges in exchange for real utility. They don't care about flashy animations, gamification, social features, or collaboration — they just need to grasp the paper for tomorrow's seminar.

## 4. Goals

- Every Q&A answer cites the specific sections of the source paper it draws from
- Generate targeted concept explanations that connect to other relevant sections of the paper
- Generate a structured summary of each uploaded paper (key arguments, methodology, findings)
- Auto-generate flashcards from paper content covering key concepts, definitions, and claims
- Implement a spaced-repetition review system (SM-2 algorithm) that schedules flashcards for optimal retention
- Provide a dedicated workspace per paper that preserves conversation context across sessions
- Support PDF uploads (up to 50 pages) as the primary document format
- Deploy the application to a live URL with persistent storage
- Provide polished dark and light mode themes that meet the minimalist-academic aesthetic

## 5. Non-Goals

- Not a paper writing or editing tool
- Not a collaborative or multi-user tool
- No support for non-PDF formats in v1 (DOCX, LaTeX, TXT deferred to Future Work)
- No mobile app or native mobile experience in v1 (web-responsive only)
- No multi-paper comparison or cross-paper synthesis in v1
- No flashcard export to external tools (Anki, Quizlet) in v1
- No user accounts or authentication in v1 (single-user / local-session scope)
- No social, sharing, or gamification features


## 6. User Stories

- **[P0]** As a Master's student doing a literature review, I want to upload a dense paper and get a structured summary, so that I can quickly decide if it's worth a deep read.
- **[P0]** As a PhD researcher, I want to ask specific questions about a paper and see which sections each answer comes from, so that I can trust the response and cross-reference the source.
- **[P0]** As an undergraduate thesis writer, I want concept explanations that connect back to other relevant sections of the paper, so that I understand how ideas in the paper build on each other.
- **[P1]** As a grad student studying for qualifying exams, I want auto-generated flashcards that resurface over days and weeks, so that I retain key concepts long after I've finished reading the paper.

---

## 7. System Architecture

PaperPal follows a classic frontend/backend split with an AI processing layer. The frontend is a static site that communicates with the backend via REST. The backend handles document processing, embedding generation, vector storage, and LLM orchestration.

┌──────────────┐      HTTP       ┌──────────────┐
│   Frontend   │ ──────────────► │   Backend    │
│ (HTML/CSS/JS)│ ◄────────────── │  (FastAPI)   │
└──────────────┘    JSON API     └──────┬───────┘
│
┌────────────────────┼────────────────────┐
▼                    ▼                    ▼
┌─────────────┐      ┌──────────────┐     ┌──────────────┐
│ Claude API  │      │  ChromaDB    │     │   SQLite     │
│ (LLM calls) │      │ (vectors)    │     │ (flashcards) │
└─────────────┘      └──────────────┘     └──────────────┘

**Data flow on paper upload:** PDF → text extraction → chunking → embeddings → stored in ChromaDB.

**Data flow on query:** User question → embedding → ChromaDB similarity search → top-k chunks → Claude API with retrieved context → response with citations.

## 8. API Design

RESTful JSON API exposed by the FastAPI backend.

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/papers/upload` | Upload PDF, returns paper_id after processing |
| GET | `/api/papers/{paper_id}` | Get paper metadata and summary |
| POST | `/api/papers/{paper_id}/ask` | Ask a question — returns answer with citations |
| POST | `/api/papers/{paper_id}/explain` | Get an explanation of a concept from the paper |
| GET | `/api/papers/{paper_id}/flashcards` | Get auto-generated flashcards for the paper |
| POST | `/api/flashcards/{card_id}/review` | Submit a flashcard review result (for spaced repetition) |
| GET | `/api/flashcards/due` | Get flashcards due for review today |

All endpoints return JSON. Errors follow standard HTTP codes (400 bad request, 404 not found, 500 server error) with `{"error": "message"}` body.

## 9. Technical Stack & Decisions

| Layer | Technology | Reason |
|---|---|---|
| Frontend | Vanilla HTML/CSS/JS | Forces fundamentals; no framework overhead for v1 |
| Backend | FastAPI (Python) | Async support for LLM streaming; same language as ML ecosystem |
| LLM | Claude API (Anthropic) | Strong reasoning on dense text; reliable citation adherence |
| Embeddings | Sentence-Transformers (local) | Free, fast, no external API dependency |
| Vector DB | ChromaDB | File-based, zero-config; sufficient for single-user scale |
| Database | SQLite | Simple, no separate server needed for v1 |
| PDF Parsing | PyMuPDF | Handles academic PDFs well (multi-column, figures) |
| Spaced Repetition | SM-2 algorithm | Proven algorithm; same one used by Anki |
| Frontend hosting | Vercel | Free tier, instant deploy for static sites |
| Backend hosting | Railway | Simple Python deploy, supports Docker |

### Key Tradeoffs Made

- **Vanilla JS over React for v1** — prioritizing DOM/fetch fluency over framework speed
- **Local embeddings over hosted** — avoids per-request cost for document processing
- **No user auth in v1** — sessions are local; reduces scope significantly
- **ChromaDB over Pinecone** — simpler deploy, no external dependencies for a solo project

---

## 10. Risks & Open Questions

### Risks
- **PDF parsing edge cases:** Multi-column layouts, equations, and scanned pages may produce noisy text that degrades RAG quality.
- **Claude API cost:** Long documents with many queries could burn through credits fast; need to implement sensible rate limits.
- **Chunking strategy:** Wrong chunk size degrades retrieval quality — may require tuning per document type.
- **Spaced repetition complexity:** SM-2 is well-documented, but integrating it cleanly with the flashcard UX adds non-trivial state management.

### Open Questions
- Should users be able to override/edit auto-generated flashcards, or treat them as read-only?
- How long is a "session" — do we persist chat history across browser refreshes?
- What's the maximum document length before we reject or warn the user?

## 11. Future Work

- Multi-format support: DOCX, LaTeX (.tex), plain text
- Multi-paper comparison and cross-paper synthesis
- Flashcard export to Anki, Quizlet, or CSV
- User accounts and authentication for multi-device sync
- Mobile-native experience (iOS/Android)
- "Related papers" recommendations via Claude + external paper databases
- Collaborative annotation (share a paper + notes with another user)
- Voice input for asking questions about a paper

## 12. Success Metrics

### Technical Metrics
- Paper upload + processing completes in < 60 seconds for a 20-page PDF
- Q&A response latency < 5 seconds end-to-end
- Retrieval relevance: manually evaluate that top-k chunks contain the answer for a test set of 10 questions

### Product Metrics (for v1 validation)
- I personally use PaperPal for every paper I read for a month — it has to be useful to me first
- At least 3 research students outside of me try the tool and provide qualitative feedback
- At least 1 feature request comes from a real user (indicates engagement)

### Portfolio Metrics
- Project is deployed and publicly accessible
- README has screenshots, 60-second demo video, and clear setup instructions
- Code is clean enough that a senior engineer could read the repo in 10 minutes and understand the architecture
