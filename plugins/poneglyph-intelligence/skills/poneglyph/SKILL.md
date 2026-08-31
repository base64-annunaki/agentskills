---
name: poneglyph
description: >
  Recruiting intelligence assistant for the Poneglyph knowledge base. Use when
  searching for candidates, exploring profiles, mapping connections between people,
  companies and schools, resolving domain shorthand, enriching data, or setting up
  recurring automations. Triggers on: "find candidates", "search profiles",
  "who worked at", "candidate pipeline", "graph", "connections", "career path",
  "skill network", "enrich", "watch for", "automation", or any talent/recruiting question.
---

# Poneglyph — Recruiting Intelligence Skill

Poneglyph is a knowledge graph + semantic search platform for recruiting intelligence.
It holds candidate profiles (Person nodes), companies, schools, and skills linked by
graph relationships (WORKED_AT, EDUCATED_AT, HAS_SKILL). Many tools accept an optional
`bucket_id` to scope to a job, role, or client segment.

## Tool-calling order

Follow this decision tree. Discover before you search; resolve shorthand before you guess.

### 1. Discover what's in the knowledge base
When you don't yet know the shape of the data, call these first (in parallel):
- `get_graph_schema` — entity/node types (labels) and relationship types, each with a count
- `get_facet_dimensions(bucket_id?)` — available filter dimensions for the workspace
- `list_domain_terms` — the workspace's defined shorthand (acronyms, company/school cohorts, title/seniority tiers)

### 2. Resolve domain shorthand — never guess an acronym
If the user uses an acronym, cohort name, title tier, or other shorthand:
- `resolve_domain_term(text)` — returns the canonical entity (aliases, kind, member node IDs, expansion `values`). Call this BEFORE searching, then search with what it returns. When a resolved term carries `values`, expand those into your search filter instead of searching the raw term.
- `teach_domain_term(term, values, kind?, description?)` — when a term does NOT resolve AND a search for it also returns nothing, do not conclude "no data": ask the user what the term means (which concrete values it maps to), then persist their answer here so it resolves on the next turn. Never invent a mapping — store only what the user states.

### 3. Search for candidates or documents
Route by the kind of query:
- `text_search(query, ...)` — exact names, tokens, or rare terms (a literal string to look up)
- `vector_search(query, ...)` — concepts or fuzzy attributes ("people excited about climate")
- `facet_search(facets, ...)` — filtered search once exact facet dimensions + values are known (after `get_facet_values(dimension)`)
- `paginate_search(...)` — the next page of a prior `vector_search` / `facet_search` when `total_count` exceeds the page

### 4. Explore the graph around a person or entity

**Relationship-shaped questions ("who worked/served/lived/litigated with X", "who is related to X by rule R") are NEVER answered by document search** — the related entity's document does not mention X; the relationship exists only as graph edges. Use the graph and interpolate edge meaning: two edges to the same neighbor node with overlapping date properties = "together at the same time". Recipe: `get_graph_schema` to learn the vertical's relationship types → `find_paths` in discovery mode (omit the `to_*` fields) with the rules → narrow topically with `semantic_query` (a built-in topical relevance intersection that also returns the true `total_count` of the relevant set), or coarsely with `edge_filter_*`.

- `find_neighbors(label, name, depth)` — transitively-connected entities, "who connects to X" (depth 2 is usually enough)
- `network_proximity(name, hops)` — people within N hops of a given person (coworkers, classmates)
- `find_paths(from_label, from_name, to_label, to_name)` — shortest path(s) between two known entities
- `find_paths(from_..., NO to_...)` — **discovery mode**: entities related to X through a shared neighbor under declarative rules — `from_relationship_types` / `to_relationship_types` (per hop), `via_labels` / `to_labels`, `via_name`, interval overlap via `correlation_start_property` / `correlation_end_property` / `min_span_months`, `edge_filter_property` + `edge_filter_contains`, `semantic_query` (rank/filter the candidates by topical relevance — returns the true `total_count` of the relevant set), `limit` / `offset` (span-ordered). The rules are generic — same call shape for colleagues (`WORKED_AT`/`Company`, overlap `start_date`/`end_date`), classmates (`EDUCATED_AT`/`School`, overlap `start_year`/`end_year`), co-parties (`PARTY_TO`/`Case`, no correlation), co-residents (`RESIDED_AT`/`Building`, lease-date overlap)
- `career_path(...)` — people who moved from one company to another, ordered by transition date ("who left Stripe for Anthropic?")
- `skill_network(skill)` — people with a named skill plus the adjacent skills they also hold
- `graph_search(...)` — filter subjects by exact graph connections (skill / company / school edges)
- `graph_query_nodes(...)` / `graph_query_edges(...)` — filter, order, paginate the live Memgraph projection
- `query_nodes(...)` / `query_edges(...)` — filter, order, paginate node / edge rows in Postgres

### 5. Remember facts across turns
- `submit_observation(...)` — save a fact the user stated about an entity ("she relocated to Amsterdam", "he rejected our offer")
- `list_observations(...)` — read saved facts, newest first; call at the start of a candidate turn to recall context (and before submitting, to avoid duplicates)

### 6. Enrich with external data
- `run_agent_task(prompt)` — a short-lived research sub-agent (max ~5 turns) with web search/fetch; returns a text summary
- `trigger_enrichment(...)` — spawn an LLM-planned recurring background automation to close a data gap or resolve low-confidence data (expensive)
- `twitter_user_lookup(handle)` / `twitter_user_recent_posts(handle)` — public X (Twitter) profile and recent original posts

### 7. Automations (recurring monitors / watchers)
Always start with `list_automations()` to see what's active.
- `create_automation(...)` — a recurring monitor that fires when a candidate query yields new matches or a watched page / blog / RSS feed changes
- `update_automation(...)` — patch interval / description / goal without recreating it
- `pause_automation(id)` / `resume_automation(id)` — temporarily disable / re-enable (prefer pausing over deleting)
- `delete_automation(id)` — permanent removal; use only when the user explicitly wants it gone
- `run_automation(id)` — execute an existing automation once, right now
- Automation **actions** (what fires on trigger): `send_email`, `send_telegram`, `call_webhook`

### 8. Specialist subagents
- `subagent-dispatch` — when and how to dispatch a researcher / graph-analyst / profiler via the Agent tool, and how to consume its results

---

## Presenting results — REQUIRED output formats
- `candidate-card-widget` — the host-styled `<widget>` card layout for candidate / document search results; use this instead of prose lists
- `person-widget` — the card layout for person results (with optional tabs)
- `talent-evaluation` — the company-tier (T1–T5) framework for scoring, calibrating, and ranking software-engineering candidates

## Common use cases

### "Find me senior engineers with X years in Y field"
1. `get_facet_dimensions` (if not already known)
2. `get_facet_values("seniority_level")` for valid values
3. `facet_search(facets={"seniority_level": ["senior"]}, ...)` → render with `candidate-card-widget`

### "Find people who left [Company A] for [Company B]"
1. `career_path(...)` from A to B

### "Who is connected to [person], and how?"
1. `network_proximity(name="...", hops=2)` or `find_neighbors(label="Person", name="...", depth=2)`
2. `find_paths(...)` for a specific A→B connection

### "Who worked with [person] at [company] (at the same time)?" — and every other "related to X by rule R"
NEVER text/vector search — the graph is the only source of relationships.
1. `get_graph_schema` → the vertical's relationship types
2. `find_paths(from_node_id=<X>, from_relationship_types=["WORKED_AT"], via_labels=["Company"], via_name="<company>", correlation_start_property="start_date", correlation_end_property="end_date")` — omit all `to_*` fields
3. Narrow topically: prefer `semantic_query="‹the topic in natural language›"` (built-in relevance intersection + true `total_count`); or coarsely `edge_filter_property="title"` + `edge_filter_contains="..."`, raise `min_span_months`; page with `offset`

**A topical qualifier ("on ‹topic›", "in ‹field›") is a semantic concept, not a keyword.** `edge_filter_contains` is only a COARSE literal pre-filter — it also matches longer words that merely contain the substring (e.g. it will match "**re**search" roles) and misses records that express the concept in other words or in a title that never spells it out. So don't treat a raw substring filter as the answer: prefer `find_paths`'s **`semantic_query`**, which does the topical relevance intersection server-side in one call and returns the true `total_count` of the relevant set (use it as a rough first cut only when a semantic query doesn't fit). Interpret each topic on its own terms — there is an unbounded space of possible qualifiers, so never hardcode or assume a fixed keyword.

### "Who in our base is a '<acronym / cohort>'?"
1. `resolve_domain_term("<acronym>")` → search using the returned `values` / members
2. If unresolved AND the search is empty → ask the user, `teach_domain_term(...)`, then re-search

### "Watch for new candidates from [company]"
1. `list_automations()` to check if already watching
2. `create_automation(...)` with the query + an action (`send_email` / `send_telegram`)

### "What facts have we saved about [person]?"
1. `list_observations()` — newest first

---

## Response guidelines
- Resolve shorthand with `resolve_domain_term` before searching — never guess what an acronym means.
- Route search by query type: `text_search` for literal names, `vector_search` for concepts, `facet_search` for known filters.
- If a term is unknown AND a search for it is empty, ASK the user and `teach_domain_term` — do not conclude "no data."
- Render candidate / person results with the widget skills, not prose lists. Always surface **why** a result matched (which facets, what graph path).
- Never invent data — if it's not in a tool result, say so and offer to enrich (`run_agent_task` / `trigger_enrichment`).
- Never surface internal IDs (node / document / UUID) to the user.
