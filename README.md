# Plataforma Unificada de Dados & IA (LLM + Big Data)  
**Seiva Digital + NOZES.ia + Herbáceas App**  
Versão: 0.1 (MVP Blueprint) — Janeiro/2026

## 1) Objetivo
Construir uma **plataforma única e robusta** de dados e Inteligência Artificial para suportar, de forma integrada, os projetos:

- **Seiva Digital** (Flutter, offline-first, coleta socioambiental + GPS + fotos)
- **NOZES.ia** (React/TS, geração/edição de chaves de identificação biológica)
- **Herbáceas App** (web, análise de vegetação por imagem com múltiplos modelos)

A plataforma deve entregar:
- **Banco único** (visão unificada e governança) para os três projetos
- **Análise de dados e imagens com IA** (LLMs e VLMs), com rastreabilidade e revisão humana
- **Operação profissional** (segurança, observabilidade, qualidade, versionamento e métricas)

---

## 2) Princípios de Arquitetura
### 2.1 Banco “único” na prática
O “banco único” será uma **Plataforma de Dados** com camadas:
1) **Object Storage** (arquivos): imagens, anexos, exports (CSV/XLSX), versões
2) **PostgreSQL** (dados transacionais + domínio)
3) **Indexação vetorial** (busca semântica / RAG) no próprio Postgres via **pgvector**
4) **Pipelines** (jobs assíncronos + filas) para processar imagem/dados
5) **AI Gateway** (API única) para serviços de IA e integração com apps

### 2.2 Separação por domínios, com “core” compartilhado
- `core.*` — entidades compartilhadas (projetos, equipes, usuários, mídia, auditoria, execuções de IA)
- `seiva.*` — dados de campo, formulários, observações, geoespacial
- `nozes.*` — taxonomia, características, estados, matrizes, chaves
- `herbaceas.*` — parcelas/subparcelas, análises, morfotipos, resultados por imagem

---

## 3) Arquitetura Alvo (Visão Geral)

### 3.1 Componentes
**Apps (Clientes)**
- Seiva Digital (Flutter, offline-first com SQLite)
- NOZES.ia (React/TypeScript)
- Herbáceas App (Web)

**Plataforma (Backend)**
- **Object Storage**: S3 (cloud) ou MinIO (self-hosted)
- **PostgreSQL**: com **PostGIS** (geodados) + **pgvector** (embeddings)
- **Filas/Jobs**: Redis + Celery (ou RQ) para processamento assíncrono
- **AI Gateway (FastAPI)**:
  - ingestão de mídia e dados
  - análise de imagem (VLM)
  - extração estruturada (JSON validado)
  - RAG (busca semântica com vetores)
- **Observabilidade**: Prometheus + Grafana + logs estruturados

### 3.2 Fluxos essenciais
**(A) Upload de imagem → IA → Resultado → Revisão**
1. App envia imagem (ou link presigned) → Object Storage  
2. App registra metadados → Postgres (`core.media`)  
3. Enfileira job → Worker processa (VLM)  
4. Resultado estruturado → Postgres (`core.ai_runs` + tabelas domínio)  
5. Revisão humana (aprovar/corrigir) → Postgres (`core.reviews`)  

**(B) Busca semântica / RAG**
1. Indexação do conteúdo (textos/relatórios/chaves/observações)  
2. Embeddings no pgvector  
3. Consulta semântica (similaridade + filtros por projeto/domínio)  
4. Resposta com rastreabilidade (referências internas)

---

## 4) Stack Recomendada (Profissional e Escalável)

### 4.1 Banco e dados
- **PostgreSQL** (transacional)
- **PostGIS** (geoespacial — Seiva Digital)
- **pgvector** (embeddings e busca semântica — RAG e similaridade)
- (Opcional) **TimescaleDB** se houver telemetria/time-series volumosa

### 4.2 Armazenamento de arquivos
- **S3** (AWS) para produção cloud
- **MinIO** para self-hosted e ambientes locais/híbridos

### 4.3 IA e integração
- **FastAPI** como AI Gateway (API única)
- **LlamaIndex** (recomendado) para indexação/RAG  
  - Alternativa: LangChain (padronizar em um só framework)
- **Modelos**: OpenAI / Gemini / Claude (estratégia multi-provedor com fallback)

### 4.4 Processamento assíncrono
- **Redis + Celery** (ou RQ) para jobs de:
  - análise de imagem
  - geração/atualização de embeddings
  - validações e consistência de dados
  - exportações e relatórios

### 4.5 Observabilidade e segurança
- **Prometheus + Grafana** (métricas)
- Logs estruturados + correlação por `request_id` / `job_id`
- RBAC e trilhas de auditoria no banco

---

## 5) Entregáveis do MVP (Prioridade Máxima)
### MVP = “Backbone comum” + 1 pipeline completo de imagem + RAG básico

**Entrega 1 — Infra base**
- Postgres com extensões PostGIS + pgvector
- Object Storage (S3 ou MinIO)
- Estrutura de schemas: `core`, `seiva`, `nozes`, `herbaceas`

**Entrega 2 — AI Gateway**
Endpoints mínimos:
- `POST /ingest/media` — registra mídia + enfileira processamento
- `POST /ai/analyze-image` — força análise imediata (debug/admin)
- `POST /ai/extract-structured` — extrai JSON validado por schema
- `POST /ai/rag-query` — consulta semântica (pgvector)

**Entrega 3 — Pipeline Herbáceas (fim-a-fim)**
- imagem → VLM → JSON estruturado → persistência → revisão humana

---

## 6) Regras de Qualidade e Governança (Obrigatório)
### 6.1 Auditoria de IA (padrão)
Toda execução de IA deve registrar:
- `model_provider`, `model_name`, `model_version` (quando aplicável)
- `prompt_version` / `template_id`
- `input_hash`, `output_hash`
- `latency_ms`, `cost_estimate` (se disponível)
- `status`: success/fail/retry
- `review_status`: pending/approved/corrected
- `confidence` (quando aplicável)

### 6.2 Contratos de saída (JSON Schema)
- Nenhuma resposta “solta”. Sempre:
  - JSON validado
  - campos obrigatórios, tipos e limites
  - tratamento de erro e reprocessamento

### 6.3 Human-in-the-loop
- Resultados críticos (taxonomia, morfotipos) passam por revisão:
  - aprovação/correção
  - histórico e versionamento
  - comparação IA vs humano (métricas)

### 6.4 Segurança e LGPD
- Dados de localização (GPS) são sensíveis:
  - controle de acesso por projeto/equipe
  - retenção e mascaramento quando necessário
  - trilhas de acesso (auditoria)

---

## 7) Roadmap (12 semanas sugeridas)
### Semanas 1–2: Design e Banco
- Modelagem `core` + schemas de domínio
- Subir Postgres (PostGIS + pgvector)

### Semanas 3–4: Storage + AI Gateway + Jobs
- Object storage + presigned URLs
- FastAPI + fila (Redis/Celery)

### Semanas 5–6: Herbáceas pipeline + validação por schema
- Extração estruturada
- Revisão humana (admin simples)

### Semanas 7–8: RAG/Busca semântica e NOZES
- Indexação de conteúdo NOZES
- Busca por similaridade e “assistente” com rastreabilidade

### Semanas 9–10: Seiva Sync + validações inteligentes
- Pipeline de sincronização offline → online
- Alertas de consistência e outliers

### Semanas 11–12: Observabilidade + Hardening + Go-live
- métricas, logs, RBAC, backups
- checklist produção e SLOs

---

## 8) Padrões de Implementação (para robustez)

### 8.1 Padronização de IDs, versionamento e idempotência
- IDs: UUID v4 (mídia, observações, execuções)
- Idempotência em ingestão (hash do arquivo + metadata)
- Jobs devem ser reexecutáveis (retry seguro)

### 8.2 Padronização de nomes de arquivos no storage
Sugestão:
`/{project}/{yyyy}/{mm}/{dd}/{media_uuid}.jpg`

### 8.3 Estratégia multi-modelo (fallback)
- Definir “prioridade” por tarefa:
  - visão (VLM): modelo A → fallback B
  - texto/RAG: modelo A → fallback B
- Guardar o “provider” no `ai_runs` para rastreabilidade

### 8.4 DevOps mínimo
- Docker Compose para dev
- CI com testes:
  - unitários (API)
  - validação de schema
  - migrações (DDL)
- Migrations (ex.: Alembic) e versionamento de banco

---

## 9) Integrações por Projeto (resumo)

### Seiva Digital
- SQLite local continua sendo fonte offline
- Sync quando online:
  - envia “eventos” (observações, formulários, mídia)
  - recebe validações e enriquecimento (IA)

### NOZES.ia
- Persistência estruturada no Postgres
- IA para:
  - sugerir características/estados
  - detectar inconsistências na matriz
  - busca semântica por entidades e características

### Herbáceas App
- Upload e processamento em lote
- Pipeline de análise por subparcela
- Export XLSX/CSV gerado a partir do banco unificado

---

## 10) Como começar (passo a passo para a equipe)
1) Aprovar este README como **baseline**
2) Criar repositório “plataforma”:
   - `ai-gateway/` (FastAPI)
   - `db/` (migrations/DDL)
   - `worker/` (jobs)
   - `infra/` (compose/helm opcional)
3) Subir ambiente DEV com Docker (Postgres + Redis + MinIO)
4) Implementar tabelas `core` e pipeline `media → job → ai_runs`
5) Entregar Herbáceas pipeline completo + revisão humana
6) Adicionar RAG NOZES
7) Adicionar sync Seiva

---

## 11) Critérios de sucesso (MVP)
- Pipeline “fim-a-fim” operando (imagem → IA → banco → revisão)
- Busca semântica funcional (pgvector) com filtros por projeto
- Auditoria completa em `ai_runs`
- Integração inicial com pelo menos 1 app (Herbáceas primeiro)

---

## 12) Próximas melhorias (Visão evolutiva)
- Dataset curado e avaliações automatizadas (regressão de prompts)
- Classificadores/segmentação especializados (quando necessário)
- Cache inteligente e processamento em lote para reduzir custo
- Painel admin completo (aprovação, métricas, qualidade)
- Migração para Lakehouse/Spark caso o volume exija

---

## Contato e Governança
- Responsável técnico: (definir)
- Reunião semanal de arquitetura: (definir)
- SLA/SLO inicial do MVP: (definir)

Instruções adicionais (recomendação prática para “profissional e robusto”)

Para aumentar a chance de sucesso com pouca fricção, recomendo impor 3 padrões desde o primeiro commit:

Contrato de dados: toda saída de IA deve ser JSON validado por schema e auditada em ai_runs.

Revisão humana: tudo que vira “verdade” (taxonomia/morfotipo) precisa de status approved/corrected.

Padronização de stack: um único “AI Gateway” (FastAPI) e um único padrão de indexação (LlamaIndex ou LangChain, escolha um e fixe).

Se você quiser, eu também posso te entregar na sequência:

um backlog de épicos e tarefas (MVP → V2) em formato pronto para GitHub Projects/Jira,

e um esqueleto de repositório (estrutura de pastas + padrões de endpoints + modelos de tabela “core”).
