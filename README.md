# RAG Chatbot n8n

A Retrieval-Augmented Generation (RAG) chatbot built in n8n. It answers questions using a knowledge base stored in Supabase with pgvector, and it only answers from that knowledge base.

## How it works

The project has two n8n workflows:

1. **RAG - Ingest Knowledge** loads text, splits it into chunks, creates embeddings with OpenAI, and saves them to the `documents` table in Supabase.
2. **RAG - Chatbot** receives a message through the Chat Trigger, lets the AI Agent (GPT-5 mini) search the knowledge base, and replies based on the results.

## Files

- `RAG_Ingest_Knowledge.json`: ingestion workflow
- `RAG_Chatbot.json`: chatbot workflow

## Requirements

- n8n (tested on version 2.45.3)
- A Supabase project
- An OpenAI API key

## Setup

### 1. Create the database table and function

Run this in the Supabase SQL Editor:

```sql
create extension if not exists vector;

create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);

create or replace function match_documents (
  query_embedding vector(1536),
  match_count int default null,
  filter jsonb default '{}'
)
returns table (id bigint, content text, metadata jsonb, similarity float)
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

### 2. Add credentials in n8n

Create these two credentials. The files do not include any keys, so you must add them yourself:

- **Supabase account:** Project URL and `service_role` key
- **OpenAI account:** your API key

After importing each workflow, open every node that shows a credential warning and select your credential.

### 3. Import the workflows

1. In n8n, go to **Workflows → Import from File**.
2. Import `RAG_Ingest_Knowledge.json` and `RAG_Chatbot.json`.

### 4. Add your knowledge

Open the **Knowledge Source** node in the Ingest workflow and replace the sample text with your own content.

### 5. Load the knowledge

1. Open the Ingest workflow.
2. Click **Execute workflow**.
3. Check the `documents` table in Supabase. It should contain your text in `content` and vectors in `embedding`.

To reload with new content, first run `delete from documents;` in the SQL Editor. Otherwise you will get duplicate rows.

### 6. Test the chatbot

1. Activate the **RAG - Chatbot** workflow.
2. Open the Chat Trigger panel and ask a question about your content.
3. Ask something outside your knowledge base. The bot should say the information is not available.

## Notes

- Both workflows use `text-embedding-3-small`. Keep the same model in ingestion and chat, or the search results will be wrong.
- The Supabase `service_role` key gives full database access. Never commit it to this repository.
- The Chat Trigger webhook is public once the workflow is active. Only share the link with people you trust.

## License

Add a license here if you plan to share this project publicly.
