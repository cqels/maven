# CQELS 2.0.0-alpha.20

A **correctness-and-truthfulness release.** Four engine fixes close public downstream reports
(CQELS4J#54, CQELS4J#55, CQELS4J#56, CQELS4J#57) — static patterns that behaved as `OPTIONAL`, prefixed names that
silently resolved to `null` inside expressions, syntax errors that registered anyway, and reasoning
that never fired for atomically pushed observations — and `sameTerm()` lands as a real
implementation rather than a silent `null` (CQELS4J#65). The multi-stream interval join takes a
capability step: per-subscription state, bounded buffers, multi-triple side patterns and a natural
join over *every* shared variable. And the largest documentation-accuracy campaign this project has
run corrected 89 wrong row-cardinality claims out of 158 enumerated, replaced 69 line-number
citations with method names, and — more durably than the corrections themselves — added gates so
the structural classes cannot rot again. Several merged changes alter results for existing queries;
see **Upgrade notes**.

### Added
- **Multi-triple side patterns and a natural-join key on the multi-stream interval join**, the
  multi-stream epic's stage S4. A `STREAM` block may now declare several triple patterns; `SideBgpMatcher`
  conjoins statements co-resident in that side's own declared window as an ordinary SPARQL BGP and
  emits each newly completed solution once, as a `BgpSolutionElement` timestamped at its newest
  constituent, with co-residency tested on the solution's `[oldest, newest]` span so a conjunction
  cannot reach further than its constituents would singly. `deriveTemporalJoinSpecFromMultiSource`
  now keys on **every** shared variable (SPARQL natural join) instead of rejecting a query with two
  or more; a pair disagreeing on a shared variable emits no row rather than throwing mid-stream.
  **Scope, stated as the PR states it:** the route keeps its *interval-join* semantics — this only
  widens what it accepts. Zero shared variables is still rejected (a cross product is a separate
  feature), three or more sources is still rejected, and on the explicit `operators.temporalJoin`
  JSON-hint path a multi-triple side now **requires** `leftPatternWindow`/`rightPatternWindow` —
  that path has no declared stream window to fall back on, unlike the auto-derived route, which
  takes each side's pattern window from its own `FROM STREAM` declaration. A multi-triple
  side on a zero-extent `[NOW]` window warns at compile time rather than being rejected — that shape
  is the documented same-timestamp decision-cycle contract, now pinned by a hot `[NOW] ⋈ [NOW]`
  regression across all three interleavings. Side-BGP fan-out is bounded by the existing
  `cqels.join.crossProductCap` in two directions — at most `cap` completed solutions and at most
  `cap × patterns` surviving partial bindings — so an unsatisfiable tail that completes nothing
  still trips the guard.
- **Watermark-driven CEP timeout evaluation for quiet streams**. A trailing
  negation (`SYN notFollowedBy ACK within 5s`) previously stayed silent forever on a stream that
  went quiet, because `NFAPatternProcessor.processWithTimeouts` only expired partials when a *later
  data event* advanced event time — continuous missing-event detectors had to inject fake data to
  surface their own deadlines. New public API: `NFAPatternProcessor.advanceWatermark(long)` expires
  every partial whose deadline precedes the watermark and emits the corresponding timeout output —
  exactly what a data event at that timestamp would have done, minus every data-event side effect —
  and `CQELSEngine.advanceCepWatermark(cepQueryId, eventTime)` feeds the compiled directional path.
  No wall clock and no scheduler anywhere (maintainer decision, option C). Data events and
  watermarks share **one serialized channel per subscription** so their arrival order is the
  caller's order; a two-source merge would let a watermark overtake a data event pushed before it
  and expire a partial that event had already killed. The legacy raw-stream CEP arm has no watermark
  input and rejects `advanceCepWatermark` loudly. Non-advancing watermarks are coalesced per
  subscription (documented idempotence), and the frontier advances only *after* a successful
  delivery, so a watermark whose delivery failed stays retryable.
- **`sameTerm(A, B)`** joins the expression function dispatch, matching CQELS4J#65,
  delegating to RDF4J `Value#equals` — verified against the pinned 4.3.8 jar to be an exact
  structural match for SPARQL §17.4.1.1 term identity. Reachable from `FILTER`, `BIND`, both
  `HAVING` paths, `OPTIONAL` inner filters, and the in-batch and `Map`-row routes; unknown arity or
  an unbound argument yields a type error, consistent with the neighbouring functions. The
  `NaN`-vs-`=` divergence holds by construction and is pinned. **Limitation pinned rather than
  hidden:** row-bound *literal* arguments are lossy through the row representation, so two
  numerics differing only in lexical form or hierarchy datatype can false-positive and tagged/typed
  literals can narrow to bare strings; IRI and blank-node arguments are exact.
- **A `cqels-UCs` module carrying tutorials 01–04**, the first slice of a larger effort, re-derived
  against current engine behaviour rather than recovered. Generator timestamps are seed-reproducible
  so window membership cannot depend on when the JVM runs. Tutorials 05–16 are not in this slice.
- **Self-hosted benchmark runner lanes — nightly screening and weekly deep bench**,
  orchestrated through `benchctl` with per-machine concurrency groups, heartbeat/liveness
  signalling, environment fingerprinting, and results pushed to orphan `bench-results/*` branches.
  **Changes nothing about the published artifacts and needs no upgrade action:** both cron triggers
  ship COMMENTED OUT, so merging activates nothing until a runner is registered per
  `scripts/bench/RUNNERS.md` and the `BENCH_JDK_VENDOR` / `BENCH_SNB_ROOT` repo variables are set;
  the bench lanes require a runner label the PR-gating jobs never carry, so they cannot block PR CI
  either way; and the supply-chain boundary holds — no publish or release-path job runs self-hosted,
  `publish.yml` is untouched, and the workflow's token permissions are minimal (`contents:write`
  reaches only the orphan branches).

### Fixed

**Public downstream reports (CQELS4J)**
- **A static BGP outside `STREAM{}` behaved as `OPTIONAL` on the composed route**. Closes the
  live residual of CQELS4J#54 and the wrong-pattern finding from CQELS4J#56.
  `enrichBoundRowWithStatic` returned at most one map and filled each needed variable
  *independently* from the forward-BFS view, so a partially-satisfiable static BGP still produced a
  filled row and a value could come from a triple that never satisfied the whole pattern. It now
  runs `staticBgpMatchesAll` over the enriched view and emits one row per satisfying assignment —
  0..N per input row, dropping the row when the conjunction has no solution — the same contract
  `mapToResultRows` already had on the lookup route. Three defects found while doing it, each
  pinned by a test proven to fail against the code it guards: the matcher seed omitted a stream
  variable shared with the static BGP but not projected (cross-zone false match) and, where present,
  carried the row's lossy Java form (so `xsd:int 42` matched no repository statement) — seeds now
  come from pre-projection `Value`s; `StaticBgpMatcher` now deduplicates candidate **triples** by
  s/p/o, never solutions, because a fixture loading one fact into both the default and a named graph
  produced two byte-identical solutions from one triple; and dedup keys are term-kind-qualified,
  since a blank node's `stringValue()` is its bare ID and `_:urn:x` collided with `<urn:x>`. The
  per-variable fill survives as `fillStaticVariablesIndependently`, used only by the temporal
  route's residual `putIfAbsent` stage, which is behaviourally untouched. The view scope is not
  widened: forward BFS only, null fallback, bounded enumeration under the existing work guard.
- **A prefixed name used as a constant inside an expression resolved to `null` for every prefix the
  query itself declared**, matching CQELS4J#55, so `FILTER(?row = c:REAR)` dropped every row while the
  same `c:` prefix resolved fine in the triple patterns two lines above — with no error at
  registration. `ExpressionEvaluator.resolvePrefixedName` consulted only `GeoVocabulary`'s seven
  built-ins and never the query's own prefix map; its two siblings in the same class,
  `resolveDatatypeIri` and `resolveFunctionIri`, already consulted the query prefixes first, and
  that asymmetry was the bug. The report named `FILTER` only; **every** expression context the
  engine evaluates through `ExpressionEvaluator` was affected — `FILTER`, `BIND` and `HAVING`, on
  both the stream-only and composed stream+static routes and inside an `OPTIONAL` block. The
  compiled-query pipeline needed no change (all seven evaluator constructions already passed the
  prefixes); the three in `CepPatternCompiler` used the no-arg constructor and now pass the prefix
  map they already held for the pattern's triple positions. **Deliberately unchanged:** an
  unresolvable prefix still yields `null` — a SPARQL type error, so `FILTER` drops the row and
  `BIND` leaves the variable unbound — matching what the other two prefixed-name positions and the
  triple-pattern side already do. Rejecting an undeclared prefix per SPARQL 1.1 §4.1.1.1 is a
  query-wide registration-time check that a per-row evaluation site cannot express; pinned as-is by
  two negative tests.
- **Rules never fired for observations pushed atomically**, matching CQELS4J#57.
  `ReactiveReteAdapter.apply` guarded on `isRDFStatement()`, which a `GraphStreamElement` returns
  `false` for by contract (its payload is a `List<Statement>`), so anything pushed through
  `DataStream.push(Collection)` passed through the adapter untouched — silently, with no warning.
  Users had to choose between atomic push (CEP and multi-pattern joins safe, reasoning dead) and
  statement-at-a-time (reasoning works, atomicity lost). `apply` now expands a multi-statement
  element and feeds each statement into the network stamped with the **element's** timestamp, so the
  whole observation shares one window slot and one eviction watermark; the element itself is
  re-emitted as the same instance rather than expanded, because atomicity is what the caller chose
  it for. Elements carrying no statement — a non-RDF payload, or a degenerate empty
  `GraphStreamElement` — still pass through unchanged.
- **Strict parse at every CQELS-QL registration surface**, matching CQELS4J#56 — see
  **Upgrade notes**. Strict parsing immediately caught a live instance of the bug in this tree:
  `cqels-cdsp`'s `energy-efficiency.cqels` read `[SLIDE 60s 10s]`, missing the mandatory `STEP`
  keyword, and the lenient parser had been registering a mangled window for as long as the file
  existed. Fixed here along with the two README rows documenting the same invalid shorthand. The
  spec's "`LATENESS` on `[NOW]`/`[TRIPLES n]` is a parse error" claim is now actually true at
  registration; previously the error was swallowed.

**Query correctness**
- **`FILTER NOT EXISTS` excluded a row only when the block's join-key variable happened to be
  materialised into the row** — SELECT list, expression reference, or an
  earlier pattern's binding. The same query on the same data returned opposite answers depending on
  the `SELECT` list, and the failure was silent-wrong: the anti-join simply went inert. SPARQL
  evaluates `NOT EXISTS` against the full solution mapping, so projection must not reach back into
  pattern evaluation. Each block's join-key variables — those shared with the outer group's stream
  pattern or static BGP, never block-local existentials — are now fetched into the row via
  `staticEnrichmentRowVariables` (memoised `computeNotExistsJoinVars`) and stripped after
  post-processing alongside the filter-only variables. Composition with the static-BGP-conjunction
  fix's matcher-authoritative enrichment was verified, not assumed: the matcher copies
  pre-projection seeds into every solution,
  so the fetch is a scalar column, never a row fan-out. **Route scope, stated honestly:** an
  `OPTIONAL` sharing the key resolves as well (pinned), while the **stream-only route** remains
  projection-dependent — pinned as a carve-out, not claimed fixed (see **Known issues /
  follow-ups**). A second carve-out, the **composed-aggregate shape**'s fold ordering, is
  closed later in this same release, below.
- **Three aggregate-projection shapes, one per aggregate route**, under a broader
  finding than that issue's title. A sweep of `{6 aggregates + 2 multi} × 5 windows ×
  {global, GROUP BY}` through `registerCqelsQuery` found the class still open on three shapes.
  `[NOW]` + a **global** (no `GROUP BY`) aggregate leaked the raw argument binding and is now
  rejected at registration (see **Upgrade notes**); a `COUNT(*)`-only projection over `[TRIPLES]`,
  `[SLIDE]` or `[RANGE … STEP]` emitted **nothing**, because `COUNT(*)` names no variable so every
  record was empty and the `!record.isEmpty()` filter annihilated the window — records are now
  retained for the all-`COUNT(*)` projection, which by construction implies `hasAggregates` so they
  fold to `{alias: value}` and no empty row can reach a consumer; and the same drop on the composed
  stream+static route is fixed below the inner-join gate, so multiplicity is unchanged. That issue's
  own reproducer (global `AVG` over `[TRIPLES 4]`) already passed on `main` — a separate, earlier fix
  had landed it in July,
  and the `@Disabled` marker that prompted the reopening was a stale carry-over from a branch based
  before that fix. Three pre-existing tests were rewritten: they pinned that the flag-off path "binds no
  `Long ?c`", which measurement showed was never true, and now pin the emission **grid**, which is
  what the flag actually governs.
- **Aggregates folded BEFORE `UNION` / `OPTIONAL` / `FILTER NOT EXISTS` instead of after**
 . SPARQL §18.2 evaluates the graph pattern before aggregation, so a solution
  a reshaper excludes must not contribute to an aggregate over its group. On the two routes that
  fold in a per-element batch — the stream-only windowed hash join and the composed stream+static
  aggregate topology — the fold ran FIRST, and the reshapers then probed a row already rebuilt from
  `GROUP BY` keys and aggregate aliases alone: `FILTER NOT EXISTS` excluded rows from the OUTPUT but
  not from the COUNT (a windowed count beside an exclusion of half the arrivals read `1,2,3,4` where
  `1,1,2,2` is correct, identically for `COUNT(?v)`/`SUM`/`AVG`); `UNION`, which is not a left outer
  join, matched neither arm on the folded row and dropped it, so the query emitted NOTHING;
  `OPTIONAL` null-filled the folded row, leaking block-introduced variables into a projection that
  never selected them. Fixed by reordering: the stream-only route now composes
  `rows → reshapers → fold → solution modifiers`; the composed route runs the reshapers between
  static enrichment and the in-batch `FILTER`/`BIND`, matching the composed `Istream` and
  non-aggregate topologies. Routes left alone, each verified rather than assumed: the incremental
  windowed-aggregate route is unreachable for these shapes (its gate rejects all three blocks and
  falls back to the hash join, inheriting the fix); the outer `collectList` fold already ran
  reshapers first; the temporal-join route rejects aggregates and all three blocks; `Istream` rejects
  aggregates at registration. **This also closes the composed-aggregate carve-out left open
  above** for `FILTER NOT EXISTS`: the fetched join key is now in the row when the anti-join probes
  it, so exclusion is effective there even for a `COUNT(*)`-only projection. **Residual, pinned
  rather than hidden:** on the stream-only route the row is the `SELECT` projection plus aggregate
  arguments with no fetch-and-strip pair, so a block key that is neither projected nor an aggregate
  argument leaves that block inert on either side of the fold — see **Known issues / follow-ups**.
- **Three engine fixes extracted from the six abandoned use-case PRs so they land on their own
  merits**. CEP `whereContext` guards in `SimplePatternProcessor` were never evaluated, and a
  negated state could match — both fixed. `ReasoningService`'s `isRDFStatement()` guard *reads* as
  "is this the kind of element I handle?" but *means* "is this a SINGLE statement?", so a
  multi-statement graph element was skipped entirely rather than reasoned over. And
  `ShaclContinuousQuery` now folds a `GraphStreamElement` into the snapshot atomically instead of
  dropping it. Recorded because the first attempt was worse than the bug: turning `statementsOf`'s
  silent drop into a pass-through collapsed two semantically **opposite** cases onto one empty list
  (main *forwarded* a non-RDF element but *dropped* an `isRDFStatement()==true` element with a null
  statement), and both callers read emptiness as "pass through" — producing an NPE raised before
  `Mono.fromCallable`, where `failClosedOnSolverError` cannot catch it. The guard now disambiguates
  at the call sites and drops. An intermediate commit message on that branch claimed the refactor
  "introduces no new failure mode"; that claim was false and was not carried forward.

**Multi-stream interval join hygiene** — the multi-stream epic's stages S1-S4
- **Interval-join runtime state was shared across every subscription to the returned `Flux`**,
  stage F5 — `IntervalJoinRuntime` was constructed once at assembly time inside `join(...)`, so
  buffers, watermarks and timers leaked between subscriptions. Its construction is now wrapped in
  `Flux.defer`, mirroring the existing per-subscription fix in
  `StreamElementToStreamEventAdapter`.
- **The left/right buffers had no cardinality bound**, stage F6(2). A cap
  (`cqels.join.intervalBufferCap`, default 100 000) now fails loudly instead of growing without
  bound. **The flag bounds one buffering STAGE, not the sum:** `SideBgpMatcher`'s raw-statement
  buffer and the interval join's own per-side buffer are capped separately by the same flag,
  so a *multi-triple* side can hold up to 2× the configured number across both stages. Size memory
  accordingly. **`F6(3)` — bounding the operator's internal event queue — is NOT implemented; it is
  an explicit documented waiver, not a resolution**, and the cap has no visibility into that queue:
  it is checked at admission, after an event has already been dequeued from `eventSink`, so a slow
  or backpressuring consumer can still accumulate unboundedly in front of it.
- **Out-of-order arrivals on the multi-stream route vanished silently**, stage F4(i):
  `TemporalJoinDispatch` always called `eventTimeJoin` with null late-data handlers. A 5-arg
  dispatch overload now carries a late-data listener, wired from `CqelsQueryCompiler`'s existing
  `lateDataDropListener`. Late drops are still **silent unless a listener is registered** —
  `setLateDataDropListener` defaults to `null`.
- **One defect class recurred across successive review rounds on the side-BGP stage, and the
  invariant it produced is now permanent**. Anything deciding whether a record or a buffered
  item is still relevant must read the *join*
  frontier — `min(left, right)`, and specifically the frontier of watermarks the operator has
  **enqueued**, not the one it has processed — because a side-local horizon discards partners the
  join still admits, and a processed-frontier read makes admission depend on downstream demand.
  Late filtering, side-BGP eviction and eviction on the watermark-forwarding path now all use that
  one anchor; only durations and spans may be per-side. The conjunction-extent check is
  overflow-safe (`nextMax > saturatingAdd(nextMin, extentMillis)` — the previous subtraction wrapped
  for timestamps more than `Long.MAX_VALUE` apart, admitting two statements that cannot co-reside
  under any window). **Documented, not fixed:** retention is governed by the *slower* side, so a
  stalled partner grows the side buffer until the cap fires; eviction is arrival-driven, so a side
  receiving nothing while the other advances retains until its next arrival or the cap; and state
  that the *other* side's later progress makes unreachable cannot be released without a cross-thread
  call into the single-threaded matcher — bounded by the cap, freed when the subscription ends.
- **The docs described the multi-stream route as RStream-cumulative; it is an interval join**,
  stage S4 track A. The cheatsheet claimed a windowed join "re-evaluates the whole window on every
  arriving element and re-emits all current rows", and told users to *omit* `Istream(...)` for that
  behaviour — omitting it gives neither. `CQELS-QL_SPEC.md` had no multi-stream section at all and
  listed "Multi-Stream Join Operators" under *Future Extensions*, while the route has compiled and
  executed since an early PR. New spec §10 states the route's real contract with every claim citing
  production code or a characterization test: the interval predicate with **closed** endpoints,
  insert-only pair emission (with the honest caveat that no retraction contract exists, so
  eviction-then-re-formation could re-emit), bag/multiset semantics, and the rejection surface by
  category. Two pre-existing doc errors fixed with it: `LATENESS` on this route is accepted at
  registration and rejected at `execute()` (two such sites), not rejected at registration; and late
  drops are silent unless a listener is registered. `CqelsQueryDefinition.isIstream()` and
  `CqelsQLASTBuilder` no longer claim "Default is RStream-cumulative" unconditionally.
- **The route had almost no result-asserting coverage before it was changed**, stage S2. Across the
  whole repository there was exactly **one** result-asserting test with a non-`[NOW]` window on a
  multi-stream join and **zero** covering the headline `RANGE ⋈ RANGE` shape.
  `MultiStreamWindowCharacterizationTest` (17 cases) pins what the route does today — expected
  values hand-derived from the traced interval predicate and cold-source event order, no oracle used
  — so S4 changed it against a recorded baseline rather than a blank one. Five behaviours are marked
  CHARACTERIZED rather than correct.

**Documentation accuracy, and the gates that keep it**
- **Every row-cardinality claim in the user-facing documentation was re-derived**, stage S5. The
  docs stated row counts that hold only for restricted query shapes: row count in this compiler is
  not a property of the window spec but is governed by `GROUP BY`, by `hasBatchSolutionModifiers()`
  (= `hasOrderBy() || hasLimit()`), by per-binding and per-window fan-out, by route, by the static
  side, by the EMIT policy and by the R2S operator. 158 claims enumerated across
  `CQELS-QL_SPEC.md`, `CQELSQL_SYNTAX_CHEATSHEET.md`, `ToolSchemas.java` and `README.md`; **89
  defects corrected over three passes** (47, then 36 from a re-audit of those corrections, then 6
  from review of the result). Notable semantic corrections: `OPTIONAL` is a left outer join and can
  never emit zero rows (it had been grouped with the inner-join shapes); a null `staticRepository`
  makes `NOT EXISTS` and `OPTIONAL` inert but makes `UNION` arms and inner/lookup/`GRAPH` joins
  **drop every row**; `k` matching triples give up to `k²` bindings on the multi-pattern hash-join
  route, not `k`; the out-of-order tolerance bound on centered windows is `P + F`, not `F`; and
  RSP-QL named windows parse and compile but throw at registration.
- **The first sweep broke shipped Java, and nothing caught it**. Audit commentary — including
  an editorial instruction to the editor — was spliced into the `register_stream_query` description,
  and one JSON string was split across two raw lines. The file still **compiled**, because a Java
  text block containing malformed JSON is a valid text block, so 2178 green tests and a clean
  compile said nothing about it: `register_stream_query` could not register at runtime.
  `ToolSchemasJsonValidityTest` now reflects every schema constant and parses each, and was proven
  to fail on the exact break before being trusted.
- **69 line-number citations replaced with method names** — 48 of them were already stale.
  Zero line numbers remain, so the convention has nothing left to drift.

### Changed
- **`validate_stream_query`'s MCP description no longer implies it catches registration-time
  rejections**. It is parse-only, and now says so, naming the constructor-time guards it
  never runs — which matters more now that strict parsing makes registration throw and this same fix
  adds a new registration-time rejection.
- **The interval join subscribes its sources on first `request()`, not on `subscribe()`**.
  Moving the trigger from `doOnSubscribe` to a once-guarded `doOnRequest` was necessary to close a
  subscribe-order race structurally — a cancelled Reactive Streams subscriber never calls
  `request()` — but it is a real, accepted behaviour change: a downstream that never requests (a
  custom `Subscriber` withholding `request()`) now leaves **both** sources permanently unsubscribed
  and the join never completes, where previously two otherwise-empty sources attached and completed
  regardless of demand. Cancellation still cleans up correctly.
- **Each emission axis note in the docs is presented as a WORKING list, not a guarantee**. It
  went A–F, then A–P, then A–S, and each revision was found incomplete by the next audit. "All axes
  checked" is a claim about a revision.
- **CI dependency bumps:** `anthropics/claude-code-action` 1.0.180 → 1.0.193,
  `dorny/paths-filter` 3.0.3 → 4.0.3, `actions/setup-java` 5.6.0 → 5.7.0,
  `docker/login-action` 3.7.0 → 4.6.0.
- **The nightly JDK compatibility probe's rationale no longer cites a JDK 26 deadline**. The
  comment justified collecting JEP 472/498 warnings "because JDK 26 turns the Unsafe ones into
  errors". The JDK never said that — its own wording is "a future release", undated — and the claim
  was stale on its face: 26 has shipped and these are still warnings. The argument for not silencing
  the flags never depended on a date and is unchanged.

### Upgrade notes

**Several merged changes alter results for existing queries.** The three below have the widest blast
radius and are spelled out in full; narrower ones follow, and none of the three lists is a total —
read all three before upgrading. Each replaces a silently wrong, silently empty or silently salvaged
result with a correct one, so none is a regression — but every one is observable.

1. **Every CQELS-QL registration surface now parses strictly**.
   `CQELSEngine.registerCqelsQuery` and `CQELSEngine.registerCepQuery`, the `cqels-cdsp`
   `QueryRegistry` `.cqels` file
   loader, and (inheriting through the engine methods, pinned by a regression test rather than
   assumed) MCP `register_stream_query` all route through `CqelsQueryParser.parseCqelsQLStrict`.
   **Before:** a genuine syntax error — a property path, a `[SLIDE 60s 10s]` missing its `STEP`
   keyword — was salvaged by ANTLR's single-token error recovery, printed to stderr, and registered
   as whatever recovery produced, which then emitted nothing. **After:** it throws
   `CqelsQLParseException` naming the offending construct. Queries that "worked" by silently doing
   nothing now fail loudly at registration; validate your registered query text before upgrading.
   `registerCypherQuery` and `registerAspQuery` are different grammars and are **out of scope** —
   they are unchanged.
2. **A static BGP outside `STREAM{}` on the composed route is now a CONJUNCTION**.
   **Before:** each needed variable was filled independently from the enriched view, so a
   partially-satisfiable static pattern still produced a filled row (`OPTIONAL`-like), and a value
   could come from a triple that did not satisfy the whole pattern. **After:** one row per complete
   satisfying assignment (so a multi-solution match fans out, bag semantics), and the row is
   **dropped** when the conjunction has no solution. Two second-order effects on the same change:
   the matcher's seed now carries pre-projection `Value`s, so a typed literal such as `xsd:int 42`
   matches statements it previously missed; and `StaticBgpMatcher` deduplicates candidate **triples**
   by s/p/o, so a repository holding one fact in both the default graph and a named graph now yields
   one solution where it previously yielded two — a SHACL fixture doing exactly that had been
   counting one violation as two. Distinct facts still fan out.
3. **`[NOW]` with a GLOBAL (no `GROUP BY`) aggregate is now rejected at registration**.
   **Before:** `SELECT (AVG(?temp) AS ?avgTemp) … [NOW]` delivered `{temp=25.0}` and never bound
   `?avgTemp` — every in-batch fold gate is guarded by `!isNow` and the outer fold requires
   `hasGroupBy()`, so it folded nowhere and the raw argument binding shipped. **After:** a loud
   rejection at registration naming the windows that accumulate. It is rejected rather than routed
   to a fold because `evictExpiredElements` has no `NOW` case: folding would never evict, so the
   aggregate would silently span the whole stream under an instantaneous-window declaration — a
   worse wrong answer with unbounded growth behind it. **Grouped `[NOW]` aggregates are unaffected**
   (they reach the outer fold and compute correctly on a finite source), and the multi-source
   temporal-join route keeps its own separate aggregate rejection.

**Further result changes, scoped narrowly:**

- **`FILTER NOT EXISTS` now excludes rows whose join key is unprojected — on the static-lookup
  route**. Rows that previously survived the anti-join because the key was absent from the
  `SELECT` list are now correctly excluded, and an `OPTIONAL` sharing that key resolves too. This is
  **route-scoped, not general**: the stream-only route remains projection-dependent and is pinned as
  a carve-out. The composed-aggregate shape was a second carve-out at the time but is closed
  later in this entry, below.
- **`COUNT(*)`-only projections that previously emitted NOTHING now aggregate**, over
  `[TRIPLES]`, `[SLIDE]` and `[RANGE … STEP]` windows and on the composed stream+static route. A
  consumer that treated the empty result as "no data" will now receive folded `{alias: value}` rows.
  `COUNT(*)` with an *unprojected* `GROUP BY` key still emits nothing — see below.
- **Aggregate results change for a query that also uses `UNION` / `OPTIONAL` / `FILTER NOT EXISTS`**
 , on the stream-only windowed hash join and the composed
  stream+static aggregate topology. A windowed `COUNT`/`SUM`/`AVG` beside a `FILTER NOT EXISTS`
  that previously excluded rows from the output but not from the count (`1,2,3,4` where `1,1,2,2`
  is correct) is now correct; a `UNION`+aggregate query whose folded row matched neither arm and
  emitted NOTHING now emits; an `OPTIONAL`+aggregate query that leaked block-introduced variables
  into the projection no longer does. The stream-only route keeps one carve-out — a block key that
  is neither projected nor an aggregate argument still leaves that block inert — see **Known
  issues**.

**Also worth checking before upgrading:**

- **A multi-stream join that previously grew without bound now fails loudly at 100 000 buffered
  elements per buffering stage**, behind the `cqels.join.intervalBufferCap` flag. Like alpha.18's three
  runtime caps, this fires at *runtime* on a data-dependent threshold, not at registration: a query
  registers clean and can throw later under load. Raise it with
  `-Dcqels.join.intervalBufferCap=<n>`, and supply a `QueryResultListener` with an `onError` if you
  register with a lambda — a tripped query otherwise stops emitting permanently with only a log line
  to show for it. **Sizing note:** the flag bounds one stage, not the sum, so a multi-triple side —
  which has both `SideBgpMatcher`'s statement buffer and the interval join's own — can hold up to
  2× that number. Multi-triple sides also make a trip likelier, because a solution is expired on its
  newest constituent but stops being usable at its oldest.
- If you subscribe to an interval join with a custom `Subscriber` that withholds `request()`, both
  sources are now left unsubscribed and the join never completes — see **Changed**.
- Rules now fire for observations pushed via `DataStream.push(Collection)`. If you worked
  around the old silence by pushing statement-at-a-time as well, drop the workaround — the atomic
  push now reasons on its own, and the statements share one window slot and one eviction watermark.
- A prefixed name inside `FILTER`/`BIND`/`HAVING` that previously matched nothing now resolves
 . Queries whose results were silently empty for this reason will start emitting rows.

### Known issues / follow-ups

- **Multi-stream / multi-window joins — revised from alpha.19.** That entry called them
  "effectively unimplemented" and listed four defects; S1/S2/S4 closed some and not others, so here
  is the corrected picture.
  - **Now fixed:** interval-join state is per-subscription rather than shared across every
    subscription to the returned `Flux`; the buffers are bounded by
    `cqels.join.intervalBufferCap` and fail loudly rather than growing without bound, so the
    "no state bound" clause no longer holds; the route is documented as the **interval join** it is
    rather than the RStream-cumulative relation join the docs promised; a `STREAM` block may
    now carry several triple patterns, conjoined by a BGP matcher; and a query sharing two or
    more variables is joined naturally on all of them instead of being rejected.
  - **Still true:** the tested, correct multi-way join machinery — `MultiWayJoinState` /
    `JoinGraph` — remains **unreachable from the query language** (verified: outside its own
    package it is referenced only by `cqels-benchmarks` and tests, never by `cqels-core`'s or
    `cqels-engine`'s production sources, and by no compiler). There is still **no retraction**:
    emission is insert-only pair emission, so eviction followed by re-formation could re-emit, and
    no delta or
    retraction contract exists. And the route remains an **interval join derived from the two window
    durations**, not general multi-way windowed-relation machinery — that is the epic's recorded
    decision D1 = (a'), not an oversight, and a windowed-relation join would arrive later on a
    separate named-window surface. Per-stream watermark control is still absent: `LATENESS` on this
    route is accepted at registration and rejected at `execute()`, and idle-source watermark advance,
    stage F6(1), is still open. `TemporalJoinSpec.Kind.WINDOWED` remains a declared-but-unimplemented
    second kind that throws on dispatch. The epic stays open.
  - **No longer true, and it was already stale when alpha.19 shipped:** alpha.19's claim that "a
    multi-stream declaration without a matching number of `STREAM` blocks silently falls back to a
    nondeterministically chosen single stream" describes behaviour that an earlier PR had
    *already* replaced with a loud `UnsupportedOperationException` naming both supported shapes —
    that PR is
    an ancestor of the `v2.0.0-alpha.19` tag. Nothing in this cycle changed it; the alpha.19 note
    was simply describing a state of the tree that no longer existed.
- **Residual from the aggregate-fold reordering — on the stream-only route, a reshaper's
  block key that is neither projected nor
  an aggregate argument still leaves that block inert, on either side of the fold.** Unlike the
  composed route, whose static-enrichment stage fetches a `FILTER NOT EXISTS`/`OPTIONAL`/`UNION`
  block's join key into the row even when it is unprojected, the stream-only route's row is
  only the `SELECT` projection plus the aggregate arguments — there is no equivalent fetch-and-strip
  mechanism there. Reordering the fold relative to the reshapers fixed the ordering
  everywhere the key IS present in the row, but does not change this: `SELECT (COUNT(*) AS ?n) … FILTER
  NOT EXISTS { ?s ex:excluded ?e }` on the stream-only route still reads `1,2,3,4` instead of
  `1,1,2,2` unless `?s` is projected or is itself an aggregate argument.
- **`COUNT(*)` with an unprojected `GROUP BY` key still emits nothing.**
  `mapBindingsToRecord` carries only `SELECT` variables and aggregate-argument variables, never an
  unprojected group key; with `COUNT(*)` naming no variable either, every record is empty and the
  `!record.isEmpty()` filter annihilates the batch. One shape past the earlier aggregate-projection
  fix above, and deliberately
  excluded there: retention alone would trade an empty result for a **wrong** one, since
  every-record-empty collapses `buildGroupKey` to a single null-keyed group counting everything.
- **The row representation still string-lifts non-core typed and language-tagged
  literals**, so `datatype()`/`lang()` lose fidelity and a bare `FILTER(?dt)` over an `xsd:dateTime`
  binding is still wrongly admitted on `Map`-row routes. This is what **bounds `sameTerm()`
  accuracy** for row-bound literal arguments: two numerics differing only in lexical form or
  hierarchy datatype can false-positive, and tagged/typed literals can narrow to bare strings. IRI
  and blank-node arguments are exact. Carried over from alpha.19, unchanged.
- **Private tracker issue numbers ship inside user-facing exception messages.** They appear
  in `throw` messages across `cqels-core` and `cqels-engine` — `CqelsQueryCompiler` carries most of
  them, with `CQELSEngine`, `CqelsQLASTBuilder`, `StaticBgpMatcher`, `TemporalJoinDispatch`,
  `WindowSpec`, `CepChannelEmitter` and `CepPatternCompiler` supplying the rest — in messages
  downstream users hit through the published artifacts and cannot look up. **The issue's own tally
  is wrong and needs recomputing before the sweep is scoped from it:** it claims 46 distinct issue
  numbers, but a fresh scan of `src/main` string literals with comments and Javadoc stripped finds
  17 distinct tokens (16 numbers plus one two-variant token counted once) across roughly two dozen
  message sites
  — and the population is a moving target, since `CepChannelEmitter` is new in this release and
  arrived carrying one. It propagates: a downstream docs PR
  quoted one verbatim into public prose and was caught only in review. Not a security issue — the
  numbers alone, no credentials, URLs or private project names — and not release-blocking; the
  messages are correct and useful apart from the reference. The durable half of the fix is extending
  the existing leak gate, itself a prior fix in this same family, to string literals in
  `src/main`, which it currently does not cover.
- **Use-case tutorials, slices 2-4 outstanding.** The tutorials module landed slices 01-04
 ; 05-16 remain.
  The six original use-case PRs are closed, not merged: they were never built or tested (their only
  workflow died at `detect-changes` on an Actions billing block, so every downstream job showed
  SKIPPED and the red X was misread as a workflow bug), and running them locally gave 84/88 with all
  four failures asserting behaviour the engine does not have.
- **Two S5 test-coverage residuals closed, two documentation residuals not.** The row-cardinality
  sweep recorded
  that the CypherQL, CEP and `SPARQLExecutor` routes were reached by code reading with no tests
  pinning them, and that processing-time mode rested on a single `eventTime` gate with no test.
  This closes both, as four pinned claims — CypherQL `SlidingWindow` and `SPARQLExecutor`'s
  `collectList` paths are completion-only (pinned at operator *and* registered-query level, with
  exact step-grid counts); directional CEP honours `LATENESS` (a control/treatment pair over one
  event sequence with opposite outcomes); processing-time interval joins consult no watermark for
  lateness. All four hold exactly as worded — no doc/engine contradictions found. **Not closed:** the
  status-and-provenance claims ("not yet merged", "not yet implemented") that the same sweep flagged
  as their
  own unswept population after three instances turned up incidentally; and no convergence is claimed
  for the docs sweep — each of the last several rounds found defects introduced by the previous
  round's fixes. What improved is that the *structural* classes (JSON validity, citations,
  duplication, paren damage, punctuation residue, table shape) are gated rather than merely fixed.
  The same PR also records a finding it did not act on: `IntervalJoinOperator.processingTimeJoin` is
  unreachable from any production API (`TemporalJoinDispatch.assemble` only ever builds event-time),
  awaiting a remove/wire/document decision.
- **The interval join expires side-BGP solutions by their newest constituent.** A solution
  stops being usable at its *oldest* constituent but is expired by the operator on its *newest*, so
  it occupies the buffer for `newest − oldest` longer than it can ever produce a row; worst case, a
  solution spanning its whole side window is dead on arrival and still held for a full window, and
  at sustained rates those entries count against `cqels.join.intervalBufferCap`. Documented rather
  than fixed, deliberately: `cleanupBuffer` is a `headMap` range cut over a timestamp-keyed
  `NavigableMap` that removes whole buckets, and two solutions sharing a `newest` but differing in
  `oldest` land in the same bucket, so the rule cannot be expressed as a cut on that key. **Liveness
  and memory, not correctness:** no wrong row results, but the buffer cap fails loudly, so at
  sustained rates the dead entries can trip it and terminate an otherwise valid join — raise
  `-Dcqels.join.intervalBufferCap=<n>` if a legitimate query hits it. Single-triple sides are
  unaffected (`newest == oldest`).
- **Fail-loud on unknown function names was assessed and deliberately NOT shipped** with `sameTerm`
 . `CqelsQueryCompilerFunctionRegistryTest` pins silent-`null` as permanent behaviour, and
  the grammar's `idToken` path would need a synchronized keyword list across every expression entry
  point. The gap is known, not overlooked — a misspelled function name still evaluates to `null`,
  which `FILTER` reads as a type error and drops the row.
- **The interval join's internal event queue is still unbounded**, epic stage F6(3), by
  explicit
  documented waiver rather than resolution. The per-side buffer cap bounds a different
  structure. Also still open on this route: `SideBgpMatcher` may drop a
  `GraphStreamElement` — the same `isRDFStatement()` guard family as the `ReasoningService` and
  SHACL fixes above — and `StreamElementToStreamEventAdapter.lift` drops the
  subscriber's Reactor context and drains its source unbounded during subscription — both
  pre-existing, both shared with the temporal-join route.
- **Carried over from alpha.19, unchanged:** modifier-free per-arrival routes still ignore their
  declared window entirely.


## Using this release

Artifacts resolve anonymously — no token, no `settings.xml`:

```xml
<repositories>
  <repository>
    <id>cqels</id>
    <name>CQELS Releases</name>
    <url>https://maven.cqels.org/releases</url>
  </repository>
</repositories>

<dependency>
  <groupId>org.cqels</groupId>
  <artifactId>cqels-engine</artifactId>
  <version>2.0.0-alpha.20</version>
</dependency>
```

## Verifying this release

The signed manifest and its signature are served from the artifact repository;
the public key is served from a different origin, so verifying with it is not
circular. Take the key from the commit pinned below.

```bash
set -euo pipefail
BASE=https://maven.cqels.org/releases/supply-chain/2.0.0-alpha.20

curl -fsSLO https://raw.githubusercontent.com/cqels/CQELS4J/5bc3d2d00a0c7a5db8e496ad22e73fc34f2a399b/cosign.pub
echo "36dd8daa9988f23eb40c4f3550fa7bdfa3796e5e58cce8d23b9cc6a99f47f30b  cosign.pub" | sha256sum --check --strict - || exit 1

curl -fsSLO "$BASE/SHA256SUMS" -O "$BASE/SHA256SUMS.bundle"
cosign verify-blob --key cosign.pub --bundle SHA256SUMS.bundle \
  --new-bundle-format=false --insecure-ignore-tlog SHA256SUMS
```

The URL is pinned to a commit rather than to `master`, so it keeps working
after a key rotation. `--insecure-ignore-tlog` is expected: these signatures
carry no transparency-log entry, which is precisely why the key's origin
matters. Full walkthrough: https://github.com/cqels/CQELS4J/blob/master/SUPPLY_CHAIN.md
