# Agentic RAG — Streamlit PDF + Web Retrieval

This repository contains a single Streamlit application (`app.py`) that builds a
Retrieval-Augmented Generation (RAG) workflow over an uploaded PDF and, when
needed, augments answers with fresh web content. The app:

1. Chunks the PDF with `RecursiveCharacterTextSplitter`
2. Embeds chunks using `sentence-transformers/all-mpnet-base-v2`
3. Stores vectors in a local FAISS index
4. Routes each query through an LLM-based guard (`check_local_knowledge`) to
   decide whether local content is sufficient
5. Falls back to a `CrewAI` duo (Serper + scraper) if web context is required
6. Synthesizes the final answer via `ChatGroq`

Everything runs inside Streamlit; there is no separate CLI entry point in the
current code.

---

## Key features

- **PDF-grounded answers**: Upload any PDF and query it via semantic search.
- **Router-aware web fallback**: If the local chunks cannot satisfy the query,
  the app dispatches a search-and-scrape crew to gather context.
- **Embeddings & index persistence**: Build a FAISS index once and optionally
  save it for reuse.
- **Modern LLM stack**: Answers are produced with Groq’s `llama-3.3-70b` model,
  while Gemini powers the CrewAI agents.
- **Streamlit UX**: Simple UI with toggles for web usage and index management.

---

## Requirements

- Python 3.10+
- `pip install -r requirements.txt`
- Environment variables (load via `.env` or your shell)

```bash
GROQ_API_KEY=your_groq_key
SERPER_API_KEY=your_serper_key
GEMINI_API_KEY=your_gemini_key
```

`python-dotenv` loads these automatically at app start. Without them, Groq,
Serper, and Gemini calls will fail.

---

## Running the app

All commands should be executed from the project root
`/Users/dineshkatiyar/Projects/PythonAIAgent/AIAgenticRAGWithUI`.

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
streamlit run app.py
```

### Streamlit workflow

1. **Upload PDF**: Builds an in-memory FAISS index. A status message confirms
   progress.
2. **Index path (optional)**: Provide a directory to load/save FAISS artifacts.
   Existing indices are loaded via `FAISS.load_local`. When “Save FAISS index”
   is checked, new indices are persisted via `save_local`.
3. **Ask a question**: Enter a query and click **Ask**.
4. **Toggle web usage**: Disable “Allow web search” to force purely local
   answers. If disabled and the router returns “No,” the app warns and skips
   the query.
5. **Review the answer**: Results appear under the “Answer” header.

Uploaded PDFs are stored temporarily using `NamedTemporaryFile`. Remove or clean
up files manually if you handle sensitive documents.

---

## Architecture overview

- `setup_vector_db(pdf_path)`: Loads the PDF via `PyPDFLoader`, splits text,
  embeds chunks, and returns a FAISS store.
- `get_local_content(vector_db, query)`: Retrieves up to five nearest chunks to
  form the context string.
- `check_local_knowledge(query, context)`: A Groq-powered router that returns a
  strict Yes/No signal about whether the local context is sufficient.
- `setup_web_scraping_agent()` + `get_web_content(query)`: Creates a Crew with
  Serper for discovery and `ScrapeWebsiteTool` for extraction, both running on a
  Gemini model supplied via the `LLM` wrapper.
- `process_query(...)`: Chooses local or web context, then calls
  `generate_final_answer`, which prompts `ChatGroq` to craft the response.
- `run_streamlit()`: Streamlit entry point wiring the UI to the pipeline.

All heavy lifting happens in `app.py`, so refer there for implementation
details.

---

## Index persistence tips

- Default index directory: `faiss_index`.
- Loading errors automatically trigger a rebuild from the uploaded PDF.
- Always reuse the exact embedding model (`all-mpnet-base-v2`) when loading an
  existing index to avoid vector-space mismatches.
- The index directory must exist (the app creates it when saving if needed).

---

## Troubleshooting

- **FAISS install (macOS/ARM)**: Prefer `conda install -c conda-forge faiss-cpu`
  or use a prebuilt wheel that matches your Python version.
- **Missing API keys**: The UI loads but queries fail. Double-check `.env`
  entries and restart Streamlit after exporting keys.
- **Rate limits**: Groq, Serper, or Gemini throttling manifests as errors in the
  Streamlit logs. Consider lowering usage or swapping keys.
- **Large PDFs**: Chunking happens in-memory. Monitor RAM usage or reduce
  `chunk_size` in `setup_vector_db` if necessary.

---

## Next steps

- Add provenance UI elements (surface which chunks or URLs informed each answer)
- Provide sample PDFs plus integration tests for CI
- Containerize the Streamlit app for reproducible deployment

Happy hacking!

