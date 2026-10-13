# Architecture diagrams

Two hand-authored [draw.io](https://www.drawio.com) diagrams of the **production** environment, each
with PNG and SVG exports. They are publishable: account ids, resource ids, domains and mailbox
addresses are replaced with placeholders (`app.example.com`, `user@example.com`). The real values
live in `infra/cdk/lib/config.ts`.

| Source | Exports | Shows |
|---|---|---|
| [`system-overview.drawio`](system-overview.drawio) | [PNG](system-overview.png) · [SVG](system-overview.svg) | Every deployed component, and how requests, data, schedules and failures move between them |
| [`email-round-trip.drawio`](email-round-trip.drawio) | [PNG](email-round-trip.png) · [SVG](email-round-trip.svg) | One send, its reply, and the reply's import, as a numbered sequence |

The prose behind both is in [ARCHITECTURE.md](../ARCHITECTURE.md) — §4 for email, §5 for background
work, §6 for infrastructure. Where a diagram and the code disagree, the code is right.

Icons are the AWS Architecture Icons in draw.io's built-in `mxgraph.aws4` library. Open a source in
[app.diagrams.net](https://app.diagrams.net), draw.io Desktop, or the VS Code *Draw.io Integration*
extension.

## Reading the edges

| Style | Meaning |
|---|---|
| Solid black | Request or data path |
| Dashed black | Async, scheduled, configuration or credential read |
| Dashed red | Failure path |

## Sandbox differs

The diagrams show production. Sandbox deploys only the Messaging, Api and Frontend stacks, so it has
**no Cognito, certificate or DNS records**: it is served on its CloudFront domain, and its API
routes run without the JWT authorizer (`authMode: 'dev'`). Its **poller is deployed but not
scheduled** — exactly one environment may poll the mailbox. Everything else is the same shape.

## Keeping them current

The diagrams do not regenerate themselves. A change to the stacks in `infra/cdk/lib/` that adds,
removes or rewires a resource updates the matching cells in the same commit — and the exports, which
are regenerated from the sources, never edited.

## Exporting

From this directory, with draw.io Desktop on `PATH` as `drawio`:

```bash
for n in system-overview email-round-trip; do
  drawio --user-data-dir="$(mktemp -d)" --no-sandbox -x -f png -s 1 -o "$n.png" "$n.drawio"
  drawio --user-data-dir="$(mktemp -d)" --no-sandbox -x -f svg       -o "$n.svg" "$n.drawio"
done
```

- **Never pass `-e`.** It embeds the full diagram XML in the export. The exports are meant to be the
  rendered picture only; the `.drawio` file is the source.
- **`--user-data-dir`** with a fresh directory avoids draw.io's single-instance lock. Without it, the
  second and later invocations hang with no output instead of exporting.
- **`-p` is 1-indexed** (since draw.io 27.0.2). Each source here has one page, so it is not needed.
