# agent.md — Answer Style Guide for This Journal

> Reference for how notes/answers in this repo (especially the `2026/` folder) are written.
> When asked to add a new topic or answer questions "in the same way," follow this.

## What this repo is
A personal **Daily Learning Journal** by a working Software Engineer. Notes are
written to be **re-read before interviews** and **used on the job** — not academic.
See `README.md` for the topic list and the stack in play.

## The voice
- **Practical over theoretical.** Explain the concept, then show where it actually bites.
- **Honest.** Call out trade-offs, gotchas, and "here's the part people get wrong."
- **Concise but concrete.** Use real numbers (`p95 400ms`, `101 queries for 100 tasks`),
  real tools, and short code/PromQL/YAML snippets where they clarify.
- **First-person where natural** ("I tuned relations to lazy…") — these are the author's notes.

## The author's real stack (ground answers in this, not generic examples)
- **Backend:** Node.js, NestJS, TypeORM
- **Data/cache:** PostgreSQL/MySQL, Redis
- **Infra:** Docker / docker-compose, AWS, Nginx / reverse proxy
- **Observability:** Prometheus + Grafana (already wired), APM tools
- **Real projects referenced:** Oceanevoke (e-commerce/WooCommerce), esolveinfotech,
  `tejas_network` compose stack, ShipRocket integrations
- **Also studying:** Java, Spring Boot, Microservices, Kafka, Kubernetes, React

## Structure patterns (pick what fits the topic)
1. **Concept notes** → for each idea: **Definition (simple)** → **Real-world use case**
   → **Why we use it** → **Market options today**. (See `30 System Design Concepts in Short.md`.)
2. **How-to / runbook** → numbered options ranked cheapest→best, then step-by-step
   setup with copy-pasteable snippets and a "secure it / validate it" step.
   (See `How to Measure p95 Latency.md`.)
3. **Scenario / interview questions** → restate the question, give a structured
   walk-through (what I'd check first → next → fix), end with an **Interview one-liner**.

## Non-negotiable ingredients
- **Interview one-liner** — most topics end with a quotable 1–2 sentence summary the
  author can say out loud in an interview.
- **Real-world use case** — tie every concept to something concrete, ideally the stack above.
- **Gotcha / honesty note** — where relevant, a ⚠️ note on what people get wrong
  (e.g. TypeORM "lazy" actually means `Promise<T>`; unbounded Prometheus label cardinality).
- **Market/tool options** — name the actual tools used in industry today.

## Formatting
- Markdown. `#` title, `>` blockquote for the format legend, `---` between sections.
- Bold the labels (**Definition:**, **Why:**, **Interview one-liner:**).
- Keep snippets minimal and language-tagged where useful.
- File names are Title Case and descriptive (e.g. `How to Measure p95 Latency.md`).

## Where things go
- New topic notes → `2026/<Descriptive Title>.md`
- Update `README.md`'s topic/log list only if the author asks.
