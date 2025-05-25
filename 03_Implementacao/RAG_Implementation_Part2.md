# 🔧 Implementação RAG com Python - Parte 2

> Continuação da implementação completa do sistema RAG

---

## 🤖 Generation System

### 🧠 LLM Response Generator

```python
# src/core/generator.py
from langchain.llms import OpenAI
from langchain.chat_models import ChatOpenAI
from langchain.schema import HumanMessage, SystemMessage, AIMessage
from langchain.prompts import PromptTemplate, ChatPromptTemplate
from typing import List, Dict, Optional, Any
import json
import time
from datetime import datetime

class ResponseGenerator:
    """Gerador de respostas contextualizado para documentação"""
    
    def __init__(self, 
                 model_name: str = "gpt-4",
                 temperature: float = 0.1,
                 max_tokens: int = 1000):
        
        self.llm = ChatOpenAI(
            model_name=model_name,
            temperature=temperature,
            max_tokens=max_tokens
        )
        
        self.prompt_templates = self._load_prompt_templates()
        self.response_cache = {}
        
    def generate_response(self, 
                         query: str,
                         context_documents: List[Dict],
                         response_type: str = "general",
                         user_context: Optional[Dict] = None) -> Dict[str, Any]:
        """Gera resposta contextualizada"""
        
        # Verifica cache
        cache_key = self._generate_cache_key(query, context_documents, response_type)
        if cache_key in self.response_cache:
            cached_response = self.response_cache[cache_key]
            cached_response['from_cache'] = True
            return cached_response
        
        # Seleciona template apropriado
        template = self.prompt_templates.get(response_type, self.prompt_templates['general'])
        
        # Prepara contexto
        formatted_context = self._format_context_documents(context_documents)
        
        # Cria prompt
        prompt = template.format(
            query=query,
            context=formatted_context,
            user_context=self._format_user_context(user_context),
            timestamp=datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        )
        
        # Gera resposta
        start_time = time.time()
        
        messages = [
            SystemMessage(content=self._get_system_message(response_type)),
            HumanMessage(content=prompt)
        ]
        
        try:
            ai_response = self.llm(messages)
            generation_time = time.time() - start_time
            
            # Estrutura resposta
            response = {
                'answer': ai_response.content,
                'sources': self._extract_source_info(context_documents),
                'confidence': self._calculate_confidence(ai_response.content, context_documents),
                'generation_time': round(generation_time, 2),
                'response_type': response_type,
                'timestamp': datetime.now().isoformat(),
                'from_cache': False
            }
            
            # Adiciona ao cache
            self.response_cache[cache_key] = response
            
            # Limita tamanho do cache
            if len(self.response_cache) > 100:
                # Remove entradas mais antigas
                oldest_key = min(self.response_cache.keys(), 
                               key=lambda k: self.response_cache[k]['timestamp'])
                del self.response_cache[oldest_key]
            
            return response
            
        except Exception as e:
            return {
                'answer': f"Desculpe, ocorreu um erro ao gerar a resposta: {str(e)}",
                'sources': [],
                'confidence': 0.0,
                'generation_time': time.time() - start_time,
                'error': str(e),
                'from_cache': False
            }
    
    def _load_prompt_templates(self) -> Dict[str, str]:
        """Carrega templates de prompt especializados"""
        
        return {
            'general': """
Você é um assistente especializado em documentação técnica. Use APENAS as informações fornecidas no contexto para responder à pergunta.

Contexto:
{context}

Pergunta: {query}

Contexto do usuário: {user_context}

Instruções:
1. Responda baseado APENAS no contexto fornecido
2. Se a informação não estiver no contexto, diga "Não encontrei essa informação na documentação fornecida"
3. Cite as fontes específicas quando possível
4. Forneça exemplos práticos quando disponíveis
5. Use formatação markdown para melhor legibilidade
6. Seja preciso e direto
7. Se houver código, inclua explicações claras

Resposta:""",

            'api': """
Você é um especialista em APIs. Use o contexto fornecido para explicar endpoints, parâmetros e exemplos de uso.

Contexto da API:
{context}

Pergunta sobre API: {query}

Contexto do usuário: {user_context}

Instruções específicas para APIs:
1. Inclua método HTTP, endpoint e parâmetros
2. Forneça exemplos de request/response quando possível
3. Explique códigos de status relevantes
4. Mencione autenticação se aplicável
5. Use formato markdown para código
6. Cite a documentação oficial como fonte

Resposta detalhada:""",

            'tutorial': """
Você é um instrutor técnico. Crie um tutorial passo-a-passo baseado no contexto fornecido.

Contexto para tutorial:
{context}

Solicitação de tutorial: {query}

Contexto do usuário: {user_context}

Instruções para tutorial:
1. Organize em passos numerados claros
2. Inclua pré-requisitos se mencionados no contexto
3. Forneça exemplos de código completos
4. Adicione dicas e avisos importantes
5. Termine com próximos passos ou recursos adicionais
6. Use linguagem clara e acessível

Tutorial passo-a-passo:""",

            'troubleshooting': """
Você é um especialista em resolução de problemas técnicos. Use o contexto para diagnosticar e resolver issues.

Contexto de troubleshooting:
{context}

Problema relatado: {query}

Contexto do usuário: {user_context}

Instruções para troubleshooting:
1. Identifique possíveis causas baseadas no contexto
2. Forneça soluções ordenadas por probabilidade
3. Inclua comandos ou código para diagnóstico
4. Mencione como prevenir o problema no futuro
5. Se não houver solução no contexto, sugira próximos passos
6. Use formatação clara com listas e código

Diagnóstico e solução:"""
        }
    
    def _get_system_message(self, response_type: str) -> str:
        """Retorna mensagem de sistema baseada no tipo de resposta"""
        
        base_message = """Você é um assistente especializado em documentação técnica. 
        Suas principais qualidades são:
        - Precisão: Responda apenas com informações verificáveis no contexto
        - Clareza: Use linguagem clara e bem estruturada
        - Praticidade: Forneça exemplos e soluções aplicáveis
        - Honestidade: Admita quando não há informação suficiente"""
        
        type_specific = {
            'api': " Você tem expertise particular em APIs REST, endpoints e integrações.",
            'tutorial': " Você é excelente em criar tutoriais passo-a-passo claros e práticos.",
            'troubleshooting': " Você é especialista em diagnóstico e resolução de problemas técnicos.",
            'general': " Você pode ajudar com qualquer aspecto da documentação técnica."
        }
        
        return base_message + type_specific.get(response_type, type_specific['general'])
    
    def _format_context_documents(self, context_documents: List[Dict]) -> str:
        """Formata documentos de contexto para o prompt"""
        
        if not context_documents:
            return "Nenhum contexto específico fornecido."
        
        formatted_context = []
        
        for i, doc_info in enumerate(context_documents, 1):
            doc_content = doc_info.get('content', '')
            doc_source = doc_info.get('source', f'Documento {i}')
            doc_type = doc_info.get('type', 'documento')
            
            formatted_doc = f"""
Fonte {i}: {doc_source} ({doc_type})
---
{doc_content}
---
"""
            formatted_context.append(formatted_doc)
        
        return '\n'.join(formatted_context)
    
    def _format_user_context(self, user_context: Optional[Dict]) -> str:
        """Formata contexto do usuário"""
        
        if not user_context:
            return "Usuário geral"
        
        context_parts = []
        
        if user_context.get('role'):
            context_parts.append(f"Função: {user_context['role']}")
        
        if user_context.get('experience_level'):
            context_parts.append(f"Nível: {user_context['experience_level']}")
        
        if user_context.get('preferred_language'):
            context_parts.append(f"Linguagem: {user_context['preferred_language']}")
        
        if user_context.get('use_case'):
            context_parts.append(f"Caso de uso: {user_context['use_case']}")
        
        return ', '.join(context_parts) if context_parts else "Usuário geral"
    
    def _extract_source_info(self, context_documents: List[Dict]) -> List[Dict]:
        """Extrai informações das fontes para citação"""
        
        sources = []
        
        for doc_info in context_documents:
            source_info = {
                'title': doc_info.get('source', 'Documento'),
                'type': doc_info.get('type', 'documentation'),
                'url': doc_info.get('url', ''),
                'section': doc_info.get('section', ''),
                'relevance_score': doc_info.get('score', 0.0)
            }
            sources.append(source_info)
        
        # Ordena por relevância
        sources.sort(key=lambda x: x['relevance_score'], reverse=True)
        
        return sources
    
    def _calculate_confidence(self, response: str, context_documents: List[Dict]) -> float:
        """Calcula confidence score da resposta"""
        
        if not response or not context_documents:
            return 0.0
        
        # Fatores de confidence
        factors = {
            'context_quality': self._assess_context_quality(context_documents),
            'response_completeness': self._assess_response_completeness(response),
            'source_alignment': self._assess_source_alignment(response, context_documents),
            'specificity': self._assess_response_specificity(response)
        }
        
        # Pesos dos fatores
        weights = {
            'context_quality': 0.3,
            'response_completeness': 0.3,
            'source_alignment': 0.2,
            'specificity': 0.2
        }
        
        # Calcula confidence final
        confidence = sum(factors[factor] * weights[factor] for factor in factors)
        
        return round(min(confidence, 1.0), 2)
    
    def _assess_context_quality(self, context_documents: List[Dict]) -> float:
        """Avalia qualidade do contexto fornecido"""
        
        if not context_documents:
            return 0.0
        
        # Número de documentos (mais contexto = melhor)
        doc_count_score = min(len(context_documents) / 5.0, 1.0)
        
        # Diversidade de tipos de documento
        doc_types = set(doc.get('type', 'unknown') for doc in context_documents)
        diversity_score = min(len(doc_types) / 3.0, 1.0)
        
        # Relevância média dos documentos
        relevance_scores = [doc.get('score', 0.5) for doc in context_documents]
        avg_relevance = sum(relevance_scores) / len(relevance_scores)
        
        # Score final
        quality_score = (doc_count_score * 0.3 + 
                        diversity_score * 0.3 + 
                        avg_relevance * 0.4)
        
        return quality_score
    
    def _assess_response_completeness(self, response: str) -> float:
        """Avalia completude da resposta"""
        
        if not response:
            return 0.0
        
        # Comprimento da resposta
        word_count = len(response.split())
        length_score = min(word_count / 200.0, 1.0)  # 200 palavras = score máximo
        
        # Presença de estrutura
        structure_indicators = ['#', '*', '1.', '-', '```']
        structure_score = sum(1 for indicator in structure_indicators if indicator in response) / len(structure_indicators)
        
        # Presença de exemplos/código
        has_examples = 1.0 if '```' in response or 'exemplo' in response.lower() else 0.5
        
        completeness = (length_score * 0.4 + structure_score * 0.3 + has_examples * 0.3)
        
        return completeness
    
    def _assess_source_alignment(self, response: str, context_documents: List[Dict]) -> float:
        """Avalia alinhamento da resposta com as fontes"""
        
        if not response or not context_documents:
            return 0.0
        
        response_lower = response.lower()
        alignment_scores = []
        
        for doc in context_documents:
            doc_content = doc.get('content', '').lower()
            
            # Calcula overlap de termos significativos
            response_terms = set(word for word in response_lower.split() if len(word) > 4)
            context_terms = set(word for word in doc_content.split() if len(word) > 4)
            
            if context_terms:
                overlap = len(response_terms.intersection(context_terms))
                alignment = overlap / len(context_terms)
                alignment_scores.append(alignment)
        
        return sum(alignment_scores) / len(alignment_scores) if alignment_scores else 0.0
    
    def _assess_response_specificity(self, response: str) -> float:
        """Avalia especificidade da resposta"""
        
        if not response:
            return 0.0
        
        # Indicadores de especificidade
        specific_indicators = [
            'exemplo', 'código', 'comando', 'endpoint', 'parâmetro',
            'função', 'método', 'classe', 'arquivo', 'diretório'
        ]
        
        response_lower = response.lower()
        specificity_count = sum(1 for indicator in specific_indicators if indicator in response_lower)
        
        # Presença de código
        code_boost = 0.3 if '```' in response else 0.0
        
        # URLs ou referências específicas
        reference_boost = 0.2 if 'http' in response or '@' in response else 0.0
        
        specificity = min((specificity_count / len(specific_indicators)) + code_boost + reference_boost, 1.0)
        
        return specificity
    
    def _generate_cache_key(self, query: str, context_documents: List[Dict], response_type: str) -> str:
        """Gera chave de cache para a consulta"""
        
        import hashlib
        
        # Cria hash baseado na query, contexto e tipo
        context_content = ''.join([doc.get('content', '')[:100] for doc in context_documents])
        cache_string = f"{query}_{context_content}_{response_type}"
        
        return hashlib.md5(cache_string.encode()).hexdigest()
    
    def generate_code_example(self, 
                            description: str, 
                            language: str,
                            context_documents: List[Dict]) -> Dict[str, Any]:
        """Gera exemplo de código específico"""
        
        code_prompt = f"""
Baseado no contexto fornecido, gere um exemplo de código em {language} para: {description}

Contexto:
{self._format_context_documents(context_documents)}

Instruções:
1. Gere código funcional e completo
2. Inclua comentários explicativos
3. Adicione tratamento de erros quando apropriado
4. Use melhores práticas da linguagem
5. Forneça explicação do código após o exemplo

Exemplo de código:"""
        
        messages = [
            SystemMessage(content="Você é um especialista em programação que gera código limpo e bem documentado."),
            HumanMessage(content=code_prompt)
        ]
        
        try:
            response = self.llm(messages)
            
            return {
                'code': response.content,
                'language': language,
                'description': description,
                'sources': self._extract_source_info(context_documents),
                'confidence': 0.8  # Código gerado tem confidence específico
            }
        except Exception as e:
            return {
                'code': f"// Erro ao gerar código: {str(e)}",
                'language': language,
                'description': description,
                'error': str(e),
                'confidence': 0.0
            }
    
    def get_generation_stats(self) -> Dict[str, Any]:
        """Retorna estatísticas do gerador"""
        
        return {
            'cache_size': len(self.response_cache),
            'available_templates': list(self.prompt_templates.keys()),
            'model_name': self.llm.model_name,
            'temperature': self.llm.temperature
        }
```

---

## 🔗 Complete RAG System Integration

### 🎭 Main RAG Orchestrator

```python
# src/rag_system.py
from src.utils.document_loader import DocumentationLoader
from src.utils.preprocessor import DocumentPreprocessor
from src.core.embeddings import EmbeddingManager
from src.core.vector_store import VectorStoreManager
from src.core.retriever import AdvancedRetriever
from src.core.generator import ResponseGenerator
from config.settings import settings
from typing import List, Dict, Optional, Any
import logging
from pathlib import Path

logger = logging.getLogger(__name__)

class DocumentationRAGSystem:
    """Sistema RAG completo para documentação"""
    
    def __init__(self):
        self.doc_loader = None
        self.preprocessor = None
        self.embedding_manager = None
        self.vector_store = None
        self.retriever = None
        self.generator = None
        self.is_initialized = False
        
    async def initialize(self, documents_directory: Optional[str] = None):
        """Inicializa todos os componentes do sistema RAG"""
        
        logger.info("Initializing RAG system...")
        
        try:
            # 1. Inicializa componentes base
            self.doc_loader = DocumentationLoader()
            self.preprocessor = DocumentPreprocessor(
                chunk_size=settings.CHUNK_SIZE,
                chunk_overlap=settings.CHUNK_OVERLAP
            )
            
            # 2. Inicializa embedding manager
            self.embedding_manager = EmbeddingManager(
                primary_model=settings.EMBEDDING_MODEL,
                cache_dir=settings.EMBEDDINGS_CACHE_DIR
            )
            
            # 3. Inicializa vector store
            self.vector_store = VectorStoreManager(
                store_type=settings.VECTOR_DB_TYPE,
                collection_name=settings.COLLECTION_NAME
            )
            
            # 4. Carrega documentos se diretório fornecido
            if documents_directory:
                await self._load_and_process_documents(documents_directory)
            
            # 5. Inicializa retriever e generator
            self.retriever = AdvancedRetriever(self.vector_store)
            self.generator = ResponseGenerator(
                model_name=settings.LLM_MODEL,
                temperature=settings.TEMPERATURE,
                max_tokens=settings.MAX_TOKENS
            )
            
            self.is_initialized = True
            logger.info("RAG system initialized successfully")
            
        except Exception as e:
            logger.error(f"Error initializing RAG system: {e}")
            raise
    
    async def _load_and_process_documents(self, documents_directory: str):
        """Carrega e processa documentos"""
        
        logger.info(f"Loading documents from {documents_directory}")
        
        # Carrega documentos
        documents = self.doc_loader.load_documents(documents_directory)
        
        if not documents:
            logger.warning("No documents found to load")
            return
        
        # Processa documentos
        processed_docs = self.preprocessor.process_documents(documents)
        logger.info(f"Processed {len(processed_docs)} document chunks")
        
        # Inicializa vector store com documentos
        self.vector_store.initialize_store(self.embedding_manager, processed_docs)
        
        logger.info("Documents loaded and indexed successfully")
    
    async def add_documents(self, documents_directory: str):
        """Adiciona novos documentos ao sistema"""
        
        if not self.is_initialized:
            raise ValueError("RAG system not initialized")
        
        # Carrega novos documentos
        new_documents = self.doc_loader.load_documents(documents_directory)
        
        if not new_documents:
            return {"message": "No new documents found"}
        
        # Processa novos documentos
        processed_docs = self.preprocessor.process_documents(new_documents)
        
        # Adiciona ao vector store
        added_ids = self.vector_store.add_documents(processed_docs)
        
        return {
            "documents_added": len(processed_docs),
            "document_ids": added_ids
        }
    
    async def query(self, 
                   question: str,
                   filters: Optional[Dict] = None,
                   user_context: Optional[Dict] = None,
                   response_type: str = "general") -> Dict[str, Any]:
        """Executa query completa no sistema RAG"""
        
        if not self.is_initialized:
            raise ValueError("RAG system not initialized")
        
        try:
            # 1. Retrieval
            relevant_docs = self.retriever.retrieve(
                query=question,
                k=settings.RETRIEVAL_TOP_K,
                filters=filters,
                user_id=user_context.get('user_id') if user_context else None
            )
            
            if not relevant_docs:
                return {
                    'answer': "Não encontrei informações relevantes na documentação para responder sua pergunta.",
                    'sources': [],
                    'confidence': 0.0,
                    'retrieval_count': 0
                }
            
            # 2. Prepara contexto para geração
            context_documents = []
            for doc in relevant_docs:
                context_doc = {
                    'content': doc.page_content,
                    'source': doc.metadata.get('source', 'Unknown'),
                    'type': doc.metadata.get('source_type', 'documentation'),
                    'section': doc.metadata.get('main_topic', ''),
                    'score': doc.metadata.get('relevance_score', 0.8)
                }
                context_documents.append(context_doc)
            
            # 3. Generation
            response = self.generator.generate_response(
                query=question,
                context_documents=context_documents,
                response_type=response_type,
                user_context=user_context
            )
            
            # 4. Adiciona informações de retrieval
            response['retrieval_count'] = len(relevant_docs)
            response['query'] = question
            response['filters_applied'] = filters or {}
            
            return response
            
        except Exception as e:
            logger.error(f"Error processing query: {e}")
            return {
                'answer': f"Ocorreu um erro ao processar sua pergunta: {str(e)}",
                'sources': [],
                'confidence': 0.0,
                'error': str(e)
            }
    
    async def generate_api_documentation(self, openapi_spec: Dict) -> Dict[str, Any]:
        """Gera documentação para API baseada em especificação OpenAPI"""
        
        if not self.is_initialized:
            raise ValueError("RAG system not initialized")
        
        # Carrega especificação da API
        api_docs = self.doc_loader.load_api_documentation(openapi_spec)
        
        # Processa documentos da API
        processed_docs = self.preprocessor.process_documents(api_docs)
        
        # Adiciona ao vector store
        self.vector_store.add_documents(processed_docs)
        
        # Gera documentação estruturada
        documentation_sections = []
        
        for endpoint_path, methods in openapi_spec.get('paths', {}).items():
            for method, spec in methods.items():
                query = f"Como usar o endpoint {method.upper()} {endpoint_path}? Inclua exemplos completos."
                
                response = await self.query(
                    question=query,
                    response_type="api",
                    filters={'source_type': 'api'}
                )
                
                documentation_sections.append({
                    'endpoint': f"{method.upper()} {endpoint_path}",
                    'documentation': response['answer'],
                    'confidence': response['confidence']
                })
        
        return {
            'api_documentation': documentation_sections,
            'total_endpoints': len(documentation_sections),
            'openapi_version': openapi_spec.get('openapi', '3.0.0')
        }
    
    async def create_tutorial(self, topic: str, user_level: str = "intermediate") -> Dict[str, Any]:
        """Cria tutorial sobre um tópico específico"""
        
        tutorial_query = f"Crie um tutorial completo sobre {topic} para usuário {user_level}"
        
        response = await self.query(
            question=tutorial_query,
            response_type="tutorial",
            user_context={'experience_level': user_level}
        )
        
        return {
            'tutorial_topic': topic,
            'target_level': user_level,
            'content': response['answer'],
            'sources': response['sources'],
            'confidence': response['confidence']
        }
    
    async def troubleshoot_issue(self, problem_description: str) -> Dict[str, Any]:
        """Ajuda a resolver problemas técnicos"""
        
        troubleshoot_query = f"Como resolver este problema: {problem_description}"
        
        response = await self.query(
            question=troubleshoot_query,
            response_type="troubleshooting",
            filters={'content_type': ['troubleshooting', 'faq', 'documentation']}
        )
        
        return {
            'problem': problem_description,
            'solution': response['answer'],
            'sources': response['sources'],
            'confidence': response['confidence']
        }
    
    def get_system_stats(self) -> Dict[str, Any]:
        """Retorna estatísticas completas do sistema"""
        
        stats = {
            'initialized': self.is_initialized,
            'settings': {
                'chunk_size': settings.CHUNK_SIZE,
                'chunk_overlap': settings.CHUNK_OVERLAP,
                'retrieval_top_k': settings.RETRIEVAL_TOP_K,
                'llm_model': settings.LLM_MODEL,
                'embedding_model': settings.EMBEDDING_MODEL
            }
        }
        
        if self.is_initialized:
            stats.update({
                'embedding_cache': self.embedding_manager.get_cache_stats(),
                'vector_store': self.vector_store.get_collection_stats(),
                'retrieval': self.retriever.get_retrieval_stats(),
                'generation': self.generator.get_generation_stats()
            })
        
        return stats
    
    async def health_check(self) -> Dict[str, Any]:
        """Verifica saúde do sistema"""
        
        health = {
            'status': 'healthy',
            'components': {},
            'issues': []
        }
        
        # Verifica inicialização
        if not self.is_initialized:
            health['status'] = 'unhealthy'
            health['issues'].append('System not initialized')
            return health
        
        # Verifica componentes
        try:
            # Testa embedding
            test_embedding = self.embedding_manager.embed_query("test")
            health['components']['embeddings'] = 'ok' if test_embedding else 'error'
        except Exception as e:
            health['components']['embeddings'] = 'error'
            health['issues'].append(f'Embeddings error: {str(e)}')
        
        try:
            # Testa vector store
            vector_stats = self.vector_store.get_collection_stats()
            health['components']['vector_store'] = 'ok' if vector_stats else 'error'
        except Exception as e:
            health['components']['vector_store'] = 'error'
            health['issues'].append(f'Vector store error: {str(e)}')
        
        try:
            # Testa retrieval
            test_results = self.retriever.retrieve("test query", k=1)
            health['components']['retrieval'] = 'ok'
        except Exception as e:
            health['components']['retrieval'] = 'error'
            health['issues'].append(f'Retrieval error: {str(e)}')
        
        # Determina status geral
        if health['issues']:
            health['status'] = 'degraded' if len(health['issues']) < 2 else 'unhealthy'
        
        return health

# Instância global do sistema RAG
rag_system = DocumentationRAGSystem()
```

---

## 🌐 API REST Interface

### 🔌 FastAPI Implementation

```python
# src/api/rag_api.py
from fastapi import FastAPI, HTTPException, UploadFile, File, Form
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field
from typing import List, Dict, Optional, Any
import uvicorn
import os
import tempfile
import shutil
from pathlib import Path

from src.rag_system import rag_system

app = FastAPI(
    title="Documentation RAG API",
    description="API para sistema RAG de documentação técnica",
    version="1.0.0"
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configure conforme necessário
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Modelos Pydantic
class QueryRequest(BaseModel):
    question: str = Field(..., description="Pergunta a ser respondida")
    filters: Optional[Dict] = Field(None, description="Filtros para busca")
    user_context: Optional[Dict] = Field(None, description="Contexto do usuário")
    response_type: str = Field("general", description="Tipo de resposta")

class QueryResponse(BaseModel):
    answer: str
    sources: List[Dict]
    confidence: float
    retrieval_count: int
    generation_time: float
    query: str

class TutorialRequest(BaseModel):
    topic: str = Field(..., description="Tópico do tutorial")
    user_level: str = Field("intermediate", description="Nível do usuário")

class APIDocRequest(BaseModel):
    openapi_spec: Dict = Field(..., description="Especificação OpenAPI")

class TroubleshootRequest(BaseModel):
    problem_description: str = Field(..., description="Descrição do problema")

# Endpoints principais
@app.get("/")
async def root():
    """Endpoint raiz com informações da API"""
    return {
        "message": "Documentation RAG API",
        "version": "1.0.0",
        "endpoints": {
            "query": "/query",
            "tutorial": "/tutorial",
            "troubleshoot": "/troubleshoot",
            "api_docs": "/generate-api-docs",
            "upload": "/upload-documents",
            "health": "/health",
            "stats": "/stats"
        }
    }

@app.post("/initialize")
async def initialize_system(documents_directory: Optional[str] = None):
    """Inicializa o sistema RAG"""
    try:
        await rag_system.initialize(documents_directory)
        return {"message": "RAG system initialized successfully"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/query", response_model=QueryResponse)
async def query_documentation(request: QueryRequest):
    """Consulta a documentação usando RAG"""
    
    if not rag_system.is_initialized:
        raise HTTPException(status_code=400, detail="RAG system not initialized")
    
    try:
        response = await rag_system.query(
            question=request.question,
            filters=request.filters,
            user_context=request.user_context,
            response_type=request.response_type
        )
        
        return QueryResponse(**response)
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/tutorial")
async def create_tutorial(request: TutorialRequest):
    """Cria tutorial sobre um tópico"""
    
    if not rag_system.is_initialized:
        raise HTTPException(status_code=400, detail="RAG system not initialized")
    
    try:
        tutorial = await rag_system.create_tutorial(
            topic=request.topic,
            user_level=request.user_level
        )
        return tutorial
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/troubleshoot")
async def troubleshoot_issue(request: TroubleshootRequest):
    """Ajuda a resolver problemas técnicos"""
    
    if not rag_system.is_initialized:
        raise HTTPException(status_code=400, detail="RAG system not initialized")
    
    try:
        solution = await rag_system.troubleshoot_issue(request.problem_description)
        return solution
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/generate-api-docs")
async def generate_api_documentation(request: APIDocRequest):
    """Gera documentação para API"""
    
    if not rag_system.is_initialized:
        raise HTTPException(status_code=400, detail="RAG system not initialized")
    
    try:
        api_docs = await rag_system.generate_api_documentation(request.openapi_spec)
        return api_docs
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/upload-documents")
async def upload_documents(files: List[UploadFile] = File(...)):
    """Upload de documentos para o sistema"""
    
    if not rag_system.is_initialized:
        raise HTTPException(status_code=400, detail="RAG system not initialized")
    
    try:
        # Cria diretório temporário
        with tempfile.TemporaryDirectory() as temp_dir:
            uploaded_files = []
            
            # Salva arquivos enviados
            for file in files:
                file_path = Path(temp_dir) / file.filename
                with open(file_path, "wb") as buffer:
                    shutil.copyfileobj(file.file, buffer)
                uploaded_files.append(str(file_path))
            
            # Adiciona documentos ao sistema
            result = await rag_system.add_documents(temp_dir)
            
            return {
                "uploaded_files": len(uploaded_files),
                "processed_documents": result.get("documents_added", 0),
                "message": "Documents uploaded and processed successfully"
            }
            
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    """Verifica saúde do sistema"""
    try:
        health = await rag_system.health_check()
        status_code = 200 if health['status'] == 'healthy' else 503
        return health
    except Exception as e:
        return {
            "status": "error",
            "message": str(e)
        }

@app.get("/stats")
async def get_system_stats():
    """Retorna estatísticas do sistema"""
    try:
        stats = rag_system.get_system_stats()
        return stats
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.delete("/reset")
async def reset_system():
    """Reset completo do sistema (cuidado!)"""
    try:
        # Esta operação deve ser usada com cuidado
        rag_system.vector_store.delete_collection()
        rag_system.is_initialized = False
        
        return {"message": "System reset successfully"}
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Middleware para logging
@app.middleware("http")
async def log_requests(request, call_next):
    import time
    start_time = time.time()
    
    response = await call_next(request)
    
    process_time = time.time() - start_time
    print(f"{request.method} {request.url.path} - {response.status_code} - {process_time:.3f}s")
    
    return response

if __name__ == "__main__":
    uvicorn.run(
        "rag_api:app",
        host="0.0.0.0",
        port=8000,
        reload=True,
        log_level="info"
    )
```

---

## 🧪 Testing & Evaluation

### 📊 RAG System Tests

```python
# tests/test_rag.py
import pytest
import asyncio
import tempfile
import os
from pathlib import Path

from src.rag_system import DocumentationRAGSystem
from src.core.embeddings import EmbeddingManager
from src.utils.document_loader import DocumentationLoader

class TestRAGSystem:
    """Testes para o sistema RAG completo"""
    
    @pytest.fixture
    async def rag_system(self):
        """Fixture para sistema RAG de teste"""
        system = DocumentationRAGSystem()
        
        # Cria documentos de teste
        with tempfile.TemporaryDirectory() as temp_dir:
            test_doc = Path(temp_dir) / "test_doc.md"
            test_doc.write_text("""
# Test Documentation

This is a test document for RAG system.

## API Endpoints

### GET /api/users
Returns list of users.

### POST /api/users
Creates new user.

## Examples

```python
import requests
response = requests.get('/api/users')
print(response.json())
```
""")
            
            await system.initialize(temp_dir)
        
        yield system
    
    @pytest.mark.asyncio
    async def test_system_initialization(self, rag_system):
        """Testa inicialização do sistema"""
        assert rag_system.is_initialized
        assert rag_system.embedding_manager is not None
        assert rag_system.vector_store is not None
        assert rag_system.retriever is not None
        assert rag_system.generator is not None
    
    @pytest.mark.asyncio
    async def test_basic_query(self, rag_system):
        """Testa query básica"""
        response = await rag_system.query("How to get list of users?")
        
        assert response['answer']
        assert 'GET /api/users' in response['answer']
        assert response['confidence'] > 0
        assert len(response['sources']) > 0
    
    @pytest.mark.asyncio
    async def test_api_query(self, rag_system):
        """Testa query específica de API"""
        response = await rag_system.query(
            "How to create a new user?",
            response_type="api"
        )
        
        assert response['answer']
        assert 'POST /api/users' in response['answer']
        assert response['confidence'] > 0
    
    @pytest.mark.asyncio
    async def test_code_example_query(self, rag_system):
        """Testa query por exemplos de código"""
        response = await rag_system.query("Show me Python code example")
        
        assert response['answer']
        assert 'python' in response['answer'].lower()
        assert 'requests' in response['answer']
    
    @pytest.mark.asyncio
    async def test_tutorial_creation(self, rag_system):
        """Testa criação de tutorial"""
        tutorial = await rag_system.create_tutorial("API usage", "beginner")
        
        assert tutorial['tutorial_topic'] == "API usage"
        assert tutorial['target_level'] == "beginner"
        assert len(tutorial['content']) > 100
    
    @pytest.mark.asyncio
    async def test_troubleshooting(self, rag_system):
        """Testa funcionalidade de troubleshooting"""
        solution = await rag_system.troubleshoot_issue("API returns 404 error")
        
        assert solution['problem'] == "API returns 404 error"
        assert len(solution['solution']) > 50
    
    @pytest.mark.asyncio
    async def test_filters(self, rag_system):
        """Testa aplicação de filtros"""
        response = await rag_system.query(
            "API information",
            filters={'content_type': 'api'}
        )
        
        assert response['filters_applied']['content_type'] == 'api'
    
    @pytest.mark.asyncio
    async def test_user_context(self, rag_system):
        """Testa personalização por contexto do usuário"""
        response = await rag_system.query(
            "How to use the API?",
            user_context={
                'role': 'developer',
                'experience_level': 'beginner'
            }
        )
        
        assert response['answer']
        # Resposta deve ser mais detalhada para iniciante
        assert len(response['answer']) > 200
    
    @pytest.mark.asyncio
    async def test_system_stats(self, rag_system):
        """Testa obtenção de estatísticas"""
        stats = rag_system.get_system_stats()
        
        assert stats['initialized'] == True
        assert 'embedding_cache' in stats
        assert 'vector_store' in stats
        assert 'retrieval' in stats
        assert 'generation' in stats
    
    @pytest.mark.asyncio
    async def test_health_check(self, rag_system):
        """Testa verificação de saúde"""
        health = await rag_system.health_check()
        
        assert health['status'] in ['healthy', 'degraded', 'unhealthy']
        assert 'components' in health

class TestEmbeddingManager:
    """Testes para gerenciador de embeddings"""
    
    @pytest.fixture
    def embedding_manager(self):
        return EmbeddingManager()
    
    def test_embed_single_text(self, embedding_manager):
        """Testa embedding de texto único"""
        text = "This is a test document"
        embedding = embedding_manager.embed_query(text)
        
        assert isinstance(embedding, list)
        assert len(embedding) > 0
        assert all(isinstance(x, float) for x in embedding)
    
    def test_embed_multiple_texts(self, embedding_manager):
        """Testa embedding de múltiplos textos"""
        texts = [
            "First test document",
            "Second test document",
            "Third test document"
        ]
        embeddings = embedding_manager.embed_documents(texts)
        
        assert len(embeddings) == len(texts)
        assert all(isinstance(emb, list) for emb in embeddings)
    
    def test_embedding_cache(self, embedding_manager):
        """Testa funcionamento do cache"""
        text = "Cache test document"
        
        # Primeira chamada
        embedding1 = embedding_manager.embed_query(text)
        
        # Segunda chamada (deve usar cache)
        embedding2 = embedding_manager.embed_query(text)
        
        assert embedding1 == embedding2
        
        # Verifica estatísticas do cache
        stats = embedding_manager.get_cache_stats()
        assert stats['cache_size'] > 0

# Função para executar testes
def run_tests():
    """Executa todos os testes"""
    pytest.main([__file__, "-v", "--asyncio-mode=auto"])

if __name__ == "__main__":
    run_tests()
```

---

## 🚀 Deployment & Production

### 🐳 Docker Configuration

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Instala dependências do sistema
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copia arquivos de requisitos
COPY requirements.txt .

# Instala dependências Python
RUN pip install --no-cache-dir -r requirements.txt

# Copia código da aplicação
COPY . .

# Cria diretórios necessários
RUN mkdir -p data/embeddings data/vectorstore logs

# Expõe porta da API
EXPOSE 8000

# Comando padrão
CMD ["uvicorn", "src.api.rag_api:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  rag-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - PINECONE_API_KEY=${PINECONE_API_KEY}
      - PINECONE_ENVIRONMENT=${PINECONE_ENVIRONMENT}
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    restart: unless-stopped
    
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  redis_data:
```

---

## 📊 Performance Monitoring

```python
# src/utils/monitoring.py
import time
import psutil
import logging
from typing import Dict, Any
from dataclasses import dataclass
from datetime import datetime

@dataclass
class PerformanceMetrics:
    timestamp: datetime
    cpu_usage: float
    memory_usage: float
    disk_usage: float
    response_time: float
    query_count: int

class RAGMonitor:
    """Monitor de performance do sistema RAG"""
    
    def __init__(self):
        self.metrics_history = []
        self.query_count = 0
        self.start_time = time.time()
        
    def record_query(self, response_time: float):
        """Registra métricas de uma query"""
        self.query_count += 1
        
        metrics = PerformanceMetrics(
            timestamp=datetime.now(),
            cpu_usage=psutil.cpu_percent(),
            memory_usage=psutil.virtual_memory().percent,
            disk_usage=psutil.disk_usage('/').percent,
            response_time=response_time,
            query_count=self.query_count
        )
        
        self.metrics_history.append(metrics)
        
        # Mantém apenas últimas 1000 métricas
        if len(self.metrics_history) > 1000:
            self.metrics_history = self.metrics_history[-1000:]
    
    def get_performance_summary(self) -> Dict[str, Any]:
        """Retorna resumo de performance"""
        if not self.metrics_history:
            return {"message": "No metrics available"}
        
        recent_metrics = self.metrics_history[-100:]  # Últimas 100
        
        return {
            "uptime_hours": (time.time() - self.start_time) / 3600,
            "total_queries": self.query_count,
            "avg_response_time": sum(m.response_time for m in recent_metrics) / len(recent_metrics),
            "avg_cpu_usage": sum(m.cpu_usage for m in recent_metrics) / len(recent_metrics),
            "avg_memory_usage": sum(m.memory_usage for m in recent_metrics) / len(recent_metrics),
            "queries_per_hour": self.query_count / ((time.time() - self.start_time) / 3600)
        }

# Instância global do monitor
monitor = RAGMonitor()
```

---

## 🔗 Relacionado

- [[🏗️ Componentes Doc 4.0]]
- [[🔍 RAG - Retrieval-Augmented Generation]]
- [[🤖 Agentes IA para Automação]]
- [[🧪 Automação de Testes]]

---

#rag #python #implementacao-completa #langchain #fastapi #docker #testing #monitoring #campus-party

*Implementação RAG completa: Do conceito à produção* 🔧
