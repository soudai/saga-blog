# ADR 002: Content Visibility and Group Administration

Status: Accepted
Date: 2026-03-11

## Context

The product requirements define:
- Google login
- article states `WIP` and `Published`
- visibility scopes `Public` and `Group-only`
- application-managed groups

The missing point that needed to be fixed was WIP visibility.

## Decision

The authorization and visibility model is defined as follows:

- Authentication is Google OAuth based.
- Users, groups, and memberships are managed inside the application, independent of Google Workspace groups.
- Group creation, group updates, group membership changes, and admin-group membership changes are restricted to users who belong to the admin group.
- Article lifecycle has exactly two states: `WIP` and `Published`.
- Article visibility has exactly two scopes: `Public` and `Group-only`.
- A `WIP` article is visible only to its author.
- A `WIP` article is excluded from shared lists, search results, notifications, comments, stars, and mentions for other users.
- A `Published` + `Public` article is visible to all authenticated users.
- A `Published` + `Group-only` article is visible only to the author and members of the target group.
- Access checks must be centralized and reused by article detail, list, search, notification, comment, and history endpoints.
- Update history follows the same visibility rule as the current article state.

## Consequences

- The backend needs one shared authorization predicate for article visibility instead of feature-specific ad hoc checks.
- Search indexing and notification fan-out must filter on the same visibility rule used by article reads.
- `WIP` content does not participate in collaborative features until publication.
