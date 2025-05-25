# Project Instructions - Campus Party 2025 Documentação 4.0

## 🎯 Meta
Criar `presentation_slides.html` completa com Reveal.js usando conteúdo do `campus_digest.txt`

## 🚀 Solução Context Window (CRÍTICA)
**Problema**: 33 arquivos MD excedem context window
**Solução**: GitIngest já processou → `campus_digest.txt` (165k tokens)
**Abordagem**: Ler digest único em vez de múltiplos arquivos

## 🛠️ MCP Servers Essenciais
1. **Context7**: Documentação atualizada Reveal.js + Mermaid
2. **Desktop Commander**: Ler digest + criar HTML + servir localhost
3. **Continuity**: Tracking progresso do projeto

## 📋 Processo Otimizado

### 1. Preparação com Digest
```bash
# ✅ FAZER: Ler digest único (165k tokens)
read_file: /Users/lucascardoso/apps/MCP/MCP_OBSIDIAN/MCP_OBSIDIAN/CAMPUS/campus_digest.txt

# ❌ NÃO FAZER: read_multiple_files (excede context)
# ❌ NÃO FAZER: Ler arquivos individuais

# ✅ Consultar docs atualizadas 
resolve-library-id: "reveal.js"
get-library-docs: context7_id_reveal
resolve-library-id: "mermaid"  
get-library-docs: context7_id_mermaid
```

### 2. Extração Estruturada do Digest
O `campus_digest.txt` contém seções claras:
- **File: arquivo.md** marca início de cada arquivo
- **8 diagramas Mermaid** prontos para renderização
- **16+ blocos código** Python/YAML funcionais
- **Roteiro completo** 60min cronometrado
- **Cases reais** com ROI $200K economia

### 3. Implementação HTML
```html
<!DOCTYPE html>
<html>
<head>
    <title>🚀 Documentação 4.0 na Era IA - Campus Party 2025</title>
    <link rel="stylesheet" href="https://unpkg.com/reveal.js@4.3.1/dist/reveal.css">
    <link rel="stylesheet" href="https://unpkg.com/reveal.js@4.3.1/dist/theme/black.css">
    <style>
        :root {
            --cp-blue: #0066CC;
            --cp-dark: #1a1a1a;
            --cp-accent: #00A8FF;
        }
        .reveal h1, .reveal h2 { color: var(--cp-blue); }
    </style>
</head>
<body>
    <div class="reveal">
        <div class="slides">
            <!-- 8 seções verticais extraídas do digest -->
        </div>
    </div>
    <script src="https://unpkg.com/reveal.js@4.3.1/dist/reveal.js"></script>
    <script src="https://unpkg.com/reveal.js@4.3.1/plugin/highlight/highlight.js"></script>
    <script src="https://unpkg.com/mermaid@10.6.1/dist/mermaid.min.js"></script>
</body>
</html>
```

### 4. Estrutura de Slides (do digest)
**Seção 1: Abertura** (slides 1-5)
- Título: Apresentadores REAIS (Áulus + Lucas)
- Agenda 60min baseada no roteiro completo
- Hook: "Quantos perderam horas procurando docs?"
- Poll inicial engajamento
- Transição evolução

**Seção 2: Evolução** (slides 6-13) 
- Timeline Doc 1.0→4.0 (do Evolucao_Documentacao.md)
- Problemas era manual → soluções IA
- Estatísticas comparativas
- Contexto histórico visual

**Seção 3: Doc 4.0** (slides 14-23)
- Definição (do Documentacao_40_Definicao.md)
- 4 pilares com diagramas Mermaid
- Características únicas
- Ciclo vida automatizado
- Stack tecnológico completo

**Seção 4: Arquiteturas** (slides 24-38)
- RAG completo (do RAG_Architecture.md)
- Código Python funcional (do RAG_Implementation.md)  
- Agentes IA (do Agentes_IA.md)
- Pipeline qualidade (do Pipeline_Qualidade.md)
- Stack detalhado (do Stack_Tecnologico.md)

**Seção 5: Implementação** (slides 39-50)
- Roadmap 4 fases (do Roadmap_Implementacao.md)
- CI/CD pipeline (do CI_CD_Pipeline.md)
- Automação testes (do Automacao_Testes.md)
- Templates e código (do Templates_Codigo.md)

**Seção 6: Cases/ROI** (slides 51-58)
- Case API (do Case_API_Documentation.md)
- Case Knowledge Base (do Case_Knowledge_Base.md)
- ROI $200K (do ROI_Metricas.md)
- Métricas before/after

**Seção 7: Próximos Passos** (slides 59-63)
- Como começar (do FAQ_Tecnico.md)
- Recursos (do Ferramentas_Lista.md)
- Timeline realista
- Miro Board (do Miro_Board_Guide.md)

**Seção 8: Q&A** (slides 64-66)
- FAQ principais (do FAQ_Tecnico.md)
- Contatos reais (do Contatos_Referencias.md)
- Recursos complementares

### 5. Componentes Especiais
**Diagramas Mermaid**: Todos 8 do 06_Mermaid/ estão no digest
**Códigos Python**: Blocos executáveis dos RAG_Implementation
**Interatividade**: Polls e demos do Recursos_Interativos.md
**Roteiro**: Timeline preciso do Roteiro_Apresentacao.md

### 6. Chunking se Necessário
```bash
# Se HTML > 50 linhas, usar chunking:
write_file: presentation_slides.html, chunk1, mode="rewrite"
write_file: presentation_slides.html, chunk2, mode="append"
write_file: presentation_slides.html, chunk3, mode="append"
```

### 7. Servir e Testar
```bash
# Servir localhost
execute_command: "cd /Users/lucascardoso/apps/MCP/MCP_OBSIDIAN/MCP_OBSIDIAN/CAMPUS/ && python -m http.server 8000"

# Testar funcionalidade
puppeteer_navigate: "http://localhost:8000/presentation_slides.html"
puppeteer_screenshot: "presentation_test"
# Validar navegação, códigos, diagramas
```

## 🎨 Design Campus Party
```css
/* Extraído do digest - tema consistente */
:root {
  --cp-blue: #0066CC;      /* Azul Campus Party */
  --cp-dark: #1a1a1a;     /* Background escuro */
  --cp-accent: #00A8FF;   /* Accent azul claro */
  --cp-text: #ffffff;     /* Texto branco */
}
```

## 🚨 Restrições Críticas
- **Usar APENAS digest**: Não ler arquivos individuais
- **ZERO ficção**: Só apresentadores Áulus e Lucas
- **Códigos funcionais**: Extrair exemplos reais do digest
- **Links verificados**: Apenas GitHub e LinkedIn confirmados
- **Métricas reais**: ROI baseado em dados do digest

## ✅ Critérios Sucesso
- [ ] 60+ slides extraídos do digest
- [ ] Navegação Reveal.js funcionando
- [ ] 8 diagramas Mermaid renderizados  
- [ ] 16+ códigos executáveis
- [ ] Apresentadores reais creditados
- [ ] Servindo em localhost:8000
- [ ] Performance < 3s carregamento

## 🔧 Comandos Práticos
```bash
# Workflow completo
read_file: campus_digest.txt
resolve-library-id: "reveal.js" + get-library-docs
resolve-library-id: "mermaid" + get-library-docs  
write_file: presentation_slides.html (chunked)
execute_command: python -m http.server 8000
puppeteer_navigate + screenshot para validar
```

**Resultado**: Apresentação HTML completa com TODO conteúdo dos 33 arquivos via digest único, pronta para Campus Party 2025! 🚀