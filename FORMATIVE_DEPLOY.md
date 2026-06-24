# Formative private backend deployment

This fork is used only as the private Neo4j LLM Graph Builder backend for Matrix cockpit ontology
proposals.

## Hardening applied

- `backend/score.py` no longer logs raw `input_text` in `/populate_graph_schema`.
- `/populate_graph_schema` requires `Authorization: Bearer <token>` matching
  `FORMATIVE_SCHEMA_API_TOKEN`.

Do not revert that change. Raw matter text must not enter application logs.

## Deploy target

Deploy the backend only to AWS, in `eu-west-1`, behind private or single-identity access. Use AWS App
Runner or ECS/Fargate; App Runner is the shortest path for this backend because the upstream service
already ships a backend Dockerfile.

Do not use Neo4j Labs' public hosted backend for real documents. It is US-hosted and not controlled
by Formative.

## AWS build path

This fork includes `.github/workflows/build-backend-image.yml`. It builds `backend/Dockerfile` in
GitHub Actions and pushes:

- `<account>.dkr.ecr.eu-west-1.amazonaws.com/matrix-graph-builder-backend:main-<sha>`
- `<account>.dkr.ecr.eu-west-1.amazonaws.com/matrix-graph-builder-backend:latest`

Required GitHub configuration:

- secret `AWS_ROLE_TO_ASSUME`: GitHub OIDC role allowed to push to the ECR repository
- optional variable `AWS_REGION`: defaults to `eu-west-1`
- optional variable `ECR_REPOSITORY`: defaults to `matrix-graph-builder-backend`

Create the App Runner service from the ECR `latest` image. Runtime port is `8000`.

## Backend env

Set at least one model config supported by the backend. Example:

```env
LLM_MODEL_CONFIG_ANTHROPIC_CLAUDE_4_6_SONNET=claude-sonnet-4-6,<anthropic-api-key>
FORMATIVE_SCHEMA_API_TOKEN=<long-random-shared-secret>
TRACK_USER_USAGE=false
GCS_FILE_CACHE=false
GCP_LOG_METRICS_ENABLED=False
```

Store `FORMATIVE_SCHEMA_API_TOKEN` and model provider keys in AWS Secrets Manager or App Runner
secrets, not in GitHub variables.

After deployment, configure cockpit:

```env
MATRIX_GRAPH_BUILDER_URL=https://<private-graph-builder-backend>
MATRIX_GRAPH_BUILDER_MODEL=anthropic_claude_4_6_sonnet
MATRIX_GRAPH_BUILDER_TOKEN=<same-secret-as-FORMATIVE_SCHEMA_API_TOKEN>
```

Then verify from `etl-cockpit`:

```bash
AWS_PROFILE=etl-cockpit .venv/bin/python -m matrix_cockpit.tools.cloud_doctor --skip-bedrock-invoke --require-graph-builder
```
