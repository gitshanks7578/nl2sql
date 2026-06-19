# NL2SQL

NL2SQL is a TypeScript + Express backend that converts natural-language questions into SQL for user-registered PostgreSQL and MySQL databases.

The main idea is simple: before asking an LLM to generate SQL, the backend first understands the connected database schema. It extracts tables, columns, primary keys, foreign keys, and relationships, converts them into a structured schema graph, and uses that context to produce more grounded SQL responses.

## What Makes It Interesting

- Connects to real PostgreSQL and MySQL databases.
- Ingests live schema instead of depending on manually written schema prompts.
- Converts database structure into a graph-style schema IR.
- Retrieves only relevant schema context for each user query.
- Supports three response modes: `SQL`, `PLAN`, and `EXPLAIN`.
- Stores conversations and recent message history for follow-up queries.
- Validates generated SQL before execution.
- Supports basic user-level isolation through JWT-authenticated database ownership.

## Tech Stack

- Node.js
- TypeScript
- Express
- Prisma ORM
- PostgreSQL
- MySQL
- Groq API
- Gemini API support
- JWT authentication

## Architecture

```text
Client
  |
  v
Express API
  |
  v
Controller Layer
  |
  +--> Database Registration
  +--> Connection Testing
  +--> Schema Ingestion
  +--> Prompt Building
  +--> LLM Generation
  +--> SQL Validation
  +--> SQL Execution
  |
  v
Prisma Database
```

## Core Pipeline

```text
User registers via AUTH API's accesstoken
     │
     ▼
database registeration (POSTGRES/MYSQL)
     │
     ▼
schema injestion - (internal pipeline 1)
     │
     ▼
Prompt Builder - (internal pipeline 2)
     │
     ▼
    LLM
     │
     ▼
SQL Validator (stops destructive queries or hallucinated tables)
     │
     ▼
Generated SQL
     │
     ▼
Optional Execution
```

## Schema Ingestion Flow (internal pipeline 1)

```text
Registered Database
  |
  v
Postgres / MySQL Extractor
  |
  v
Schema Normalizer
  |
  v
Type Mapper (to normalize datatypes of postgres/mysql)
  |
  v
Relation Resolver
  |
  v
Schema IR
```

The schema ingestion pipeline extracts:

- Tables
- Columns
- Data types
- Nullable fields
- Primary keys
- Foreign keys

Then it converts the schema into an intermediate representation:

```ts
{
  nodes: [
    {
      table: "users",
      columns: [
        { name: "id", type: "string", nullable: false }
      ],
      primaryKeys: ["id"]
    }
  ],
  edges: [
    {
      fromTable: "orders",
      fromColumn: "user_id",
      toTable: "users",
      toColumn: "id"
    }
  ],
  adjacency: {
    users: [
      { table: "orders", via: "id", targetColumn: "user_id" }
    ]
  }
}
```

This makes the database easier to reason about as a relationship graph.


## Prompt builder Flow (internal pipeline 2)

```text
receives query,schema IR and history (last 5 messages)
  |
  v
schema cleaner (cleans schema of the system tables)
  |
  v
context retriever (retrieves relevant tables and relations)
  |
  v
final prompt
```

## Query Modes

### SQL

Generates SQL from a natural-language question.

```json
{
  "mode": "SQL",
  "message": "Generated SQL successfully.",
  "sql": "SELECT ...",
  "tablesUsed": ["users", "orders"]
}
```

### PLAN

Explains how the query should be solved without generating SQL.

```json
{
  "mode": "PLAN",
  "message": "Use customers and orders, connect them through the customer id, then aggregate total revenue.",
  "sql": null,
  "tablesUsed": ["customers", "orders"]
}
```

### EXPLAIN

Explains the query or result in plain language.

```json
{
  "mode": "EXPLAIN",
  "message": "This question asks for revenue grouped by each customer.",
  "sql": null,
  "tablesUsed": ["customers", "orders"]
}
```

## Data Model

```text
User
  -> Database
      -> Conversation
          -> Message
```

This structure keeps registered databases and conversations scoped to a user.

## API Overview

Base path:

```text
/api/v1
```

Routes:

```text
POST /register-user
POST /database-registeration-postgres
POST /database-registeration-mysql
POST /schema-injestion
POST /query
POST /execute
```

Health check:

```text
GET /
```

> Full request examples are available in the Postman collection.

## Environment Variables

```env
DATABASE_URL=[connection_link]
JWT_SECRET=[same secret from AUTH API]
GROQ_API_KEY="your-groq-api-key"
GEMINI_API_KEY="your-gemini-api-key"
PORT=3000
```

## Setup

Install dependencies:

```bash
npm install
```

Generate Prisma client:

```bash
npx prisma generate
```

Run migrations:

```bash
npx prisma migrate dev
```

Start the development server:

```bash
npm run dev
```

Server:

```text
http://localhost:3000
```

## Project Structure

```text
src/
  controller/
    database.controller.ts

  middleware/
    verifyjwt.ts
    error.middleware.ts

  orchestration/
    schema_injestion/
      pipeline/
        1_postgres_extractor.ts
        1_mysql_extractor.ts
        2_normalizer.ts
        2_typeMapper.ts
        3_relationResolver.ts

    prompt_builder/
      pipeline/
        1_schemaCleaner.ts
        2_contextRetriever.ts

  service/
    sql_groq_service.ts
    plan_groq_Service.ts
    explain_groq_Service.ts
    validator.ts

  utils/
    connectionTest.ts
    executor.ts
    encrypter.ts
    decrypter.ts

prisma/
  schema.prisma
```

## Security Note

Database passwords are currently handled with a simple Base64 helper for development purposes.

## Current Limitations

- SQL validation is lightweight and regex-based.
- Production-grade credential encryption is not implemented yet.
- LLMs can still provide a slightly wrong query due to the INTENT problem as current architecture translates query into SQL but doesn't provide accurate INTENT for enhanced precision.

## Future Scope

- Stronger SQL parser-based validation
- Real credential encryption
- Docker setup
- Unit and integration tests
- Frontend dashboard
- Support for more databases
