# NoSQL Database Design — Lab Sheet 5

This ZIP contains the complete questions and solutions in `LAB_SHEET_5_QUESTIONS_AND_SOLUTIONS.md`.

## Requirements
- Docker (for local Riak KV and Neo4j examples)
- curl
- Python 3.9+
- Python packages: `requests`, `neo4j`

Install Python packages:
```bash
pip install requests neo4j
```

Notes:
- Riak quorum and horizontal-scaling exercises require a multi-node cluster for a real distributed demonstration.
- The Riak examples use the HTTP API and assume a compatible Riak KV 2.x setup.
- Neo4j examples assume Neo4j 5 and local credentials `neo4j/password123`; change these in your environment as needed.
- Do not use sample credentials in production.
