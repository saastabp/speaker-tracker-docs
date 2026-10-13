# Speaker Tracker — Design (redirect)

This document was the original design of record. In 2026-10 its content was split into documents
that each have one job, and this page now only maps the old section numbers — which code comments
and applied migrations still cite, and which cannot be edited — to where that material lives now.

| Old section | Now |
|---|---|
| §1 Purpose and scope, and the boundary with the listener-conversion system | [PRODUCT.md §1](PRODUCT.md#1-purpose) |
| §2 Features | [PRODUCT.md §2](PRODUCT.md#2-features), with the rules in [§3](PRODUCT.md#3-rules-worth-knowing) |
| §3 Architecture — request path, email topology, scope, import, threading, credentials | [ARCHITECTURE.md](ARCHITECTURE.md), email in [§4](ARCHITECTURE.md#4-email) |
| §3 "Write invariant — one transaction per send" | [ARCHITECTURE.md §5.2](ARCHITECTURE.md#52-dual-writes-we-accept) |
| §4 Data model | [DATABASE.md](DATABASE.md) — the only schema reference; where §4 and a migration disagreed, the migration was right |
| §5 Research ("Kindling") fields | [PRODUCT.md §2](PRODUCT.md#venues-and-contacts) and [DATABASE.md `organizations`](DATABASE.md#organizations) |
| §6 Build order | [archive/DEV-PLAN.md](archive/DEV-PLAN.md) |
| §7 Decisions | [DECISIONS.md](DECISIONS.md) |
| §8 Mockup | [PRODUCT.md §5](PRODUCT.md#5-look-and-feel) and `samples/README.md` |
| §9 Branding | [PRODUCT.md §5](PRODUCT.md#5-look-and-feel) |
| §10 Baselines surveyed | [DECISIONS.md](DECISIONS.md) (the platform choices) |

The full original text is in git history.
