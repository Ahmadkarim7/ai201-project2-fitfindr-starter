# FitFindr — planning.md

A multi-tool agent that finds secondhand clothing, styles it against the user's wardrobe, and writes a shareable caption.

---

## Tools

### Tool 1: search_listings

**What it does:**
Searches the mock listings dataset for items matching keywords, an optional size, and an optional price ceiling, and ranks the matches by relevance.

**Input parameters:**
- `description` (str): keywords describing what the user wants (e.g. "vintage graphic tee").
- `size` (str | None): size to filter by, matched case-insensitively as a substring of the listing's size field. None skips size filtering.
- `max_price` (float | None): inclusive maximum price. None skips price filtering.

**What it returns:**
A list of listing dicts sorted by relevance, highest first. Each dict has: `id`, `title`, `description`, `category`, `style_tags` (list), `size`, `condition`, `price` (float), `colors` (list), `brand`, `platform`. Returns an empty list if nothing matches.

**What happens if it fails or returns nothing:**
Returns an empty list (never raises). The agent detects the empty list, sets an error message telling the user to loosen filters, and stops without calling the other tools.

---

### Tool 2: suggest_outfit

**What it does:**
Given a found item and the user's wardrobe, asks the LLM to suggest 1–2 complete outfits using named pieces the user already owns.

**Input parameters:**
- `new_item` (dict): the listing dict the user is considering.
- `wardrobe` (dict): a dict with an `items` key holding a list of wardrobe-item dicts. May be empty.

**What it returns:**
A non-empty string with 1–2 outfit suggestions referencing the new item and specific wardrobe pieces.

**What happens if it fails or returns nothing:**
If the wardrobe's `items` list is empty, it prompts the LLM for general styling advice for the item instead of referencing nonexistent pieces, so it always returns a useful string.

---

### Tool 3: create_fit_card

**What it does:**
Turns an outfit suggestion into a short, casual social-media caption for the thrifted find.

**Input parameters:**
- `outfit` (str): the outfit suggestion string from suggest_outfit.
- `new_item` (dict): the listing dict for the thrifted item.

**What it returns:**
A 2–4 sentence caption mentioning the item name, price, and platform naturally, with a vibe that varies each run (high LLM temperature).

**What happens if it fails or returns nothing:**
If `outfit` is empty or whitespace-only, it returns a descriptive error message string instead of raising or calling the LLM.

---

## Planning Loop

The agent runs a conditional loop in `run_agent()`. It parses the query, then calls `search_listings`. It checks the result: **if the list is empty, it sets `session["error"]` and returns early — the downstream tools are never called.** If results exist, it selects the top one, calls `suggest_outfit` with that item plus the wardrobe, then calls `create_fit_card` with the resulting suggestion. Behavior changes based on what search returns, so an impossible query and a normal query produce different paths. The loop is done once a fit card is produced or an error is set.

---

## State Management

A single `session` dict is the source of truth for one interaction. It holds the query, parsed parameters, search results, the selected item, the wardrobe, the outfit suggestion, the fit card, and an error field. Each tool writes its output into the session; the next tool reads from it. The top search result is stored in `session["selected_item"]` and passed directly into `suggest_outfit`; that output is stored in `session["outfit_suggestion"]` and passed into `create_fit_card`. The user never re-enters the item — it flows through the session automatically.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Returns `[]`; agent sets an error message ("No listings matched — try removing the size filter, raising your price limit, or using more general keywords") and stops before calling other tools. |
| suggest_outfit | Wardrobe is empty | Detects empty `items` list and returns general styling advice for the item instead of referencing pieces the user doesn't have. |
| create_fit_card | Outfit input is missing or incomplete | Guards against empty/whitespace `outfit` and returns a descriptive error string instead of crashing. |

---

## Architecture

User query

│

▼

Planning Loop (run_agent)

│

├─► _parse_query(query) → {description, size, max_price}

│

├─► search_listings(description, size, max_price)

│       │ results == []

│       ├──► [ERROR] session["error"] = "No listings found..." → return

│       │

│       │ results == [item, ...]

│       ▼

│   Session: selected_item = results[0]

│       │

├─► suggest_outfit(selected_item, wardrobe)

│       │

│   Session: outfit_suggestion = "..."

│       │

└─► create_fit_card(outfit_suggestion, selected_item)

│

Session: fit_card = "..."

│

▼

Return session

---

## AI Tool Plan

**Milestone 3 — Individual tool implementations:**
I used Claude. For each tool I provided that tool's spec block from this file (inputs, return value, failure mode) plus the actual field names from `listings.json` and `wardrobe_schema.json`, and asked it to implement the function using `load_listings()` from the data loader. Before running each one I checked that it filtered by all three parameters (for search_listings) and handled its documented failure case, then tested with real queries.

**Milestone 4 — Planning loop and state management:**
I gave Claude this Planning Loop section, the State Management section, and the architecture diagram, and asked it to implement `run_agent()` wiring the tools through the `session` dict. I verified it branched on the empty-search result and returned early rather than calling all three tools unconditionally, then ran `python agent.py` to confirm both the happy path and the no-results path behaved correctly.

---

## A Complete Interaction (Step by Step)

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:** `_parse_query` extracts `max_price=30.0`, `size=None`, and uses the full query as the description. The agent calls `search_listings(description=<query>, size=None, max_price=30.0)`, which returns a ranked list of matching tops. The agent stores them in `session["search_results"]` and picks the top one as `session["selected_item"]`.

**Step 2:** The agent calls `suggest_outfit(new_item=<selected item>, wardrobe=<example wardrobe>)`. The LLM returns a 2–4 sentence suggestion pairing the item with named pieces (e.g. baggy jeans, combat boots). Stored in `session["outfit_suggestion"]`.

**Step 3:** The agent calls `create_fit_card(outfit=<suggestion>, new_item=<selected item>)`. The LLM returns a casual caption mentioning the item, price, and platform. Stored in `session["fit_card"]`.

**Final output to user:** The UI shows three panels — the top listing's details, the outfit idea, and the fit card caption.