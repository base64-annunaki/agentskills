---
name: poneglyph
description: >
  Read-only recruiting intelligence assistant for the Poneglyph knowledge base.
  Use when searching for candidates, exploring profiles, filtering or counting
  cohorts in the graph, mapping connections between people and companies, or
  enriching a profile with public web data.
  Triggers on: "find candidates", "search profiles", "who worked at",
  "who worked with", "candidate pipeline", "graph", "connections", "enrich",
  or any talent/recruiting question. Do not use to create, edit, or monitor
  anything — this surface has no write tools and no signals or watchers.
---

# Poneglyph — Recruiting Intelligence Skill

Poneglyph is a knowledge graph + semantic search platform for recruiting intelligence.
It holds candidate profiles (Person nodes), companies, schools, and skills
linked by graph relationships (WORKED_AT, STUDIED_AT, HAS_SKILL).

Every tool on this surface reads. Nothing creates, updates, or watches data.

## Tool calling order

Always follow this decision tree — never skip steps.

### 1. If you don't know what's in the knowledge base yet
Call these in parallel before doing anything else:
- `get_graph_schema()` — node labels and relationship types, with counts
- `get_facet_dimensions()` — what filter dimensions are available

Known facet dimensions: `current_company`, `education_level`, `location`, `seniority_level`, `skill`, `total_companies`, `years_experience`

Scope comes from your API key — leave `vertical_id` unset unless the user names one.

### 2. Describing a person — semantic search
```
search(query, facets?, fields?, limit?, offset?)
```
- `search` is semantic over each person's OWN profile text. It does not traverse the graph, so relational questions ("who worked with X", "colleagues of Y") cannot be answered here — go to step 5.
- Start broad, then narrow with `facets` if too many results
- Use `get_facet_values(dimension)` to see valid values before filtering
- After a promising result, `find_similar(document_id)` surfaces related profiles
- Paginate with `offset` — e.g. `limit=10, offset=10` for page 2. Check `totalCount` to know when you've reached the end.

### 3. Hard predicates and counts — query the graph
```
query_graph_nodes(label, filters?, has_neighbors?, order_by?, limit?, cursor?)
aggregate_graph(cohort_filter, group_by, order?, limit?, offset?)
```
- Use `query_graph_nodes` when the question is a rule, not a vibe: each `has_neighbors` entry requires a 1-hop relationship to exist, and several are ANDed — so "has both these skills" or "founder AND advisor" is one call. Matched neighbours are not returned, only the nodes themselves.
- Use `aggregate_graph` for "most common / rarest / top N X across a cohort". `cohort_filter` defines the population as `{dimension: [values]}`, `group_by` names the dimension to count.
- Both need real labels and edge types — call `get_graph_schema` first.

### 4. Exploring a specific person or entity
```
resolve_object(signals=[{signal_type, signal_value}])  # email, linkedin_url, full_name, … -> existing objects
batch_get_nodes(node_ids=[...], type?)                 # up to 50 node UUIDs in one round trip
get_documents(document_ids=[...])                      # full document records by document ID
find_neighbors(node_id | label+name, depth?, relationship_types?, limit?)
```
- Once search, `query_graph_nodes`, `find_paths` or `resolve_object` has handed you node IDs, hydrate them with one `batch_get_nodes` call — not one `find_neighbors` per person.
- `batch_get_nodes` accepts internal node UUIDs only. Resolve an email or LinkedIn URL through `resolve_object` first.
- Its `graph_context` counts are not vertical-scoped — treat them as advisory, never quote one as a scoped total.

### 5. Relational questions — "who worked with X"
```
find_paths(from_label, from_name, from_relationship_types?, via_labels?, to_labels?, ...)
```
- Omit every `to_*` field to get DISCOVERY mode: entities reached through a shared neighbour. This is the route for colleagues and classmates — e.g. `from_relationship_types=["WORKED_AT"]`, `via_labels=["Company"]`, `to_labels=["Person"]`. Add `correlation_start_property` / `correlation_end_property` to require overlapping dates ("at the same time").
- Supplying `to_*` gives pairwise shortest-path mode, which currently under-returns: two people connected only through a shared employer come back empty rather than as an error. Never read an empty pairwise result as "no connection" — re-ask it as discovery, or cross-check with `find_neighbors` / `query_graph_nodes`.

### 6. Enriching with external data
```
web_search(query, limit?, provider?)
web_extract(url)
list_observations(page_size?, offset?)
```
- The web tools read public sources. Nothing you find is written back to the knowledge base — put it in your answer.
- `list_observations` is read-only: review what has already been flagged instead of repeating it.

---

## Common use cases

### "Find me candidates with X years in Y field"
1. `get_facet_dimensions` (if not already known)
2. `get_facet_values("seniority_level")` to see valid values
3. `search(query, facets={"seniority_level": ["senior"]}, limit=10)`

### "What's the background of [person]?"
1. `search(query="[person name]", limit=3)` — or `resolve_object` if you have an email or LinkedIn URL
2. `batch_get_nodes(node_ids=[...])` for the full record
3. `find_neighbors(node_id="...", depth=2)` for what they're attached to

### "Who worked with [person]?"
1. `get_graph_schema()` to confirm the labels and edge type
2. `find_paths(from_label="Person", from_name="...", from_relationship_types=["WORKED_AT"], via_labels=["Company"], to_labels=["Person"])`
3. Add `correlation_start_property` / `correlation_end_property` to keep only overlapping tenures

### "Who from [company] is in the base?"
1. `find_neighbors(label="Company", name="...", relationship_types=["WORKED_AT"])`
2. OR `query_graph_nodes(label="Person", has_neighbors=[{"edge_type": "WORKED_AT", "neighbor_label": "Company", "neighbor_filters": [{"field": "canonical_name", "op": "eq", "value": "..."}]}])`

### "What's the most common employer among senior [skill] engineers?"
1. `get_facet_values` on each dimension you plan to filter on
2. `aggregate_graph(cohort_filter={"skill": ["Rust"], "seniority_level": ["senior"]}, group_by="current_company")`

### "What data quality issues have been flagged?"
1. `list_observations()` — returns newest first
2. Filter by `kind` (struggle, data_gap, suggested_change, observation) in your response

---

## Response guidelines

- When your answer contains a **list of entities**, call `present_entity_cards(groups=[...])` — one group per category, each entity with `title`, `subtitle`, `body`, `fields`, `tags` — and keep the analysis in your prose. The cards only render in the Poneglyph web UI; in a terminal client the call returns a bare acknowledgement, so write the list there as a short table: name | current role | key signal.
- Always surface **why** a result is relevant (which facets matched, what graph path was found)
- If a search returns 0 results, broaden the query and try again before reporting no results
- Never invent data — if it's not in a tool result, say so and offer to enrich via `web_search`
- If the user asks to save, correct, or monitor something, say the surface is read-only rather than attempting a call
