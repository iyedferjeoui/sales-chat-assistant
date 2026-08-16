# Sales Chat Assistant

An AI-powered sales/lead assistant built with n8n, Supabase, and Google Gemini. It answers customer questions about store policies (delivery, hours, exchanges) using RAG over uploaded documents, and answers product questions (price, size, color, stock) by querying a live products database — all while replying in whichever language the customer used.

## How it works

The project is split into two independent n8n workflows that share data through Supabase.

### 1. Load Data Flow — builds the knowledge base

```
Upload a document → Default Data Loader (splits into chunks)
                   → Embeddings (Google Gemini)
                   → Supabase Vector Store (insert mode)
                   → stored in the `documents` table
```

Run this once per document (e.g. a policies PDF). It never talks to customers — its only job is to turn a document into searchable embeddings.

### 2. Retriever Flow — answers customer questions

```
Customer message → AI Agent (Google Gemini)
                       ├── Tool: Supabase Vector Store (retrieve mode)
                       │     → searches the `documents` table for policy/FAQ answers
                       └── Tool: Postgres — Execute SQL Query
                             → runs a live SQL query against the `products` table
                             → handles price, stock, size, color, comparisons, "cheapest", etc.
```

The AI Agent decides which tool (or both) to call based on the system prompt's routing rules, then answers using only what the tools return — it never invents prices, stock, or policy details.

## Repo structure

```
sales-chat-assistant/
├── n8n-workflows/
│   ├── load-data-flow.json      Import into n8n to set up the knowledge base loader
│   └── retriever-flow.json      Import into n8n to set up the customer-facing agent
├── database/
│   ├── schema.sql                documents table + match_documents() function (pgvector)
│   └── products-schema.sql       products table + example rows
├── knowledge-base/
│   └── store-policies.pdf        Example source document for the knowledge base
├── prompts/
│   └── system-prompt.md          The AI Agent's system message (tool routing, tone, language rules)
└── docs/
    └── setup-guide.md            Step-by-step environment setup
```

## Practical example

**Customer asks:** "Do you have the Men's Classic Robe in large, and can you deliver it to Sfax?"

1. AI Agent recognizes this needs both tools.
2. Calls the **product tool** → `SELECT * FROM products WHERE name ILIKE '%Men''s Classic Robe%' AND size = 'L'` → gets price, color, stock.
3. Calls the **knowledge base tool** → searches the `documents` table for delivery info → gets delivery time and coverage.
4. Combines both into one grounded, direct answer:
   > "Yes, we have the Men's Classic Robe in large (85 TND, in stock). We deliver to Sfax, usually within 1 to 3 business days."

**Customer asks (in Arabic):** "شنوة أرخص روب عندكم؟" ("what's the cheapest robe you have?")

1. AI Agent detects Arabic and commits to replying in Arabic.
2. Calls the product tool with no name filter → gets the full product list → reasons over price.
3. Replies in Arabic with the correct cheapest item and price.

## How to run it

### 1. Set up Supabase

1. Create a project at [supabase.com](https://supabase.com).
2. Go to **Database → Extensions**, enable `vector`.
3. Open the **SQL Editor** and run `database/schema.sql`, then `database/products-schema.sql`.
4. Note your **Project URL** and **service_role key** (Project Settings → API) — needed for the Supabase credential in n8n.
5. Note your **pooler connection string** (Project Settings → Database → Connect → Session pooler) — needed for the Postgres credential in n8n. Use the pooler, not the direct connection, or you may hit IPv6 resolution issues.

### 2. Set up n8n

1. Run n8n (Docker, npm, or n8n Cloud).
2. In n8n, create credentials:
   - **Supabase account** (Project URL + service_role key)
   - **Postgres account** (pooler host, database `postgres`, pooler user, your DB password, port from the pooler connection string)
   - **Google Gemini(PaLM) Api account** (API key from [Google AI Studio](https://aistudio.google.com))
3. Import `n8n-workflows/load-data-flow.json` and `n8n-workflows/retriever-flow.json` (Workflows → Import from File).
4. In both imported workflows, reassign each node's credential to the ones you just created (imported workflows reference credentials by name, so you'll need to re-select them).

### 3. Load your knowledge base

1. Open the **Load Data Flow**.
2. Run it once with your own policy/FAQ document (or use `knowledge-base/store-policies.pdf` as a test).
3. Check Supabase's Table Editor — the `documents` table should now have rows with content and embeddings.

### 4. Add your products

Edit the example rows in `products-schema.sql`, or insert your own directly in Supabase's SQL Editor / Table Editor into the `products` table.

### 5. Configure the system prompt

Open `prompts/system-prompt.md`, adjust the business description at the top for your store, and paste the full prompt into the AI Agent node's **System Message** field in the Retriever Flow.

### 6. Test

Open the Retriever Flow, click **Open Chat**, and ask a few questions — mix product questions, policy questions, and questions in different languages to confirm both tools and the routing logic work end to end.

### 7. (Optional) Connect Gmail

Replace the chat trigger with a **Gmail Trigger** node to turn this into a live email-answering assistant instead of a chat demo. Requires a Google Cloud OAuth client (Web application type) with the correct n8n redirect URI.

## Notes

- The product tool only filters by column name, never by AI-guessed columns — always check that AI-controlled fields in n8n use `$fromAI()` with an explicit, example-driven description, or tool calls can fail silently.
- pgvector's HNSW index caps out at 2000 dimensions — if you change embedding models and hit a dimension mismatch, either resize your `vector()` column to match the new model's output, or drop the HNSW index if it exceeds 2000 dimensions.
- This project intentionally avoids letting the AI write anything but read-only `SELECT` queries. If you extend this to writes (e.g. creating orders), use a restricted, read-only-by-default database role wherever possible.
