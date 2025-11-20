<h1><span style="color:#2b8a3e">Agentic RAG (app.py)</span></h1>

<p>This README documents the `app.py` implementation in this folder. It explains what the script does, how to configure environment variables, install dependencies, run it, and adapt it for your own Agentic Retrieval-Augmented Generation (RAG) workflows. Headings use different colors for readability.</p>

<h2><span style="color:#1f77b4">Overview</span></h2>

`app.py` implements a small Agentic RAG prototype that:

- Loads a PDF, chunks it, and builds a FAISS vector store with Hugging Face sentence-transformer embeddings.
- Uses a router function to determine whether a user query can be answered from local (PDF) content.
- If local knowledge is insufficient, spins up a web-search + scraping crew (via `crewai` tools) to fetch context from the web.
- Uses a combo of LLMs (Groq `ChatGroq` and a `crew_llm`) to decide routing and to synthesize the final answer.

This is a demonstration/prototype. The code uses several third-party LLM tool wrappers and search/scraping tools and expects API keys and a local PDF file.

<h2><span style="color:#d62728">Quick features</span></h2>

- PDF -> FAISS vector store (via `PyPDFLoader`, `RecursiveCharacterTextSplitter`, `HuggingFaceEmbeddings`).
- Local routing: `check_local_knowledge(query, context)` decides whether to use local vector DB or web scraping.
- Web search & scraping: configured agents using `crewai` and `SerperDevTool` / `ScrapeWebsiteTool`.
- Final answer generation with `ChatGroq` LLM and a Crew-managed LLM for agent tasks.

<h2><span style="color:#9467bd">Requirements</span></h2>

- Python 3.10+ recommended.
- A GPU or sufficiently large CPU if using large LLMs locally (the script references large models; some require hosted APIs).
- Environment variables (set in a `.env` file or in your environment):
  - `GROQ_API_KEY` — API key for Groq/ChatGroq usage (if required)
  - `SERPER_API_KEY` — API key for the Serper web search tool
  - `GEMINI` — API key for the Gemini LLM used by `crew_llm`

<h2><span style="color:#17becf">Python dependencies</span></h2>

The script imports from a number of libraries. A minimal set (subject to your environment and package names) includes:

- python-dotenv
- langchain
- faiss-cpu (or faiss-gpu) and/or `langchain` vectorstore bindings
- sentence-transformers
- langchain-huggingface (or `langchain_huggingface.embeddings` as used)
- crewai and crewai_tools (project-specific wrappers used in this repo)
- langchain_groq (ChatGroq wrapper)
- PyPDFLoader (from `langchain.document_loaders`)

Example pip install line (adjust versions and packages per your environment):

```bash
python -m venv .venv
source .venv/bin/activate
pip install python-dotenv langchain faiss-cpu sentence-transformers langchain-huggingface crewai crewai-tools langchain-groq
```

Note: Some packages have different names or require extra installation steps (e.g., `faiss` can be tricky on macOS). Adapt as needed.

<h2><span style="color:#ff7f0e">Configuration</span></h2>

1. Place the PDF you want to use in the same directory or update `pdf_path` in `main()` (default: `genai-principles.pdf`).
2. Create a `.env` file with the required API keys or export them in your shell, e.g.: 

```
GROQ_API_KEY=your_groq_key
SERPER_API_KEY=your_serper_key
GEMINI=your_gemini_key
```

3. Ensure you have adequate compute and API access for the LLMs referenced. The example LLM models in `app.py` are large and may require hosted endpoints or different credentials.

<h2><span style="color:#8c564b">How it works (contract)</span></h2>

- Inputs: a query (string), a vector DB (built from a PDF). The code constructs a `local_context` from the vector DB (empty string initially) and uses `process_query(query, vector_db, local_context)`.
- Outputs: a text answer (string) printed to stdout and returned by `process_query`.
- Data shapes: `vector_db` is a FAISS vector store produced by `setup_vector_db(pdf_path)`; `query` is a plain text string.
- Error modes: missing PDF file, missing API keys, dependency import errors, or network timeouts when calling web tools/LLMs.
- Success criteria: function returns a coherent string answer and prints it.

<h2><span style="color:#e377c2">Edge cases & considerations</span></h2>

- Empty PDF or improperly loaded documents will produce empty vectors and empty `local_context`.
- The route decision relies on `check_local_knowledge` which queries an LLM — that can be noisy and sometimes wrong. Consider adding a confidence threshold or fallback rules.
- Web scraping might return long raw HTML or require rate-limiting and respect for robots.txt.
- Large LLM usage may be costly or require batching, streaming, or token limits.
- FAISS and embeddings memory needs grow with document size — monitor memory usage.

<h2><span style="color:#7f7f7f">How to run</span></h2>

1. Create and activate a virtual environment.
2. Install dependencies (see section above).
3. Ensure `.env` is present and required keys are set.
4. Place `genai-principles.pdf` or adjust `pdf_path` in `main()`.
5. Run the script:

```bash
python app.py
```

You should see messages for "Setting up vector database...", retrieval source (local or web), and a printed "Final Answer:".

<h2><span style="color:#bcbd22">Notes about models & resources</span></h2>

- `ChatGroq` and the `crew_llm` use hosted models; verify your API keys and quotas.
- If you don't have access to those providers, replace the LLM instantiation with a local or other hosted model supported by your environment (e.g., `OpenAI`, `HuggingFaceHub`, or a smaller local LLM wrapper).
- The code uses `HuggingFaceEmbeddings` with `sentence-transformers/all-mpnet-base-v2`. That embedding model is small and suitable for many tasks.

<h2><span style="color:#1b9e77">Extending & next steps</span></h2>

- Add caching for web scraping results to reduce API calls and rate limits.
- Add unit tests for `check_local_knowledge`, `setup_vector_db`, and `process_query` — include a small sample PDF for fast tests.
- Add a CLI interface (argparse or Typer) to supply PDF path, query, and toggles for web search.
- Add safety limits (time limits, token budgets) to LLM calls.

<h2><span style="color:#9467bd">Troubleshooting</span></h2>

- Import errors: ensure correct package names and versions. Some packages use different distribution names.
- FAISS build problems on macOS: prefer `faiss-cpu` via conda or use a prebuilt wheel.
- API key errors: confirm keys are set and valid. The script reads env vars via `python-dotenv`'s `load_dotenv()`.

<h2><span style="color:#2ca02c">Example change: small CLI</span></h2>

You can quickly add `argparse` to allow `--pdf` and `--query` to be provided from the command line. The main logic already accepts `pdf_path` and `query` variables — refactor `main()` to parse CLI args and pass them through.

<h2><span style="color:#d62728">License & contact</span></h2>

This repository does not include an explicit license file. Add a `LICENSE` (for example MIT) if you intend to publish.

For questions about this script, inspect `app.py` and open an issue or contact the repository owner.

---

*README generated from the code in `app.py`.*
