# PDF-Bot 🦜📄

A local, privacy-first chatbot that lets you **upload a PDF and have a conversation with it** — no cloud APIs, no data leaving your machine. Built with Streamlit, LangChain, ChromaDB, and the open-source **LaMini-T5-738M** model from MBZUAI.

---

## How It Works

```
PDF Upload → Text Extraction (PDFMiner)
         → Chunking (RecursiveCharacterTextSplitter)
         → Embeddings (all-MiniLM-L6-v2)
         → Stored in ChromaDB (local vector store)
         → Query → Retrieval → LaMini-T5 generates answer
```

1. You upload a PDF through the Streamlit UI.
2. The document is split into 500-token chunks and embedded using `sentence-transformers`.
3. Embeddings are persisted locally in a ChromaDB (DuckDB + Parquet) vector store under `db/`.
4. When you ask a question, the top relevant chunks are retrieved and passed to **LaMini-T5-738M** via a LangChain `RetrievalQA` chain.
5. The answer is displayed in a chat interface alongside a live PDF preview.

---

## Tech Stack

| Layer | Library / Model |
|---|---|
| UI | Streamlit + streamlit-chat |
| PDF Parsing | PDFMiner, PyPDFLoader (LangChain) |
| Text Splitting | RecursiveCharacterTextSplitter |
| Embeddings | all-MiniLM-L6-v2 (sentence-transformers) |
| Vector Store | ChromaDB (DuckDB + Parquet backend) |
| LLM | MBZUAI/LaMini-T5-738M (HuggingFace) |
| Inference | transformers pipeline (text2text-generation) |
| Orchestration | LangChain RetrievalQA |
| Containerisation | Docker |

All inference runs on **CPU** — no GPU required.

---

## Project Structure

```
PDF-Bot/
├── chatbot_app.py      # Main Streamlit app — UI, ingestion, QA pipeline
├── ingest.py           # Standalone script to pre-ingest PDFs into ChromaDB
├── constants.py        # ChromaDB settings (DuckDB+Parquet, persist dir)
├── requirements.txt    # Python dependencies with pinned versions
├── Dockerfile          # Docker image definition (Python 3.10)
└── docs/               # Drop your PDF files here (created at runtime)
```

---

## Getting Started

### Prerequisites

- Python 3.10+
- ~4 GB of disk space (model download on first run)
- Docker (optional, for containerised setup)

---

### Option 1 — Run Locally

**1. Clone the repo**

```bash
git clone https://github.com/patilsujal0911-source/PDF-Bot.git
cd PDF-Bot
```

**2. Create a virtual environment**

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
```

**3. Install dependencies**

```bash
pip install -r requirements.txt
```

**4. Create the docs directory**

```bash
mkdir docs
```

**5. Launch the app**

```bash
streamlit run chatbot_app.py
```

Open your browser at `http://localhost:8501`, upload a PDF, and start chatting.

---

### Option 2 — Pre-ingest PDFs (optional)

If you want to load PDFs into the vector store before starting the app:

```bash
# Place your PDFs in the docs/ folder, then:
python ingest.py
```

This creates the `db/` directory with the ChromaDB vector store. The app will load directly from the existing store without re-ingesting.

---

### Option 3 — Docker

**Build the image**

```bash
docker build -t pdf-bot .
```

**Run the container**

```bash
docker run -p 8501:8501 pdf-bot
```

Access the app at `http://localhost:8501`.

> **Note:** The LaMini-T5-738M model (~1.4 GB) is downloaded from HuggingFace on first run. For Docker, consider pre-downloading it and baking it into the image to avoid repeated downloads on every container start.

---

## Usage

1. **Upload a PDF** using the file uploader in the UI.
2. Wait for the **"Embeddings are created successfully!"** confirmation.
3. Type your question in the chat input box.
4. The bot retrieves relevant context from your document and generates an answer.
5. The full PDF preview is shown on the left panel alongside the chat.

---

## Configuration

Key settings are centralised in `constants.py`:

```python
CHROMA_SETTINGS = Settings(
    chroma_db_impl='duckdb+parquet',
    persist_directory='db',
    anonymized_telemetry=False
)
```

Inference parameters (temperature, max length, etc.) can be tuned in the `llm_pipeline()` function inside `chatbot_app.py`:

```python
pipe = pipeline(
    'text2text-generation',
    model=base_model,
    tokenizer=tokenizer,
    max_length=256,
    temperature=0.3,
    top_p=0.95,
)
```

---

## Known Limitations

- **Single PDF at a time** — the ingestion loop processes the last PDF found in `docs/`; uploading multiple files simultaneously may not work as expected.
- **CPU-only inference** — responses may be slow on lower-spec machines since the model runs in `torch.float32` on CPU.
- **Heavy chunk overlap** — `chunk_size=500, chunk_overlap=500` in the app means every chunk is essentially a duplicate; consider setting overlap to ~50–100 for better retrieval precision.
- **ChromaDB version lock** — pinned to `0.3.26` which uses the older `chroma_db_impl` API; not compatible with newer ChromaDB versions without changes to `constants.py`.

---

## Dependencies (pinned)

See [`requirements.txt`](requirements.txt) for the full list. Key versions:

| Package | Version |
|---|---|
| langchain | 0.0.267 |
| streamlit | 1.25.0 |
| transformers | 4.31.0 |
| torch | 2.0.1 |
| chromadb | 0.3.26 |
| sentence-transformers | 2.2.2 |

---

## License

This project is licensed under the terms in the [LICENSE](LICENSE) file.

---

## Acknowledgements

- [LaMini-T5-738M](https://huggingface.co/MBZUAI/LaMini-T5-738M) by MBZUAI
- [LangChain](https://github.com/langchain-ai/langchain) for the RAG orchestration primitives
- [ChromaDB](https://www.trychroma.com/) for local vector storage
- [AI Anytime](https://github.com/AIAnytime) — original inspiration
