<h1><span style="color:#2b8a3e">Agentic RAG (app.py) — With Streamlit UI</span></h1>

<p>This README documents the `app.py` implementation in the `AIAgenticRAGWithUI` folder. It explains how to use the CLI and Streamlit UI, the new FAISS index persistence behavior, and the updated requirements.</p>

<h2><span style="color:#1f77b4">Overview</span></h2>

This version of `app.py` provides two modes:

- CLI mode: run with `python app.py` and optional flags to build/load a FAISS index and run a query.
- Streamlit UI: run with `streamlit run app.py` to upload a PDF, enter a query, and view responses interactively.

Both use the same core pipeline: PDF -> chunk -> embeddings -> FAISS vector store -> router -> LLM answer generation (local or web-sourced).

<h2><span style="color:#d62728">New & important features</span></h2>

- Streamlit UI: PDF upload, query input, option to load/save FAISS index, result display, and caching via `st.cache_resource`.
- CLI flags: `--pdf`, `--query`, `--no-web`, `--save-index`, `--index-path`.
- FAISS persistence: build and save `FAISS` index to disk; CLI and UI can load an existing index to skip rebuilding.
- Single embedding model constant: `EMBEDDING_MODEL` used consistently for save/load compatibility.
- Requirements updated to include `streamlit` and `google-generativeai`.
- CLI flags: `--pdf`, `--query`, `--no-web`, `--save-index`, `--index-path` to control behavior from the terminal.
- FAISS index persistence: the app can save a built FAISS index to disk and reload it to avoid rebuilding on every run.
- Consistent embedding model: `EMBEDDING_MODEL` constant is used so indexes built/loaded use the same embedding model.
- Requirements updated with `streamlit` and `google-generativeai` entries to support the UI and Gemini client.

<h2><span style="color:#9467bd">Requirements</span></h2>

- Python 3.10+ recommended (the project has been tested with modern Python 3.11/3.12 environments).
- Environment variables (set in a `.env` file or in your environment):
  - `GROQ_API_KEY` — API key for Groq/ChatGroq usage (if required)
  - `SERPER_API_KEY` — API key for the Serper web search tool
  - `GEMINI_API_KEY` — API key for the Gemini LLM used by `crew_llm` (note name change from earlier `GEMINI`)
````markdown
<h1><span style="color:#2b8a3e">Agentic RAG (app.py) — With Streamlit UI</span></h1>

<p>This README documents the `app.py` implementation in this folder (`AIAgenticRAGWithUI`). It includes updated instructions for the Streamlit UI, CLI flags, FAISS index persistence, and updated requirements.</p>

<h2><span style="color:#1f77b4">Overview</span></h2>

This version of `app.py` extends the original Agentic RAG prototype with two modes of operation:

- Command-line (CLI) mode: run with `python app.py` and optional flags to load/build a FAISS index and run a single query.
- Streamlit UI mode: run with `streamlit run app.py` to open a browser UI that accepts a PDF upload, lets you enter a query, and shows the answer interactively.

Both modes reuse the same core functions:

- PDF -> FAISS vector store (via `PyPDFLoader`, `RecursiveCharacterTextSplitter`, `HuggingFaceEmbeddings`).
- Local routing: `check_local_knowledge(query, context)` decides whether to answer from local PDF content.
- Web search & scraping: `crewai` agents (`SerperDevTool`, `ScrapeWebsiteTool`) are used if local context does not contain the answer.
- Final answer generation uses `ChatGroq` and a crew-managed `crew_llm` (Gemini) where configured.

<h2><span style="color:#d62728">New/Updated Features</span></h2>

- Streamlit UI (browser): upload a PDF, optionally load/save a FAISS index, enter queries, and press Ask. The UI caches the vector DB per uploaded file for faster repeated queries.
- CLI flags: `--pdf`, `--query`, `--no-web`, `--save-index`, `--index-path` to control behavior from the terminal.
- FAISS index persistence: the app can save a built FAISS index to disk and reload it to avoid rebuilding on every run.
- Consistent embedding model: `EMBEDDING_MODEL` constant is used so indexes built/loaded use the same embedding model.
- Requirements updated with `streamlit` and `google-generativeai` entries to support the UI and Gemini client.

<h2><span style="color:#9467bd">Requirements</span></h2>

- Python 3.10+ recommended (the project has been tested with modern Python 3.11/3.12 environments).
- Environment variables (set in a `.env` file or in your environment):
  - `GROQ_API_KEY` — API key for Groq/ChatGroq usage (if required)
  - `SERPER_API_KEY` — API key for the Serper web search tool
  - `GEMINI_API_KEY` — API key for the Gemini LLM used by `crew_llm` (note name change from earlier `GEMINI`)

<h2><span style="color:#17becf">Python dependencies (recommended)</span></h2>

The repository includes a `requirements.txt` in `AIAgenticRAGWithUI/` with the main dependencies used by the UI variant. Key packages:

- python-dotenv
- langchain (or langchain-community variants used here)
- faiss-cpu (or faiss-gpu)
- sentence-transformers
- langchain-huggingface
- langchain-groq
- crewai and crewai-tools
- streamlit==1.26.0
- google-generativeai==0.3.0

Install (example):

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r AIAgenticRAGWithUI/requirements.txt
```

Note: FAISS can be tricky on macOS; consider installing `faiss-cpu` via conda if you hit issues.

<h2><span style="color:#ff7f0e">Configuration</span></h2>

1. Create a `.env` file in the project root (or export env vars). Example:

```
GROQ_API_KEY=your_groq_key
SERPER_API_KEY=your_serper_key
GEMINI_API_KEY=your_gemini_key
```

2. Decide whether you want to run the Streamlit UI or CLI. The CLI supports persistent FAISS indexes and quick one-off queries; the UI is better for interactive exploration.

<h2><span style="color:#8c564b">How it works (contract)</span></h2>

- Inputs: a query (string) and a FAISS vector DB (built from a PDF or loaded from disk). The UI accepts a PDF upload and optional index path; the CLI accepts `--pdf` and `--index-path`.
- Outputs: a text answer (string) displayed in the UI or printed to stdout.
- Data shapes: `vector_db` is a FAISS wrapper from LangChain-community; documents are chunked strings returned by `PyPDFLoader`.
- Error modes: missing PDF, invalid index, missing API keys, dependency errors, network timeouts.

<h2><span style="color:#e377c2">Edge cases & considerations</span></h2>

- Uploaded PDF may be large — chunking and embedding will use memory/time. Consider smaller documents for quick iteration.
- The router (`check_local_knowledge`) uses an LLM call to decide whether local content suffices — this can be noisy. Use `--no-web` to force local-only behavior when desired.
- Index compatibility: ensure the same `EMBEDDING_MODEL` is used when saving and loading FAISS indexes.

<h2><span style="color:#7f7f7f">How to run</span></h2>

CLI mode (recommended for quick scripted runs):

```bash
python AIAgenticRAGWithUI/app.py \
  --pdf path/to/your.pdf \
  --query "Summarize agentic RAG" \
  --save-index \
  --index-path my_faiss_index
```

Options:
- `--no-web` : disable web search/scraping; use local documents only.
- `--save-index` : save built FAISS index to `--index-path` to reuse later.

Streamlit UI (recommended for interactive use):

1. Install dependencies (see above).
2. Run:

```bash
streamlit run AIAgenticRAGWithUI/app.py
```

3. In the browser:
- Upload a PDF (or provide a path to an existing FAISS index in the UI input).
- Optionally set an index path and choose to save the built index.
- Enter your query and click Ask.

<h2><span style="color:#bcbd22">Index persistence details</span></h2>

- The CLI attempts to load a FAISS index from `--index-path` using `FAISS.load_local( index_path, embeddings )`.
- If the index doesn't exist or fails to load, the CLI will rebuild from the provided PDF.
- In Streamlit, the UI can also load an index from the provided index path or build from the uploaded PDF and optionally save the index for reuse.
- Ensure the same `EMBEDDING_MODEL` constant is used across runs so saved indexes remain compatible.

<h2><span style="color:#1b9e77">Files changed / new artifacts</span></h2>

- `AIAgenticRAGWithUI/app.py` — added Streamlit UI, CLI flags, FAISS load/save, and embedding model constant.
- `AIAgenticRAGWithUI/requirements.txt` — added `streamlit` and `google-generativeai` entries.

<h2><span style="color:#9467bd">Troubleshooting</span></h2>

- If Streamlit fails to import, ensure `streamlit` is installed in the active environment.
- If FAISS fails to build on macOS, try installing a prebuilt wheel or use conda to install `faiss-cpu`.
- If embedding or model calls fail, double-check API keys and network access.

<h2><span style="color:#2ca02c">Example changes you can make next</span></h2>

- Add deterministic index file naming (e.g., hash the PDF contents) so the index path can be auto-derived.
- Add provenance/tracing in the UI showing which chunks or web sources contributed to the answer.
- Add a background worker for long-running embedding/index builds so the UI remains responsive.

<h2><span style="color:#d62728">License & contact</span></h2>

This repository does not include an explicit license file. Add a `LICENSE` (for example MIT) if you intend to publish.

For questions about this script, inspect `app.py` and open an issue or contact the repository owner.


