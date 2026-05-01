# Docker stacks

This repo contains self-hosted Docker Compose stacks.

## Maintaining README files

When you modify a `docker-compose.yml` in any stack directory, update that
directory's `README.md` to reflect the current state of services: ports,
purpose, dependencies, volume paths, and required environment variables.

Keep the architecture diagram in `core/README.md` in sync with the actual
data flow between services and the `~/ai-briefing` pipeline.

## Secrets

`.env` files and `*-cookies.json` files are gitignored. Always provide an
`.env.example` alongside any `.env` so others can replicate the setup.
