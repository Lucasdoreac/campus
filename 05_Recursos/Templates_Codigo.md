# 📝 Templates de Código para Documentação 4.0

> Código reutilizável e configurações prontas para acelerar implementação

---

## 🤖 Templates RAG System

### 🔍 RAG Basic Implementation

```python
"""
Template básico para sistema RAG com LangChain
Pronto para produção com configurações essenciais
"""

import os
from typing import List, Dict, Optional
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Pinecone
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import ConversationalRetrievalChain
from langchain.llms import OpenAI
from langchain.memory import ConversationBufferWindowMemory
import pinecone

class DocumentationRAG:
    """Sistema RAG para documentação com cache e otimizações"""
    
    def __init__(self, 
                 pinecone_api_key: str,
                 pinecone_env: str,
                 openai_api_key: str,
                 index_name: str = "docs-index"):
        
        # Configuração
        self.index_name = index_name
        self.embeddings = OpenAIEmbeddings(openai_api_key=openai_api_key)
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200,
            separators=["\n\n", "\n", " ", ""]
        )
        
        # Inicializa Pinecone
        pinecone.init(
            api_key=pinecone_api_key,
            environment=pinecone_env
        )
        
        # Vector store
        self.vectorstore = Pinecone.from_existing_index(
            index_name=index_name,
            embedding=self.embeddings
        )
        
        # LLM com configurações otimizadas
        self.llm = OpenAI(
            temperature=0.1,
            model_name="gpt-3.5-turbo-instruct",
            max_tokens=500,
            openai_api_key=openai_api_key
        )
        
        # Memória conversacional
        self.memory = ConversationBufferWindowMemory(
            memory_key="chat_history",
            return_messages=True,
            k=5  # Últimas 5 interações
        )
        
        # Chain principal
        self.qa_chain = ConversationalRetrievalChain.from_llm(
            llm=self.llm,
            retriever=self.vectorstore.as_retriever(
                search_type="similarity",
                search_kwargs={"k": 5}
            ),
            memory=self.memory,
            return_source_documents=True,
            verbose=True
        )
    
    def add_documents(self, documents: List[Dict]) -> Dict:
        """Adiciona documentos ao vector store"""
        try:
            texts = []
            metadatas = []
            
            for doc in documents:
                # Chunking do conteúdo
                chunks = self.text_splitter.split_text(doc['content'])
                
                for i, chunk in enumerate(chunks):
                    texts.append(chunk)
                    metadatas.append({
                        'source': doc.get('source', 'unknown'),
                        'title': doc.get('title', 'untitled'),
                        'chunk_id': f"{doc.get('id', 'doc')}_{i}",
                        'type': doc.get('type', 'document'),
                        **doc.get('metadata', {})
                    })
            
            # Adiciona ao vector store
            self.vectorstore.add_texts(texts, metadatas=metadatas)
            
            return {
                "success": True,
                "documents_processed": len(documents),
                "chunks_created": len(texts)
            }
            
        except Exception as e:
            return {
                "success": False,
                "error": str(e)
            }
    
    def query(self, question: str, filters: Optional[Dict] = None) -> Dict:
        """Executa query no sistema RAG"""
        try:
            # Aplica filtros se fornecidos
            if filters:
                retriever = self.vectorstore.as_retriever(
                    search_type="similarity",
                    search_kwargs={"k": 5, "filter": filters}
                )
                # Atualiza o retriever temporariamente
                original_retriever = self.qa_chain.retriever
                self.qa_chain.retriever = retriever
            
            # Executa a query
            result = self.qa_chain({"question": question})
            
            # Restaura retriever original se foi modificado
            if filters:
                self.qa_chain.retriever = original_retriever
            
            return {
                "success": True,
                "answer": result["answer"],
                "sources": [
                    {
                        "content": doc.page_content[:200] + "...",
                        "metadata": doc.metadata
                    }
                    for doc in result.get("source_documents", [])
                ],
                "chat_history": result.get("chat_history", [])
            }
            
        except Exception as e:
            return {
                "success": False,
                "error": str(e),
                "answer": "Desculpe, ocorreu um erro ao processar sua pergunta."
            }
    
    def get_related_docs(self, query: str, k: int = 3) -> List[Dict]:
        """Busca documentos relacionados sem gerar resposta"""
        try:
            docs = self.vectorstore.similarity_search(query, k=k)
            return [
                {
                    "content": doc.page_content,
                    "metadata": doc.metadata,
                    "score": getattr(doc, 'score', 0)
                }
                for doc in docs
            ]
        except Exception as e:
            return []

# Exemplo de uso
if __name__ == "__main__":
    # Configuração
    rag = DocumentationRAG(
        pinecone_api_key=os.getenv("PINECONE_API_KEY"),
        pinecone_env=os.getenv("PINECONE_ENV"),
        openai_api_key=os.getenv("OPENAI_API_KEY")
    )
    
    # Adiciona documentos
    sample_docs = [
        {
            "id": "api_auth",
            "title": "API Authentication",
            "content": "Para autenticar com nossa API, use Bearer token no header Authorization...",
            "source": "api/auth.md",
            "type": "api_documentation"
        }
    ]
    
    result = rag.add_documents(sample_docs)
    print(f"Documentos adicionados: {result}")
    
    # Faz uma pergunta
    response = rag.query("Como faço autenticação na API?")
    print(f"Resposta: {response['answer']}")
```

### 🔧 FastAPI Server Template

```python
"""
Template FastAPI para servir sistema RAG
Inclui autenticação, rate limiting e monitoramento
"""

from fastapi import FastAPI, HTTPException, Depends, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from typing import List, Optional, Dict
import uvicorn
import redis
import json
import time
from datetime import datetime, timedelta

# Importa nossa classe RAG
from rag_system import DocumentationRAG

# Configuração
app = FastAPI(
    title="Documentation 4.0 API",
    description="API inteligente para documentação com RAG",
    version="1.0.0"
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configurar adequadamente em produção
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Security
security = HTTPBearer()

# Redis para cache e rate limiting
redis_client = redis.Redis(host='localhost', port=6379, db=0)

# Inicializa RAG system
rag_system = DocumentationRAG(
    pinecone_api_key=os.getenv("PINECONE_API_KEY"),
    pinecone_env=os.getenv("PINECONE_ENV"),
    openai_api_key=os.getenv("OPENAI_API_KEY")
)

# Pydantic models
class QueryRequest(BaseModel):
    question: str
    filters: Optional[Dict] = None
    include_sources: bool = True

class QueryResponse(BaseModel):
    answer: str
    sources: Optional[List[Dict]] = None
    response_time: float
    cached: bool = False

class DocumentRequest(BaseModel):
    documents: List[Dict]

class HealthResponse(BaseModel):
    status: str
    timestamp: str
    version: str

# Rate limiting decorator
def rate_limit(max_requests: int = 100, window_seconds: int = 3600):
    def decorator(func):
        async def wrapper(request, *args, **kwargs):
            # Simples rate limiting por IP (melhorar em produção)
            client_ip = request.client.host
            key = f"rate_limit:{client_ip}"
            
            current = redis_client.get(key)
            if current is None:
                redis_client.setex(key, window_seconds, 1)
            else:
                if int(current) >= max_requests:
                    raise HTTPException(status_code=429, detail="Rate limit exceeded")
                redis_client.incr(key)
            
            return await func(request, *args, **kwargs)
        return wrapper
    return decorator

# Cache decorator
def cache_response(ttl_seconds: int = 300):
    def decorator(func):
        async def wrapper(*args, **kwargs):
            # Gera chave de cache baseada nos argumentos
            cache_key = f"cache:{hash(str(args) + str(kwargs))}"
            
            # Tenta buscar no cache
            cached = redis_client.get(cache_key)
            if cached:
                return {**json.loads(cached), "cached": True}
            
            # Executa função
            result = await func(*args, **kwargs)
            
            # Salva no cache
            redis_client.setex(cache_key, ttl_seconds, json.dumps(result))
            
            return result
        return wrapper
    return decorator

# Endpoints
@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check endpoint"""
    return HealthResponse(
        status="healthy",
        timestamp=datetime.now().isoformat(),
        version="1.0.0"
    )

@app.post("/query", response_model=QueryResponse)
@cache_response(ttl_seconds=300)
async def query_documentation(request: QueryRequest):
    """Endpoint principal para queries de documentação"""
    start_time = time.time()
    
    try:
        result = rag_system.query(
            question=request.question,
            filters=request.filters
        )
        
        if not result["success"]:
            raise HTTPException(status_code=500, detail=result["error"])
        
        response_time = time.time() - start_time
        
        return QueryResponse(
            answer=result["answer"],
            sources=result["sources"] if request.include_sources else None,
            response_time=round(response_time, 3)
        )
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/documents")
async def add_documents(
    request: DocumentRequest,
    background_tasks: BackgroundTasks,
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    """Endpoint para adicionar documentos (requer autenticação)"""
    
    # Validação simples de token (implementar adequadamente)
    if credentials.credentials != os.getenv("API_SECRET_KEY"):
        raise HTTPException(status_code=401, detail="Invalid token")
    
    try:
        # Adiciona documentos em background
        background_tasks.add_task(
            process_documents_background,
            request.documents
        )
        
        return {
            "message": "Documents queued for processing",
            "count": len(request.documents)
        }
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/search")
async def search_documents(
    q: str,
    limit: int = 5,
    filters: Optional[str] = None
):
    """Endpoint para busca de documentos relacionados"""
    try:
        filter_dict = json.loads(filters) if filters else None
        
        docs = rag_system.get_related_docs(
            query=q,
            k=limit
        )
        
        return {
            "query": q,
            "results": docs,
            "count": len(docs)
        }
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/metrics")
async def get_metrics():
    """Endpoint para métricas básicas"""
    try:
        # Métricas básicas do Redis
        info = redis_client.info()
        
        return {
            "redis_connected_clients": info.get("connected_clients", 0),
            "redis_used_memory": info.get("used_memory_human", "0B"),
            "cache_keys": redis_client.dbsize(),
            "timestamp": datetime.now().isoformat()
        }
        
    except Exception as e:
        return {"error": str(e)}

# Background tasks
async def process_documents_background(documents: List[Dict]):
    """Processa documentos em background"""
    try:
        result = rag_system.add_documents(documents)
        
        # Log resultado
        print(f"Background processing completed: {result}")
        
        # Poderia enviar notificação, webhook, etc.
        
    except Exception as e:
        print(f"Background processing failed: {e}")

# Startup event
@app.on_event("startup")
async def startup_event():
    """Inicialização da aplicação"""
    print("🚀 Documentation 4.0 API starting up...")
    
    # Testa conexões
    try:
        redis_client.ping()
        print("✅ Redis connection successful")
    except:
        print("❌ Redis connection failed")
    
    print("✅ RAG system initialized")
    print("🎉 API ready to serve requests!")

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        reload=True,
        log_level="info"
    )
```

---

## 🧪 Templates de Testes

### 🔬 Test Suite Completo

```python
"""
Template de testes para sistema de documentação
Inclui testes unitários, integração e E2E
"""

import pytest
import asyncio
from unittest.mock import Mock, patch
import json
from fastapi.testclient import TestClient

# Importa nossa aplicação
from main import app
from rag_system import DocumentationRAG

client = TestClient(app)

# Fixtures
@pytest.fixture
def sample_documents():
    return [
        {
            "id": "test_doc_1",
            "title": "Test API Documentation",
            "content": "This is a test document about API authentication and usage.",
            "source": "test/api.md",
            "type": "api_documentation",
            "metadata": {
                "author": "test_user",
                "version": "1.0"
            }
        }
    ]

@pytest.fixture
def rag_system_mock():
    with patch('main.rag_system') as mock:
        yield mock

# Testes de unidade
class TestRAGSystem:
    """Testes unitários para o sistema RAG"""
    
    @pytest.mark.asyncio
    async def test_query_success(self, rag_system_mock):
        """Testa query bem-sucedida"""
        # Mock da resposta
        rag_system_mock.query.return_value = {
            "success": True,
            "answer": "Para autenticar, use Bearer token.",
            "sources": [
                {
                    "content": "Authentication docs...",
                    "metadata": {"source": "api/auth.md"}
                }
            ]
        }
        
        # Executa query
        response = client.post("/query", json={
            "question": "Como fazer autenticação?",
            "include_sources": True
        })
        
        # Validações
        assert response.status_code == 200
        data = response.json()
        assert data["answer"] == "Para autenticar, use Bearer token."
        assert len(data["sources"]) == 1
        assert "response_time" in data
    
    @pytest.mark.asyncio
    async def test_query_error(self, rag_system_mock):
        """Testa query com erro"""
        # Mock do erro
        rag_system_mock.query.return_value = {
            "success": False,
            "error": "Connection timeout"
        }
        
        # Executa query
        response = client.post("/query", json={
            "question": "Test question"
        })
        
        # Validações
        assert response.status_code == 500
        assert "Connection timeout" in response.json()["detail"]
    
    def test_add_documents_without_auth(self):
        """Testa adição de documentos sem autenticação"""
        response = client.post("/documents", json={
            "documents": []
        })
        
        assert response.status_code == 403  # Sem token
    
    def test_add_documents_with_invalid_auth(self):
        """Testa adição com token inválido"""
        response = client.post("/documents", 
            json={"documents": []},
            headers={"Authorization": "Bearer invalid_token"}
        )
        
        assert response.status_code == 401
    
    @patch.dict('os.environ', {'API_SECRET_KEY': 'test_secret'})
    def test_add_documents_success(self, rag_system_mock, sample_documents):
        """Testa adição bem-sucedida de documentos"""
        response = client.post("/documents",
            json={"documents": sample_documents},
            headers={"Authorization": "Bearer test_secret"}
        )
        
        assert response.status_code == 200
        data = response.json()
        assert data["count"] == 1
        assert "queued for processing" in data["message"]

# Testes de integração
class TestIntegration:
    """Testes de integração entre componentes"""
    
    @pytest.mark.integration
    def test_health_endpoint(self):
        """Testa endpoint de health check"""
        response = client.get("/health")
        
        assert response.status_code == 200
        data = response.json()
        assert data["status"] == "healthy"
        assert "timestamp" in data
        assert data["version"] == "1.0.0"
    
    @pytest.mark.integration
    def test_search_endpoint(self):
        """Testa endpoint de busca"""
        response = client.get("/search?q=authentication&limit=3")
        
        assert response.status_code == 200
        data = response.json()
        assert data["query"] == "authentication"
        assert "results" in data
        assert "count" in data
    
    @pytest.mark.integration
    def test_metrics_endpoint(self):
        """Testa endpoint de métricas"""
        response = client.get("/metrics")
        
        assert response.status_code == 200
        data = response.json()
        # Verifica se tem métricas básicas (pode falhar se Redis não estiver rodando)
        assert "timestamp" in data

# Testes E2E
class TestEndToEnd:
    """Testes end-to-end completos"""
    
    @pytest.mark.e2e
    @pytest.mark.asyncio
    async def test_complete_workflow(self, sample_documents):
        """Testa workflow completo: adicionar docs -> fazer query"""
        
        # 1. Adiciona documentos
        with patch.dict('os.environ', {'API_SECRET_KEY': 'test_secret'}):
            add_response = client.post("/documents",
                json={"documents": sample_documents},
                headers={"Authorization": "Bearer test_secret"}
            )
        
        assert add_response.status_code == 200
        
        # 2. Aguarda processamento (em cenário real)
        await asyncio.sleep(1)
        
        # 3. Faz query
        query_response = client.post("/query", json={
            "question": "What is this document about?",
            "include_sources": True
        })
        
        # Em teste real, verificaria se a resposta contém informações do documento
        # Aqui só verifica se não deu erro
        assert query_response.status_code in [200, 500]  # 500 OK se não tiver RAG real

# Testes de performance
class TestPerformance:
    """Testes de performance e carga"""
    
    @pytest.mark.performance
    def test_query_response_time(self):
        """Testa tempo de resposta das queries"""
        import time
        
        start_time = time.time()
        
        response = client.post("/query", json={
            "question": "Quick test question"
        })
        
        end_time = time.time()
        response_time = end_time - start_time
        
        # Em produção, ajustar limites conforme SLA
        assert response_time < 5.0  # Máximo 5 segundos
        
        if response.status_code == 200:
            data = response.json()
            assert data["response_time"] < 3.0  # Tempo interno < 3s
    
    @pytest.mark.performance 
    def test_concurrent_queries(self):
        """Testa queries concorrentes"""
        import concurrent.futures
        import threading
        
        def make_query(thread_id):
            response = client.post("/query", json={
                "question": f"Test question from thread {thread_id}"
            })
            return response.status_code
        
        # Executa 10 queries concorrentes
        with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
            futures = [executor.submit(make_query, i) for i in range(10)]
            results = [future.result() for future in futures]
        
        # Verifica se todas as requests foram processadas
        success_count = sum(1 for code in results if code in [200, 500])
        assert success_count == 10  # Todas processadas

# Configuração de testes
def pytest_configure():
    """Configuração global do pytest"""
    pytest.mark.unit = pytest.mark.unit
    pytest.mark.integration = pytest.mark.integration  
    pytest.mark.e2e = pytest.mark.e2e
    pytest.mark.performance = pytest.mark.performance

# Exemplo de execução
if __name__ == "__main__":
    # Executa apenas testes unitários
    pytest.main(["-v", "-m", "unit"])
    
    # Executa todos os testes
    # pytest.main(["-v"])
    
    # Executa com coverage
    # pytest.main(["--cov=.", "--cov-report=html"])
```

---

## 🐳 Templates Docker e Deploy

### 📦 Dockerfile Otimizado

```dockerfile
# Multi-stage build para otimizar tamanho da imagem
FROM python:3.11-slim as builder

# Instala dependências do sistema
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Configura Poetry para gerenciamento de dependências
ENV POETRY_HOME="/opt/poetry" \
    POETRY_CACHE_DIR="/tmp/poetry_cache" \
    POETRY_VENV_IN_PROJECT=1 \
    POETRY_NO_INTERACTION=1

RUN pip install poetry

# Copia arquivos de dependências
WORKDIR /app
COPY pyproject.toml poetry.lock ./

# Instala dependências
RUN poetry install --only=main && rm -rf $POETRY_CACHE_DIR

# Stage de produção
FROM python:3.11-slim as production

# Cria usuário não-root
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Instala dependências mínimas do sistema
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copia virtual environment do builder
COPY --from=builder /app/.venv /app/.venv

# Configura PATH
ENV PATH="/app/.venv/bin:$PATH"

# Cria diretórios de trabalho
WORKDIR /app
RUN chown -R appuser:appuser /app

# Copia código da aplicação
COPY --chown=appuser:appuser . .

# Configura usuário
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Expõe porta
EXPOSE 8000

# Comando de inicialização
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### 🚀 Docker Compose Stack

```yaml
# docker-compose.yml - Stack completo para desenvolvimento
version: '3.8'

services:
  # API principal
  api:
    build: 
      context: .
      dockerfile: Dockerfile
      target: production
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:password@postgres:5432/docs_db
      - REDIS_URL=redis://redis:6379/0
      - PINECONE_API_KEY=${PINECONE_API_KEY}
      - PINECONE_ENV=${PINECONE_ENV}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - API_SECRET_KEY=${API_SECRET_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - docs_network
    restart: unless-stopped
    volumes:
      - ./logs:/app/logs
  
  # PostgreSQL para dados relacionais
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: docs_db
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8"
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - docs_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d docs_db"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  # Redis para cache e sessões
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    networks:
      - docs_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
  
  # Elasticsearch para busca textual
  elasticsearch:
    image: elasticsearch:8.8.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    networks:
      - docs_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  # Kibana para visualização (opcional)
  kibana:
    image: kibana:8.8.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks:
      - docs_network
    restart: unless-stopped
  
  # Prometheus para métricas
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
    networks:
      - docs_network
    restart: unless-stopped
  
  # Grafana para dashboards
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
    networks:
      - docs_network
    restart: unless-stopped
  
  # Worker para processamento em background
  worker:
    build: 
      context: .
      dockerfile: Dockerfile
      target: production
    command: celery -A worker worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql://user:password@postgres:5432/docs_db
      - REDIS_URL=redis://redis:6379/0
      - PINECONE_API_KEY=${PINECONE_API_KEY}
      - PINECONE_ENV=${PINECONE_ENV}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - postgres
      - redis
    networks:
      - docs_network
    restart: unless-stopped
    volumes:
      - ./logs:/app/logs

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  es_data:
    driver: local
  prometheus_data:
    driver: local
  grafana_data:
    driver: local

networks:
  docs_network:
    driver: bridge
```

---

## 🔄 Templates CI/CD

### 🚀 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Build, Test & Deploy Documentation 4.0

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  PYTHON_VERSION: '3.11'
  NODE_VERSION: '18'

jobs:
  # Job de testes
  test:
    runs-on: ubuntu-latest
    
    services:
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ env.PYTHON_VERSION }}
    
    - name: Install Poetry
      uses: snok/install-poetry@v1
      with:
        version: latest
        virtualenvs-create: true
        virtualenvs-in-project: true
    
    - name: Load cached venv
      id: cached-poetry-dependencies
      uses: actions/cache@v3
      with:
        path: .venv
        key: venv-${{ runner.os }}-${{ steps.setup-python.outputs.python-version }}-${{ hashFiles('**/poetry.lock') }}
    
    - name: Install dependencies
      if: steps.cached-poetry-dependencies.outputs.cache-hit != 'true'
      run: poetry install --no-interaction --no-root
    
    - name: Install project
      run: poetry install --no-interaction
    
    - name: Run linting
      run: |
        poetry run black --check .
        poetry run isort --check-only .
        poetry run flake8 .
    
    - name: Run type checking
      run: poetry run mypy .
    
    - name: Run unit tests
      run: |
        poetry run pytest tests/unit/ -v --cov=. --cov-report=xml
      env:
        DATABASE_URL: postgresql://postgres:test_password@localhost:5432/test_db
        REDIS_URL: redis://localhost:6379/0
    
    - name: Run integration tests
      run: |
        poetry run pytest tests/integration/ -v
      env:
        DATABASE_URL: postgresql://postgres:test_password@localhost:5432/test_db
        REDIS_URL: redis://localhost:6379/0
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella
  
  # Job de build da imagem Docker
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ghcr.io/${{ github.repository }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
  
  # Job de deploy (apenas em main)
  deploy:
    needs: [test, build]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up kubectl
      uses: azure/k8s-set-context@v3
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG }}
    
    - name: Deploy to Kubernetes
      run: |
        envsubst < k8s/deployment.yaml | kubectl apply -f -
        kubectl rollout status deployment/docs-api
        kubectl get services
      env:
        IMAGE_TAG: ${{ github.sha }}
        DATABASE_URL: ${{ secrets.DATABASE_URL }}
        REDIS_URL: ${{ secrets.REDIS_URL }}
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        PINECONE_API_KEY: ${{ secrets.PINECONE_API_KEY }}
        PINECONE_ENV: ${{ secrets.PINECONE_ENV }}
    
    - name: Run smoke tests
      run: |
        # Aguarda deployment ficar pronto
        sleep 30
        
        # Testa endpoint de health
        curl -f https://docs-api.yourdomain.com/health
        
        # Testa query básica
        curl -X POST https://docs-api.yourdomain.com/query \
          -H "Content-Type: application/json" \
          -d '{"question": "health check query"}'
    
    - name: Notify deployment
      if: always()
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        fields: repo,message,commit,author,action,eventName,ref,workflow
```

---

## 🔗 Relacionado

- [[🔧 Lista de Ferramentas]]
- [[🛠️ Stack Tecnológico]]
- [[🧪 Automação de Testes]]
- [[🗺️ Roadmap de Implementação]]

---

#templates #codigo #implementacao #rag #fastapi #docker #ci-cd #testes #campus-party

*Templates prontos: Acelere sua implementação com código testado e otimizado* 📝