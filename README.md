# Far parlare i tuoi contenuti — demo RAG con WordPress

Materiale tecnico del talk **"Far parlare i tuoi contenuti: introduzione pratica alla RAG con WordPress"** — WordCamp Torino, 9 maggio 2026.

> Speaker: [Luca Baroncini](https://lucabaroncini.com) — PM @ [Net7](https://netseven.it), lead organizer WordCamp Pisa 2026.

## Cosa c'è in questa repo

```
.
├── animations/              # 3 animazioni HTML mostrate dal palco
│   ├── 01-ingest-pipeline.html
│   ├── 02-query-pipeline.html
│   └── 03-embedding-space.html
├── contenuti/               # 20 markdown del sito demo "Le Meraviglie Segrete di Pisa"
├── docker/                  # Docker Compose per Qdrant + n8n
├── workflows/               # Workflow n8n esportati (ingestione + query)
│   ├── ingestione-contenuti.json
│   └── query-chatbot.json
├── slides/                  # Placeholder per le slide finali (PDF post-talk)
├── chat.html                # Interfaccia chat HTML che chiama il webhook n8n
├── .mcp.json.example        # Config MCP n8n per Claude Code (opzionale)
└── docker/.env.example      # Template variabili d'ambiente
```

## Stack della demo

- **WordPress** + plugin [AI Ready Content](https://github.com/Lukid/ai-ready-content) per esporre i post in markdown su `/<slug>.md` e una sitemap su `/airc-sitemap.json`
- **n8n** come orchestratore visuale (workflow di ingestione + workflow di query)
- **Qdrant** come vector database, con ricerca **ibrida** (dense embedding + BM25 sparse) e Reciprocal Rank Fusion
- **OpenAI** per embedding (`text-embedding-3-large`) e generazione (`gpt-5.4-mini`)
- **Cohere Rerank** (`rerank-v4.0-pro`) come reranker finale

L'idea del talk: mostrare che con strumenti accessibili (n8n + un vector DB) un team WordPress può costruire un assistente che risponde davvero sui propri contenuti, **senza** passare da Python o da framework che richiedono un data scientist.

## Come replicare la demo a casa tua

### 1. Prerequisiti

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Una API key [OpenAI](https://platform.openai.com/) (per embedding + generazione)
- Una API key [Cohere](https://cohere.com/) (per il reranker — c'è un free tier sufficiente per la demo)
- Un sito WordPress con il plugin [AI Ready Content](https://github.com/Lukid/ai-ready-content) attivo (oppure usa i markdown pronti in `contenuti/`)

### 2. Avvia Qdrant + n8n

```bash
cd docker
cp .env.example .env       # poi edita .env con le tue chiavi
docker compose up -d
```

- n8n: http://localhost:5678
- Qdrant dashboard: http://localhost:6333/dashboard

Dentro a n8n, crea le credenziali (Settings → Credentials):

- **OpenAI** (API key)
- **Cohere** (API key)
- **Qdrant** — URL: `http://qdrant:6333`, API key vuota (in locale Qdrant non richiede auth)

### 3. Importa i workflow

Da n8n, **Workflows → Import from File**, carica:

- `workflows/ingestione-contenuti.json`
- `workflows/query-chatbot.json`

Riassocia le credenziali che hai appena creato a ogni nodo che le richiede.

### 4. Indicizza i contenuti

Hai due opzioni.

**Opzione A — Usa i markdown già pronti in questa repo.** Adatta il workflow `Ingestione Contenuti` per leggere da `contenuti/` invece che dal sito WP.

**Opzione B — Punta al tuo WordPress.** Se il plugin AI Ready Content è installato sul tuo sito, modifica nel workflow l'URL della sitemap (`/airc-sitemap.json`) e lascia che n8n recuperi i `.md` direttamente dal sito.

In entrambi i casi: esegui il workflow una volta. Vedrai la collection `meraviglie_pisa` (o quella che scegli tu) popolata su Qdrant.

### 5. Apri la chat

```bash
open chat.html
```

Il file punta al webhook `http://localhost:5678/webhook/rag-chat` esposto dal workflow di query. Scrivi una domanda e dovresti vedere i nodi accendersi in n8n.

## Animazioni

Tre animazioni HTML autocontenute, controllate da tastiera (`SPAZIO` o frecce per avanzare):

- **01-ingest-pipeline.html** — visualizza la pipeline di ingestione (post WP → chunk → embedding → Qdrant)
- **02-query-pipeline.html** — visualizza la pipeline di query (domanda → ricerca ibrida → rerank → LLM → risposta)
- **03-embedding-space.html** — mostra lo spazio degli embedding (parole vicine per significato)

Aprile in un browser, niente build, niente dipendenze.

## (Opzionale) Claude Code + MCP n8n

Se usi [Claude Code](https://www.anthropic.com/claude-code), puoi attivare l'MCP server di n8n per chiedere a Claude di costruire/correggere workflow direttamente dall'editor.

```bash
cp .mcp.json.example .mcp.json
# poi edita .mcp.json e incolla la tua N8N_API_KEY (Settings → API → Create API Key)
```

Le skill n8n che ho usato durante la preparazione vengono dal progetto upstream [`czlonkowski/n8n-mcp`](https://github.com/czlonkowski/n8n-mcp): se vuoi le stesse capacità, segui le istruzioni del repo per installarle nel tuo `~/.claude/skills/`.

## Q&A — domande tipiche dei partecipanti

**Devo per forza usare n8n?** No. La pipeline che vedi nei due workflow puoi scriverla in Python, in PHP, dentro WordPress stesso. n8n l'ho scelto perché è visuale, mostra bene il flusso dal palco e abbassa la barriera per chi non scrive Python.

**Devo per forza usare Qdrant?** No. Funzionano altrettanto bene Weaviate, pgvector, LanceDB, Chroma. Qdrant ha ricerca ibrida nativa con BM25 sparse, che è il motivo per cui l'ho scelto qui.

**Posso usare modelli locali invece di OpenAI / Cohere?** Sì. Sostituisci i nodi OpenAI con Ollama (o un endpoint compatibile) e il reranker Cohere con `bge-reranker` self-hosted. La demo è pensata per essere accessibile, non per imporre uno stack.

**Quanto costa girare la demo?** Per i 20 contenuti del sito → centesimi di embedding una tantum. Una conversazione intera (5-10 domande) costa qualche centesimo di euro tra reranker e LLM.

## Licenza

[MIT](LICENSE) — usa, modifica, prendi spunto.

## Crediti

Sito demo "Le Meraviglie Segrete di Pisa" — sito turistico inventato per la demo, con humor toscano e campanilismo Pisa-Livorno.

Stack ispirato dal lavoro fatto in Net7 sul chatbot della Regione Toscana ("Chiedilo a me") basato su Cheshire Cat.
