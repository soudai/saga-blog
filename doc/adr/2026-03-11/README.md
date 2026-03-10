# ADR Bundle: 2026-03-11

Status: Accepted

This directory records the initial implementation decisions consolidated from GitHub issues `#1`, `#2`, and `#3`, plus follow-up clarifications made on 2026-03-11.

Files:
- `001-application-architecture-baseline.md`
- `002-content-visibility-and-group-administration.md`
- `003-bootstrap-admin-and-slack-delivery.md`

These ADRs supersede earlier ambiguous notes where needed. In particular:
- PostgreSQL version is fixed to `18`.
- Application data access uses `2way-sql`.
- Schema migration uses `golang-migrate`.
- Slack delivery uses a Slack bot, not webhook-only delivery.
