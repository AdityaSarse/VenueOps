# AGENTS.md — VenueOps

## Purpose
VenueOps: venue and incident management platform. The project owner is the only authority.

## Tech stack (per PRD)
React + JavaScript (frontend), Node.js + Express modular backend, PostgreSQL, Redis + BullMQ, Socket.IO, Nginx, Docker Compose, GitHub Actions, Azure VM, Prometheus/Grafana. JavaScript only — no TypeScript.

## Folder structure
- docs/ — PRD (docs/PRD.md) and progress log (docs/PROGRESS.md).
- App source folders — not created yet; document each here as it is added.

## Run / test / lint
TBD — no tooling configured yet. Agents may not run commands; give the owner exact commands to run.

## Conventions
- Small, reviewable changes; follow the existing structure and conventions.
- Do not refactor unrelated code.
- No comments unless asked.
- Never invent APIs, files, or table names; ask or read the file.
- Say "not run" for anything not executed, and list commands for the owner to verify.

## Rules (summary)
- Owner decides. If unclear, stop and ask. The PRD in docs/ is the source of truth.
- Do NOT: run commands/scripts; touch the database; use git/GitHub; create, rename, delete, or move files (except AGENTS.md and docs/PROGRESS.md); touch cloud/infra; read or write secrets/.env; add/remove dependencies; call external services; weaken security.
- Risky areas (schema, auth, RBAC, status transitions, outbox/notification worker, escalation) need a written plan and owner approval before changing code.
- May read project files and edit existing file contents, only for the current task.
- Record progress in docs/PROGRESS.md.

## PRD
- docs/PRD.md — VenueOps Product Requirements Document (source of truth).
