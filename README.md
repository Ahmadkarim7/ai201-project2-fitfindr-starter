# FitFindr

A multi-tool AI agent that helps users find secondhand clothing and figure out how to wear it. Given a natural-language request, FitFindr searches a mock listings dataset, suggests outfits using the user's existing wardrobe, and generates a shareable "fit card" caption.

## Tool Inventory

### `search_listings(description, size, max_price) → list[dict]`
- **Inputs:** `description` (str) — keywords to search for; `size` (str | None) — size filter, case-insensitive substring match; `max_price` (float | None) — inclusive price ceiling.
- **Output:** a list of matching listing dicts sorted by relevance (highest keyword-overlap score first). Returns an empty list when nothing matches.
- **Purpose:** filters and ranks the mock listings. Pure Python, no LLM call.

### `suggest_outfit(new_item, wardrobe) → str`
- **Inputs:** `new_item` (dict) — a listing dict; `wardrobe` (dict) — a dict with an `items` list.
- **Output:** a 2–4 sentence string suggesting 1–2 outfits using named wardrobe pieces. With an empty wardrobe, returns general styling advice instead.
- **Purpose:** styles the found item against the user's closet. Calls Groq `llama-3.3-70b-versatile`.

### `create_fit_card(outfit, new_item) → str`
- **Inputs:** `outfit` (str) — the suggestion from `suggest_outfit`; `new_item` (dict) — the listing dict.
- **Output:** a casual 2–4 sentence social caption mentioning the item, price, and platform. Returns an error message string if `outfit` is empty.
- **Purpose:** turns the outfit into a shareable post. Calls the LLM at high temperature so output varies each run.

## How the Planning Loop Works

The loop lives in `run_agent()` in `agent.py`. It is conditional, not a fixed sequence:

1. Initialize a `session` dict (the single source of truth).
2. Parse the query into `description`, `size`, and `max_price` via regex (`_parse_query`).
3. Call `search_listings`. **Branch point:** if results are empty, set `session["error"]` to a helpful message and **return early** — `suggest_outfit` and `create_fit_card` are never called.
4. If results exist, select the top one as `selected_item`.
5. Call `suggest_outfit` with the selected item and wardrobe.
6. Call `create_fit_card` with the outfit suggestion and selected item.
7. Return the session.

Because the loop returns early on empty search results, the agent's behavior genuinely differs between a successful query and an impossible one.

## State Management

A single `session` dict carries all state through the run. `search_listings` writes to `session["search_results"]`; the top result is stored in `session["selected_item"]` and passed directly into `suggest_outfit`; its output is stored in `session["outfit_suggestion"]` and passed into `create_fit_card`. The user never re-enters the item — it flows automatically from one tool to the next. `session["error"]` records early termination.

## Error Handling Strategy

- **`search_listings` — no matches:** returns `[]` (never raises). The agent detects this and stops with a message telling the user to loosen filters. Example: querying "designer ballgown size XXS under $5" returns an empty list, and the agent responds with guidance instead of crashing or calling downstream tools.
- **`suggest_outfit` — empty wardrobe:** detects an empty `items` list and prompts the LLM for general styling advice rather than referencing nonexistent pieces.
- **`create_fit_card` — empty outfit:** guards against an empty/whitespace `outfit` string and returns a descriptive error message string instead of raising.

## Spec Reflection

- **How the spec helped:** designing each tool's inputs, outputs, and failure mode in `planning.md` before coding meant the planning loop was mostly wiring pre-defined pieces together rather than improvising.
- **How implementation diverged:** rather than stripping filler words out of the query for the `description`, the whole query string is passed as the description. Because `search_listings` scores by keyword overlap, irrelevant words score zero and are harmless, so the extra parsing wasn't worth the complexity.

## AI Usage

1. **Tool implementations:** I gave the AI each tool's spec block (inputs, return value, failure mode) plus the data field names from `listings.json` and `wardrobe_schema.json`, and asked it to implement the functions using `load_listings()`. I reviewed each one to confirm it filtered by all parameters and handled its empty/error case before running it.
2. **Planning loop:** I gave the AI the planning-loop section and asked it to wire the tools through the `session` dict. I verified it branched on the empty-search result and returned early instead of calling all three tools unconditionally.

## Running It

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
# add GROQ_API_KEY to a .env file
python app.py
```