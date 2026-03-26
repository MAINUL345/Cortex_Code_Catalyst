---
name: cortex-code
description: "Use this skill when the user wants to build a Snowflake Cortex AI chatbot demo through a conversational, step-by-step workflow — mirroring the Catalyst AI Demo Builder app. Triggers include: any mention of 'Cortex Code', 'CoCo', 'Catalyst', 'build a demo', 'build a Cortex Analyst', 'create a semantic model', 'create a Cortex Search service', 'build a Cortex Agent', 'set up an agent', 'NL-to-SQL', 'document search', 'RAG', 'chatbot', or any request to interactively configure and deploy Cortex Analyst, Cortex Search, or Cortex Agents by providing inputs step-by-step (company name, database, schema, tables, stage, warehouse, etc.). Also triggers on 'start'."
license: Proprietary
---

# Cortex Code — Catalyst AI Demo Builder (Conversational)

## Overview

This skill replicates the **Catalyst AI Demo Builder** Streamlit app as a conversational chat flow. It has three stages — **Build**, **Deploy**, **Chat** — that you walk the user through one question at a time, collecting inputs like form fields in a Streamlit app.

The skill builds:
1. **Cortex Analyst** — semantic model YAML uploaded to a stage, for NL-to-SQL over structured tables
2. **Cortex Search** — search service over documents/PDFs parsed with PARSE_DOCUMENT and chunked
3. **Combined** — both Analyst + Search, with smart routing between them

---

## Naming Convention

All object names use a prefix derived from the client name:
```
CATALYST_<CLIENT_NAME_UPPERCASED_ALPHANUMERIC>
```
Example: client "Acme Corp" → prefix = `CATALYST_ACME_CORP`

Objects created:
- Analyst stage: `{prefix}_ANALYST_STAGE`
- Semantic model file: `{prefix}_SEMANTIC_VIEW.yaml` (inside the stage)
- Chunks table: `{prefix}_CHUNKS`
- Search service: `{prefix}_SEARCH_SERVICE`
- Agent: `{prefix}_AGENT`

---

## STAGE 1: BUILD — Collect Inputs One-by-One

When the user starts, ask these questions **one at a time**. After each answer, confirm and move to the next. If the user provides multiple answers at once, accept them all and skip ahead.

### Step 1: Client & Connection

Ask in this order:

1. **Client name** — "What is your client or project name? (e.g., Acme Corp)"
2. **Database** — "Which Snowflake database? (e.g., SALES_DB)"
   - If unsure, run `SHOW DATABASES` and list options
3. **Schema** — "Which schema? (e.g., PUBLIC)"
   - If unsure, run `SHOW SCHEMAS IN DATABASE {db}` and list options
4. **Warehouse** — "Which warehouse? (default: COMPUTE_WH)"
5. **Business description** — "Give me a one-line business context (e.g., 'SaaS company wanting to query sales pipeline and support documents')"

### Step 2: Data Type Selection

Ask:
> "What data does this client have?"
> 1. **Structured Tables** — Cortex Analyst (NL-to-SQL)
> 2. **Documents / PDFs** — Cortex Search (RAG)
> 3. **Both — Tables + Docs** — Combined

### Step 3a: Structured Tables (if analyst or combined)

1. **Auto-discover** — Run `SHOW TABLES IN SCHEMA {db}.{schema}` automatically and present results (limit 8)
2. Ask: "Which tables should the semantic model cover? (max 4 recommended, list them)"
   - Also allow user to type table names manually if auto-discover didn't find them

### Step 3b: Document Stage (if search or combined)

1. **Stage name** — "What is the Snowflake stage name containing PDFs/text files? (e.g., DOCS_STAGE)"
2. **Auto-discover files** — Run `LIST @{db}.{schema}.{stage_name}` and filter for `.pdf`, `.txt`, `.docx`, `.md`
3. Show found files and ask: "Which files should I index? (default: all)"

### Confirmation

After all inputs collected, display a summary:
```
Client: {client_name}
Database: {db}
Schema: {schema}
Warehouse: {warehouse}
Mode: {analyst|search|combined}
Tables: {table1, table2, ...}       (if analyst/combined)
Stage: {stage_name}                  (if search/combined)
Files: {file1, file2, ...}          (if search/combined)
Business: {business_desc}
```
Ask: "Ready to build? (yes/no)"

---

## STAGE 2: DEPLOY — Execute Build Steps

Once confirmed, execute all build steps sequentially, reporting progress after each step.

### Path A: Cortex Analyst Backend (if analyst or combined)

#### A1: Read table schemas

For each table (max 4):
```sql
DESCRIBE TABLE {db}.{schema}.{table}
```
Collect column names and types. Report: "✓ {table} — {N} columns read"

If no tables can be read, stop and report the error.

#### A2: Generate semantic model YAML programmatically

Use this type-mapping logic:
- VARCHAR/TEXT/STRING/CHAR → `VARCHAR`
- NUMBER/INT/FLOAT/DECIMAL/NUMERIC → `NUMBER`
- DATE → `DATE`
- TIMESTAMP → `TIMESTAMP`
- BOOL → `BOOLEAN`
- Everything else → `VARCHAR`

Generate YAML with this structure:
```yaml
name: {PREFIX}_SEMANTIC_VIEW
tables:
  - name: {table_lowercase}
    base_table:
      database: {db}
      schema: {schema}
      table: {TABLE}
    dimensions:
      - name: {col_lowercase}
        expr: {COLUMN_NAME}
        data_type: {mapped_type}
    measures:
      - name: total_{col_lowercase}
        expr: SUM({COLUMN_NAME})
        data_type: NUMBER
```

Rules:
- ALL columns go into `dimensions`
- Only NUMBER columns ALSO get a corresponding `measures` entry with `SUM()` aggregation named `total_{col_lowercase}`
- Table names in YAML use lowercase with underscores
- Column names in `expr` preserve original casing
- Column names in `name` use lowercase with underscores

Show the generated YAML to the user.

#### A3: Upload YAML to a Snowflake stage

```sql
CREATE STAGE IF NOT EXISTS {db}.{schema}.{PREFIX}_ANALYST_STAGE
```

If this fails (read-only database), fall back to the current session's database/schema:
```sql
CREATE STAGE IF NOT EXISTS {app_db}.{app_schema}.{PREFIX}_ANALYST_STAGE
```

Then upload the YAML content to:
```
@{db_or_fallback}.{schema_or_fallback}.{PREFIX}_ANALYST_STAGE/{PREFIX}_SEMANTIC_VIEW.yaml
```

**IMPORTANT**: Since we are in a workspace (not Streamlit), write the YAML to a local file first, then use PUT to upload:
```sql
PUT file:///tmp/{PREFIX}_SEMANTIC_VIEW.yaml @{db}.{schema}.{PREFIX}_ANALYST_STAGE AUTO_COMPRESS=FALSE OVERWRITE=TRUE
```

If PUT is not available, create the stage and write using a stored procedure or COPY approach. The key output is: **the stage file path** (e.g., `@DB.SCHEMA.STAGE/FILE.yaml`) which becomes the `analyst_sv_fqn`.

#### A4: Generate suggested questions

Use CORTEX.COMPLETE to generate 5 suggested questions from the YAML:
```sql
SELECT SNOWFLAKE.CORTEX.COMPLETE('snowflake-arctic',
  'Extract the 5 best questions a user would ask about this data model. Return ONLY a JSON array of strings. No markdown.\n\n{yaml_first_3000_chars}'
) AS R
```

Parse the JSON array. If parsing fails, use these defaults:
- "What are the key metrics for this business?"
- "Show me the top performers by value"
- "What are the main trends over time?"
- "Give me a summary breakdown by category"
- "What changed most compared to last period?"

### Path B: Cortex Search Backend (if search or combined)

#### B1: Create chunks table

```sql
CREATE TABLE IF NOT EXISTS {db}.{schema}.{PREFIX}_CHUNKS (
    chunk_id    VARCHAR DEFAULT UUID_STRING(),
    source_file VARCHAR,
    chunk_text  VARCHAR,
    chunk_index INTEGER
)
```
```sql
TRUNCATE TABLE IF EXISTS {db}.{schema}.{PREFIX}_CHUNKS
```

#### B2: Parse and chunk documents

Build the file filter:
- If specific files selected: `WHERE RELATIVE_PATH IN ('file1.pdf', 'file2.pdf')`
- If all files: `WHERE RELATIVE_PATH ILIKE ANY ('%.pdf','%.txt','%.docx','%.md')`

Execute:
```sql
INSERT INTO {db}.{schema}.{PREFIX}_CHUNKS (source_file, chunk_text, chunk_index)
WITH parsed AS (
    SELECT
        RELATIVE_PATH AS source_file,
        SNOWFLAKE.CORTEX.PARSE_DOCUMENT(
            '@{db}.{schema}.{stage_name}',
            RELATIVE_PATH,
            {'mode': 'LAYOUT'}
        ):content::STRING AS doc_text
    FROM DIRECTORY(@{db}.{schema}.{stage_name})
    {file_filter}
)
SELECT
    source_file,
    f.value::VARCHAR  AS chunk_text,
    f.index           AS chunk_index
FROM parsed,
     LATERAL FLATTEN(
         input => SNOWFLAKE.CORTEX.SPLIT_TEXT_RECURSIVE_CHARACTER(
             doc_text, 'markdown', 1500, 150
         )
     ) f
WHERE f.value::VARCHAR IS NOT NULL
  AND LENGTH(TRIM(f.value::VARCHAR)) > 20
```

Report chunk count:
```sql
SELECT COUNT(*) AS N FROM {db}.{schema}.{PREFIX}_CHUNKS
```

#### B3: Create Cortex Search Service

```sql
CREATE OR REPLACE CORTEX SEARCH SERVICE {db}.{schema}.{PREFIX}_SEARCH_SERVICE
    ON chunk_text
    ATTRIBUTES source_file, chunk_index
    WAREHOUSE = {warehouse}
    TARGET_LAG = '1 hour'
AS (
    SELECT chunk_id, chunk_text, source_file, chunk_index
    FROM {db}.{schema}.{PREFIX}_CHUNKS
)
```

Report success/failure. The search service name is `{PREFIX}_SEARCH_SERVICE`.

#### B4: Generate suggested questions (if not already generated by Path A)

Use CORTEX.COMPLETE:
```sql
SELECT SNOWFLAKE.CORTEX.COMPLETE('snowflake-arctic',
  $$For a business described as: '{business_desc}', generate 5 questions a user would ask when chatting with their documents. Return ONLY a JSON array of strings. No markdown.$$
) AS R
```

### Build Summary

After all steps complete, present a summary:
```
⚡ Catalyst Demo Ready
Client: {client_name}
Mode: {ANALYST|SEARCH|COMBINED}
Objects created:
  - Semantic Model Stage: {PREFIX}_ANALYST_STAGE (if analyst/combined)
  - Search Service: {PREFIX}_SEARCH_SERVICE (if search/combined)
  - Chunks Table: {PREFIX}_CHUNKS (if search/combined)
Semantic model file: @{db}.{schema}.{PREFIX}_ANALYST_STAGE/{PREFIX}_SEMANTIC_VIEW.yaml
Suggested questions: [list them]

Ready to chat! Ask me anything about your data.
```

---

## STAGE 3: CHAT — Answer User Questions

Once deployed, switch to chat mode. Answer user questions using the backends built above.

### Analyst Query Function (ask_analyst)

Uses the Cortex Analyst REST API via `_snowflake.send_snow_api_request`:

**Request:**
```json
{
  "semantic_model_file": "@{stage_path}/{PREFIX}_SEMANTIC_VIEW.yaml",
  "messages": [{"role": "user", "content": [{"type": "text", "text": "{question}"}]}]
}
```

**Endpoint:** `POST /api/v2/cortex/analyst/message`

**Process the response:**
1. Extract `text` items from `message.content` → interpretation
2. Extract `sql` items from `message.content` → SQL statement
3. If SQL found, **execute it** and get results as a dataframe
4. Generate a natural language summary of the results using CORTEX.COMPLETE:
   ```sql
   SELECT SNOWFLAKE.CORTEX.COMPLETE('snowflake-arctic',
     $$The user asked: '{question}'. The database returned these results:
     {data_preview_csv_first_5_rows}
     Please provide a brief, professional natural language summary of this data.$$
   ) AS R
   ```
5. Return: answer text + SQL + data results

**IMPORTANT — In workspace context (not Streamlit):** Since `_snowflake` module is not available in workspaces, use the `snowflake_multi_cortex_analyst` tool directly to query the semantic model file. Pass `semantic_model_file` = the stage path. Then execute the returned SQL via `snowflake_sql_execute`.

### Search Query Function (ask_search)

Uses `snowflake.core.Root` to call the Cortex Search service:

```python
from snowflake.core import Root
root = Root(session)
svc = root.databases[db].schemas[schema].cortex_search_services[svc_name]
resp = svc.search(question, columns=["chunk_text", "source_file"], limit=5)
```

Then generate an answer using CORTEX.COMPLETE with the retrieved context:
```sql
SELECT SNOWFLAKE.CORTEX.COMPLETE('snowflake-arctic',
  $$Answer the following question using ONLY the document extracts below.
  If the answer is not in the documents, say so clearly. Be concise.

  Documents:
  {context_chunks_joined_by_separator}

  Question: {question}

  Answer:$$
) AS R
```

Return: answer + sources (unique source_file values)

**IMPORTANT — In workspace context:** Since `snowflake.core` is not available, query the search service via SQL:
```sql
SELECT SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
  '{db}.{schema}.{PREFIX}_SEARCH_SERVICE',
  '{"query": "{question}", "columns": ["chunk_text", "source_file"], "limit": 5}'
) AS results
```
Then parse the JSON results and use CORTEX.COMPLETE for the answer.

### Combined Query Function (ask_combined)

Route the question to the right backend:

1. Call CORTEX.COMPLETE to classify:
   ```sql
   SELECT SNOWFLAKE.CORTEX.COMPLETE('snowflake-arctic',
     $$Is this question best answered by querying a SQL database (return 'data') or by reading documents/text files (return 'docs')? Return ONLY one word.
     Question: {question}$$
   ) AS R
   ```
2. If result contains "doc" AND search service exists → call ask_search first
   - If search returns meaningful results (has sources, answer doesn't say "no relevant" or "could not"), use it
   - Otherwise, fall through to ask_analyst
3. Default → call ask_analyst

### Chat Response Format

For every answer, present:
1. **Answer text** (natural language summary)
2. **SQL** (if from Analyst) — show in an expandable section
3. **Data table** (if SQL returned results) — show the dataframe
4. **Chart** (if data has 2+ columns with at least one numeric) — suggest a bar chart of first non-index column vs first numeric column, limited to 20 rows
5. **Sources** (if from Search) — list unique source_file values

---

## Conversation Rules

1. **Ask ONE question at a time** unless the user volunteers multiple answers
2. **Always confirm** the collected value before moving on: "Got it, database = `SALES_DB`. Next..."
3. **Offer help** when user is unsure: run SHOW DATABASES, SHOW SCHEMAS, SHOW TABLES, DESCRIBE TABLE, LIST to list options
4. **Use the user's execution context** (role, warehouse, database from workspace context) as smart defaults
5. **Validate inputs** before generating SQL — check that databases/schemas/tables exist by running SQL
6. **Execute SQL directly** — do not just show SQL and ask the user to run it. Execute it and report results
7. **Handle errors gracefully** — if a step fails, show the error, suggest a fix, and offer to retry
8. **After build completion**, automatically transition to chat mode
9. **In chat mode**, always execute analyst SQL and return actual data — never just return SQL alone
10. **Track all created objects** and report fully qualified names at the end

---

## Smart Defaults

Use the workspace execution context to pre-fill defaults:
- **Database**: from `executionContext.database` or ask
- **Schema**: from `executionContext.schema` or ask
- **Warehouse**: from `executionContext.warehouse` or `COMPUTE_WH`
- **Role**: from `executionContext.role`
- **Naming**: derive prefix from client name → `CATALYST_<UPPER_ALPHANUM>`

---

## Privilege Requirements

Before starting, ensure:
```sql
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE <role>;
-- For cross-region inference (if needed):
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```

For Search:
```sql
GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA <db>.<schema> TO ROLE <role>;
```

For Agents (if building one):
```sql
GRANT CREATE AGENT ON SCHEMA <db>.<schema> TO ROLE <role>;
```

---

## Full Example Conversation

```
User: start
Assistant: Welcome to Catalyst AI Demo Builder! Let's build your AI chatbot step by step.
  What is your client or project name?

User: Acme Corp
Assistant: Got it — client = Acme Corp. Prefix will be CATALYST_ACME_CORP.
  Which Snowflake database should we use?

User: SALES_DB
Assistant: Database = SALES_DB. Which schema?

User: PUBLIC
Assistant: Schema = PUBLIC. Which warehouse? (default: COMPUTE_WH)

User: COMPUTE_WH
Assistant: Warehouse = COMPUTE_WH. Give me a one-line business context.

User: SaaS company wanting to query sales pipeline and support documents
Assistant: Got it. What data does this client have?
  1. Structured Tables (Cortex Analyst)
  2. Documents / PDFs (Cortex Search)
  3. Both — Tables + Docs (Combined)

User: 3
Assistant: Combined mode — we'll build both Analyst and Search.
  Let me check what tables are available...
  [runs SHOW TABLES IN SCHEMA SALES_DB.PUBLIC]
  Found: OPPORTUNITIES, ACCOUNTS, CONTACTS, PIPELINE_HISTORY
  Which tables should the semantic model cover? (max 4)

User: OPPORTUNITIES and ACCOUNTS
Assistant: Got it — 2 tables. Now for documents:
  What is the Snowflake stage name containing your PDFs/text files?

User: DOCS_STAGE
Assistant: Let me check the files...
  [runs LIST @SALES_DB.PUBLIC.DOCS_STAGE]
  Found 12 files: policy_guide.pdf, sales_playbook.pdf, ...
  Which files should I index? (default: all 12)

User: all
Assistant: Great! Here's your build summary:
  Client: Acme Corp | DB: SALES_DB | Schema: PUBLIC | Warehouse: COMPUTE_WH
  Mode: COMBINED
  Tables: OPPORTUNITIES, ACCOUNTS
  Stage: DOCS_STAGE (12 files)
  Ready to build? (yes/no)

User: yes
Assistant:
  [A1] Reading table schemas...
    ✓ OPPORTUNITIES — 15 columns
    ✓ ACCOUNTS — 10 columns
  [A2] Generating semantic model YAML...
    ✓ YAML generated (2,340 chars)
  [A3] Uploading to stage CATALYST_ACME_CORP_ANALYST_STAGE...
    ✓ Uploaded to @SALES_DB.PUBLIC.CATALYST_ACME_CORP_ANALYST_STAGE/CATALYST_ACME_CORP_SEMANTIC_VIEW.yaml
  [A4] Generating suggested questions...
    ✓ 5 questions ready
  [B1] Creating chunks table...
    ✓ CATALYST_ACME_CORP_CHUNKS ready
  [B2] Parsing and chunking 12 documents...
    ✓ 847 chunks created
  [B3] Creating Cortex Search Service...
    ✓ CATALYST_ACME_CORP_SEARCH_SERVICE created (indexing in background)

  ⚡ Catalyst Demo Ready!
  Objects: CATALYST_ACME_CORP_ANALYST_STAGE, CATALYST_ACME_CORP_SEARCH_SERVICE, CATALYST_ACME_CORP_CHUNKS
  Ask me anything about your data!

User: What's our total pipeline value by stage?
Assistant: [uses Cortex Analyst → gets SQL → executes → summarizes]
  Your total pipeline value is $12.4M across 5 stages:
  - Prospecting: $2.1M (17%)
  - Qualification: $3.2M (26%)
  - Proposal: $4.1M (33%)
  - Negotiation: $2.0M (16%)
  - Closed Won: $1.0M (8%)
  SQL: SELECT STAGE, SUM(AMOUNT) ...
  [shows data table + bar chart]

User: What does our sales playbook say about handling objections?
Assistant: [routes to Search → retrieves chunks from sales_playbook.pdf → summarizes]
  According to the sales playbook, the recommended objection handling framework is...
  Sources: sales_playbook.pdf (chunks 12, 13, 14)
```
