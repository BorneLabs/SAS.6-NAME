# NAME Backend for Hybrid AI Agent Prototype

Welcome to the NAME backend repository by Bornelabs. This project implements the backend for our NAME mobile app prototype, which lets users create their own AI agents by uploading documents and chatting with a custom-named agent.

We use a hybrid approach: a Kodular-built mobile app communicates with our cloud-based backend, which handles heavy AI processing using a Retrieval-Augmented Generation (RAG) method.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
  - [Running Locally](#running-locally)
  - [API Endpoints](#api-endpoints)
- [Code Explanation](#code-explanation)
  - [Environment Setup](#environment-setup)
  - [Document Upload & Preprocessing](#document-upload--preprocessing)
  - [Embedding Generation & FAISS Index](#embedding-generation--faiss-index)
  - [Query Processing (RAG Workflow)](#query-processing-rag-workflow)
- [Future Enhancements](#future-enhancements)
- [Contributing](#contributing)
- [License](#license)

## Overview

We (Bornelabs) are developing **NAME**, a mobile app that lets users create their own AI agents by simply uploading documents and chatting with the agent they’ve named. This repository contains our backend implementation built with FastAPI. Our backend:

- Accepts document uploads (plain text and PDFs)
- Extracts and preprocesses text
- Splits text into manageable chunks
- Generates vector embeddings using a pre-trained SentenceTransformer model
- Indexes embeddings with FAISS for efficient similarity search
- Processes user queries by retrieving relevant context and generating responses using a text-generation pipeline (DialoGPT-small)

## Features

- **Document Upload:** Accepts plain text and PDF files.
- **Text Processing:** Extracts text from PDFs and splits it into chunks.
- **Embeddings:** Uses SentenceTransformer (`all-MiniLM-L6-v2`) to generate vector embeddings.
- **Similarity Search:** Implements FAISS for fast similarity search.
- **Query Processing:** Processes queries via a RAG-like workflow, combining retrieved context with the query to generate responses using DialoGPT-small.
- **Open-Source:** All components are free and open-source.

## Architecture

Our backend leverages:
- **FastAPI** for the RESTful API.
- **Uvicorn** as the ASGI server.
- **SentenceTransformer** for text embeddings.
- **FAISS** for vector indexing.
- **Transformers (DialoGPT-small)** for text generation.
- **PyPDF2** for PDF text extraction.

## Requirements

- Python 3.8 or later
- Dependencies:
  - `fastapi`
  - `uvicorn[standard]`
  - `sentence-transformers`
  - `faiss-cpu`
  - `transformers`
  - `PyPDF2`
  - `numpy`
  - `re`

## Installation

1. **Clone the Repository:**

   ```bash
   git clone https://github.com/yourusername/NAME-backend.git
   cd NAME-backend
   ```

2. **Create and Activate a Virtual Environment:**

   ```bash
   python -m venv env
   source env/bin/activate  # On Windows: env\Scripts\activate
   ```

3. **Install Dependencies:**

   ```bash
   pip install fastapi uvicorn[standard] sentence-transformers faiss-cpu transformers PyPDF2 numpy
   ```

## Usage

### Running Locally

To run our FastAPI application locally, execute:

```bash
uvicorn main:app --reload
```

This will start the server at `http://127.0.0.1:8000`.

### API Endpoints

- **POST `/upload`**  
  **Description:** Upload a document (plain text or PDF).  
  **Response:** JSON indicating the number of text chunks processed.

- **POST `/query`**  
  **Description:** Process a user query by retrieving relevant context from uploaded documents and generating a response.  
  **Request:** JSON payload with a `query` field (string).  
  **Response:** JSON containing the generated response and the list of retrieved text chunks.

## Code Explanation

### Environment Setup

We initialize our FastAPI app, load the SentenceTransformer model, and set up global variables to store document chunks, embeddings, and the FAISS index.

```python
from fastapi import FastAPI, File, UploadFile, HTTPException
from pydantic import BaseModel
import uvicorn
import io, re
import numpy as np

from sentence_transformers import SentenceTransformer
import faiss
from transformers import pipeline
from PyPDF2 import PdfReader

app = FastAPI(title="NAME Backend API", description="Backend for our hybrid NAME app", version="0.1")

# Load our embedding model
we_embedding_model = SentenceTransformer('all-MiniLM-L6-v2')

# Global variables for storing text chunks, embeddings, and FAISS index
we_document_chunks = []       # List of text chunks
we_document_embeddings = None # Numpy array of embeddings
we_faiss_index = None         # FAISS index

# Load text-generation pipeline using DialoGPT-small
we_generator = pipeline("text-generation", model="microsoft/DialoGPT-small")
```

### Document Upload & Preprocessing

We define a helper function to split text into manageable chunks, then create the `/upload` endpoint. This endpoint accepts plain text and PDF files, extracts text, splits it into chunks, generates embeddings, and updates our FAISS index.

```python
def split_text_into_chunks(text: str, max_length: int = 500):
    """
    We split the text into chunks of at most `max_length` characters.
    """
    sentences = re.split(r'(?<=[.!?])\s+', text)
    chunks = []
    current_chunk = ""
    for sentence in sentences:
        if len(current_chunk) + len(sentence) + 1 <= max_length:
            current_chunk = f"{current_chunk} {sentence}".strip() if current_chunk else sentence
        else:
            chunks.append(current_chunk)
            current_chunk = sentence
    if current_chunk:
        chunks.append(current_chunk)
    return chunks
```

```python
@app.post("/upload")
async def upload_document(file: UploadFile = File(...)):
    """
    We handle document uploads by accepting 'text/plain' and 'application/pdf' files.
    For PDFs, we extract text using PyPDF2. Then we split the text into chunks,
    generate embeddings using our SentenceTransformer model, and update our FAISS index.
    """
    if file.content_type not in ["text/plain", "application/pdf"]:
        raise HTTPException(status_code=400, detail="Invalid file type. Only plain text or PDF files are accepted.")
    
    content = await file.read()
    if file.content_type == "text/plain":
        text = content.decode("utf-8")
    elif file.content_type == "application/pdf":
        pdf_reader = PdfReader(io.BytesIO(content))
        text = ""
        for page in pdf_reader.pages:
            page_text = page.extract_text()
            if page_text:
                text += page_text + "\n"
    if not text.strip():
        raise HTTPException(status_code=400, detail="No extractable text found in the file.")
    
    # Split text into manageable chunks
    chunks = split_text_into_chunks(text)
    
    # Generate embeddings for each chunk
    embeddings = we_embedding_model.encode(chunks)
    
    # Update our global storage
    global we_document_chunks, we_document_embeddings, we_faiss_index
    we_document_chunks.extend(chunks)
    embeddings_np = np.array(embeddings).astype('float32')
    if we_document_embeddings is None:
        we_document_embeddings = embeddings_np
    else:
        we_document_embeddings = np.vstack((we_document_embeddings, embeddings_np))
    
    # Initialize or update the FAISS index
    dim = we_document_embeddings.shape[1]
    if we_faiss_index is None:
        we_faiss_index = faiss.IndexFlatL2(dim)
        we_faiss_index.add(we_document_embeddings)
    else:
        we_faiss_index.add(embeddings_np)
    
    return {"message": f"Successfully uploaded document and processed {len(chunks)} text chunks."}
```

### Embedding Generation & FAISS Index

In the upload endpoint, we generate vector embeddings for each text chunk using our SentenceTransformer model and update our FAISS index, allowing for fast similarity searches when processing user queries.

### Query Processing (RAG Workflow)

Our `/query` endpoint processes user queries by generating an embedding for the query, searching our FAISS index for the top similar text chunks, and generating a response using our text-generation pipeline.

```python
class QueryRequest(BaseModel):
    query: str

@app.post("/query")
async def process_query(query_req: QueryRequest):
    """
    We process the user query by:
    1. Generating an embedding for the query.
    2. Searching our FAISS index for the top similar text chunks.
    3. Combining retrieved context with the query.
    4. Generating a response using our text-generation pipeline.
    """
    global we_document_chunks, we_document_embeddings, we_faiss_index
    if we_faiss_index is None or len(we_document_chunks) == 0:
        raise HTTPException(status_code=400, detail="No documents have been uploaded yet.")
    
    # Generate an embedding for the query
    query_embedding = we_embedding_model.encode([query_req.query]).astype('float32')
    
    # Retrieve top 5 similar text chunks
    k = 5
    distances, indices = we_faiss_index.search(query_embedding, k)
    retrieved_chunks = [we_document_chunks[i] for i in indices[0] if i < len(we_document_chunks)]
    
    # Combine the retrieved context with the query
    context = " ".join(retrieved_chunks)
    prompt = f"Based on the context: {context}\nAnswer the query: {query_req.query}\nResponse:"
    
    # Generate the response using our text-generation pipeline
    generated = we_generator(prompt, max_length=200, do_sample=True)
    response_text = generated[0]['generated_text']
    
    return {"response": response_text, "retrieved_context": retrieved_chunks}
```

### Running Our Application

We run our FastAPI application using Uvicorn:

```python
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Future Enhancements

- **Fine-Tuning:** Integrate fine-tuning for personalized responses.
- **Improved Security & Logging:** Add authentication, detailed error logging, and HTTPS.
- **Scalability:** Explore other vector databases (e.g., Chroma, Weaviate) as our dataset grows.
- **Persistence:** Implement persistent storage for the FAISS index and document metadata.

## Contributing

We welcome contributions from the community! Please fork this repository, make your changes, and submit a pull request. For major changes, open an issue first to discuss your ideas.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

**The Bornelabs Team**
