1. Eager / lazy loading of TypeORM relations
When an entity has a relation (like your TaskEntity has visits, agent, deposits), TypeORM has to decide when to fetch that related data.

Eager = load the relation automatically, every time you load the parent.


@OneToMany(() => VisitEntity, v => v.task, { eager: true })
visits: VisitEntity[];
Now every find on a task also pulls all its visits — even on a list screen where you only need the customer name. Wasteful, and a common cause of slow endpoints.

Lazy = don't fetch it until you actually ask for it (default behavior when eager: false). You pull the relation only when needed, explicitly:


// only join visits when THIS endpoint needs them
const task = await taskRepo.findOne({
  where: { id },
  relations: { visits: true },   // opt-in per query
});
"Tuning" = fixing the mismatch. In your code you saw relations like:


@OneToMany('VisitEntity', 'task_id')
visits: VisitEntity[];

@ManyToOne(() => FieldAgent, ...)
agent: FieldAgent;
Tuning them means:

Turn OFF eager on heavy relations so list endpoints don't drag unnecessary joins.
Load relations per-query (relations: {...} or query builder .leftJoinAndSelect) only on the endpoints that need them.
This is also how you kill N+1 queries — instead of TypeORM firing 1 query for tasks + 1 per task for its agent (101 queries for 100 tasks), you do a single JOIN.
Interview one-liner:

"Eager loading fetches relations automatically with the parent; lazy loading defers until accessed. I tuned relations to lazy/opt-in and loaded only what each endpoint needed via explicit joins — that removed unnecessary data fetching and N+1 query patterns on list endpoints."

⚠️ One honesty note: in TypeORM specifically, "lazy relations" technically means typing the field as Promise<VisitEntity[]>. In interviews people use "lazy vs eager" loosely to mean "auto-load vs load-on-demand," which is the concept above. If asked, mention eager: true/false and per-query relations — that's what you actually control.

2. p95 (95th percentile latency)
It's a way to measure API response time that's honest about your slow requests, unlike an average.

Say 20 requests hit an endpoint. Sort their response times fastest → slowest. p95 = the value below which 95% of requests fall — basically your "second-worst" experience.

p50 (median) = 100ms → half your requests are faster than this
p95 = 400ms → 95% are faster; the slowest 5% are worse
p99 = 900ms → only 1% are worse
Why not just use average? Averages hide pain. If 95 requests take 50ms and 5 take 3000ms, the average looks okay-ish but 5% of your users are suffering. p95/p99 expose that "tail latency."

Why it's the standard metric: teams set SLAs like "p95 < 200ms" because it reflects what most users actually feel, while ignoring one-off freak spikes.

Interview one-liner:

"p95 is the response time under which 95% of requests complete. I track it instead of the average because averages hide tail latency — after optimization my p95 dropped from ~Xms to ~Yms."