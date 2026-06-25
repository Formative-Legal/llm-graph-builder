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
LLM_MODEL_CONFIG_BEDROCK_CLAUDE_SONNET_4_6=eu.anthropic.claude-sonnet-4-6,eu-west-1
LLM_MODEL_CONFIG_BEDROCK_NOVA_LITE_V1=eu.amazon.nova-lite-v1:0,eu-west-1
FORMATIVE_SCHEMA_API_TOKEN=<long-random-shared-secret>
TRACK_USER_USAGE=false
GCS_FILE_CACHE=false
GCP_LOG_METRICS_ENABLED=False
```

Attach an App Runner instance role with `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream`
for the selected Bedrock model. Store `FORMATIVE_SCHEMA_API_TOKEN` in AWS Secrets Manager or App Runner
secrets, not in GitHub variables. If Claude model access is blocked by AWS billing/Marketplace setup,
`bedrock_nova_lite_v1` is the AWS-native fallback model verified for the schema endpoint.

For Anthropic models served through AWS Marketplace, the App Runner instance role also needs
`aws-marketplace:ViewSubscriptions` and `aws-marketplace:Subscribe`; AWS may use these on first
runtime invocation to activate the model for that role/account.

After deployment, configure cockpit:

```env
MATRIX_GRAPH_BUILDER_URL=https://<private-graph-builder-backend>
MATRIX_GRAPH_BUILDER_MODEL=bedrock_claude_sonnet_4_6
MATRIX_GRAPH_BUILDER_TOKEN=<same-secret-as-FORMATIVE_SCHEMA_API_TOKEN>
```

The deployed AWS App Runner service is currently:

```env
MATRIX_GRAPH_BUILDER_URL=https://nytuxdnv6f.eu-west-1.awsapprunner.com
```

Then verify from `etl-cockpit`:

```bash
AWS_PROFILE=etl-cockpit .venv/bin/python -m matrix_cockpit.tools.cloud_doctor --skip-bedrock-invoke --require-graph-builder
```
