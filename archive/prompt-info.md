# CLAUDE PROMPT

> **Archived 2026-10-12.** The original kickoff prompt, kept for the record. Several of its
> suggestions were not taken — the database is MySQL, not Postgres; sign-in uses Cognito Managed
> Login, not the Hosted UI; and Zod, Zustand and TanStack Table are not used. What was actually
> decided is in [DECISIONS.md](../DECISIONS.md).

I am creating a new python app, AWS back end, web front end.  This is a fairly 
light-weight CRM-style, bespoke for managing live speaking and podcast gigs, 
with a pipeline, target tracking, client/company contact management.  
Not intended for public consumption at this point.

I presented this to claude online, with this additonal prompt:
> What are considered the best-practice front end packages in 
> terms of support, flexibility, best UI components.  Back end is serverless.

I got this response:

For a bespoke, behind-auth internal CRM sitting on a serverless Python backend, the pragmatic best-practice shape is a **React SPA built with Vite**, shipped as static files to S3 + CloudFront (the same hosting pattern you already run for the podcast), hitting API Gateway + Lambda, with Cognito for auth. Worth stating up front because it's a common trap: skip Next.js / SSR here. An internal tool with no SEO gains nothing from a server-rendered meta-framework, and a plain static SPA keeps deployment to "sync a build folder to a bucket," which stays true to the serverless model. Reach for a meta-framework only if you later want file-based routing plus server functions, in which case TanStack Start is the lighter fit over Next.js.

On React vs. the alternatives: for a data-heavy CRM (tables, pipeline board, forms, date pickers, contact management), React has the deepest ecosystem of exactly those building blocks and the best component-library support. Vue is perfectly capable but thinner for this class of app; Svelte is lighter but you'll hit gaps in the CRM-shaped component space.

**Component library — the core of your question.** In early 2026 the landscape has split into three camps: fully-loaded design systems (MUI, Ant Design), headless accessibility primitives (Radix, React Aria), and the copy-paste "own your code" style popularized by shadcn/ui. For your situation — backend-first, want to ship an internal tool without assembling a design system from primitives — my lead recommendation is **Mantine**. It's a comprehensive library with 120+ components and 70+ hooks, built around developer experience, accessibility, and a strong theming system with dark mode. Critically for a CRM, it excels at form management with built-in validation, and its hook library is valuable even if you don't use the UI components, and its docs are widely considered the best in the ecosystem. It gets you tables, forms, modals, date pickers, and notifications with the least assembly.

Two alternatives worth knowing:

- **shadcn/ui** — the top pick for most new React projects in 2026: Tailwind-native, copy-paste components on Radix primitives, zero runtime overhead. You own every file, which is great for a bespoke look, but it's more assembly and assumes Tailwind fluency. Pick this if you want maximum control and don't mind wiring more yourself.
- **Ant Design** — the most "CRM-shaped out of the box," with mature enterprise data grids and form systems. Best for B2B/admin panels where time-to-market beats design differentiation. The trade-off is a heavier bundle and opinionated styling you'll fight if you want a custom look later.

**The packages that matter more than the component library for a CRM.** This is where a pipeline/CRM app lives or dies, and it's largely library-agnostic:

- **TanStack Query (React Query)** — server state against your API: caching, background refetch, mutations, optimistic updates. Non-negotiable for CRUD-heavy work; it eliminates most of the manual loading/error state you'd otherwise hand-roll.
- **TanStack Table** (or `mantine-react-table`, which marries the two) — headless sorting/filtering/pagination for your target and contact lists.
- **Forms + validation** — Mantine's built-in `@mantine/form` covers most cases; if you want the ecosystem standard, **React Hook Form + Zod**. The Zod schemas also give you a single source of truth for shape/validation you can mirror on the Python side.
- **dnd-kit** — the modern, well-supported choice for the pipeline kanban drag-and-drop (status columns, cards, reordering).
- **Routing** — **TanStack Router** for end-to-end type safety, or **React Router** if you want the mainstream default.
- **TypeScript** — strongly recommended even coming from Python. For a CRM with real domain models (gigs, companies, contacts, pipeline stages), the type safety pays for itself quickly and makes AI-assisted refactoring far more reliable.
- **Auth** — for Cognito, `react-oidc-context` against the Hosted UI is clean and minimal for a few-user internal tool; Amplify Auth is the heavier, more batteries-included option.
- **Global client state** — you likely won't need much beyond TanStack Query + React state. If you do, reach for **Zustand**, not Redux.

If I had to compress it to one line: **Vite + React + TypeScript + Mantine + TanStack Query + dnd-kit**, on S3/CloudFront + Cognito. That gives you the fastest path to a working internal CRM with the least frontend yak-shaving, and every piece has strong support and an escape hatch if you outgrow it.

One thing worth deciding early that shapes the frontend: your data store on the backend (DynamoDB vs. RDS/Postgres). A relational pipeline CRM with lots of joins between contacts, companies, and gigs is often smoother on Postgres, and it affects how you design the API contracts the frontend consumes. Happy to go deeper on any layer — component library trade-offs, the Cognito wiring, or how to structure the API for the pipeline views.

Review the sibling legacy-tracker and job-tracker projects for possible baseline
architectures, review the prompt-info.md for additional info about the purpose.
This is not a gamified version, keep as more of a straight business app, but should not be
ugly.

Email capability should allow attachments and full formatting.  It should include
target goals.