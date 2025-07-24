# Guide: Adding Metadata Filtering to Your RAG System

This guide explains how to upgrade your local RAG (Retrieval-Augmented Generation) system to filter searches by data source (e.g., searching only within files or only within commit history). This gives you precise control over the context used to answer your questions.

---

### **The Goal: Precise Context Control**

By default, a simple RAG system searches its entire vector database. This can be problematic if you want to ask a question specifically about documentation (like a `README.md` file) but the most similar-sounding text comes from a commit message.

By adding metadata, you can explicitly tell your query script: "Only search for answers within my code files" or "Only look at my commit history."

---

### **Step 1: Update `ingest-data.py` to Add Metadata**

First, we'll modify the ingestion script to add a `source_type` tag to every piece of data it sends to the vector database. This tag will be either `"file"` or `"commit"`.

Replace the content of your `ingest-data.py` with this updated version.

```python
# Updated ingest-data.py

import os
import git
from dotenv import load_dotenv
from pinecone import Pinecone
import google.generativeai as genai
from tqdm import tqdm

# --- 1. Configuration & Initialization ---
print("Loading environment variables and initializing services...")
load_dotenv()
PINECONE_API_KEY = os.environ.get("PINECONE_API_KEY")
GOOGLE_API_KEY = os.environ.get("GOOGLE_API_KEY")
PINECONE_INDEX_NAME = os.environ.get("PINECONE_INDEX_NAME")

if not all([PINECONE_API_KEY, GOOGLE_API_KEY, PINECONE_INDEX_NAME]):
    print("Error: Missing one or more required environment variables.")
    exit()

EMBEDDING_MODEL = "models/embedding-001"

# Initialize services
pc = Pinecone(api_key=PINECONE_API_KEY)
genai.configure(api_api_key=GOOGLE_API_KEY)
index = pc.Index(PINECONE_INDEX_NAME)
print("Services initialized.")

# --- 2. Data Gathering (with Metadata) ---
def get_repository_data(repo_path='.'):
    """Gathers data and adds source_type metadata."""
    print(f"Gathering data from repository at: {os.path.abspath(repo_path)}")
    repo = git.Repo(repo_path)
    
    all_data = []

    # A. Gather commit logs
    commits = list(repo.iter_commits('main'))
    for commit in tqdm(commits, desc="Processing Commits"):
        commit_text = (
            f"Commit: {commit.hexsha}\n"
            f"Author: {commit.author.name}\n"
            f"Date: {commit.committed_datetime}\n"
            f"Message: {commit.message}"
        )
        # Add metadata for filtering
        all_data.append({
            "id": f"commit-{commit.hexsha}",
            "text": commit_text,
            "metadata": {"source_type": "commit", "text": commit_text}
        })

    # B. Gather file contents
    for item in tqdm(repo.tree().traverse(), desc="Processing Files"):
        if item.type == 'blob':
            try:
                file_content = item.data_stream.read().decode('utf-8')
                file_text = f"File: {item.path}\n\n{file_content}"
                # Add metadata for filtering
                all_data.append({
                    "id": f"file-{item.path}",
                    "text": file_text,
                    "metadata": {"source_type": "file", "path": item.path, "text": file_text}
                })
            except (UnicodeDecodeError, git.exc.GitCommandError):
                continue
                
    print(f"Found {len(all_data)} total data chunks to process.")
    return all_data

# --- 3. Embedding and Upserting (with Metadata) ---
def create_embeddings_and_upsert(data):
    """Creates embeddings and upserts data with metadata into Pinecone."""
    print("Creating embeddings and upserting to Pinecone...")
    batch_size = 100

    for i in tqdm(range(0, len(data), batch_size), desc="Upserting Batches"):
        batch = data[i:i + batch_size]
        
        texts_to_embed = [item['text'] for item in batch]
        
        result = genai.embed_content(
            model=EMBEDDING_MODEL,
            content=texts_to_embed,
            task_type="retrieval_document"
        )
        embeddings = result['embedding']
        
        vectors_to_upsert = []
        for j, item in enumerate(batch):
            # The metadata object is now taken directly from the data
            vectors_to_upsert.append((item['id'], embeddings[j], item['metadata']))
            
        index.upsert(vectors=vectors_to_upsert)

# --- Main Execution ---
if __name__ == "__main__":
    print("Starting RAG ingestion process...")
    print("Clearing out the index before starting to avoid duplicates...")
    index.delete(delete_all=True)
    
    repo_data = get_repository_data()
    if repo_data:
        create_embeddings_and_upsert(repo_data)
        print("\n--- Ingestion Complete! ---")
        print("Index stats:")
        print(index.describe_index_stats())
    else:
        print("No data found to ingest.")
```

**IMPORTANT:** After updating the file, you must run it again to re-process your repository. This will upload the new data with the required metadata tags.

```bash
python ingest-data.py
```

---

### **Step 2: Update `ask-question.py` to Allow Filtering**

Next, replace `ask-question.py` with this upgraded version. It uses Python's `argparse` library to accept a new `--source` command-line argument and builds a filter for the Pinecone query based on its value.

```python
# Updated ask-question.py

import os
import sys
import argparse
from dotenv import load_dotenv
from pinecone import Pinecone
import google.generativeai as genai

# --- 1. Configuration & Initialization ---
load_dotenv()
PINECONE_API_KEY = os.environ.get("PINECONE_API_KEY")
GOOGLE_API_KEY = os.environ.get("GOOGLE_API_KEY")
PINECONE_INDEX_NAME = os.environ.get("PINECONE_INDEX_NAME")

if not all([PINECONE_API_KEY, GOOGLE_API_KEY, PINECONE_INDEX_NAME]):
    print("Error: Missing one or more required environment variables.")
    exit()

EMBEDDING_MODEL = "models/embedding-001"
GENERATION_MODEL = "gemini-1.5-flash"

# Initialize services
pc = Pinecone(api_key=PINECONE_API_KEY)
genai.configure(api_key=GOOGLE_API_KEY)
index = pc.Index(PINECONE_INDEX_NAME)
llm = genai.GenerativeModel(GENERATION_MODEL)

# --- 2. The RAG Query Function (with Filtering) ---
def answer_question(query, source_filter=None):
    """Finds context from Pinecone, applying a filter if provided."""
    
    query_embedding = genai.embed_content(
        model=EMBEDDING_MODEL,
        content=query,
        task_type="retrieval_query"
    )['embedding']
    
    # Build the query arguments
    query_args = {
        "vector": query_embedding,
        "top_k": 7,
        "include_metadata": True
    }
    
    # Add the metadata filter if the user specified one
    if source_filter:
        query_args["filter"] = {"source_type": {"$eq": source_filter}}
        print(f"Applying filter: searching only in source_type '{source_filter}'")

    query_results = index.query(**query_args)
    
    context = ""
    if not query_results['matches']:
        context = "No relevant context found in the specified source."
    else:
        for match in query_results['matches']:
            context += match['metadata']['text'] + "\n\n---\n\n"
        
    prompt = f"""
    Based on the following context from my code repository, please answer the question.
    The context may contain file contents and git commit messages.
    
    CONTEXT:
    {context}
    
    QUESTION:
    {query}
    
    ANSWER:
    """
    
    print("\nGenerating answer...")
    response = llm.generate_content(prompt)
    
    return response.text

# --- Main Execution ---
if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Ask questions about your codebase.")
    parser.add_argument("query", type=str, help="The question you want to ask.")
    parser.add_argument("--source", type=str, choices=['file', 'commit'], help="Optional: Filter the search to a specific source ('file' or 'commit').")
    
    args = parser.parse_args()
    
    print(f"Querying with: '{args.query}'")
    
    final_answer = answer_question(args.query, args.source)
    print("\n--- Answer ---")
    print(final_answer)
```

---

### **Step 3: How to Use Your New Filterable RAG System**

You can now run queries with much more control.

#### 1. Search Only in Files (including `.md` files)
This is the perfect way to ask about your PKI documentation.

```bash
python ask-question.py "how do I create a pki for tls in istio" --source=file
```

#### 2. Search Only in Commit History
Useful for questions about project history, bug fixes, or feature timelines.

```bash
python ask-question.py "what was the last bug fix related to the API" --source=commit
```

#### 3. Search Everything (Default Behavior)
If you don't provide a `--source` flag, it will search both files and commits, just as it did before.

```bash
python ask-question.py "tell me everything about the authentication system"
```