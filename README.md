<!-- Agentic RAG — README generated from app.py -->

# Agentic RAG (app.py) — With Streamlit UI

This project demonstrates a small Retrieval-Augmented Generation (RAG) pipeline that can run as a CLI or as a Streamlit app. It
builds a FAISS vector store from an uploaded PDF (or local PDF), uses an embedding model for semantic search, routes queries
to local knowledge or web-scraped content, and synthesizes answers with an LLM.

The single source of truth is `app.py` which contains both frontends (CLI + Streamlit) and the RAG pipeline.

## Quick overview

- Input sources: local PDF (primary) and optional web search/scraping when local knowledge is insufficient.
- Pipeline: PDF -> chunking -> embeddings -> FAISS similarity search -> router decides local vs web -> LLM answer synthesis.
- Frontends: CLI and Streamlit UI share the same processing functions.

## What this README documents

- How to configure env vars and dependencies
- How to run the CLI and Streamlit UI
- How FAISS index persistence works (save/load)
- Troubleshooting tips (faiss on macOS, dependency issues)

## Requirements & configuration

- Python: 3.10+ recommended (3.11/3.12 are fine).
- Required environment variables (create a `.env` or export in shell):

```bash
GROQ_API_KEY=your_groq_key
SERPER_API_KEY=your_serper_key
GEMINI_API_KEY=your_gemini_key
```

- The code uses an `EMBEDDING_MODEL` constant. Current default in `app.py`:

```
sentence-transformers/all-mpnet-base-v2
```

## Dependencies

See `requirements.txt` in this folder. Key libraries used by `app.py` include:

- langchain-community FAISS wrapper (`langchain_community.vectorstores.FAISS`)
- `PyPDFLoader` (PDF loading)
- `RecursiveCharacterTextSplitter` (chunking)
- `HuggingFaceEmbeddings` (embeddings)
- `ChatGroq` (Groq LLM wrapper)
- `crewai` / `crewai_tools` (web search & scraping agents)
- `google.generativeai` (Gemini client used by crew)
- `streamlit` (for the UI)

If you hit install problems for FAISS on macOS, see Troubleshooting below.

## Running the app

All examples assume you run commands from the `AIAgenticRAGWithUI` folder.

### CLI mode

The CLI is the non-Streamlit branch inside `app.py`. It supports building/loading a FAISS index and running a single query.

Example:

```bash
python app.py \
  --pdf path/to/your.pdf \
  --query "Summarize agentic RAG" \
  --save-index \
  --index-path my_faiss_index
```

CLI flags (as implemented in `app.py`):

- `--pdf` : Path to PDF to index (default `Agent Quality.pdf`).
- `--query` : Query string to ask the agent.
- `--no-web` : Disable web search/scraping; only local knowledge will be used.
- `--save-index` : Save the built FAISS index to disk.
- `--index-path` : Directory path to save/load the FAISS index (default `faiss_index`).

Behavior notes:

- On start, the code attempts to load an existing FAISS index from `--index-path`. If loading fails or the path doesn't exist, it will try to build from the provided `--pdf`.
- If `--no-web` is passed and the router decides local knowledge is insufficient, the program exits with a message.

### Streamlit UI

Run the Streamlit UI with:

```bash
streamlit run app.py
```

UI features:

- Upload a PDF to build a local vector DB (or set an index path to reuse a previously saved index).
- Enter a query in the text input and press Ask.
- Toggle "Allow web search" to enable/disable web scraping when local knowledge is insufficient.
- Optionally save the FAISS index to the given index path so subsequent runs can load it instead of rebuilding.

Implementation details:

- The Streamlit branch uses `st.cache_resource` to cache loading/building the vector DB during a session.
- Uploaded PDFs are saved to a temporary file and used to build the index via `setup_vector_db`.

## How the pipeline works (contract)

Inputs:

- PDF file (path or uploaded stream)
- Query string
- Optional: index path (existing FAISS index directory)

Outputs:

- Human-readable answer text from the LLM printed to stdout (CLI) or shown in the Streamlit UI.

Success criteria:

- For queries answerable from the PDF, the system should return an accurate, context-grounded answer without calling the web scraper.
- For queries not found locally, and if web search is enabled, the system should fetch web content, then synthesize an answer.

Error modes:

- Missing or invalid PDF -> index build fails and the program prints an error.
- FAISS load/save errors -> index rebuild fallback executed (if PDF available) or readable error.

## Important functions (where to look in `app.py`)

- `setup_vector_db(pdf_path)` : load PDF, chunk, create embeddings, build FAISS index.
- `get_local_content(vector_db, query)` : similarity search to gather local context.
- `check_local_knowledge(query, context)` : router using LLM to decide if local content suffices (returns Yes/No).
- `get_web_content(query)` : runs the crew (SerperDevTool + ScrapeWebsiteTool) to fetch web content.
- `process_query(query, vector_db, local_context)` : orchestration that chooses local vs web and calls the LLM for final answer.

## Index persistence

- The code uses `FAISS.save_local(index_path)` and `FAISS.load_local(index_path, embeddings)` to persist and reload indices.
- Important: always use the same `EMBEDDING_MODEL` when saving and loading an index.

### Example index workflow

1. Build index from `my.pdf` and save:

```bash
python app.py --pdf my.pdf --save-index --index-path my_faiss_index
```

2. Later, load existing index and run queries:

```bash
python app.py --index-path my_faiss_index --query "Explain X"
```

## Troubleshooting

- FAISS installation on macOS: `faiss` can be difficult to pip-install on macOS/ARM. Recommended options:
  - Use conda/miniforge and `conda install -c conda-forge faiss-cpu`
  - Or install a compatible pre-built wheel for your Python version/arch
  - Alternatively run on x86_64 environment or Docker image that bundles faiss

- If `pip install -r requirements.txt` fails with native build errors, try:

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip setuptools wheel
# prefer conda for faiss or install faiss via conda
```

- Missing API keys: ensure `GROQ_API_KEY`, `SERPER_API_KEY`, and `GEMINI_API_KEY` are set. The Streamlit UI and crew tools rely on these for web queries and some LLMs.

## Security & privacy notes

- Uploaded PDFs may be temporarily stored on disk in a temporary file by the Streamlit flow. Remove any sensitive documents or adapt the code to encrypt/remove files after processing.
- When enabling web search, the query is sent to external services and scraped content may include sensitive material. Review privacy policies for the services used (Serper/Gemini/Groq).

## Developer notes & next steps

- Pin dependency versions in `requirements.txt` to avoid installation surprises across platforms.
- Add a small sample PDF and a pytest that builds an index and runs a short query to verify the end-to-end pipeline.
- Add provenance in the UI: show which chunks/documents contributed to the answer (for traceability).
- Consider moving web scraping to an asynchronous background worker if large or slow operations are frequent.

## Where to look next

- Main code: `app.py` (both CLI and Streamlit logic live here).
- Edit embeddings model: change `EMBEDDING_MODEL` constant in `app.py` if you want a different model.
- Dependency list: `requirements.txt` in this folder.

---

If you want, I can now:

1. Pin the dependencies in `requirements.txt` to a tested set of versions.
2. Add a minimal test harness + sample PDF to exercise the pipeline.
3. Diagnose any `pip install` errors you saw (paste the pip output) and propose a platform-specific fix.

Pick one and I'll proceed.

---

For questions, inspect `app.py` or open an issue.
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
<h1><span style="color:#2b8a3e">Agentic RAG (app.py) — With Streamlit UI</span></h1>

This README documents the `app.py` implementation in the `AIAgenticRAGWithUI` folder. It explains how to use the CLI and Streamlit UI, FAISS index persistence, and updated requirements.

<h2><span style="color:#1f77b4">Overview</span></h2>

Two run modes are supported:

- CLI mode: run with `python app.py` and flags to build/load a FAISS index and run a query.
- Streamlit UI: run with `streamlit run app.py` to upload a PDF, enter a query, and view answers interactively.

Both modes use the same pipeline: PDF -> chunk -> embeddings -> FAISS vector store -> routing -> LLM answer generation (local or web).

<h2><span style="color:#d62728">Key features</span></h2>

- Streamlit UI: PDF upload, query input, optional load/save FAISS index, and caching via `st.cache_resource`.
- CLI flags: `--pdf`, `--query`, `--no-web`, `--save-index`, `--index-path`.
- FAISS index persistence: save/load FAISS index on disk to skip rebuilding.
- Stable embedding model: `EMBEDDING_MODEL` constant ensures saved indexes remain compatible.

<h2><span style="color:#9467bd">Requirements</span></h2>

- Python 3.10+ (3.11/3.12 recommended).
- Environment variables (in `.env` or your shell): `GROQ_API_KEY`, `SERPER_API_KEY`, `GEMINI_API_KEY`.

<h2><span style="color:#17becf">Dependencies</span></h2>

See `AIAgenticRAGWithUI/requirements.txt`. Important packages include:

- python-dotenv
- langchain or langchain-community variants
- faiss-cpu (or faiss-gpu)
- sentence-transformers
- langchain-huggingface
- langchain-groq
- crewai, crewai-tools
- streamlit==1.26.0
- google-generativeai==0.3.0

Install example:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r AIAgenticRAGWithUI/requirements.txt
```

<h2><span style="color:#ff7f0e">Configuration</span></h2>

1. Create a `.env` file (or export env vars):

```
GROQ_API_KEY=your_groq_key
SERPER_API_KEY=your_serper_key
GEMINI_API_KEY=your_gemini_key
```

2. Choose CLI or Streamlit. Provide an `--index-path` (CLI) or index path in the UI to persist/reuse FAISS indexes.

<h2><span style="color:#8c564b">How to run</span></h2>

CLI example:

```bash
python AIAgenticRAGWithUI/app.py \
  --pdf path/to/your.pdf \
  --query "Summarize agentic RAG" \
  --save-index \
  --index-path my_faiss_index
```

Streamlit UI:

```bash
streamlit run AIAgenticRAGWithUI/app.py
```

Upload a PDF, optionally set an index path, enter a query, and click Ask.

<h2><span style="color:#bcbd22">Index persistence details</span></h2>

- The app uses `FAISS.load_local(index_path, embeddings)` to load an existing index and `vector_db.save_local(index_path)` to save.
- Keep `EMBEDDING_MODEL` consistent when saving/loading.

<h2><span style="color:#1b9e77">Troubleshooting</span></h2>

- Streamlit import errors: install `streamlit` in your active venv.
- FAISS build issues on macOS: prefer conda + `faiss-cpu` or use a prebuilt wheel.
- Index API differences: adapt `save_local`/`load_local` for your LangChain version.

<h2><span style="color:#2ca02c">Next steps / ideas</span></h2>

- Derive deterministic index filenames from PDF content hashes.
- Add provenance tracing to show which chunks/web sources contributed to an answer.
- Run index builds in a background worker for UI responsiveness.

<h2><span style="color:#d62728">License & contact</span></h2>

Add a `LICENSE` file if you plan to publish. For questions, inspect `app.py` or open an issue.
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


