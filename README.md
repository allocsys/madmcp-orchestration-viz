# madmcp orchestration viz

A single-file Three.js visualization (`index.html`) of how the `madmcp` agent orchestration server routes traffic: agent → core → connectors, plus the delegation layer's sub-agents.

## Graph structure

**Anchors**
- `agent` — the calling agent, top of the graph.
- `madmcp` core — the orchestration hub everything routes through.
- Security gate — a static double-ring boundary around core representing the server's IP allowlist, shared-key auth, and proxy-hop trust checks (`connectors/security.js`). Not tied to per-request pulses; it's a standing boundary.

**Direct-call services** (single hop off core, orbit core independently)
- `github`, `notion`, `cloudflare`, `mem0`, `context7` — one connector each, direct request/response.
- `cloudflare-observability` — split out from `cloudflare` as its own node with a visibly thinner, dimmer connector, since it's read-only telemetry querying (`cf_workers_observability_query`/`keys`/`values`/`compare`) rather than control-plane management (Workers/D1/R2/Hyperdrive).

**Cross-service sync**
- A dedicated rose-colored edge runs directly between `mem0` and `notion` (bypassing core), representing `sync_mem0_to_notion` as a real cross-service orchestration edge rather than two independent direct calls.

**Delegation layer** (core hands off the whole task; each sub-agent has distinct geometry)
- `delegate_gemini` node (labeled `delegate_gemini` in the viz; the underlying server tool is `delegate_agent` — see Known gaps) — octahedron core, two intersecting rings. Autonomous, open-ended fan-out: dispatched once, then independently reaches every repo and every service in parallel and returns one consolidated result.
- `delegate-designer` — dodecahedron core, single violet ring. Represents a **bounded loop**, not an open fan-out: generate → validate → fix → regenerate, cycling through its own `design-generate` / `design-validate` / `design-fix` phase nodes until done.
- `delegate-research` — tetrahedron core, single sky ring. Represents `delegate_research`'s two mutually exclusive, single-shot modes, each its own child node:
  - `research-precision` (sky) — fetch a URL + question, one Gemini call, returns a compact answer.
  - `research-wide` (sage) — one Exa `/answer` call (search + synthesis combined), web-only.

**Repo leaves**
- `api-service`, `worker-queue`, `shared-types` — example fan-out targets under `delegate_gemini`.

## Animation / edge legend
- **Direct call** (sage) — core → service, one hop.
- **Telemetry** (sky, thinner/dimmer) — core → cf-observability, read-only.
- **Delegation** (amber) — core → sub-agent handoff.
- **Bounded loop** (violet) — delegate_designer's internal generate/validate/fix cycle.
- **Cross-service sync** (rose) — mem0 → notion direct edge.

All service/delegate/repo nodes orbit their parent (core, or their owning delegate) on independently-tuned radii and speeds; orbit rings trace each path so the motion reads as deliberate rather than random drift.

## Known gaps (not yet represented)
- **`fetch` connector** — a real madmcp capability (`connectors/fetch/`) with no node or edge in the viz. It's also used internally by `delegate_research`'s precision mode (fetch before handing content to Gemini) but isn't shown there either.
- **Naming mismatch** — the viz's generic delegate node is id'd/labeled `delegate_gemini`; the actual server tool is registered as `delegate_agent` (`connectors/gemini/tools.js`). A future pass should rename the node/labels to match what developers actually see.

## Source of truth
Capabilities are driven by `allocsys/madmcp`'s `connectors/` directory. When the server's connector list changes, re-diff it against this graph before assuming the viz is current.
