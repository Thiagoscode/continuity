> Published in this public mirror as dated prior-art evidence for the mechanisms it describes,
> ahead of the v2.1 revision of the defensive technical disclosure (v2.0, May 2026).
> Code citations reference the private development repository `continuity-ultimate`.

# Whitepaper v2.1 rewrite — code-verified ground truth

**Verified:** 2026-08-12, against `main` (post-`ce208a98`)
**Target:** "Continuity: Persistent Decision-Centric AI Cognition with Runtime …" — Defensive Technical Disclosure v2.0 (May 2026)
**Method:** every claim in the May-2026 audit re-checked against current source with `file:line` citations. Verdicts: ✅ audit confirmed · ✳️ confirmed with refinement · ❌ audit itself wrong.

Use this file as the single source of truth when rewriting. Where the paper and this file disagree, the paper is wrong.

---

## Critical (blocks republication)

### 1. Method D — Markov chain ✅ CONFIRMED: dedup thresholds, not retrieval

- `packages/mcp-server/src/tools/decision/handlers.ts:449-460` — `markovChain.checkNewDecision(tags)` feeds `adjustedThresholds`; consumed as `BLOCK_THRESHOLD = markovResult?.adjustedThresholds.blockThreshold ?? 0.85` and `WARNING_THRESHOLD = … ?? 0.70` inside **log_decision deduplication**.
- `handlers.ts:718-721` — `markovChain.setLastState(decision.tags)` after a successful log (model update).
- **Zero** Markov references in `middleware/AutoRetrievalMiddleware.ts`, any search handler, or semantic-search path (grepped `packages/mcp-server/src` recursively).

**Rewrite VIII.D as:** tag-sequence Markov model that *adapts pre-log deduplication thresholds* (base block 0.85 / warn 0.70) when a tag pattern is predicted; state updated post-log. Delete every "retrieval ranking bias" sentence, including in the Abstract.

### 2. Section VIII.B — relationship types & weights ✳️ CONFIRMED, plus a math nuance the audit missed

Type union (`src/services/RelationshipDetector.ts:35`):
`'supersedes' | 'superseded_by' | 'related_to' | 'causes' | 'caused_by'` — **3 detector families, 5 directional labels. No `contradicts`, no `depends_on`.** The "contradicts" vocabulary at `RelationshipDetector.ts:88` is keyword helper text, not an inferred relation.

Verified weights:

| Relation | Actual weights (code) | Cite |
|---|---|---|
| supersedes | keyword **0.40** · semantic **0.30** · tag **0.20** · temporal **0.10** | `:267-276` |
| causes / caused_by | keyword **0.50** · mention **0.20** · semantic **0.20** · temporal **0.10** | `:346-349` |
| related_to | semantic **0.30** · file **0.25** · entity **0.20** · tag **0.15** · keyword **0.10** | `:445-449` (paper already matches) |

**Nuance the paper must not omit:** weights are **renormalised over applicable dimensions** via `blend()` (`:263-266`) — tag applies only when *both* decisions have tags, semantic only when embeddings actually ran (`this.semanticRan`). They are not fixed coefficients; with embeddings unavailable, supersedes effectively becomes keyword 0.57 / tag 0.29 / temporal 0.14. Presenting the table as a static linear combination is itself an overstatement.

Direction resolution: temporal ordering flips `causes`↔`caused_by` (`:355-356`, `:732-734`).

### 3. Section V.B — TOP_K clamp ✅ CONFIRMED: middleware-only

- `packages/mcp-server/src/middleware/AutoRetrievalMiddleware.ts:33,35` — `DEFAULT_MAX_DECISIONS = 3`, `TOP_K_CEILING = 10`; clamp at `:83` (`Math.max(1, Math.min(TOP_K_CEILING, …))`) applies to the config-read path of **in-loop auto-retrieval only**.
- `packages/mcp-server/src/tools/decision/handlers.ts:1137` — `search_decisions`: `const limit = args.limit || 10` — **no ceiling**; a caller passing `limit: 500` gets 500.
- `handlers.ts:1043,1085` — `list_decisions` pagination default **50**.
- `get_quick_context` truncates instead of clamping: question → 100 chars, answer → 150 chars (`tools/context/handlers.ts:633-634,649-650`; also `:179,345`).

**Rewrite:** scope the invariant to the auto-retrieval middleware, or ship a global clamp first and then claim it. The per-path constants table below belongs in the paper.

### 4. Section V.C — 17,500-token closed form ✅ CONFIRMED: not a global invariant

- MCP write path: `MAX_TEXT_LENGTH = 2000`, `MAX_ANSWER_LENGTH = 5000` (`packages/mcp-server/src/utils/logger.ts:80,86`).
- Extension write path: question **500** + answer 5000, silent truncation (`src/services/DecisionCRUD.ts:520-525`) and hard validation errors (`:816,823`).
- Retrieval surfaces return 100/150-char previews (above), far below the full-field figures the formula multiplies.

**Rewrite:** label as "theoretical upper bound for the full-field MCP injection path," and add the per-surface table. The bound is honest only when its surface is named.

### 5. Sections IV, V.F — persistent store ✅ CONFIRMED: journal is the source of truth

`docs/ARCHITECTURE.md:51-56,97-98,118-119`: `decisions.jsonl` = append-only NDJSON journal, source of truth (collapse-by-id, newest-rev-wins, tombstones, `merge=union`); `decisions.json` = derived cache, regenerated *from* the journal, never authoritative. All architecture diagrams naming `decisions.json` as the store must be redrawn.

---

## High

### 6. Section VI.A — "OS never receives the syscall" ✅ overstatement confirmed

Interception is at the MCP handler-dispatch layer (`SecurityCheckPipeline.interceptToolCall` via ToolCallDispatcher). **Correct wording:** "the tool invocation does not proceed to handler execution." Keep XII.B's seccomp disclaimer; VI.A must stop contradicting it.

### 7. Section VI.C — OWASP mapping ✅ conceptual only

Zero `LLM0x`/`OWASP` identifiers anywhere in `packages/mcp-server/src`. Present VI.C as a *conceptual coverage mapping by the authors*, or add a traceability matrix to the repo first.

### 8. Section VI.E — warn vs strict ✳️ confirmed, with counts

- Default is `'warn'` (`packages/mcp-server/src/index.ts:2145,2154` — `config.governance.enforcementMode ?? 'warn'`).
- **65** `allowed: false` sites in `SecurityCheckPipeline.ts` return blocks regardless of mode.
- The mode-sensitive branch is the SENTINEL semantic check (Step 4), unreachable unless `governance.semanticCheck.enabled === true` (`SecurityCheckPipeline.ts:1237-1256`).

**Rewrite:** "warn mode governs the opt-in semantic-violation check; pattern-based checks block in both modes."

### 9. Section VII.D — scrubber benchmark ✅ caveats confirmed

CI gate is **≥ 90% recall**, not 100% (`packages/mcp-server/__tests__/core-security/benchmark/Scrubber.benchmark.test.ts:23-26`). The 53/53 run is a point measurement on a synthetic, shape-accurate corpus (decision `a594fe82`, commit `2db9f82b`). Footnote exactly that: *synthetic corpus, high-confidence patterns, commit-pinned, CI floor 90%*. Also note `DecisionCRUD` carries extra legacy regexes (SSN, credit card, private IP) beyond the shared 27-pattern catalog — the "five-boundary shared primitive" story needs that asterisk.

### 10. Section XI — embodiment paths, corrected table

| Paper | Correct (2026-08-12) |
|---|---|
| `src/services/governance/GovernanceLock.ts` | `src/services/GovernanceLock.ts` (no `governance/` subdir) |
| `src/services/governance/ToolInterceptor.ts` | **removed** — consolidated into `packages/mcp-server/src/services/SecurityCheckPipeline.ts` |
| `src/services/MarkovDecisionChain.ts` | `packages/mcp-server/src/services/MarkovDecisionChain.ts` |
| bounded retrieval "via DecisionLogger.ts" | extension facade → `src/services/DecisionSearchEngine.ts`; MCP paths: `AutoRetrievalMiddleware`, `search_decisions`, `get_quick_context` |
| `ToolResultScrubber.ts` | ✅ `packages/mcp-server/src/security/ToolResultScrubber.ts` |
| `InputScrubber.ts` | ✅ `packages/mcp-server/src/middleware/InputScrubber.ts` |
| `CredentialStore.ts` | ✅ `packages/core/src/services/CredentialStore.ts` |

---

## Medium

### 11. Corpus statistics — and a unit the audit itself conflated ❌→✳️

The audit's "~4,174 decisions" is the **journal line count** (revisions, incl. tombstones). Measured 2026-08-12: 4,174 journal lines → **3,983 unique collapsed decisions**. If refreshing the paper's n=2,200, use the collapsed figure and define the unit ("unique decisions after journal collapse, as of YYYY-MM-DD"). The static-embedding token estimate scales with n either way.

### 12. Publication date — pick one
Header says May 18, metadata May 17, identifier `…-2026-05-17`. Align all three (identifier suggests the 17th).

### 13. Repository naming
`Thiagoscode/continuity` is the public mirror; dev repo is `continuity-ultimate`(-dev). One footnote: "public defensive-publication mirror of a private development repository."

### 14. Pattern catalog ✅ exactly 27

`packages/core/src/security/secret-patterns.ts` — **27** `name:` entries in `PATTERNS` (recounted by parsing, 2026-08-12). Presentation fixes:
- AWS is **one** pattern covering **nine** prefixes: `(?:AKIA|ASIA|AGPA|AIDA|AROA|AIPA|ANPA|ANVA|ASCA)[0-9A-Z]{16}` (`:136`) — the paper's separate "AWS temporary (ASIA*)" row must merge, and the audit's "AKIA|ASIA|…" undersold it too.
- Atlassian is `ATATT3[A-Za-z0-9_=\-]{180,250}` (`:45`), not a literal-prefix example.
- Entropy fallback: minLength 32 · maxLength 512 · threshold 4.5 (`ENTROPY_DEFAULTS`, `:277-283`) — paper matches. ✅

### 15. DEFAULT_MAX_DECISIONS = 3
Describe as "default for in-loop auto-retrieval (bash/edit/write interception)"; `search_decisions` defaults to 10, `list_decisions` to 50.

### 16–17. External claims
ClawdBot / CVE-2026-25253 and `arxiv:2604.16548` were **not** verified in this pass. Footnote primary sources or soften before republication; verify the arXiv ID exists.

---

## Per-path constants table (drop into the paper, replaces V.B's single-column claims)

| Constant | Value | Location | Scope |
|---|---|---|---|
| `DEFAULT_MAX_DECISIONS` | 3 | `AutoRetrievalMiddleware.ts:33` | in-loop injection only |
| `TOP_K_CEILING` (clamped) | 10 | `AutoRetrievalMiddleware.ts:35,83` | in-loop injection only |
| `search_decisions` limit | `args.limit \|\| 10`, **unclamped** | `tools/decision/handlers.ts:1137` | MCP search |
| `list_decisions` limit | 50 default | `tools/decision/handlers.ts:1043` | MCP pagination |
| quick-context previews | Q 100 / A 150 chars | `tools/context/handlers.ts:633-650` | `get_quick_context` |
| `MAX_TEXT_LENGTH` | 2,000 | `packages/mcp-server/src/utils/logger.ts:80` | MCP writes |
| `MAX_ANSWER_LENGTH` | 5,000 | `logger.ts:86` + `DecisionCRUD.ts:816-823` | MCP + extension |
| Question cap (extension) | 500 | `DecisionCRUD.ts:520-525,816` | extension UI path |
| Dedup thresholds (base) | block 0.85 / warn 0.70, Markov-adjusted | `tools/decision/handlers.ts:459-460` | log_decision |
| Secret patterns | 27 | `packages/core/src/security/secret-patterns.ts` | shared scrubber |
| Entropy fallback | 32–512 chars, threshold 4.5 | same, `:277-283` | medium-confidence |
| `enforcementMode` default | `'warn'` | `index.ts:2145,2154` | governance |

## What holds up (keep, unchanged)

Prior-art acknowledgments (II.D, VI.B, VII.D, XII.B) · five scrub boundaries · `_meta.credential_warnings` (`InputScrubber.ts`) · scrub idempotence + `<REDACTED:pattern-name>` · entropy defaults · `audit_secrets` tool · Memory-Amplifier threat framing with dual read/write scrub on the `DecisionCRUD` load path.

## v2.1 edit order

1. Rewrite VIII.D (Markov → dedup) — including the Abstract sentence.
2. Fix VIII.B tables + add the `blend()` renormalisation caveat; drop `contradicts`/`depends_on` or mark *planned*.
3. Insert the per-path constants table; scope V.B/V.C claims per surface.
4. Redraw architecture diagrams: `decisions.jsonl` journal + `decisions.json` cache.
5. Correct Section XI paths (table above).
6. Soften VI.A ("does not proceed to handler execution") and VI.E (65 hard blocks; warn governs SENTINEL only).
7. Footnote VII.D benchmark scope; note DecisionCRUD legacy regexes.
8. Reconcile XII.A's "four-signal" wording with VIII.B's actual five-signal related_to / four-signal supersedes-causes reality.
9. Refresh corpus stats with the collapsed-unique figure + as-of date.
10. Align the publication date; footnote the repo mirror; verify the two external citations.
