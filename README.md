# WisprLink: Product + System Design Spec (build this in stages)

## 0. HOW TO USE THIS DOC
You are a senior full-stack + applied-ML engineer. Build a public, no-auth MVP web app called **WisprLink**. Build in the stages defined in Section 3 (P0, then P1, then P2). Do not start a later stage until the earlier stage's acceptance criteria pass. Prefer a working vertical slice and a clean UI over breadth. Ask at most 3 clarifying questions only if blocked; otherwise make sensible assumptions and record them in the README.

## 1. PRODUCT CONCEPT
The user types or dictates (for example with Wispr Flow, a system-wide dictation tool that types into whatever field is focused) into a text editor. WisprLink enriches that text in real time against a persistent **memory graph** (a "memory palace" of entities, notes, file paths, images, URLs and glossary terms, grouped into clusters), by:
1. **Linking**: hyperlinking spans of text to the exact memory they refer to.
2. **Deixis resolution**: resolving phrases like "that photo" or "the file I selected" against a working-context cache, so the user doesn't re-specify context every session.
3. **Note proposals**: detecting salient, novel terms and proposing new notes attached to a cluster.

### Problem being proved
Users repeatedly re-specify context ("the hero image is at /assets/hero.png") in every new session. The MVP proves that a memory layer can recognize what the user means as they write or speak, and attach the right reference, without altering their words.

### Honest scope statement (show in README and on the demo page)
This MVP is a **workbench for proving the enrichment works**. A web app cannot see what is selected or dictated in other apps (such as VS Code). Context from other apps enters only through manual Working Context bindings and the **Copy Context Pack** export. The full "never re-specify" payoff requires an editor or browser extension, which is a later step.

### NON-NEGOTIABLE INVARIANT
**The original transcript is never mutated.** Enrichment is a standoff annotation layer (character-offset spans plus stable IDs). Add an automated test: sha256(raw text) must equal sha256(text reconstructed from the enriched render with annotations stripped). The UI shows a green "Original text intact ✓ (hash)" badge.

## 2. INPUT MODEL
### MVP mode: post-transcription enrichment
- The primary input is a plain textarea that receives typed text or text from any system-wide dictation tool (such as Wispr Flow). There is no Wispr-specific code.
- Wispr Flow writes the cleaned text only after the utterance ends, so dictated text arrives as one large insert. Enrichment runs automatically as soon as the insert lands. Typing arrives as small inserts. The same `input.delta` pipeline handles both.
- **Optional mode (P1):** Browser mic via the Web Speech API (interim + final results) for true as-you-speak links. It is mutually exclusive with system-wide dictation, with a warning not to run both at once, because both would capture the voice and produce duplicate text.
- **Replay mode:** plays a fixture transcript at a configurable WPM, with seedable simulated ASR noise, for reliable demos and benchmarks.
- **Wispr Flow adapter:** a no-op stub behind a flag, same interface as the other adapters.
- Record observed insert sizes and timings in the Metadata Inspector so thresholds can be tuned from real Wispr Flow behavior later.

### Roadmap note: integration layer (no change to MVP scope)
- The enrichment core (matchers, context scoring, fusion, deixis, auto-notes, persona, metadata schema, strategy registry, experiment harness) must be **input-agnostic**. It consumes only `input.delta` events through an input adapter interface.
- Future: a direct Wispr Flow integration, if one becomes available, would change only the input/output layer, moving from post-transcription to **pre-transcription** enrichment: (a) consume a streaming transcript directly; (b) feed persona terms and aliases back as vocabulary hints so "hero dot png" is transcribed correctly in the first place; (c) attach metadata at the point of transcription. This is contingent on an API that has not been verified. Do not build against it.
- Constraint: namespaces, schema registry, the standoff invariant and the benchmark fixtures must not depend on the input source. The same benchmarks must run unchanged on `typed_stream`, `utterance_batch` and any future adapter.

## 3. SCOPE AND STAGING
### P0: Core proof (build first)
- Live Studio: textarea with overlay highlights, Enriched View with hyperlinked spans, hover "why this link" card, 👍/👎/dismiss, link states (proposed, committed, retracted).
- Seeded demo palace (a VS Code project: `hero.png`, `auth/middleware.ts`, an API docs URL, a teammate, glossary terms, 3 clusters).
- Strategies S0 `baseline_exact`, S1 `fuzzy`, S2 `phonetic`, S4 `semantic_embed`, S5 `hybrid_rrf`.
- Deixis resolution via Working Context bindings (manual bind, editable, persisted across sessions in the sandbox) plus a recency fallback.
- Auto-note proposals with Accept / Edit / Dismiss.
- Invariant badge and test.
- Copy Context Pack export.
- Metadata Inspector (live nested JSON, schema-validated).
- Memory Palace CRUD with import (JSON/CSV/Markdown paste) and a description field on every memory.
- Feedback capture (👍/👎/dismiss, "never link this phrase to this memory").

### P1: Measurement story
- Arena (A/B side by side on the same input, with a link diff).
- Benchmark runner with fixtures, a leaderboard, precision/recall/F1, MRR, latency percentiles, a calibration plot, per-fixture drill-down, and JSON/CSV export.
- S3 `lexical_bm25` and S6 `hybrid_graph` (cluster activation, spreading activation, recency prior).
- Memory graph visualization and cluster activation.
- Browser mic mode.
- Memory maintenance: merge duplicates, pin, TTL, last-used signal.

### P2: Defer unless time remains
- S7 LLM rerank (behind a flag, with cost and latency tracking).
- Live A/B sticky bucketing, shadow mode with multiple concurrent strategies, Pareto plots, McNemar/paired-bootstrap significance, CPU/memory sampling, shareable read-only replay links, calibration refitting from live feedback.

**Gate rule:** the first thing to validate is whether semantic links are accurate enough to trust. Do not build P1 or P2 analytics on top of an unvalidated P0.

## 4. SEMANTIC LINKING REQUIREMENTS
This is the core of the product and drives most of the scope.
1. **Memories need embeddable content.** A bare path like `hero.png` has almost no semantic content. Every memory has: `canonical`, `aliases[]`, a short `description`, and optional `how_i_refer_to_it[]` phrases (for example "landing page banner", "the login thing"). The UI nudges the user to fill these in, and the import flow accepts them.
2. **Two kinds of "semantic", reported separately in benchmarks:**
   - *Surface-form matching* (ASR errors, homophones, typos such as "hero dot png"): handled by exact, fuzzy and phonetic matchers.
   - *Conceptual matching* ("the login thing" → `auth/middleware.ts`): handled by embeddings plus context scoring.
3. **Short spans embed poorly.** Embed the candidate span **together with its sentence context** (and a small left/right window), not in isolation. Enforce a minimum-evidence rule: no link on semantic similarity alone below a configured threshold unless a second signal agrees (cluster activation, alias, phonetic).
4. **Right-context disambiguation:** for large inserts (utterances), score candidates using tokens both before and after the mention within the same insert.
5. **Ambiguity policy:** when multiple memories match, show the top candidate with a visible reason and a **one-click swap** to alternatives. If top scores are within a margin, stay silent or mark "ambiguous" instead of guessing.
6. **Precision over recall.** A wrong link costs more trust than a missed one. Defaults are conservative. Dismiss is one click. Every link must show why it fired (matcher scores, cluster context, strategy and version).
7. **Embedding backend:** a small sentence-embedding model if feasible. Fall back to char-n-gram TF-IDF vectors if unavailable. Record the backend in `provenance` and show a degradation badge in the UI.

## 5. ENRICHMENT PIPELINE
1. **Normalize + tokenize:** case-fold, strip punctuation, preserve offsets; handle code-like tokens, paths, camelCase and snake_case.
2. **Candidate spans:** n-grams (1 to 5) over a sliding window of the last W tokens (default 64), or over the whole insert for utterances.
3. **Matchers** (each returns a score in [0,1] plus evidence): Aho-Corasick/trie exact and alias; fuzzy (normalized Levenshtein, Jaro-Winkler); phonetic (Double Metaphone, Soundex); BM25; semantic (cosine over embeddings).
4. **Highlight/keyword detection:** per-token surprisal vs. the persona unigram model with smoothing (`-log p(w | persona)`); YAKE/RAKE-style keyness; salience tracked by EWMA; threshold crossing with hysteresis.
5. **Context scoring:** cluster activation as a leaky integrator (configurable half-life) with spreading activation over graph edges (damping factor), plus recency and frequency priors. Used to disambiguate.
6. **Fusion and thresholding:** Reciprocal Rank Fusion and/or a weighted linear blend (configurable weights). Emit a link when score ≥ θ_on; retract when it falls below θ_off (hysteresis).
7. **Calibration:** Platt/isotonic fitted from feedback and benchmark labels; show raw vs. calibrated confidence.
8. **Deixis resolution:** detect deictic phrases ("that photo/file/function/one", "it", "the thing I selected"). Resolve against (a) Working Context bindings, then (b) a recency/salience stack filtered by the type implied by the head noun. Return top-k with scores. Unresolved phrases show "needs binding" with a one-click bind.
9. **Auto-note proposals:** a span that is high-salience, novel (not linked) and repeated ≥ N times becomes a `memory_candidate` with a suggested type and cluster. Accept creates a memory and attaches it to the cluster.
10. **Persona update:** on committed text, update term frequencies, IDF, co-occurrence edges and cluster centroids with time decay.
11. **Tiered progressive enhancement:** Tier 0 (sync, <50ms p95): exact/alias. Tier 1 (<150ms): fuzzy + phonetic. Tier 2 (async, <500ms): BM25, embeddings, graph re-ranking. Upgrades keep the **same span ID** so the UI never jumps.

### Streaming protocol and commit policy
- Client to server: `input.delta {session_id, seq, base_rev, ops:[{op:"insert"|"delete", pos, text}], source:"typed"|"dictation"|"asr_interim"|"asr_final"}`.
- Server keeps canonical text, a `rev` counter and offset-aware tokenization. Spans re-anchor across edits via diff mapping with stable span IDs.
- Server to client: `enrichment.patch {rev, ops:[add|update|remove keyed by span_id], metadata}`. Discard results computed for stale revs.
- Commit only on stable text. Interim results are "proposed" only. Use hysteresis plus a minimum dwell time to prevent flicker, and track churn.

## 6. MEMORY MODEL (MongoDB)
- `memories`: id, type (file|image|url|snippet|person|term|note|binding), canonical, aliases[], description, how_i_refer_to_it[], pointer (path/uri/text), tags[], cluster_id, phonetic_keys[], embedding, suppressed_phrases[], created_at, last_used_at, use_count, pinned, ttl.
- `clusters`: id, label, centroid, top_terms[], created_at ("rooms" in the palace).
- `edges`: src, dst, weight, kind (co-occurrence|explicit|session), last_updated.
- `sessions`, `events` (every patch, for replay), `feedback`, `experiments`, `benchmark_runs`.
- **Cold start:** seeded palace, plus paste-a-glossary, Markdown/JSON/CSV import, and promote-from-note-proposal. (Future: seed from the user's existing notes and meetings via a connector.)
- **Staleness:** merge duplicates, pin, TTL, a "last used" signal, and a stale-memory indicator.

## 7. STRATEGIES (plugin registry; zero changes elsewhere to add one)
Each strategy implements `process(delta, state) -> patch`, registered in `STRATEGY_REGISTRY` with id, version and JSON-configurable default params (editable in a UI params editor).
- S0 `baseline_exact`, S1 `fuzzy`, S2 `phonetic`, S3 `lexical_bm25`, S4 `semantic_embed`, S5 `hybrid_rrf`, S6 `hybrid_graph`, S7 `hybrid_graph_llm_rerank` (P2, behind a flag).

## 8. EXPERIMENTATION AND BENCHMARKING (P1, with P2 extras)
**Modes:** offline deterministic benchmark (seeded RNG for ASR noise: homophone swaps, dropped words, interim-revision storms); Arena (two strategies side by side on the same input); P2: shadow mode and live sticky-bucket A/B (hash of session_id and experiment_id).

**Fixtures:** a synthetic generator plus ≥ 5 hand-built fixtures (coding session, meeting notes, research notes, noisy ASR, deixis-heavy), each with gold labels (span → memory_id, deictic → memory_id, and "should-not-link" spans). Two fixture modes: `typed_stream` and `utterance_batch` (whole sentences inserted at once). Fixtures and the evaluation code must not depend on the input source.

**Metrics** (per strategy, with 95% bootstrap CIs; P2 adds paired significance):
- Span precision/recall/F1 (exact and overlap), link accuracy (correct memory_id), false positives per 100 tokens.
- Ranking: MRR, nDCG@k, recall@k.
- Deixis accuracy (top-1, top-3).
- **Report surface-form and conceptual matches separately.**
- Time-to-link; for utterances, insert-arrival-to-full-enrichment latency (p50/p95) and time-to-first-link vs. time-to-final-link.
- Stability: churn/flicker rate, robustness under ASR noise.
- Calibration: ECE and reliability diagram.
- Latency p50/p95/p99 per tier and end-to-end.
- Online: 👍, 👎, dismissal and bind-correction rates.
- P2: cost (LLM tokens/$), CPU/memory.

**Reproducibility:** benchmarks are deterministic. Pin strategy and schema versions in every run record, so results are comparable over time.
**Outputs:** sortable leaderboard, per-fixture drill-down, latency histograms, JSON/CSV export, a "Run all strategies on all fixtures" button. P2: Pareto plot (quality vs. latency).

**Ground truth beyond synthetic fixtures:** synthetic fixtures overfit easily. Provide an opt-in "contribute my feedback as benchmark labels" toggle so 👍/👎 and dismissals can become labeled data.

## 9. METADATA SCHEMA (the core contract)
Every enrichment event carries an envelope plus `metadata = { <namespace>: { <feature_id>: <value> } }`.
- Top-level keys are **namespaces** (feature categories).
- Keys inside a namespace are **feature IDs**, stable and unique within the namespace, prefixed per namespace.
- Each value validates against that namespace's registered Pydantic/JSON Schema. Schemas are versioned in the registry (not inside the data), exposed at `GET /api/schemas`, and every emitted patch is validated.

Namespaces:
- `tokens` (`tok_<rev>_<i>`): {text, start, end, stable, surprisal}
- `keywords` (`kw_*`): {span, text, score, method, novelty, state}
- `links` (`lnk_*`): {span, text, memory_id, cluster_id, confidence, calibrated_confidence, matcher_scores:{exact,fuzzy,phonetic,bm25,semantic,graph}, match_kind: surface|conceptual, state: proposed|committed|dismissed|retracted, first_seen_rev, committed_rev, alternatives:[{memory_id,score}]}
- `deixis` (`dx_*`): {span, phrase, resolved_memory_id, candidates:[{memory_id,score}], binding_source: working_context|recency|none}
- `memory_candidates` (`mc_*`): {span, text, reason, suggested_type, suggested_cluster_id, occurrences}
- `clusters` (`cl_*`): {label, activation, member_count, top_terms[]}
- `persona` (`ps_*`): {term, tf, idf, half_life_s, last_seen}
- `signal` (`sig_*`): {window_tokens, ewma_salience, novelty_rate, link_density, churn}
- `strategy` (`st_*`): {strategy_id, version, params, role: primary|shadow, links_emitted}
- `latency` (`lat_<stage>`): {ms, tier}
- `experiment` (`ex_*`): {experiment_id, arm, bucket_hash}
- `provenance` (`pv_*`): {rev, text_sha256, source, input_adapter, insert_chars, insert_gap_ms, embedding_backend}

Example:
```json
{
  "event_id": "evt_0192", "session_id": "s_ab12", "seq": 57, "rev": 31,
  "metadata": {
    "links": {
      "lnk_0007": {"span":[24,33],"text":"hero.png","memory_id":"mem_hero","cluster_id":"cl_ui",
        "confidence":0.91,"calibrated_confidence":0.88,
        "matcher_scores":{"exact":1.0,"phonetic":0.0,"semantic":0.42,"graph":0.7},
        "match_kind":"surface","state":"committed","first_seen_rev":29,"committed_rev":31,
        "alternatives":[]}
    },
    "deixis": {
      "dx_0002": {"span":[40,50],"phrase":"that photo","resolved_memory_id":"mem_hero",
        "candidates":[{"memory_id":"mem_hero","score":0.93}],"binding_source":"working_context"}
    },
    "strategy": {"st_s5": {"strategy_id":"hybrid_rrf","version":"1.0.0","params":{"theta_on":0.7,"theta_off":0.5},"role":"primary","links_emitted":4}},
    "latency": {"lat_tier0": {"ms":6,"tier":0}, "lat_tier2": {"ms":212,"tier":2}},
    "provenance": {"pv_01": {"rev":31,"text_sha256":"9f2c...","source":"dictation","input_adapter":"textarea","insert_chars":78,"insert_gap_ms":1450,"embedding_backend":"ngram_tfidf"}}
  }
}
```

## 10. UX SURFACES (single-page app, tabs)
1. **Live Studio (P0):** left: plain textarea with a mirrored highlight overlay. Right: read-only Enriched View with hyperlinked spans. Hover shows a "why this link" card (type, pointer, cluster, confidence, matcher scores, strategy and version). Click opens the memory. 👍/👎/dismiss on each link. Swap-to-alternative control on ambiguous links. Side panel: Notes Feed (Accept/Edit/Dismiss), Working Context (editable pinned bindings, e.g. "that photo → /assets/hero.png"), and (P1) a live Memory Graph. "Copy Context Pack" exports the original text plus an appended expansion of every link and deixis into full memory content, for pasting into an LLM or IDE. The original text is untouched.
2. **Memory Palace (P0 CRUD, P1 graph):** CRUD for memories and clusters, import, description fields, staleness indicators, persona stats.
3. **Experiments / Arena (P1).**
4. **Benchmarks (P1).**
5. **Metadata Inspector (P0):** live nested-JSON viewer, collapsible by namespace, schema-validated, copy/download, showing insert size and timing per event.

## 11. ARCHITECTURE
- Frontend: React + Tailwind + a graph library (react-force-graph or d3) + Recharts.
- Backend: FastAPI (Python), MongoDB, WebSocket for streaming, REST for CRUD, experiments and benchmarks.
- API (minimum): WS `/ws/session/{id}`; REST `/api/memories`, `/api/clusters`, `/api/bindings`, `/api/strategies`, `/api/experiments`, `/api/benchmarks/run` and `/api/benchmarks/{id}`, `/api/feedback`, `/api/schemas`, `/api/export/context-pack`, `/api/sessions/{id}/replay`.

## 12. PUBLIC MVP CONSTRAINTS
- **No auth.** Anonymous sessions with a random session ID. Each session gets an isolated sandbox copy of the seeded palace, a "Reset sandbox" button, and a 7-day data TTL.
- **Privacy:** a visible banner ("Public demo: don't paste secrets or personal data"). Embeddings default to a local model or the n-gram fallback; if a hosted embedding service is used, disclose it plainly in the UI and README. Never log raw text beyond the sandbox TTL.
- Rate limiting and input size caps.
- Graceful degradation: if embeddings or an LLM are unavailable, fall back and show a badge.
- Clean dark/light UI, responsive, no console errors.

## 13. ACCEPTANCE CRITERIA
**P0**
1. Typing or dictating into the textarea produces live links; Tier-0 latency < 50ms p95 on the seed data. A dictated sentence that lands as one insert is fully enriched automatically.
2. The invariant test passes (green badge) across all fixtures.
3. "I selected that photo" resolves via a Working Context binding, and the binding persists in a new session within the sandbox.
4. A repeated novel term produces a note proposal; Accept adds it to a cluster.
5. A conceptual phrase ("the login thing") links to the right memory when the memory has a description, and does not link below the evidence threshold.
6. Ambiguous matches show a top candidate plus a one-click swap. Every link shows why it fired.
7. Every patch validates against the namespace schemas; the Inspector shows the nested JSON live.
8. Dismissing a link with "never link this phrase to this memory" suppresses it in future sessions.

**P1**
9. The Arena shows two strategies side by side on the same input with a link diff.
10. The Benchmark page runs all strategies on all fixtures and shows a leaderboard with CIs, latency histograms and a calibration plot, with surface-form and conceptual results reported separately.
11. Benchmarks are reproducible: the same seed and pinned versions produce identical results.

## 14. BUILD ORDER
1. Data model, seed palace, schema registry. 2. WebSocket streaming, revision/anchoring, invariant test. 3. S0 to S2 plus Live Studio UI. 4. S4/S5, deixis, auto-notes, feedback and suppression, Memory Palace CRUD and import, Context Pack, Inspector (**P0 complete; validate semantic accuracy before continuing**). 5. S3/S6, context scoring, graph view, Arena. 6. Fixtures, metrics, benchmark runner and charts, mic mode (**P1 complete**). 7. P2 items as time allows. 8. README documenting every strategy, metric, schema, assumption, the scope statement from Section 1, and the privacy approach.
