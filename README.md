**CATALYST**
Conversion Demo Builder — Kipi.ai

**What It Is**
Catalyst builds a working Cortex Code chatbot on any client's Snowflake data in under 20 minutes. Not a demo on sample data. Not a slide about what AI can do. Their actual tables, their actual documents, answering their actual business questions — live, in the first meeting.

**The Business Flow**
Any engagement becomes a demo opportunity. Whether you are mid-way through a data engineering project, running a thrive, or walking into a first client meeting — if the client has data in Snowflake, Catalyst runs.

Four inputs. That's it. Client name → Database and schema → Tables or document stage → One line describing the business. Everything else is automatic.

The chatbot is live before the meeting ends. A Cortex Code Chatbot in Snowflake URL is generated. The client opens it, types their own question in plain English, and gets an answer from their own data in seconds. That moment — not a slide, not a script — is what converts.
No specialist required. Any account manager with Snowflake access can trigger this. No data science team. No 2-day sprint. No waiting.

**The Technical Flow**
Client's Snowflake account
         │
         ▼
 Catalyst detects data type
         │
    ┌────┴────┐
    │         │
 Tables     Documents
    │         │
    ▼         ▼
Cortex     Cortex
Analyst    Search
    │         │
    └────┬────┘
         │
         ▼
  Streamlit Chat UI
  (live URL, deployed inside Snowflake)
  
If tables → CoCo reads the schema, calls SNOWFLAKE.CORTEX.COMPLETE with a master prompt to auto-generate a Semantic View YAML, runs SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML, and wires it to Cortex Analyst. Natural language questions become SQL. SQL becomes answers and charts.

If documents → CoCo lists the stage, runs SNOWFLAKE.CORTEX.PARSE_DOCUMENT on each file, flattens the content into a chunks table, and creates a CORTEX SEARCH SERVICE over it. Questions are answered via RAG — retrieval from chunks, then CORTEX.COMPLETE for the final answer.

If both → An LLM classifier routes each question to the right tool at runtime. One chat interface, two engines, invisible to the user.

The Cortex Code uses _snowflake.send_snow_api_request() — the runtime-injected SiS module — to call both the Cortex Analyst API and the Cortex Search API. Zero external packages. Zero hosting. Runs entirely inside Snowflake.



Stack at a Glance

  **Layer**                             **Technology**
Build engine                            Cortex Code (CoCo)
Structured data                     Cortex Analyst + Semantic Views
Documents                           Cortex Search + PARSE_DOCUMENT
LLM                                 CORTEX.COMPLETE (snowflake-arctic)
UI                                  Cortex Code in Snowflake
Runtime API                         _snowflake.send_snow_api_request()


**The One Line**
We don't show clients a slide about AI. We build them a working one in 20 minutes, on their own data, before the meeting ends.


