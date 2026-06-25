# Changelog

All notable changes to genesis-architect are documented here.

---

## [1.0.0] — 2025

### Initial release

**SKILL.md** — 7-phase build protocol covering intake, project structure, memory system, code generation standards, brain map generation, security consciousness, and Vercel deployment.

**references/memory-schema.md** — Supabase schema for four persistent tables: `genesis_projects`, `genesis_components`, `genesis_decisions`, `genesis_lessons`. Includes TypeScript types, Supabase client setup, all CRUD operations, full-text search across memory, and session sync script with `--generate-map` flag.

**references/brainmap.md** — Galaxy visualization spec: node types and visual rules, force-directed layout algorithm, generation script that converts `brain.json` to `brainmap.html`, side panel content per node type, and the Synthesize button that streams a knowledge summary from the Anthropic API.

**references/security.md** — Six-layer framework: IDOR prevention, Zod input validation at every trust boundary, Upstash rate limiting via Vercel Edge Middleware, HTTP security headers with full CSP, server secret isolation and leak detection, dependency audit protocol. Includes a threat model template.

**references/vercel-deploy.md** — Complete deployment guide from Supabase project creation through GitHub Actions CI/CD. Covers Supabase branching for preview environments, post-deploy verification, structured logging, and instant rollback.

**assets/galaxy-template.html** — Standalone HTML brain map with force-directed layout, drag-and-pan canvas, zoom, click-to-detail side panel, and streaming Synthesize button. No build step. One file, works offline.

---

## Planned

- `references/testing.md` — Testing patterns for modular projects (Vitest, Playwright, MSW)
- `references/data-pipeline.md` — Stack decisions and patterns for Python/data-heavy projects
- Component library seed entries from real project usage
- Decision log seed entries with observed outcomes
