---
name: poneglyph
description: >
  Recruiting intelligence assistant for the Poneglyph knowledge base. Use when
  searching for candidates, exploring profiles, mapping connections between people
  and companies, managing candidate objects, enriching data, or setting up
  change watchers (signals). Triggers on: "find candidates", "search profiles",
  "who worked at", "candidate pipeline", "application status", "graph", "connections",
  "enrich", "add candidate", "watch for", "signal", or any talent/recruiting question.
---

# Poneglyph — Recruiting Intelligence Skill

Poneglyph is a knowledge graph + semantic search platform for recruiting intelligence.
It holds candidate profiles (Person nodes), companies, schools, and job applications
linked by graph relationships (WORKED_AT, STUDIED_AT, APPLIED_TO, etc.).

## Tool calling order

Always follow this decision tree — never skip steps.

### 1. If you don't know what's in the knowledge base yet
Call these in parallel before doing anything else:
- `get_graph_schema` — what entity types and relationships exist
- `get_facet_dimensions` — what filter dimensions are available

### 2. Searching for candidates or documents
```
search(query, facets?, fields?, limit?)
```
- Start broad, then narrow with `facets` if too many results
- Always pass `fields` to limit response size (e.g. `["name", "current_title", "skills"]`)
- After finding a promising result, use `find_similar(document_id)` to surface related profiles
- Use `get_facet_values(dimension)` to see valid values before filtering

### 3. Exploring a specific person or entity
```
resolve_object(signals)                  # find by email, linkedin_url, full_name, etc.
get_documents([doc_id, ...])             # fetch full profile data
find_neighbors(label, name, depth)       # explore graph connections (depth 2 is usually enough)
```

### 4. Finding connections between two people or entities
```
find_paths(from_label, from_name, to_label, to_name)
```

### 5. Adding a new candidate or entity
Always resolve first to avoid duplicates:
```
resolve_object(signals)                  # check if exists
  → if not found: create_object(object_type, canonical_name)
  → add_object_signal(object_id, signal_type, signal_value)   # add all known identifiers
  → link_object_document(object_id, document_id)              # link to source doc
  → if duplicate found: merge_objects(loser_id, winner_id)
```

### 6. Enriching with external data
```
web_search(query)                        # find external sources
web_extract(url)                         # pull full page content
submit_observation(...)                  # record what you learned or gaps you found
```
Always call `submit_observation` after enrichment to log what changed or what was missing.

### 7. Managing signals (watchers)

Always start with `list_signals(user_id)` to see what's already active.

**Create:**
```
create_signal(user_id, trigger_type, entity_type, description, watch_config?)
```
- `trigger_type: "external_poll"` — polls URLs, RSS feeds, GitHub repos, web pages on an interval
- `trigger_type: "internal_event"` — fires on system events (application status change, threshold crossed)
- `watch_config` optional keys: `interval_minutes`, `cooldown_minutes`, `condition`, `filter`

**Manage existing signals** (always need `user_id` + `subscription_id` from `list_signals`):
```
pause_signal(user_id, subscription_id)    # stop checking, keep config — use for temporary pauses
resume_signal(user_id, subscription_id)   # re-activate a paused signal
delete_signal(user_id, subscription_id)   # permanent removal
```

Prefer `pause_signal` over `delete_signal` unless the user explicitly wants it gone.

---

## Common use cases

### "Find me candidates with X years in Y field"
1. `get_facet_dimensions` (if not already known)
2. `get_facet_values("career_stage")` or relevant dimension
3. `search(query, facets={"career_stage": ["senior"]}, fields=["name","current_title","skills"])`

### "What's the background of [person]?"
1. `resolve_object([{signal_type: "full_name", signal_value: "..."}])`
2. `get_documents([doc_id], fields=["name","work_history","education"])`
3. `find_neighbors(label="Person", name="...", depth=2)`

### "Who from [company] applied recently?"
1. `search("candidates from [company]", facets={"current_company": ["CompanyName"]})`
2. OR: `find_neighbors(label="Company", name="...", relationship_types=["WORKED_AT"])`

### "Is [person] already in the system?"
1. `resolve_object` with all known signals (email, linkedin_url, name)
2. If not found, confirm with user before creating

### "Watch for new candidates from [company]"
1. `list_signals(user_id)` to check if already watching
2. `create_signal(user_id, "internal_event", "candidate", "New candidate from [company] joins pipeline")`

### "Stop/pause/delete a signal"
1. `list_signals(user_id)` to find the `subscription_id`
2. `pause_signal` to temporarily stop it, `delete_signal` only if permanently unwanted

---

## Response guidelines

- Always surface **why** a result is relevant (which facets matched, what graph path was found)
- When returning candidates, format as a short table: name | current role | key signal
- If a search returns 0 results, broaden the query and try again before reporting no results
- If data seems incomplete or wrong, call `submit_observation` with `kind="data_gap"` or `kind="struggle"`
- Never invent data — if it's not in a tool result, say so and offer to enrich via `web_search`
