# 🔧 Implementação RAG com Python

> Guia prático completo para implementar um sistema RAG para documentação usando Python

---

## 🎯 Visão Geral

Este guia apresenta uma implementação **completa e funcional** de um sistema RAG (Retrieval-Augmented Generation) especificamente otimizado para documentação técnica, usando Python e as melhores bibliotecas disponíveis.

### 🏗️ Arquitetura da Implementação

```python
# Estrutura do projeto RAG
project_structure = """
doc_rag_system/
├── src/
│   ├── core/
│   │   ├── embeddings.py      # Geração de embeddings
│   │   ├── vector_store.py    # Gerenciamento vector DB
│   │   ├── retriever.py       # Sistema de busca
│   │   └── generator.py       # Geração de respostas
│   ├── agents/
│   │   ├── content_agent.py   # Agente de conteúdo
│   │   └── quality_agent.py   # Agente de validação
│   ├── utils/
│   │   ├── document_loader.py # Carregamento de docs
│   │   ├── preprocessor.py    # Pré-processamento
│   │   └── evaluator.py       # Avaliação de qualidade
│   └── api/
│       └── rag_api.py         # API REST
├── config/
│   └── settings.py            # Configurações
├── data/
│   ├── raw/                   # Documentos originais
│   ├── processed/             # Documentos processados
│   └── embeddings/            # Cache de embeddings
├── tests/
│   └── test_rag.py           # Testes automatizados
└── requirements.txt           # Dependências
"""
```

---

## 📦 Setup e Dependências

### 🛠️ Instalação de Dependências

```bash
# requirements.txt
# Core RAG Libraries
langchain==0.1.0
langchain-openai==0.0.5
langchain-community==0.0.12

# Vector Databases
pinecone-client==3.0.0
chromadb==0.4.22
faiss-cpu==1.7.4

# Document Processing
pypdf==3.17.4
python-docx==1.1.0
markdown==3.5.2
beautifulsoup4==4.12.2

# ML & NLP
openai==1.12.0
sentence-transformers==2.2.2
numpy==1.24.3
pandas==2.1.4

# API & Web
fastapi==0.108.0
uvicorn==0.25.0
streamlit==1.29.0

# Utilities
python-dotenv==1.0.0
pydantic==2.5.0
loguru==0.7.2
tqdm==4.66.1

# Development & Testing
pytest==7.4.3
black==23.12.1
flake8==7.0.0
```

### ⚙️ Configuração Inicial

```python
# config/settings.py
from pydantic import BaseSettings
from typing import Optional
import os

class Settings(BaseSettings):
    """Configurações do sistema RAG"""
    
    # API Keys
    OPENAI_API_KEY: str = os.getenv("OPENAI_API_KEY")
    PINECONE_API_KEY: Optional[str] = os.getenv("PINECONE_API_KEY")
    PINECONE_ENVIRONMENT: Optional[str] = os.getenv("PINECONE_ENVIRONMENT")
    
    # Model Settings
    EMBEDDING_MODEL: str = "text-embedding-ada-002"
    LLM_MODEL: str = "gpt-4"
    
    # Vector Database Settings
    VECTOR_DB_TYPE: str = "chromadb"  # chromadb, pinecone, faiss
    COLLECTION_NAME: str = "documentation"
    
    # Chunking Settings
    CHUNK_SIZE: int = 1000
    CHUNK_OVERLAP: int = 200
    
    # Retrieval Settings
    RETRIEVAL_TOP_K: int = 5
    SIMILARITY_THRESHOLD: float = 0.7
    
    # Generation Settings
    MAX_TOKENS: int = 1000
    TEMPERATURE: float = 0.1
    
    # Paths
    DATA_DIR: str = "data"
    EMBEDDINGS_CACHE_DIR: str = "data/embeddings"
    LOG_LEVEL: str = "INFO"
    
    class Config:
        env_file = ".env"

settings = Settings()
```

---

## 📚 Document Processing Pipeline

### 📥 Document Loader

```python
# src/utils/document_loader.py
from langchain.document_loaders import (
    DirectoryLoader,
    MarkdownLoader,
    PyPDFLoader,
    Docx2txtLoader,
    TextLoader
)
from langchain.schema import Document
from typing import List, Dict, Any
from pathlib import Path
import logging

logger = logging.getLogger(__name__)

class DocumentationLoader:
    """Carregador de documentação com suporte a múltiplos formatos"""
    
    def __init__(self):
        self.loaders = {
            '.md': MarkdownLoader,
            '.pdf': PyPDFLoader,
            '.docx': Docx2txtLoader,
            '.txt': TextLoader
        }
    
    def load_documents(self, directory: str) -> List[Document]:
        """Carrega todos os documentos de um diretório"""
        
        documents = []
        directory_path = Path(directory)
        
        if not directory_path.exists():
            raise FileNotFoundError(f"Directory {directory} not found")
        
        for file_path in directory_path.rglob("*"):
            if file_path.is_file() and file_path.suffix in self.loaders:
                try:
                    loader_class = self.loaders[file_path.suffix]
                    loader = loader_class(str(file_path))
                    file_documents = loader.load()
                    
                    # Enriquece metadados
                    for doc in file_documents:
                        doc.metadata.update({
                            'source_type': file_path.suffix,
                            'file_name': file_path.name,
                            'file_path': str(file_path),
                            'last_modified': file_path.stat().st_mtime
                        })
                    
                    documents.extend(file_documents)
                    logger.info(f"Loaded {len(file_documents)} documents from {file_path}")
                    
                except Exception as e:
                    logger.error(f"Error loading {file_path}: {e}")
        
        logger.info(f"Total documents loaded: {len(documents)}")
        return documents
    
    def load_api_documentation(self, openapi_spec: Dict[str, Any]) -> List[Document]:
        """Carrega documentação de APIs a partir de especificação OpenAPI"""
        
        documents = []
        
        # Processa endpoints
        for path, methods in openapi_spec.get('paths', {}).items():
            for method, spec in methods.items():
                content = self._format_api_endpoint(path, method, spec)
                
                doc = Document(
                    page_content=content,
                    metadata={
                        'source_type': 'api',
                        'endpoint': path,
                        'method': method.upper(),
                        'tags': spec.get('tags', []),
                        'summary': spec.get('summary', ''),
                        'deprecated': spec.get('deprecated', False)
                    }
                )
                documents.append(doc)
        
        # Processa schemas/models
        for name, schema in openapi_spec.get('components', {}).get('schemas', {}).items():
            content = self._format_api_schema(name, schema)
            
            doc = Document(
                page_content=content,
                metadata={
                    'source_type': 'schema',
                    'schema_name': name,
                    'type': schema.get('type', 'object')
                }
            )
            documents.append(doc)
        
        return documents
    
    def _format_api_endpoint(self, path: str, method: str, spec: Dict) -> str:
        """Formata endpoint da API para documentação"""
        
        content = f"""
# {method.upper()} {path}

## Summary
{spec.get('summary', 'No summary available')}

## Description  
{spec.get('description', 'No description available')}

## Parameters
"""
        
        # Adiciona parâmetros
        for param in spec.get('parameters', []):
            content += f"- **{param['name']}** ({param.get('in', 'query')}): {param.get('description', '')}\n"
        
        # Adiciona exemplos de resposta
        for status, response in spec.get('responses', {}).items():
            content += f"\n## Response {status}\n{response.get('description', '')}\n"
        
        return content
    
    def _format_api_schema(self, name: str, schema: Dict) -> str:
        """Formata schema da API para documentação"""
        
        content = f"""
# Schema: {name}

## Type
{schema.get('type', 'object')}

## Description
{schema.get('description', 'No description available')}

## Properties
"""
        
        for prop_name, prop_spec in schema.get('properties', {}).items():
            content += f"- **{prop_name}** ({prop_spec.get('type', 'unknown')}): {prop_spec.get('description', '')}\n"
        
        return content
```

### 🔪 Text Preprocessing

```python
# src/utils/preprocessor.py
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.schema import Document
from typing import List, Dict, Any
import re
import hashlib

class DocumentPreprocessor:
    """Pré-processador de documentos para RAG"""
    
    def __init__(self, chunk_size: int = 1000, chunk_overlap: int = 200):
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=chunk_size,
            chunk_overlap=chunk_overlap,
            separators=["\n\n", "\n", ".", "!", "?", ",", " ", ""],
            keep_separator=True
        )
    
    def process_documents(self, documents: List[Document]) -> List[Document]:
        """Processa documentos: limpeza + chunking + enriquecimento"""
        
        processed_docs = []
        
        for doc in documents:
            # Limpeza do texto
            cleaned_content = self._clean_text(doc.page_content)
            
            # Extração de metadados adicionais
            enhanced_metadata = self._extract_metadata(cleaned_content, doc.metadata)
            
            # Chunking
            chunks = self.text_splitter.split_text(cleaned_content)
            
            for i, chunk in enumerate(chunks):
                if len(chunk.strip()) < 50:  # Skip chunks muito pequenos
                    continue
                
                chunk_metadata = enhanced_metadata.copy()
                chunk_metadata.update({
                    'chunk_id': f"{self._generate_doc_id(doc)}_{i}",
                    'chunk_index': i,
                    'total_chunks': len(chunks),
                    'chunk_hash': hashlib.md5(chunk.encode()).hexdigest()
                })
                
                processed_doc = Document(
                    page_content=chunk,
                    metadata=chunk_metadata
                )
                processed_docs.append(processed_doc)
        
        return processed_docs
    
    def _clean_text(self, text: str) -> str:
        """Limpeza básica do texto"""
        
        # Remove caracteres especiais de controle
        text = re.sub(r'[\x00-\x08\x0b\x0c\x0e-\x1f\x7f-\x9f]', '', text)
        
        # Normaliza espaços em branco
        text = re.sub(r'\s+', ' ', text)
        
        # Remove linhas vazias excessivas
        text = re.sub(r'\n\s*\n\s*\n', '\n\n', text)
        
        return text.strip()
    
    def _extract_metadata(self, content: str, existing_metadata: Dict) -> Dict:
        """Extrai metadados adicionais do conteúdo"""
        
        metadata = existing_metadata.copy()
        
        # Extrai headings
        headings = re.findall(r'^#+\s+(.+)$', content, re.MULTILINE)
        if headings:
            metadata['headings'] = headings[:5]  # Primeiros 5 headings
            metadata['main_topic'] = headings[0]
        
        # Extrai código
        code_blocks = re.findall(r'```(\w+)?\n(.*?)```', content, re.DOTALL)
        if code_blocks:
            metadata['has_code'] = True
            metadata['languages'] = list(set([lang for lang, _ in code_blocks if lang]))
        
        # Extrai links
        links = re.findall(r'\[([^\]]+)\]\(([^)]+)\)', content)
        if links:
            metadata['has_links'] = True
            metadata['link_count'] = len(links)
        
        # Análise de complexidade
        metadata['word_count'] = len(content.split())
        metadata['complexity'] = self._assess_complexity(content)
        
        return metadata
    
    def _assess_complexity(self, content: str) -> str:
        """Avalia complexidade do conteúdo"""
        
        word_count = len(content.split())
        code_count = len(re.findall(r'```', content))
        link_count = len(re.findall(r'\[([^\]]+)\]\(([^)]+)\)', content))
        
        complexity_score = word_count/100 + code_count*5 + link_count*2
        
        if complexity_score < 10:
            return "basic"
        elif complexity_score < 25:
            return "intermediate"
        else:
            return "advanced"
    
    def _generate_doc_id(self, doc: Document) -> str:
        """Gera ID único para documento"""
        
        source = doc.metadata.get('source', 'unknown')
        content_hash = hashlib.md5(doc.page_content.encode()).hexdigest()[:8]
        return f"{Path(source).stem}_{content_hash}"
```

---

## 🔢 Embedding System

### 🧠 Embeddings Manager

```python
# src/core/embeddings.py
from langchain.embeddings import OpenAIEmbeddings
from sentence_transformers import SentenceTransformer
from typing import List, Dict, Optional, Union
import numpy as np
import pickle
import os
from pathlib import Path
import hashlib

class EmbeddingManager:
    """Gerenciador de embeddings com cache e fallback"""
    
    def __init__(self, 
                 primary_model: str = "text-embedding-ada-002",
                 fallback_model: str = "all-MiniLM-L6-v2",
                 cache_dir: str = "data/embeddings"):
        
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(parents=True, exist_ok=True)
        
        # Primary model (OpenAI)
        try:
            self.primary_embeddings = OpenAIEmbeddings(
                model=primary_model,
                chunk_size=1000
            )
            self.use_primary = True
        except Exception as e:
            print(f"Warning: Could not initialize OpenAI embeddings: {e}")
            self.use_primary = False
        
        # Fallback model (SentenceTransformers)
        self.fallback_embeddings = SentenceTransformer(fallback_model)
        
        self.embedding_cache = {}
        self._load_cache()
    
    def embed_documents(self, texts: List[str]) -> List[List[float]]:
        """Gera embeddings para lista de textos com cache"""
        
        embeddings = []
        texts_to_embed = []
        cached_indices = []
        
        # Verifica cache
        for i, text in enumerate(texts):
            text_hash = self._hash_text(text)
            if text_hash in self.embedding_cache:
                embeddings.append(self.embedding_cache[text_hash])
                cached_indices.append(i)
            else:
                texts_to_embed.append((i, text, text_hash))
        
        # Gera embeddings para textos não cacheados
        if texts_to_embed:
            batch_texts = [text for _, text, _ in texts_to_embed]
            
            try:
                if self.use_primary:
                    batch_embeddings = self.primary_embeddings.embed_documents(batch_texts)
                else:
                    batch_embeddings = self.fallback_embeddings.encode(batch_texts).tolist()
                
                # Adiciona ao cache e resultado
                for (original_idx, text, text_hash), embedding in zip(texts_to_embed, batch_embeddings):
                    self.embedding_cache[text_hash] = embedding
                    embeddings.insert(original_idx - len([i for i in cached_indices if i < original_idx]), embedding)
                
                self._save_cache()
                
            except Exception as e:
                print(f"Error generating embeddings: {e}")
                # Fallback para modelo local
                batch_embeddings = self.fallback_embeddings.encode(batch_texts).tolist()
                for (original_idx, text, text_hash), embedding in zip(texts_to_embed, batch_embeddings):
                    embeddings.insert(original_idx - len([i for i in cached_indices if i < original_idx]), embedding)
        
        return embeddings
    
    def embed_query(self, text: str) -> List[float]:
        """Gera embedding para query"""
        
        text_hash = self._hash_text(text)
        
        if text_hash in self.embedding_cache:
            return self.embedding_cache[text_hash]
        
        try:
            if self.use_primary:
                embedding = self.primary_embeddings.embed_query(text)
            else:
                embedding = self.fallback_embeddings.encode([text])[0].tolist()
            
            self.embedding_cache[text_hash] = embedding
            self._save_cache()
            return embedding
            
        except Exception as e:
            print(f"Error generating query embedding: {e}")
            return self.fallback_embeddings.encode([text])[0].tolist()
    
    def _hash_text(self, text: str) -> str:
        """Gera hash do texto para cache"""
        return hashlib.md5(text.encode()).hexdigest()
    
    def _load_cache(self):
        """Carrega cache de embeddings"""
        cache_file = self.cache_dir / "embeddings_cache.pkl"
        if cache_file.exists():
            try:
                with open(cache_file, 'rb') as f:
                    self.embedding_cache = pickle.load(f)
                print(f"Loaded {len(self.embedding_cache)} cached embeddings")
            except Exception as e:
                print(f"Error loading embedding cache: {e}")
                self.embedding_cache = {}
    
    def _save_cache(self):
        """Salva cache de embeddings"""
        cache_file = self.cache_dir / "embeddings_cache.pkl"
        try:
            with open(cache_file, 'wb') as f:
                pickle.dump(self.embedding_cache, f)
        except Exception as e:
            print(f"Error saving embedding cache: {e}")
    
    def get_cache_stats(self) -> Dict:
        """Retorna estatísticas do cache"""
        return {
            "cache_size": len(self.embedding_cache),
            "cache_file_size": self._get_cache_file_size(),
            "primary_model_available": self.use_primary
        }
    
    def _get_cache_file_size(self) -> str:
        """Retorna tamanho do arquivo de cache"""
        cache_file = self.cache_dir / "embeddings_cache.pkl"
        if cache_file.exists():
            size_bytes = cache_file.stat().st_size
            if size_bytes < 1024:
                return f"{size_bytes} B"
            elif size_bytes < 1024*1024:
                return f"{size_bytes/1024:.1f} KB"
            else:
                return f"{size_bytes/(1024*1024):.1f} MB"
        return "0 B"
```

---

## 💾 Vector Store Implementation

### 📊 Vector Database Manager

```python
# src/core/vector_store.py
from langchain.vectorstores import Chroma, FAISS
from langchain.schema import Document
from typing import List, Dict, Optional, Tuple, Any
import chromadb
from chromadb.config import Settings
import pickle
import os
from pathlib import Path

class VectorStoreManager:
    """Gerenciador de vector store com múltiplos backends"""
    
    def __init__(self, 
                 store_type: str = "chromadb",
                 collection_name: str = "documentation",
                 persist_directory: str = "data/vectorstore"):
        
        self.store_type = store_type
        self.collection_name = collection_name
        self.persist_directory = Path(persist_directory)
        self.persist_directory.mkdir(parents=True, exist_ok=True)
        
        self.vectorstore = None
        self.embedding_manager = None
    
    def initialize_store(self, embedding_manager, documents: Optional[List[Document]] = None):
        """Inicializa o vector store"""
        
        self.embedding_manager = embedding_manager
        
        if self.store_type == "chromadb":
            self.vectorstore = self._init_chromadb(documents)
        elif self.store_type == "faiss":
            self.vectorstore = self._init_faiss(documents)
        else:
            raise ValueError(f"Unsupported store type: {self.store_type}")
    
    def _init_chromadb(self, documents: Optional[List[Document]] = None):
        """Inicializa ChromaDB"""
        
        client = chromadb.PersistentClient(
            path=str(self.persist_directory),
            settings=Settings(anonymized_telemetry=False)
        )
        
        try:
            # Tenta carregar coleção existente
            collection = client.get_collection(name=self.collection_name)
            vectorstore = Chroma(
                client=client,
                collection_name=self.collection_name,
                embedding_function=self.embedding_manager
            )
            print(f"Loaded existing ChromaDB collection: {self.collection_name}")
            
        except Exception:
            # Cria nova coleção
            if documents:
                vectorstore = Chroma.from_documents(
                    documents=documents,
                    embedding=self.embedding_manager,
                    client=client,
                    collection_name=self.collection_name,
                    persist_directory=str(self.persist_directory)
                )
                print(f"Created new ChromaDB collection: {self.collection_name}")
            else:
                # Cria coleção vazia
                collection = client.create_collection(name=self.collection_name)
                vectorstore = Chroma(
                    client=client,
                    collection_name=self.collection_name,
                    embedding_function=self.embedding_manager
                )
                print(f"Created empty ChromaDB collection: {self.collection_name}")
        
        return vectorstore
    
    def _init_faiss(self, documents: Optional[List[Document]] = None):
        """Inicializa FAISS"""
        
        faiss_index_path = self.persist_directory / "faiss_index"
        
        if faiss_index_path.exists():
            # Carrega índice existente
            vectorstore = FAISS.load_local(
                str(faiss_index_path),
                self.embedding_manager
            )
            print(f"Loaded existing FAISS index from {faiss_index_path}")
        else:
            if documents:
                # Cria novo índice
                vectorstore = FAISS.from_documents(
                    documents,
                    self.embedding_manager
                )
                vectorstore.save_local(str(faiss_index_path))
                print(f"Created new FAISS index at {faiss_index_path}")
            else:
                raise ValueError("Cannot create empty FAISS index")
        
        return vectorstore
    
    def add_documents(self, documents: List[Document]) -> List[str]:
        """Adiciona documentos ao vector store"""
        
        if not self.vectorstore:
            raise ValueError("Vector store not initialized")
        
        # Filtra documentos duplicados
        unique_docs = self._filter_duplicates(documents)
        
        if unique_docs:
            ids = self.vectorstore.add_documents(unique_docs)
            
            # Persiste mudanças
            if self.store_type == "faiss":
                faiss_index_path = self.persist_directory / "faiss_index"
                self.vectorstore.save_local(str(faiss_index_path))
            
            print(f"Added {len(unique_docs)} unique documents to vector store")
            return ids
        else:
            print("No new documents to add (all were duplicates)")
            return []
    
    def similarity_search(self, 
                         query: str, 
                         k: int = 5, 
                         filters: Optional[Dict] = None) -> List[Document]:
        """Busca por similaridade"""
        
        if not self.vectorstore:
            raise ValueError("Vector store not initialized")
        
        if filters and self.store_type == "chromadb":
            # ChromaDB suporta filtros nativos
            results = self.vectorstore.similarity_search(
                query=query,
                k=k,
                filter=self._convert_filters_to_chroma(filters)
            )
        else:
            # Busca básica + filtro pós-processamento
            results = self.vectorstore.similarity_search(query=query, k=k*2)  # Busca mais para compensar filtros
            
            if filters:
                results = self._apply_post_filters(results, filters)
            
            results = results[:k]  # Limita resultado final
        
        return results
    
    def similarity_search_with_score(self, 
                                   query: str, 
                                   k: int = 5,
                                   score_threshold: float = 0.0) -> List[Tuple[Document, float]]:
        """Busca por similaridade com scores"""
        
        if not self.vectorstore:
            raise ValueError("Vector store not initialized")
        
        results = self.vectorstore.similarity_search_with_score(query=query, k=k)
        
        # Filtra por threshold de score
        filtered_results = [
            (doc, score) for doc, score in results 
            if score >= score_threshold
        ]
        
        return filtered_results
    
    def _filter_duplicates(self, documents: List[Document]) -> List[Document]:
        """Filtra documentos duplicados baseado em hash do conteúdo"""
        
        unique_docs = []
        seen_hashes = set()
        
        for doc in documents:
            doc_hash = doc.metadata.get('chunk_hash')
            if not doc_hash:
                # Gera hash se não existir
                import hashlib
                doc_hash = hashlib.md5(doc.page_content.encode()).hexdigest()
                doc.metadata['chunk_hash'] = doc_hash
            
            if doc_hash not in seen_hashes:
                unique_docs.append(doc)
                seen_hashes.add(doc_hash)
        
        return unique_docs
    
    def _convert_filters_to_chroma(self, filters: Dict) -> Dict:
        """Converte filtros para formato ChromaDB"""
        
        chroma_filters = {}
        
        for key, value in filters.items():
            if isinstance(value, list):
                chroma_filters[key] = {"$in": value}
            elif isinstance(value, dict):
                if 'min' in value or 'max' in value:
                    chroma_filters[key] = {}
                    if 'min' in value:
                        chroma_filters[key]["$gte"] = value['min']
                    if 'max' in value:
                        chroma_filters[key]["$lte"] = value['max']
                else:
                    chroma_filters[key] = value
            else:
                chroma_filters[key] = {"$eq": value}
        
        return chroma_filters
    
    def _apply_post_filters(self, documents: List[Document], filters: Dict) -> List[Document]:
        """Aplica filtros pós-busca para backends que não suportam filtros nativos"""
        
        filtered_docs = []
        
        for doc in documents:
            matches_all_filters = True
            
            for key, expected_value in filters.items():
                doc_value = doc.metadata.get(key)
                
                if isinstance(expected_value, list):
                    if doc_value not in expected_value:
                        matches_all_filters = False
                        break
                elif isinstance(expected_value, dict):
                    if 'min' in expected_value and doc_value < expected_value['min']:
                        matches_all_filters = False
                        break
                    if 'max' in expected_value and doc_value > expected_value['max']:
                        matches_all_filters = False
                        break
                else:
                    if doc_value != expected_value:
                        matches_all_filters = False
                        break
            
            if matches_all_filters:
                filtered_docs.append(doc)
        
        return filtered_docs
    
    def get_collection_stats(self) -> Dict[str, Any]:
        """Retorna estatísticas da coleção"""
        
        if not self.vectorstore:
            return {"error": "Vector store not initialized"}
        
        try:
            if self.store_type == "chromadb":
                collection = self.vectorstore._collection
                return {
                    "total_documents": collection.count(),
                    "store_type": self.store_type,
                    "collection_name": self.collection_name
                }
            else:
                # Para FAISS, não temos stats detalhadas facilmente
                return {
                    "store_type": self.store_type,
                    "index_file": str(self.persist_directory / "faiss_index")
                }
        except Exception as e:
            return {"error": f"Could not retrieve stats: {e}"}
    
    def delete_collection(self):
        """Deleta a coleção (cuidado!)"""
        
        if self.store_type == "chromadb":
            client = chromadb.PersistentClient(path=str(self.persist_directory))
            try:
                client.delete_collection(name=self.collection_name)
                print(f"Deleted ChromaDB collection: {self.collection_name}")
            except Exception as e:
                print(f"Error deleting collection: {e}")
        
        elif self.store_type == "faiss":
            faiss_index_path = self.persist_directory / "faiss_index"
            if faiss_index_path.exists():
                import shutil
                shutil.rmtree(faiss_index_path)
                print(f"Deleted FAISS index: {faiss_index_path}")
```

---

## 🔍 Advanced Retrieval System

```python
# src/core/retriever.py
from typing import List, Dict, Optional, Tuple, Any
from langchain.schema import Document
import numpy as np
from collections import defaultdict
import re

class AdvancedRetriever:
    """Sistema de retrieval avançado com re-ranking e filtros contextuais"""
    
    def __init__(self, vector_store_manager):
        self.vector_store = vector_store_manager
        self.query_history = []
        self.user_preferences = defaultdict(dict)
    
    def retrieve(self, 
                query: str,
                k: int = 5,
                filters: Optional[Dict] = None,
                user_id: Optional[str] = None,
                rerank: bool = True) -> List[Document]:
        """Recuperação avançada com re-ranking opcional"""
        
        # Enriquece query com contexto
        enriched_query = self._enrich_query(query, user_id)
        
        # Busca inicial (busca mais documentos para re-ranking)
        initial_k = k * 3 if rerank else k
        
        initial_results = self.vector_store.similarity_search_with_score(
            query=enriched_query,
            k=initial_k,
            score_threshold=0.5
        )
        
        if not initial_results:
            return []
        
        # Aplica filtros contextuais
        if filters:
            filtered_results = self._apply_contextual_filters(initial_results, filters)
        else:
            filtered_results = initial_results
        
        # Re-ranking se solicitado
        if rerank and len(filtered_results) > k:
            reranked_results = self._rerank_results(query, filtered_results, user_id)
            final_results = reranked_results[:k]
        else:
            final_results = filtered_results[:k]
        
        # Registra query para learning
        self._record_query(query, final_results, user_id)
        
        return [doc for doc, score in final_results]
    
    def _enrich_query(self, query: str, user_id: Optional[str] = None) -> str:
        """Enriquece query com contexto histórico e preferências"""
        
        enriched_query = query
        
        # Adiciona contexto de queries recentes
        if self.query_history:
            recent_queries = [q['query'] for q in self.query_history[-3:]]
            context_terms = self._extract_context_terms(recent_queries)
            if context_terms:
                enriched_query += f" Context: {' '.join(context_terms[:3])}"
        
        # Adiciona preferências do usuário
        if user_id and user_id in self.user_preferences:
            prefs = self.user_preferences[user_id]
            if 'preferred_topics' in prefs:
                enriched_query += f" Topics: {' '.join(prefs['preferred_topics'][:2])}"
        
        return enriched_query
    
    def _apply_contextual_filters(self, 
                                results: List[Tuple[Document, float]], 
                                filters: Dict) -> List[Tuple[Document, float]]:
        """Aplica filtros contextuais avançados"""
        
        filtered_results = []
        
        for doc, score in results:
            passes_filters = True
            
            # Filtro por tipo de conteúdo
            if 'content_type' in filters:
                expected_types = filters['content_type']
                if isinstance(expected_types, str):
                    expected_types = [expected_types]
                
                doc_type = self._detect_content_type(doc)
                if doc_type not in expected_types:
                    passes_filters = False
            
            # Filtro por complexidade
            if 'complexity' in filters:
                expected_complexity = filters['complexity']
                doc_complexity = doc.metadata.get('complexity', 'intermediate')
                if doc_complexity != expected_complexity:
                    passes_filters = False
            
            # Filtro por recência
            if 'max_age_days' in filters:
                max_age = filters['max_age_days']
                doc_age = self._calculate_document_age(doc)
                if doc_age > max_age:
                    passes_filters = False
            
            # Filtro por qualidade
            if 'min_quality_score' in filters:
                min_quality = filters['min_quality_score']
                quality_score = self._calculate_quality_score(doc)
                if quality_score < min_quality:
                    passes_filters = False
            
            if passes_filters:
                filtered_results.append((doc, score))
        
        return filtered_results
    
    def _rerank_results(self, 
                       query: str, 
                       results: List[Tuple[Document, float]], 
                       user_id: Optional[str] = None) -> List[Tuple[Document, float]]:
        """Re-ranking baseado em múltiplos fatores"""
        
        reranked_results = []
        
        for doc, similarity_score in results:
            # Fatores de re-ranking
            factors = {
                'similarity': similarity_score,
                'quality': self._calculate_quality_score(doc),
                'relevance': self._calculate_relevance_score(query, doc),
                'freshness': self._calculate_freshness_score(doc),
                'user_preference': self._calculate_user_preference_score(doc, user_id)
            }
            
            # Pesos dos fatores
            weights = {
                'similarity': 0.4,
                'quality': 0.2,
                'relevance': 0.2,
                'freshness': 0.1,
                'user_preference': 0.1
            }
            
            # Calcula score final
            final_score = sum(factors[factor] * weights[factor] for factor in factors)
            
            reranked_results.append((doc, final_score))
        
        # Ordena por score final
        reranked_results.sort(key=lambda x: x[1], reverse=True)
        
        return reranked_results
    
    def _detect_content_type(self, doc: Document) -> str:
        """Detecta o tipo de conteúdo do documento"""
        
        content = doc.page_content.lower()
        metadata = doc.metadata
        
        # Verifica metadados primeiro
        if 'source_type' in metadata:
            return metadata['source_type']
        
        # Detecção baseada em conteúdo
        if '```' in content or 'def ' in content or 'function' in content:
            return 'code'
        elif 'get /' in content or 'post /' in content or 'endpoint' in content:
            return 'api'
        elif 'tutorial' in content or 'step' in content or 'how to' in content:
            return 'tutorial'
        elif '?' in content and len(content.split('?')) > 2:
            return 'faq'
        else:
            return 'documentation'
    
    def _calculate_document_age(self, doc: Document) -> int:
        """Calcula idade do documento em dias"""
        
        import time
        
        last_modified = doc.metadata.get('last_modified')
        if last_modified:
            current_time = time.time()
            age_seconds = current_time - last_modified
            return int(age_seconds / (24 * 3600))  # Converte para dias
        
        return 365  # Assume 1 ano se não há informação
    
    def _calculate_quality_score(self, doc: Document) -> float:
        """Calcula score de qualidade do documento"""
        
        content = doc.page_content
        metadata = doc.metadata
        
        score = 0.5  # Base score
        
        # Fatores de qualidade
        word_count = len(content.split())
        if 50 <= word_count <= 1000:  # Tamanho ideal
            score += 0.2
        
        # Presença de exemplos de código
        if '```' in content:
            score += 0.1
        
        # Presença de links/referências
        if '[' in content and '](' in content:
            score += 0.1
        
        # Estrutura (headings)
        if '#' in content:
            score += 0.1
        
        # Metadados de qualidade
        if metadata.get('has_code'):
            score += 0.05
        if metadata.get('has_links'):
            score += 0.05
        
        return min(score, 1.0)  # Limita a 1.0
    
    def _calculate_relevance_score(self, query: str, doc: Document) -> float:
        """Calcula relevância específica do documento para a query"""
        
        query_terms = set(query.lower().split())
        content_terms = set(doc.page_content.lower().split())
        
        # Term overlap
        overlap = len(query_terms.intersection(content_terms))
        max_possible = len(query_terms)
        
        if max_possible == 0:
            return 0.5
        
        overlap_score = overlap / max_possible
        
        # Boost for exact phrase matches
        if query.lower() in doc.page_content.lower():
            overlap_score += 0.3
        
        return min(overlap_score, 1.0)
    
    def _calculate_freshness_score(self, doc: Document) -> float:
        """Calcula score de frescor do documento"""
        
        age_days = self._calculate_document_age(doc)
        
        # Score diminui com a idade
        if age_days <= 7:
            return 1.0
        elif age_days <= 30:
            return 0.8
        elif age_days <= 90:
            return 0.6
        elif age_days <= 180:
            return 0.4
        else:
            return 0.2
    
    def _calculate_user_preference_score(self, doc: Document, user_id: Optional[str]) -> float:
        """Calcula score baseado em preferências do usuário"""
        
        if not user_id or user_id not in self.user_preferences:
            return 0.5
        
        prefs = self.user_preferences[user_id]
        score = 0.5
        
        # Tópicos preferidos
        if 'preferred_topics' in prefs:
            doc_topics = self._extract_topics(doc)
            for topic in prefs['preferred_topics']:
                if topic in doc_topics:
                    score += 0.1
        
        # Complexidade preferida
        if 'preferred_complexity' in prefs:
            doc_complexity = doc.metadata.get('complexity', 'intermediate')
            if doc_complexity == prefs['preferred_complexity']:
                score += 0.2
        
        return min(score, 1.0)
    
    def _extract_context_terms(self, queries: List[str]) -> List[str]:
        """Extrai termos de contexto de queries anteriores"""
        
        all_terms = []
        for query in queries:
            terms = re.findall(r'\b\w{4,}\b', query.lower())  # Palavras com 4+ caracteres
            all_terms.extend(terms)
        
        # Remove duplicatas mantendo ordem
        unique_terms = []
        seen = set()
        for term in all_terms:
            if term not in seen:
                unique_terms.append(term)
                seen.add(term)
        
        return unique_terms
    
    def _extract_topics(self, doc: Document) -> List[str]:
        """Extrai tópicos principais do documento"""
        
        content = doc.page_content.lower()
        topics = []
        
        # Tópicos técnicos comuns
        tech_topics = ['api', 'database', 'authentication', 'security', 'performance', 
                      'deployment', 'testing', 'monitoring', 'scaling', 'integration']
        
        for topic in tech_topics:
            if topic in content:
                topics.append(topic)
        
        # Tópicos dos metadados
        if 'tags' in doc.metadata:
            topics.extend(doc.metadata['tags'])
        
        return topics
    
    def _record_query(self, query: str, results: List[Tuple[Document, float]], user_id: Optional[str]):
        """Registra query para learning futuro"""
        
        query_record = {
            'query': query,
            'timestamp': time.time(),
            'results_count': len(results),
            'user_id': user_id
        }
        
        self.query_history.append(query_record)
        
        # Mantém apenas últimas 100 queries
        if len(self.query_history) > 100:
            self.query_history = self.query_history[-100:]
        
        # Atualiza preferências do usuário
        if user_id and results:
            self._update_user_preferences(user_id, query, results)
    
    def _update_user_preferences(self, user_id: str, query: str, results: List[Tuple[Document, float]]):
        """Atualiza preferências do usuário baseado no histórico"""
        
        if user_id not in self.user_preferences:
            self.user_preferences[user_id] = {
                'preferred_topics': [],
                'preferred_complexity': 'intermediate',
                'query_count': 0
            }
        
        prefs = self.user_preferences[user_id]
        prefs['query_count'] += 1
        
        # Extrai tópicos dos resultados
        for doc, score in results[:3]:  # Top 3 resultados
            doc_topics = self._extract_topics(doc)
            for topic in doc_topics:
                if topic not in prefs['preferred_topics']:
                    prefs['preferred_topics'].append(topic)
        
        # Mantém apenas top 10 tópicos
        prefs['preferred_topics'] = prefs['preferred_topics'][:10]
    
    def get_retrieval_stats(self) -> Dict[str, Any]:
        """Retorna estatísticas do sistema de retrieval"""
        
        return {
            'total_queries': len(self.query_history),
            'unique_users': len(self.user_preferences),
            'avg_results_per_query': np.mean([q['results_count'] for q in self.query_history]) if self.query_history else 0,
            'recent_queries': [q['query'] for q in self.query_history[-5:]]
        }
```

*[Continuará na parte 2 devido ao limite de comprimento...]*

---

## 🔗 Relacionado

- [[🔍 RAG - Retrieval-Augmented Generation]]
- [[🏗️ Componentes Doc 4.0]]
- [[🤖 Agentes IA para Automação]]
- [[🧪 Automação de Testes]]

---

#rag #python #implementacao #langchain #vector-database #embeddings #retrieval #campus-party
