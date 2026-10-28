<!-- Replace every {{placeholder}}. Keep section order. Delete this comment. -->
<p align="center">
  <img src="docs/assets/hero.gif" alt="{{PROJECT_NAME}} demo" width="800" />
</p>

<h1 align="center">{{PROJECT_NAME}}</h1>

<p align="center">
  {{ONE_LINE_PITCH}}
</p>

<p align="center">
  <a href="{{LIVE_URL}}">Live demo</a> ·
  <a href="docs/architecture.md">Architecture</a> ·
  <a href="docs/adr/">Decisions</a> ·
  <a href="docs/api/openapi.json">API</a> ·
  <a href="docs/case-study.md">Case study</a>
</p>

<p align="center">
  <img alt="CI" src="https://github.com/BiancaRomeroG/pulse/actions/workflows/ci.yml/badge.svg" />
  <img alt="License" src="https://img.shields.io/badge/license-MIT-black" />
  <img alt="Node" src="https://img.shields.io/badge/node-20.x-339933?logo=node.js&logoColor=white" />
  <img alt="TypeScript" src="https://img.shields.io/badge/TypeScript-strict-3178C6?logo=typescript&logoColor=white" />
</p>

## What it is

{{TWO_PARAGRAPHS: what the system does, for whom, and what makes it non-trivial. Mention Nimbus Pay context in one sentence.}}

## Why I built it

{{ONE_PARAGRAPH: the engineering question this project answers and how it relates to my experience (payments, event-driven systems, AI-assisted development).}}

## Architecture

<p align="center"><img src="docs/assets/architecture.svg" alt="Architecture" width="800" /></p>

{{THREE_BULLETS: the key moving parts.}} Full detail in [docs/architecture.md](docs/architecture.md).

## Quickstart

```bash
git clone https://github.com/BiancaRomeroG/pulse && cd pulse
cp .env.example .env
docker compose up
```

Then open {{LOCAL_URLS}}. Seeded credentials and demo script: [docs/demo.md](docs/demo.md).

For local development without Docker: [docs/runbook.md](docs/runbook.md).

> The hosted demo API runs on a free tier that sleeps when idle. First request may take ~30 s.

## Features

{{BULLETS: 6–10, each one verifiable in the demo.}}

## Tech

| Area | Choice |
|------|--------|
| Runtime | Node 20, TypeScript strict, pnpm, Turborepo |
| API | NestJS, Prisma, zod, pino, Swagger |
| Web | Next.js 16, React 19, Tailwind v4, Motion |
| Data | PostgreSQL 16 {{EXTRAS}} |
| Quality | Vitest, Supertest, Playwright, fast-check, Lighthouse CI |
| Infra | Docker, GitHub Actions {{EXTRAS}} |

## Documentation

- [Architecture](docs/architecture.md) · [Data model](docs/data-model.md) · [ADRs](docs/adr/) · [Runbook](docs/runbook.md) · [Demo script](docs/demo.md) · [Case study](docs/case-study.md)
- [Contributing](CONTRIBUTING.md) · [Security](SECURITY.md) · [Changelog](CHANGELOG.md)

## Status

{{PHASE_AND_ROADMAP_BULLETS}}

## Español

{{RESUMEN_EN_ESPAÑOL: qué es, por qué lo construí, cómo correrlo en 3 comandos. 6–10 líneas.}}

## License

MIT © Bianca Romero
