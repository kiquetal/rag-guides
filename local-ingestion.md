# Local Ingestion for RAG

As described in [first-step-rag.md](first-step-rag.md), the goal is to build a system that can answer questions using your local code and documentation files (like Markdown) as a knowledge base.

The first step is to process your local repository and store it in a way that a RAG system can understand. This is called "ingestion."

### Local Ingestion Flow

The process works by running a Python script (`ingest-data.py`) on your local machine. This script reads your Git repository, including all code files, markdown files, and commit history, and sends it to a vector database.

Here is a diagram of the ingestion process:

```ascii
+------------------+      +------------------+      +-----------------+
|                  |      |                  |      |                 |
|  Local Git Repo  |----->|  ingest-data.py  |----->|  Vector Store   |
| (files, commits) |      |  (on your machine) |      | (e.g. Pinecone) |
|                  |      |                  |      |                 |
+------------------+      +------------------+      +-----------------+
```

The `ingest-data.py` script:
1.  Scans the files and commit messages in your current directory.
2.  Adds metadata to each piece of data (e.g., marking it as a `file` or a `commit`).
3.  Uses an embedding model to convert the text into vectors.
4.  Uploads these vectors and their metadata to a cloud-based vector store.
