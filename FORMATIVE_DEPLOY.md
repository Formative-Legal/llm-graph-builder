# Formative private backend deployment

This fork is used only as the private Neo4j LLM Graph Builder backend for Matrix cockpit ontology
proposals.

## Hardening applied

- `backend/score.py` no longer logs raw `input_text` in `/populate_graph_schema`.

Do not revert that change. Raw matter text must not enter application logs.

## Deploy target

Deploy the backend only, in an EU region, behind private or single-identity access.

Acceptable targets:

- Google Cloud Run `europe-west1` or another EU region
- AWS ECS/App Runner in `eu-west-1`

Do not use Neo4j Labs' public hosted backend for real documents. It is US-hosted and not controlled
by Formative.

## Backend env

Set at least one model config supported by the backend. Example:

```env
LLM_MODEL_CONFIG_ANTHROPIC_CLAUDE_4_6_SONNET=claude-sonnet-4-6,<anthropic-api-key>
TRACK_USER_USAGE=false
GCS_FILE_CACHE=false
GCP_LOG_METRICS_ENABLED=False
```

After deployment, configure cockpit:

```env
MATRIX_GRAPH_BUILDER_URL=https://<private-graph-builder-backend>
MATRIX_GRAPH_BUILDER_MODEL=anthropic_claude_4_6_sonnet
```

Then verify from `etl-cockpit`:

```bash
AWS_PROFILE=etl-cockpit .venv/bin/python -m matrix_cockpit.tools.cloud_doctor --skip-bedrock-invoke --require-graph-builder
```
