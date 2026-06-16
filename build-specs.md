Build Spec v2
FRIDAY, JUNE 12 2026
BRD Analyzer
Build Prompt
A browser-based tool that builds an entity-relationship graph from a folder of markdown BRDs, then traverses that graph to surface connected features, shared definitions, and affected flows when you paste a new or modified BRD. Inspired by GitNexus's knowledge graph for code — but instead of functions and call chains, the graph maps domain entities, their attributes, business rules, and the relationships between them. The graph is the primary intelligence layer; text search is a fallback for recall.

I. What You're Building
1. Core loop
User loads a folder of markdown BRDs. The tool parses every doc, uses an LLM to extract entities and relationships, and builds a domain knowledge graph. Then the user pastes a draft BRD, clicks "Analyze," and the tool extracts entities from the draft, finds them in the graph, traverses their connections, and returns a structured impact report.

Three-pane layout: left pane is the input area, center pane is the context panel with results, right pane is an interactive graph visualization showing the entity neighborhood.

2. Context panel
Four sections, ordered by connection strength:

Directly Referenced — docs the pasted BRD explicitly links to. Deterministic, parsed from markdown.
Entity Connections — docs that share entities with the pasted BRD, found via graph traversal. Shows the connection path: "Your BRD modifies User.email → User.email is REQUIRED_BY Notification Preferences (BRD-042) → User.email is VALIDATED_AT Signup Flow (BRD-018)." This is the primary value.
Shared Definitions — definition/spec docs that define entities your BRD touches. Shows the current field list, constraints, and which other BRDs reference the same definition.
Weakly Related — docs found via text similarity (TF-IDF fallback) that didn't surface through the graph. A safety net for connections the LLM extraction missed.
3. Non-goals for v1
No live-updating editor — paste + click is enough.
No persistent backend — everything client-side, graph cached in localStorage/IndexedDB.
II. The Domain Graph
1. Node types
Four kinds of nodes in the graph:

Node Type	Examples	Extracted From
Entity	User, Walker, Job, Lock, MiniHub, CPF Contribution, Notification	LLM extraction from BRD content — the core domain objects
Attribute	User.email, User.first_name, Job.status, Walker.kyc_status	LLM extraction — fields, properties, and data points of entities
Rule	"Email is required for notifications," "Walker must complete KYC before accepting jobs"	LLM extraction — business rules, constraints, validation logic, preconditions
Document	01-app-init.md, 02.01-sign-in-as-a-walker.md	File system — each markdown BRD is a document node
2. Edge types
Relationships between nodes:

Edge	From → To	Meaning
HAS_ATTRIBUTE	Entity → Attribute	User HAS email, Job HAS status
RELATES_TO	Entity → Entity	Walker RELATES_TO Job (walker completes jobs), User RELATES_TO Notification
CONSTRAINED_BY	Entity or Attribute → Rule	User.email CONSTRAINED_BY "required for notifications"
DEFINES	Document → Entity/Attribute/Rule	BRD-042 DEFINES the notification preferences feature
REFERENCES	Document → Entity/Attribute	BRD-018 REFERENCES User.email (uses it, doesn't define it)
LINKS_TO	Document → Document	Explicit markdown link between docs
The key distinction: DEFINES means the doc is the source of truth for that entity/attribute. REFERENCES means it depends on or uses it. This matters for impact analysis — changing a definition affects all references.

3. Document metadata
Each document node carries structural metadata from the file system:

id — relative file path
title — from first heading or cleaned filename
platform — "web-app", "mobile-app", or "shared" (inferred from path)
featureArea — "authentication", "users", "transactions", etc. (inferred from path)
docType — "specification", "definition", or "flow" (inferred by LLM: does this doc define entities, or describe a user flow?)
III. Indexing Pipeline
1. Parse markdown
Use a markdown parser (marked or remark) to extract structure:

Heading hierarchy → sections array
Internal links → LINKS_TO edges between documents
Tables and field lists → hints for the LLM extraction step
Image paths → stored for display alongside results
2. LLM entity extraction
For each parsed doc, send its content to an LLM with a structured extraction prompt. The prompt asks the LLM to return JSON with:

{
  "docType": "specification" | "definition" | "flow",
  "entities": [
    {
      "name": "User",
      "role": "defines" | "references",
      "attributes": [
        { "name": "email", "type": "string",
          "constraints": ["required", "unique"] }
      ]
    }
  ],
  "relationships": [
    { "from": "Walker", "to": "Job",
      "type": "RELATES_TO",
      "description": "Walker completes jobs" }
  ],
  "rules": [
    { "description": "Email required for notification preferences",
      "applies_to": ["User.email", "Notification"] }
  ]
}
Use gpt-4o-mini or equivalent — cheap enough to run on 500 docs (roughly $1-2 total). Batch docs in groups of 5-10 per request to reduce latency. Include a few-shot example in the prompt with domain-specific examples from the OOLE codebase.

3. Entity resolution
Different BRDs will refer to the same entity with different names. After extraction, run a merge pass:

Normalize entity names: lowercase, singular form, strip articles ("the user profile" → "user profile" → "user_profile").
Merge aliases: if two docs extract "User" and "User Profile" and both define email and first_name, merge into one entity node with both names as aliases.
Use a second LLM call on the full entity list to resolve ambiguities: "Are 'Walker' and 'Service Provider' the same entity?" Return a merge map.
Deduplicate attributes across entities — User.email from BRD-018 and User.email from BRD-042 become one attribute node with two document edges.
4. Extraction review gate
Entity resolution auto-merges entities based on an LLM merge map. A wrong merge ("Walker" = "Service Provider" when they're distinct) silently poisons the graph and every downstream impact analysis. The review gate surfaces the extracted model before it's committed to the graph, so the user can catch bad extractions and merges.

The gate is configurable via a "Review mode" setting (top bar settings, see UI spec). Three modes:

Mode	Behavior	When to use
After extraction (default)	Pause after extraction + resolution. Show the full extracted model — entity list, attributes, relationships, rules, and the proposed merges — in an editable review panel. Nothing is written to the graph until the user clicks "Approve & build."	First index of a folder, or when extraction quality is unknown.
Merges only	Run automatically, but pause only on ambiguous merge decisions from the second LLM call (e.g. "Are 'Walker' and 'Service Provider' the same?"). User confirms or rejects each proposed merge; everything else proceeds untouched.	Re-indexing a folder you've vetted before, where only new merges are risky.
Off	Fully automatic. No gate — trust the LLM end-to-end and build the graph immediately.	Trusted folders, CI/headless runs, or the MCP reindex path.

Review panel actions:
- Approve/reject each proposed merge individually.
- Edit an entity's role (DEFINES ↔ REFERENCES), rename it, or split a wrongly-merged entity back apart.
- Delete a hallucinated entity, attribute, relationship, or rule.
- "Approve & build" commits the reviewed model to the graph.

The setting persists in localStorage and defaults to "After extraction." The MCP brd_reindex tool always runs in "Off" mode since it has no interactive surface.
5. Build graph
Assemble all nodes and edges into an in-memory graph.

Use a lightweight graph library (graphology) or a simple adjacency list.
Store the full graph as JSON in IndexedDB for fast reload — keyed by a content hash of the folder so stale caches auto-invalidate.
Build a secondary TF-IDF index across all doc content as a text-search fallback.
6. LLM extraction prompt
Use this system prompt for the extraction step:

You are analyzing a Business Requirements Document
(BRD). Extract the structured domain model from
this document.

For each entity mentioned:
- Identify whether this doc DEFINES it (source of
  truth for its structure) or REFERENCES it (uses
  it but defines it elsewhere).
- Extract its attributes with types and constraints.

For relationships:
- Only extract relationships explicitly stated or
  strongly implied by the doc.
- Use verb phrases: "Walker COMPLETES Job",
  "User RECEIVES Notification".

For business rules:
- Extract constraints, preconditions, validation
  rules, and behavioral requirements.
- Link each rule to the entities/attributes it
  applies to.

Return valid JSON matching the schema provided.
Do not invent entities or relationships not
supported by the document text.
IV. Query Flow (Analyze)
When "Analyze" is clicked
Run these steps on the pasted BRD:

Step 1 — Extract from draft: Send the pasted BRD through the same LLM extraction prompt. Get its entities, attributes, relationships, and rules.
Step 2 — Match to graph: For each extracted entity/attribute, find its node in the graph (fuzzy match on name + aliases). This is the set of "touched nodes."
Step 3 — Traverse connections: From each touched node, traverse outgoing and incoming edges up to depth 2. Collect all connected documents, with the connection path. Score by: (a) edge distance (depth 1 = high, depth 2 = medium), (b) edge type (CONSTRAINED_BY and DEFINES edges are highest priority), (c) folder proximity boost.
Step 4 — Resolve explicit links: Parse markdown links in the pasted text, match against indexed docs.
Step 5 — Text fallback: Run TF-IDF similarity on the pasted text against all indexed docs. Remove any docs already found in steps 3-4. Return top 5 as "Weakly Related."
Step 6 — Assemble results: Deduplicate, group into the four context panel sections, and render.
Step 7 — Judge conflict severity: For each surfaced connection, use a lightweight LLM call to assess the likely impact. Returns a severity level and a one-line rationale. The judgment is advisory — the user makes the final call.
Conflict severity assessment
After the graph traversal finds connected docs, the tool runs a final LLM pass to judge how hard the proposed change will hit each connection. This is not a binary "will break / won't break" — it's a prioritization signal so the user knows where to look first.

Severity levels:

Level	Meaning	Example
LOW	No conflict expected, or the change is purely additive.	Adding a new "walker profile photo" field to Walker — it doesn't touch existing attributes, and no rules constrain it yet.
MEDIUM	Related docs exist but the change is compatible with current functionality.	Updating Walker.timeslot formatting — job scheduling references timeslots but the scheduling logic doesn't depend on the format, only the values.
HIGH	The change touches constraints, definitions, or validation rules in connected docs that may need updating.	Changing Walker.timeslot from 30-min blocks to 60-min blocks — job scheduling assumes 30-min granularity, and the "timeslots must not overlap" rule needs review.

The judgment prompt receives:
- The pasted BRD text (or a summary)
- The connection path for a surfaced doc
- The relevant section/entity/rule content from that doc
- The edge types on the path (CONSTRAINED_BY edges → weight toward HIGH)

It returns:
{
  "severity": "LOW" | "MEDIUM" | "HIGH",
  "rationale": "Timeslot format change doesn't affect scheduling logic which only reads start/end values, not the block granularity."
}

Severity is displayed as a color-coded badge (green/yellow/red) on each result card, with the rationale in a tooltip. Cards are sorted by severity within each section.
Connection path display
The key differentiator: show WHY each doc surfaced, as a traversal path.

Your BRD modifies: User.email
  ↓ CONSTRAINED_BY
  Rule: "Email required for notification preferences"
  ↓ APPLIES_TO
  Entity: Notification
  ↓ DEFINED_IN
  📄 BRD-042: Notification Preferences
     mobile-app / notifications
This lets the user see the reasoning chain, not just "here's a related doc." They can immediately judge whether the connection is relevant.

V. UI Specification
1. Layout
Single page, three-pane.

Top bar: App name, "Load Repo" button, index status ("247 docs → 89 entities → 312 relationships"), API key input (collapsible), Settings (gear icon), "Re-index" button.
Settings panel (gear icon, collapsible): "Review mode" selector — After extraction (default) / Merges only / Off (see Indexing Pipeline §4). Controls whether indexing pauses for the extraction review gate. Persisted in localStorage.
Left pane (30%): Textarea for pasting BRD. Below: optional path input ("Where will this BRD live?") and "Analyze" button.
Center pane (40%): Context panel — four collapsible sections with result cards.
Right pane (30%): Interactive graph visualization (using D3.js force layout or Sigma.js). Shows the entity neighborhood of the pasted BRD — the touched nodes, their connections, and the connected documents. Clicking a node in the graph highlights its result card in the center pane.
2. Result cards
Each result card shows:

Title — doc title, clickable to expand full content
Severity badge — color-coded: green (LOW), yellow (MEDIUM), red (HIGH). Shows rationale on hover.
Path breadcrumb — mobile-app / authentication
Connection path — the graph traversal that found this doc (e.g. "via User.email → Notification Preferences")
Shared entities — pills/tags showing which entities overlap between this doc and the pasted BRD
Relevant snippet — the section of the doc most relevant to the connection
Platform badge — "Web" / "Mobile" / "Shared"
3. Graph visualization
Color-code by node type: entities (blue), attributes (teal), rules (amber), documents (gray).
Edge thickness by type: DEFINES thick, REFERENCES medium, RELATES_TO thin.
Highlight the "touched" nodes (from the pasted BRD) in a contrasting color.
Hovering a node shows a tooltip with its details. Clicking a document node scrolls the center pane to its result card.
Zoom and pan. Filter toggles for node types.
VI. MCP Server
The tool ships an MCP (Model Context Protocol) server alongside the browser UI. This lets Claude Code, VS Code agents, and other MCP clients query the BRD domain graph directly — the same way GitNexus exposes query/context/impact tools for code, this exposes them for business requirements.

The MCP server is optional — the browser tool works standalone. But if the user starts the MCP server, they can run impact analysis from Claude Code without opening the UI.

MCP Tools
Tool	Parameters	What it does
brd_query	query, limit=5	Search the domain graph for entities, rules, and docs matching a natural-language query. Returns ranked results with connection paths.
brd_context	name, file_path?	360-degree view of a single entity, attribute, or rule. Shows incoming/outgoing edges, which docs define and reference it, and its connected entities.
brd_impact	entity_name	Given an entity or attribute name, compute blast radius: traverse edges up to depth 2 and return all connected docs, grouped by impact strength. Includes conflict severity judgments.
brd_index_status	(no params)	Return current index status: doc count, entity count, relationship count, last indexed timestamp, folder hash.
brd_reindex	(no params)	Trigger a full re-index of the loaded folder. Returns progress updates.

MCP Server Architecture
The MCP server wraps the same graph engine the browser UI uses. It loads the cached graph from IndexedDB (if available) or from a shared JSON file.

For the browser tool: the graph engine runs client-side in the browser.
For the MCP server: the same graph engine runs as a local Node.js process with stdio transport, reading the same graph cache.

This means indexing happens once (through the browser UI), and both the UI and MCP server consume the same graph. The folder hash in the cache key ensures they stay in sync.

Example usage from Claude Code:
> mcp__brd-analyzer__impact entity_name=Walker.timeslot
→ Returns: HIGH severity — 3 docs affected, 2 rules need review

VII. Tech Stack
Dependencies
Layer	Choice	Why
Framework	React or vanilla JS	Three-pane layout benefits from component structure
Markdown parsing	marked or remark	Extract AST, headings, links
LLM extraction	OpenAI API (gpt-4o-mini)	Structured JSON output, cheap at scale
Graph library	graphology (in-memory graph)	Same lib GitNexus uses; fast traversal, clustering
Graph viz	Sigma.js (WebGL) or D3 force layout	Sigma handles larger graphs; D3 simpler for v1
Text fallback	Custom TF-IDF	Lightweight, no dependency needed at this scale
Folder access	File System Access API + ZIP fallback	Chrome-native folder picker
Cache	IndexedDB (for graph JSON)	Larger than localStorage allows; structured data
MCP server	Node.js (TypeScript), stdio transport	Matching GitNexus; thin wrapper around graph engine

VIII. Folder Structure Awareness
Repo-specific config
The tool should understand this folder layout:

OOLE-docs/
  business-requirements/
    01-specifications/
      01-web-app/
        03-jobs/
        04-users/
        05-locks/
        06-minihubs/
        07-transactions/
        08-cpf-management/
        09-settings/
      02-mobile-app/
        01-authentication/
          01-app-init.md
          02.01-sign-in-as-a-walker...md
          03.01-sign-up-as-a-walker...md
Infer platform from 01-web-app / 02-mobile-app.
Infer featureArea from the next folder level.
Strip numeric prefixes for display.
Group files sharing a numeric prefix stem as variants of the same flow.
Only index .md files. Track sibling images for display.
Use folder distance as a graph edge weight modifier: docs in the same feature area get connection scores boosted 1.2×.
VIII. Build Order
Sequence
Build in this order — each step is independently testable:

Step 1 — Folder loading + parsing: Load .md files, parse markdown structure, extract explicit links. Display a file tree and doc count in the UI.
Step 2 — LLM extraction pipeline: Add API key input. Send each doc through the extraction prompt. Display raw extraction results (entity list, relationship list) in a debug panel. This is where you validate extraction quality.
Step 3 — Entity resolution + graph assembly: Merge duplicate entities, build the graph with graphology. Show the full domain graph in the right pane (all entities, all connections). This is the "wow, I can see my whole domain" moment.
Step 4 — Query/analyze flow: Implement the paste → extract → match → traverse pipeline. Show the context panel with connection paths. This is where the tool becomes useful.
Step 5 — Text fallback: Add TF-IDF index, implement the "Weakly Related" section for docs the graph missed.
Step 6 — Conflict severity: Implement the severity judgment pass. Add severity badges to result cards with rationale tooltips. Sort cards by severity within sections. Validate that the LLM judgments are directionally reasonable (HIGH cards should involve CONSTRAINED_BY or DEFINES edges).
Step 7 — Polish: Result card expand/collapse, graph ↔ panel interaction, IndexedDB caching, loading states, progress indicators during indexing.
