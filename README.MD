Este repositório contém a camada de indexação e armazenamento vetorial (Vector Store) utilizada pelo agente de IA empresanho Operações, responsável por responder perguntas relacionadas aos POPs e documentos internos do time de Operações.

A indexação é realizada a partir do Google Drive e sincronizada periodicamente com este banco de vetores hospedado no Supabase.

🏗️ Arquitetura do RAG

O pipeline completo segue o fluxo:

Google Drive  →  n8n (Ingestão)  
             →  OpenAI Embeddings  
             →  Supabase Vector Store  
             →  Agente LLM (LangChain / LangGraph)  

🔹 Componentes

Google Drive → fonte original dos POPs

n8n → orquestra ingestão, detecção de mudanças e reindexação

OpenAI text-embedding-3-large → geração dos vetores

Supabase (pgvector) → armazenamento dos embeddings

LangChain / LangGraph → camada de retrieval + resposta do agente

📦 Fluxo de Ingestão (n8n)

A ingestão no n8n realiza:

✔ Rastreamento recursivo de pastas do Drive

Com controle de profundidade e processamento em batches.

✔ Filtros de qualidade

blacklist de pastas e arquivos

mimetypes permitidos (ex.: Google Docs)

✔ Verificação de atualização

O sistema só reindexa arquivos quando:

são novos

ou foram modificados no Drive (updated_at > metadata.updated_at)

✔ Reindexação inteligente

Ao detectar mudanças:

remove o embedding antigo

extrai o conteúdo atualizado

gera novo embedding

salva no Supabase

✔ Metadados armazenados

Cada documento inclui:

id do Drive

file_name

url

path

agent

created_at

updated_at

tipo de documento

conteúdo text-only processado

🗃️ Estrutura da Tabela documents

A tabela segue o padrão do LangChain Vector Store:

id                TEXT PRIMARY KEY
embedding         VECTOR(3072)
content           TEXT
metadata          JSONB

Exemplo de metadata:
{
  "id": "1aB2C3d4E5",
  "agent": "agente_operacoes",
  "file_name": "POP - Abertura de Chamados",
  "url": "https://drive.google.com/file/d/1aB2C3d4E5/view",
  "path": "/Operações/POPs/Chamados",
  "created_at": "2024-02-11T14:22:04",
  "updated_at": "2024-02-18T09:12:54"
}

🔍 Como funciona a busca (Retrieval)

A camada LLM utiliza Hybrid Retrieval:

Busca vetorial (similaridade semântica via pgvector)

Busca lexical (BM25 opcional em outra tabela)

Reranking (Cohere Reranker opcional)

O objetivo é retornar sempre os trechos mais relevantes e recentes para as respostas.

🧪 Exemplo de Query Vetorial (Supabase)
SELECT
  id,
  content,
  metadata,
  embedding <#> '[0.23, -0.11, ...]' AS similarity
FROM documents
ORDER BY similarity ASC
LIMIT 5;

🤖 Integração com o Agente

O agente LLM pode ser implementado em:

LangChain

LangGraph

FastAPI

Node.js

Python puro

Exemplo simples com LangChain:

from langchain.chat_models import ChatOpenAI
from langchain.vectorstores import SupabaseVectorStore
from langchain.embeddings import OpenAIEmbeddings

llm = ChatOpenAI(model="gpt-4o-mini")
emb = OpenAIEmbeddings(model="text-embedding-3-large")

vectorstore = SupabaseVectorStore(
    table_name="documents",
    embedding=emb,
    query_name="match_documents"
)

retriever = vectorstore.as_retriever()


🔄 Atualizações

O fluxo é atualizado automaticamente via n8n Schedule Trigger, garantindo que novas alterações no Drive sejam refletidas no Supabase sem necessidade manual.

🛠️ Tecnologias Utilizadas

Supabase (PostgreSQL + pgvector)

n8n

OpenAI Embeddings

LangChain

LangGraph

Google Drive API

📞 Suporte / Manutenção

O agente está registrado como:
agent: "empresanho_operacoes"

Para adicionar novos documentos, basta inserir no Google Drive dentro da pasta monitorada.

Para apagar ou reprocessar documentos manualmente, consulte a tabela documents.