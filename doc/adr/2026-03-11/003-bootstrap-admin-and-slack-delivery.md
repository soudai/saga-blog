# ADR 003: Bootstrap Admin and Slack Delivery

Status: Accepted
Date: 2026-03-11

## Context

Two operational gaps needed concrete rules before implementation:
- how the first administrator is created
- how Slack notifications are delivered while preserving article visibility constraints

## Decision

The bootstrap and notification model is defined as follows:

- `make init` is the bootstrap entrypoint for local and first-time environment setup.
- `make init` must create the admin group if it does not exist.
- `make init` must create a bootstrap user record for the initial administrator and add that user to the admin group.
- The bootstrap administrator is identified by email so that the account can later be linked to Google login on the first successful sign-in.
- `make init` should accept at least `ADMIN_EMAIL`, and may also accept `ADMIN_DISPLAY_NAME`.
- `make init` must be idempotent.
- Notification events are `article published`, `comment created`, and `mention created`.
- Every notification event is stored as an in-app notification before Slack delivery is attempted.
- Slack delivery uses a Slack bot and the Slack Web API, not webhook-only delivery.
- Slack delivery must never expose content to users who cannot read the target article.
- For public article publication, the initial implementation may post to a configured announcement channel.
- For group-only articles, broadcast channel posting is disabled by default. Delivery is limited to in-app notifications and direct Slack delivery to explicitly authorized recipients.
- Comment and mention notifications use recipient-specific delivery only, and only when the recipient both has article access and has a linked Slack identity.
- Slack delivery failure does not roll back the persisted in-app notification.

## Consequences

- User records need a way to link a Google identity and, separately, a Slack identity.
- Initialization logic must pre-provision an admin user before first login rather than requiring a manual database edit.
- The notification system needs a fan-out step that runs after authorization checks and before delivery channel selection.
