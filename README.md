# n8n Workflow Finder (NocoDB + Supabase Vector Search)
 
An n8n project that builds a **searchable database of n8n workflow templates** and puts an **AI assistant** on top of it.
 
It pulls AI workflow templates from the public n8n.io API, stores them as structured records in **NocoDB**, turns every record into an embedding in a **Supabase (pgvector)** vector store, and lets you ask in plain language: *"Which workflows send Telegram messages with GPT?"* or *"How do I build a RAG chatbot?"* – the agent finds relevant templates and answers with links.

## What it does
 
| Part | What happens |
|---|---|
| **1. Sync catalog** | Fetches AI templates from n8n.io, maps them to a clean schema, and **creates or updates** one NocoDB record per template (no duplicates) |
| **2. Embed** | Finds records that are not in the vector store yet and stores their embeddings in Supabase with metadata (name, template ID, URL) |
| **3. Ask** | A chat agent runs a semantic search over the vector store and either lists matching workflows or explains how to build one |
 
## How it works
 
```
Part 1 ─ Manual Trigger (Catalog)
           ├─► Fetch n8n Templates ─► Split ─► Map fields ───────────────┐
           └─► Get Existing Records ─► Keep ID + template ID ────────────┴─► Match by template ID
                                                                                     │
                                                                              Record Exists?
                                                                          ├─ yes ─► Update Record
                                                                          └─ no ──► Create Record
 
Part 2 ─ Manual Trigger (Embeddings)
           ├─► Get All Records (NocoDB) ──────────┐
           └─► Get Embedded Documents (Supabase) ─┴─► Find Not Yet Embedded ─► Already Embedded?
                                                                          ├─ yes ─► Skip
                                                                          └─ no ──► Store Embeddings (Supabase)
                                                                                     ▲ OpenAI embeddings + document loader
 
Part 3 ─ Chat Trigger ─► n8n Expert Agent ◄── OpenAI Chat Model
                                          ◄── Workflow Search (Supabase vector store tool, top 15)
```
 
### Part 1 – Sync catalog → NocoDB
- `Fetch n8n Templates (AI)` calls the public n8n.io templates API (category `AI`, 20 rows per page).
- `Map Template Fields` produces: `n8n_id`, `Workflow Name`, `Total Views`, `price`, `Purchase Url`, `User`, `description`, `Created At`, `Node List` (comma-separated node names used in the template), `Category`, `Workflow URL`.
- Existing NocoDB records are matched by `n8n_id`: found → **update**, not found → **create**. Re-running the sync never duplicates data.
### Part 2 – Embeddings → Supabase
- NocoDB records are compared with documents already in Supabase (by `n8n_id` in the metadata).
- Only **new** records are embedded (OpenAI embeddings) and inserted, so you don't pay twice for the same template.
- Each document stores metadata: workflow name, `n8n_id`, workflow URL – so the agent can return links.
### Part 3 – AI assistant
The agent (OpenAI `gpt-5-mini`) has one tool – semantic search over Supabase – and two modes:
1. **Find workflows** – returns title, URL and a short reasoning for each match
2. **Explain how to build** – gives a detailed how-to plus related workflows
## Tech stack
 
- [n8n](https://n8n.io) – HTTP Request, Split Out, Merge, Switch, Set, AI Agent, Supabase Vector Store, Embeddings, Chat Trigger
- [NocoDB](https://nocodb.com) – structured database / table UI
- [Supabase](https://supabase.com) – Postgres + pgvector as vector database
- OpenAI – chat model and embeddings
## Setup
 
### Requirements
- An n8n instance
- A NocoDB account (cloud or self-hosted) with an API token
- A Supabase project
- OpenAI API key
### 1. NocoDB table
 
Create a base and a table (any name) with these columns:
 
| Column | Type |
|---|---|
| `n8n_id` | Number |
| `Workflow Name` | Single line text |
| `Total Views` | Number |
| `price` | Number |
| `Purchase Url` | Single line text |
| `User` | JSON (or long text) |
| `description` | Long text |
| `Created At` | Single line text |
| `Node List` | Long text |
| `Category` | Single line text |
| `Workflow URL` | Single line text |
 
### 2. Supabase vector table
 
Run in the Supabase SQL editor:
 
```sql
create extension if not exists vector;
 
create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);
 
create function match_documents (
  query_embedding vector(1536),
  match_count int default null,
  filter jsonb default '{}'
) returns table (id bigint, content text, metadata jsonb, similarity float)
language plpgsql as $$
#variable_conflict use_column
begin
  return query
  select id, content, metadata,
         1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```
 
### 3. Import and configure
 
1. **Import** `n8n-workflow-finder.json` into n8n.
2. **Create credentials**: NocoDB API Token, Supabase API, OpenAI API – select them in the matching nodes.
3. **Replace placeholders** in all NocoDB nodes (`Get Existing Records`, `Get All Records`, `Create Record`, `Update Record`): `YOUR_NOCODB_WORKSPACE_ID`, `YOUR_NOCODB_BASE_ID`, `YOUR_NOCODB_TABLE_ID` (or pick them from the dropdown lists).
4. **Run Part 1**: click `Manual Trigger (Catalog)` → *Execute workflow*. Check that rows appear in NocoDB.
5. **Run Part 2**: click `Manual Trigger (Embeddings)` → *Execute workflow*. Check the `documents` table in Supabase.
6. **Chat**: open the chat window of `Chat Trigger` and ask a question.
> 🔐 Keep all API tokens in n8n credentials only – never inside node parameters.
 
## Limitations
 
- Part 1 reads a single page of **20** templates in the `AI` category. To build a bigger database, increase `rows` and loop over `page`.
- Parts 1 and 2 are started manually; the assistant has no conversation memory yet.
- Templates are only added or updated; templates removed from n8n.io are not deleted.
- The embedding model used on insert and on search must be the same (both use the OpenAI default here).
- Uses the public n8n.io API – keep request volume reasonable.
## Possible improvements
 
- Schedule Trigger for a daily sync, with pagination over all categories
- Postgres Chat Memory so the assistant remembers the conversation
- Re-embed records when a template's description changes
- Telegram or Slack as the chat interface
- Filter the search by category or by nodes used (using the stored metadata)
## Author
 
Built by [@Anatolii33](https://github.com/Anatolii33). Available for n8n automation and AI agent projects.
 
