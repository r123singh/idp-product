# Deployment Notes

## Jobs API
- Local: `python local_runner.py`
- AWS: Lambda + layer + HTTP API + DynamoDB via `infra/deploy.ps1`
- Auth: `x-api-key` on deployed environments

## Extraction engine
- Local runner for schema/LLM iteration
- Lambda handler packaging for the parser endpoint consumed by Jobs API

## Configuration
All secrets and upstream URLs are environment-driven. Published samples use empty keys and example hostnames.
