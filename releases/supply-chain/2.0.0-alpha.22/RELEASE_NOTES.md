# CQELS 2.0.0-alpha.22

A **query-language release**: CQELS-QL gains SPARQL 1.1 property paths, and CypherQL gains
variable-length relationships and joins across two streams. A property path in the one `STREAM` block of
a single-stream query over a `RANGE`, `SLIDE` or `TRIPLES` window, with no static graph pattern, is now
evaluated on the windowed hash join, under a work budget that a pre-flight bound enforces before any row
of the path is built; every other CQELS-QL route refuses a path at registration and names it. A CypherQL
`-[r:T*n..m]->` is evaluated once per window batch with openCypher trail semantics, and a `MATCH` over
two declared streams that share a variable now joins them; the join can be aggregated, or enriched from
a declared static graph, though not both. `VariableLengthPathOperator`, which nothing in the engine
called, is removed, a compile-time break for its callers.

The emission model is now something a query states rather than something its window keyword implies:
`EMIT ON_CLOSE` / `EMIT ON_UPDATE` on a legacy window choose the model of a single-stream aggregate,
single-pattern, multi-pattern or static-bearing, and an aggregate on a legacy window that states no
`EMIT` logs one warning naming the model its keyword chose. Directional `[FUTURE]` / `[PAST … FUTURE]`
windows run multi-triple joins and aggregates within one stream by default. `LATENESS` is honoured on
multi-stream `[RANGE]` joins, and on `[SLIDE]` windows on the single-stream on-close aggregate and the
multi-stream join; durations can be written in `ms`; `DataStream.advanceTo` closes the event-time
windows of a quiet stream's CQELS-QL queries without pushing an element (a custom stream processor must
forward the `StreamPunctuation` it produces); and the elements a CQELS-QL query drops as late are
observable per query through the engine API, on the routes its Added entry lists. CypherQL `RANGE` and
`SLIDE` windows now close on a live stream instead of only when the source completes.

**SPARQL conformance:** the outer group's `FILTER` now runs after its `OPTIONAL` and `UNION` blocks and
its `BIND`s on every route, with the blocks and `BIND`s in the order written (CQELS4J#85, CQELS4J#83). A
CQELS-QL query that declares `FROM` reads only its `FROM` graphs on every route, an explicit `GRAPH`
block on the ordinary single-stream lookup routes reads only its graph, and CypherQL
`FROM STATIC` / `FROM GRAPH` scope the static lookup to the declared graphs. A quoted literal, IRI or
prefixed-name constant in a `STREAM` block's triple pattern matches the RDF term it denotes. Eight
standard functions that were missing from the evaluator (`COALESCE`, `REGEX`, `REPLACE`, `STRDT`, `MD5`,
`ENCODE_FOR_URI`, `STRBEFORE`, `TIMEZONE`) now evaluate, a string literal in an expression decodes the
escapes SPARQL defines, `STRLEN`, `UCASE`, `LCASE` and `SUBSTR` refuse a non-string literal and the last
three keep its language tag, `NOW()` is one instant per evaluation, a CQELS-QL `FILTER` compares
durations by value, and a row-bound language-tagged or non-core typed literal keeps its term through the
result row. Aggregates gain per-aggregate `DISTINCT` and `SELECT` expressions over aggregate aliases;
`FILTER` inside `FILTER NOT EXISTS`, and `FILTER EXISTS`, now work (CQELS4J#92); and every variable the
`SELECT` list does not name is projected away on the composed route. Shapes that used to register and
answer wrongly are now refused at registration: a grouped query that projects, tests in `HAVING` or
orders by a variable outside its group, a top-level `BIND` whose target is already bound, and a CEP
`SEQ` argument that is not the subject of a stream triple pattern, which rules out named-window CEP.
Result values change type as well: a language-tagged or non-core typed literal comes back as an RDF4J
`Literal` instead of a `String`, and a declared `resultMapping` now decides the Java value of a literal
at more of the places a row value is written, including what a `FILTER` or `BIND` on the static-lookup
routes reads (its Fixed entry lists the exceptions). Several of these change
rows or types; the Upgrade notes say what to change.

The stream-to-static lookup routes answer more of what SPARQL asks and stay bounded while doing it. A
static BGP's answer no longer depends on the order its triples are written for the reported shapes (a
leading open triple, a predicate-only anchor, two mutually anchoring triples, cycles of up to four
triples); a pattern whose only constant is a literal object, and a fully-open pattern that shares no
variable, now read the static scope instead of answering from a rooted view alone; and the composed
windowed route joins a row whose join key binds a literal. For eligible queries, a static BGP whose
every pattern is anchored is evaluated cheapest pattern first (CQELS4J#99), and a geospatial `FILTER` on
the per-arrival lookup route probes a spatial index over the static side (CQELS4J#97): about 3 ms per
element instead of about 210 ms in the reported case. The wider reads come with bounds: the repository
candidate fetch stops at the work guard's bound, the rows of one arrival share one static work budget on
the composed route, the windowed hash join checks its cross-product cap while it builds bindings, the
enrichment cache has a maximum entry count, and repeated `execute()` calls on one compiled query no
longer accumulate lookup operators.

**Durability and security:** a storage directory now has one writer, so a second provider-built backend
on it fails to open instead of deleting records, and the four storage providers require an explicit
`path`. Java deserialization of journal payloads and checkpoint blobs is filtered: a class outside the
admitted set fails the append, the snapshot or the restore; `cqels.storage.serialFilter` admits further
classes. The durable stream catalog no longer needs the whole journal in memory, the catalog read and
the recovery reads honour the replay deadline, and `cqels://engine/status` reports how many recovered
events were delivered and how many were correctly skipped. A stream's replay history is bounded in bytes
as well as in elements, and a new stream's name is bounded at 256 UTF-8 bytes. **MCP:** `forget_memory`
refuses a raw SPARQL `LOAD`, which made the server fetch a URL the client chose; a client that stops
reading its event stream no longer holds the engine's result thread; three multi-session races under the
Streamable-HTTP transport are closed; and an unrecognized `format` is an error instead of JSON.

**Elsewhere:** `cqels-triegs`, factorized incremental view maintenance for windowed joins, is a released
module whose `model`, `maintain`, `window`, `dictionary` and `aggregate` packages are frozen from this
release, though no query is routed to it yet. `geof:distance` is no longer limited to points and
`geof:buffer` buffers in metres through a local projection (CQELS4J#94, CQELS4J#95), and an EWKT SRID
other than 4326 (outside `geof:getSRID`) and a NaN, infinite or non-numeric buffer radius, which used to
answer silently, are now type errors. Concurrent ASP solves no longer return wrong answer sets, an ASP
solve no longer pays a fixed classpath scan, and Apache POI no longer reaches the classpath of modules
that depend on `cqels-asp`. The semantic index no longer grows when a fact is stored again, and refuses
to open an index stamped for another embedding, or not stamped; two API changes need source changes: a
custom `EmbeddingProvider` must implement `id()`, and `createSearcher()` is replaced by paired
`acquireSearcher()` / `releaseSearcher()` calls. Opt-in additions include delete-and-re-derive
retraction for cascading RETE truth maintenance, an overgrounding ASP solver backend, inert-rule pruning
for registered query predicates, a bound on the CEP matches one input emits, and protection of
target-defining triples in the SHACL repair search. A grype scan of the staged release jars reports no
High-severity dependency findings, down from 23, and the engine's user-facing messages no longer cite
private tracker issue numbers.

Several of alpha.21's known issues are closed: CypherQL `RETURN DISTINCT` is applied and its
`RANGE` / `SLIDE` windows close on a live stream; a `[NOW]` `SELECT DISTINCT … ORDER BY … LIMIT` no
longer re-emits an unchanged projected snapshot; an `OPTIONAL` that probes an unprojected `BIND` target
correlates; the per-arrival lookup route joins an arrival's stream BGP once instead of twice; the
recovered-event figures and the result-signal stall are the ones named above; and string literals decode
SPARQL's escapes. New known issues are listed at the end of this section, among them two shapes that
drop rows: a static `UNION` whose join variable is not projected, and an aggregate on the windowed hash
join over solutions whose projection is empty.

### Added

- **CypherQL variable-length relationships over the window (`cqels-core`).** A MATCH with
  `-[r:T*n..m]->` is now evaluated once per window batch (`RANGE` tumbling, `SLIDE`, `TRIPLES`; each arrival under
  `[NOW]`) over the statements of every element in it, with openCypher ALL TRAIL bag semantics: one row per path, no
  relationship used twice, nodes may repeat, and `r` bound to the list of the path's relationships (a one-element
  list for an exact `*1`). The DIAMOND gives `(a,d)` twice, where CQELS-QL `p+` gives it once
  (`PathDialectDivergenceTest`). `shortestPath(...)` and `allShortestPaths(...)` keep one or all of the shortest
  paths per pair. An unbounded `*` or `*n..` is refused at registration, as is any variable-length relationship
  (an exact `*1` included) in a static-lookup or cross-stream-join query, a variable-length relationship variable that names another relationship or a node of the MATCH,
  and a named path `p = ...`. Two loud failures bound one window batch: path work over `-Dcqels.path.workBudget`,
  including the checks run on each path found, throws `PathBudgetExceededException`, and more than
  `-Dcqels.join.crossProductCap` (default 100,000) bindings throws `IllegalStateException`. Queries without a
  variable-length relationship take the unchanged per-element route.
- **Opt-in incremental retraction for cascading truth maintenance** (slice 1, `cqels-reasoning-rete`) —
  `ReasoningConfig.incrementalTmsRetraction(true)` makes `ReteNetwork.retract` in the TMS + recursive-inference
  configuration delete and re-derive (DRed) instead of replaying the surviving base stream: it removes the
  derivations that used the fact, transitively, re-derives what still has a grounded support (circular support
  does not keep a fact alive), and settles the fixpoint without the per-element depth bound, so the supported set
  is the least fixpoint of the surviving base (the retract contract), including facts the bounded forward
  path had truncated. A seeded differential test checks every step of random insert/retract sequences against the
  least-fixpoint oracle, with and without window eviction. Default off; replay is unchanged.
- **Opt-in overgrounding ASP solver backend** (step 1, `cqels-asp`) — `OvergroundingAspSolverBackend`
  keeps the ground program across window updates: an added fact grounds only the rule instances it reaches, a
  retracted fact removes only the instances that lose support (delete and re-derive), and Delta searches the
  maintained program. Programs outside the fragment (aggregates, interval facts, recursion whose head builds a
  new term, non-fact window strings) are solved by the cold `DeltaAspSolverBackend`, and so is every update after
  the maintained program, or the join work of one update, outgrows a ground-instance budget. A differential test checks identical answer sets against the cold
  solve at every step of a sliding-window sequence. Default backend unchanged.
- **CQELS-QL property-path conformance suite at the engine level (`cqels-engine`, tests only).** The design
  note's T-1 to T-13, T-18 and T-20 to T-23 register real CQELS-QL through `CQELSEngine.registerCqelsQuery`, push the
  CHAIN, DIAMOND, CYCLE, MIXED, CDIAMOND and DECOY fixtures as stream elements and assert the exact row multiset the
  listener sees (the grammar has no `COUNT(DISTINCT ...)`), over RANGE, TRIPLES and SLIDE windows. A seeded random
  comparison runs random paths (sequence, alternative, inverse, negated set, and the three closures, nested) over
  random streams and compares every row multiset with a from-scratch evaluator of the SPARQL 1.1 algebra and with the
  RDF4J SPARQL engine. T-17 stays disabled until the pre-flight bound lands.
- **Path queries are refused before they blow up, not partway through (T-17, `cqels-core`).** When the
  windowed hash join reaches a path, `PathPreflight` bounds the path's work per partial binding from the window's
  index (its triples per predicate, the busiest node's edges, the distinct terms a step can start or end at) and
  multiplies it by the partial bindings the patterns before it actually built; a path whose bound is over the budget
  fails the query with `PathBudgetExceededException` before any row of it is built, naming the path as written and
  the bound. A closure whose body is one edge step (`ex:knows+`, `(ex:p|^ex:q)*`, `(!ex:r)+`) is bounded by the
  body's own graph, which for `?x ex:knows+ ?y` with neither end bound is its exact work on a chain, a star or a
  tree, so it is refused only when its walks would cost over the budget; a longer body is bounded from the index
  sizes alone. The bound never undercounts (a randomized test runs every operator at a budget equal to its bound),
  so it also rejects some work that would have fitted; the evaluator's own budget stays as the backstop. The budget
  is now settable: `-Dcqels.path.workBudget=<n>` (default 10,000,000 units, read once at
  class load; a non-positive or non-numeric value warns and keeps the default). Non-path queries are unchanged and
  keep their windowed-join cap behaviour.
- **Property-path queries run on the windowed hash join (`cqels-core`).** A path in the triple patterns of
  the one STREAM block of a single-stream query over a RANGE, SLIDE or TRIPLES window, with no static graph pattern,
  is now evaluated: `SELECT ?x ?y FROM STREAM S [RANGE 10s] WHERE { ?x ex:knows+ ?y }` answers from the edges in the
  window and forgets a pair once an edge on its only path expires. The path is joined after every plain pattern of
  the block (whatever the `cqels.bgp.reorder` flag says), never takes the per-subject parallel join, and an aggregate
  over it folds on the windowed hash join like a multi-pattern aggregate: per arrival, or once per closed window
  under `EMIT ON_CLOSE`. One evaluation may do 10,000,000 units of path work; past that the query fails with
  `PathBudgetExceededException` instead of answering with a subset. The rows a path builds count against the
  windowed join's cross-product cap and search budget as they are built. Every other route still refuses a
  path at registration (and in the MCP validate tool), now naming it: "property paths (...) are not supported yet"
  in a CEP query, in a multi-stream query, outside the STREAM block, beside a static graph pattern, or over a NOW
  or directional window. A path endpoint that is a bare number or boolean is refused too.
- **Property paths are desugared at parse time.** A triple pattern whose predicate is a path is now a
  `PathPattern` (a `TriplePattern` subclass, so the predicate is still the path as written) carrying a `PathPlan` from
  the new `PathRewriter`: `/` chains through fresh `__pN` variables, `^` swaps subject and object, `|` becomes arms
  whose answers are concatenated (never a UNION block), an alternative of distinct links becomes one predicate-set
  step (`(:a|:b|:c)/(:d|:e|:f)` is one arm, while `:p|:p` stays two), `!(...)` becomes a negated predicate-set step,
  and `*`, `+`, `?` stay closure steps for the evaluator. Fresh variables are numbered once per query and skip every
  name the query uses. A path that expands to more than 1,024 arms (`(:p|^:p)/(:p|^:p)/...` doubles per hop) or
  16,384 steps is refused at parse time with "property path ... expands to more than 1024 alternatives", counted
  before any arm is built. Path queries are still refused when compiled until the evaluator lands.
- **Property-path evaluator over one window (`cqels-core`).** `PathExtension` extends partial bindings with
  the answers of a `PathPlan` over a `WindowTripleIndex`, per SPARQL 1.1 §18.4: sequences and alternatives are bags
  (arms that start with the same steps share them), and `*`, `+`, `?` are sets found by a level-synchronous frontier
  whose visited set starts empty, so `:a :p+ :a` holds only on a cycle and CYCLE `?x :p* ?y` is 4 rows, not 6. A
  bound start walks forward, a bound end backward, both bound search from both ends; the zero-length pairs of
  `?x :p* ?y` cover every window term, literals included. Over its work budget an evaluation throws
  `PathBudgetExceededException` rather than truncating. The closure memo (frontiers kept while a closure's edges
  are unchanged, at most 4096 starts per direction) is off by default; a memo hit is charged the work it saved, so
  the budget throws the same with it on or off. Not wired yet: path queries are still refused when compiled.
- **CQELS-QL parses SPARQL 1.1 property paths.** The `predicate` position accepts `/`, `|`, `^`, `*`, `+`,
  `?`, `!(...)` and grouping at the spec's precedence, and a variable is one `VAR_REF` token, so `:p? ?y` reads the
  `?` as the path modifier. Paths are not evaluated yet: the compiler, CEP registration and the MCP validate tool
  refuse such a query with "property paths are not executable yet" instead of the earlier syntax error (the
  lenient parser used to drop the operator and match a different pattern). Every other query parses to the same
  definition as before, checked over the 3,842 queries the cqels-core test suite and the `.cqels` files parse:
  `VAR_REF` admits exactly the variable names the old `?`/`$` plus identifier form did.
- **FILTER compares QUDT quantity literals in a common unit.** A literal typed with a QUDT unit
  (`"5"^^<http://qudt.org/vocab/unit/KiloW>`) is converted to the SI base unit of its dimension before `= != < > <= >=`,
  so `5 kW < 200 W` is false and `1 km = 1000 m` is true (temperature offsets included: `212 °F = 100 °C`). Operands of
  different dimensions, a quantity against a plain number, a unit outside the built-in table and an ill-formed magnitude
  are a type error (`null`, the row is excluded) instead of a numeric or label compare of the bare magnitudes. Slice 1
  of the accepted design note `engine-units-provenance-context-actuation-431.md`; arithmetic on quantities and the
  `cqels-rs` mirror are not part of it. Pinned by `ExpressionEvaluatorQudtUnitComparisonTest`.
- **Multi-query replication over N engine instances (T3-1 slice 1).** `ShardedDeployment` (cqels-engine)
  places each registered CQELS-QL query on exactly one of N `CQELSEngine` instances by
  `shardOf(queryKey, n) = floorMod(queryKey.hashCode(), n)`, a pure function N processes can share, feeds
  `push(stream, element)` to every instance holding a query that reads the stream, and delivers each query's answers
  to its own listener. Queries are never split across instances; intra-query sharding stays out of scope until its
  S3.3/S3.4 gauges exist.
- **cqels-reasoning-rete: opt-in inert-rule pruning for registered query predicates (G1).**
  `ReasoningConfig.Builder.queryPredicates(Set<IRI>)` makes `ReteNetwork.compile` build the network only from the
  rules that can derive, directly or through other rules, a fact of one of those predicates
  (`RuleSet.pruneInert`: reachability over the rule-dependency graph at predicate granularity; a variable
  predicate in a body demands every rule, one in a head can produce any predicate). Query answers over the
  given predicates are unchanged; inferences of other predicates that only a pruned rule would have made are no
  longer emitted. Off by default (every rule is instantiated). Head-reachability, not data-inertness, so no
  latency claim. G2 (squared chain rules) and G3 (demand views) of that issue are parked. Pinned by
  `InertRulePruningTest`.
- **cqels-mcp bounds on long-lived servers.** `recall_memory(entail: true)` reuses the entailment
  closure between writes: it is memoized on a mutation generation that a `SailChangedListener` bumps on every commit
  that changes the RDF store (tool writes and the embedding application's alike), one entry, recomputed on the first
  recall after a write or a `register_reasoning`; a governed recall still computes its own, and a repository that
  does not notify recomputes every call as before. The stream-query registration cap (1000) is configurable
  (`streamQueryMaxRegistrations`, `CQELS_MCP_STREAM_QUERY_MAX_REGISTRATIONS`); a refusal names the setting and is
  counted as `cqels_mcp_stream_query_registrations_refused_total`. Opt-in, off by default: idle stream-query
  eviction (`streamQueryIdleTimeoutSeconds`, `CQELS_MCP_STREAM_QUERY_IDLE_TIMEOUT` — a registration not drained
  within it is released as `forget_stream_query` releases it, counted as
  `cqels_mcp_stream_query_idle_evictions_total`) and retention of the episodic and decision-lineage graphs by age
  and/or count (`episodicRetention` / `decisionRetention`, `CQELS_MCP_RETENTION_{EPISODIC,DECISIONS}_MAX_{AGE,COUNT}`),
  both applied by a periodic maintenance sweep. Documented in `MCP_MEMORY_PROFILE.md`.
- **CEP: a `NOT` directly after an optional argument (ADR-0004 M2 (b)).** `SEQ(?a; ?b?; NOT ?c; ?d)`
  and `SEQ(?a; NOT ?b; ?c?; NOT ?d)` were refused at registration (M2 (a), Flink's "a NOT pattern cannot be
  preceded by an optional one"). They now register, and the negation's scope starts at the last argument present
  in the match: at `?b`'s event when `?b` matched, at `?a`'s when it did not — the same open interval A6 already
  defined (`p = prev(j)`). `SEQ(?a; ?b?; NOT ?c; ?d)` over `a c b d` answers `{a, b, d}` only (the `c` lies between
  `a` and `d`, not between `b` and `d`); `SEQ(?a; NOT ?b; ?c?; NOT ?d)` within `W` over `a` answers `{a}` at the
  window close. A `NOT` with no mandatory argument before it (`SEQ(?a?; NOT ?b; ?c)`) is still refused: when
  every argument before it is empty its scope has no start. The direct `Pattern` API (`notNext` /
  `notFollowedBy` after `optional()`, `times(0)`, `timesOrMore(0)`) follows the same rule. The conformance gate
  generates these shapes and adds a bounded-exhaustive family for them
  (`CepReferenceConformanceTest.negationAfterOptionalExhaustive`, 161 280 cases); pinned by `CepNegationAfterOptionalTest`.
- **CQELS-QL SELECT expressions over aggregate aliases.** A non-aggregate `(expr AS ?v)` in a query
  with an aggregate is now evaluated over each aggregated group row, after the aggregates. Example:
  `(SUM(?a) AS ?x) (SUM(?b) AS ?y) (ABS(?x - ?y) AS ?diff)`. This works on every route that folds aggregates,
  including the incremental (on-close) route, so `?v` is projected and can be an `ORDER BY` key. Before this
  change the expression was dropped. Since the earlier fix such a query was refused with a misleading
  "Projected variable ... is not a GROUP BY variable" message; before that fix every row lacked `?v`. The expression may read the
  `GROUP BY` variables, the aggregate aliases and earlier SELECT expression aliases. These shapes are refused at
  registration, with a message that names the alias:
  - an expression that reads any other variable;
  - an expression that reads a later SELECT expression alias;
  - an alias that is already in the group row.

  A SELECT expression in a query with no aggregate is also refused. There it was silently absent from every row;
  `BIND(expr AS ?v)` in `WHERE` is the equivalent. Pinned by `PostAggregationSelectExpressionTest`.
- **`LATENESS` on multi-stream `[RANGE d]` joins (arm (a)).** A two-stream CQELS-QL query
  (`STREAM A {…} STREAM B {…}`) whose `[RANGE d]` window declares `LATENESS l` was refused at execution assembly; the
  budget now becomes the derived temporal join's `maxOutOfOrderness`, holding both sides' watermarks back by `l`. The
  join decides lateness against one joint watermark (the `min` of both sides'), so the larger of the two streams'
  budgets applies to both sides. An arrival within the budget joins with exactly the rows it would have produced on
  time; one beyond it is dropped and reported to `setLateDataDropListener` as before. Without `LATENESS` the derived
  spec is unchanged (zero out-of-orderness). `LATENESS` on a stream of a query driven by an explicit
  `operators.temporalJoin` hint is still refused, now naming the hint's own `maxOutOfOrderness` field. `[SLIDE]`
  (arm (b)) is unchanged. Pinned by `MultiStreamWindowCharacterizationTest` section F2 and
  `CqelsQueryCompilerLatenessOnRangeTest.latenessOnTemporalJoinRange_isAdmittedAtRouting`.
- **Directional windows run a multi-triple join within one stream by default.** A non-aggregate CQELS-QL
  `SELECT` over a `[FUTURE f]` or `[PAST p FUTURE f]` window (with or without `STEP`) whose single `STREAM` block holds
  two or more triple patterns, e.g. `STREAM S { ?s ex:p ?v . ?s ex:q ?w . }`, with optional `FILTER` / `BIND` and
  `ORDER BY` / `LIMIT`, now executes instead of failing with "General directional query execution lands in a later
  milestone". Each closed anchor is one evaluation: its window's elements are joined by the same
  `indexedHashJoinBindings` core the windowed hash join uses, then `BIND` / `FILTER` and the modifiers run on that
  anchor's rows. Rows surface only when an anchor closes; a join held by two overlapping `STEP` anchors surfaces once
  per anchor. This is the replay added opt-in earlier (`executeDirectionalMultiPatternSelect`), now the default route;
  `-Dcqels.directional.multiPatternSelect=false` restores the refusal. Still refused: a directional join across two
  streams, a directional window beside a static graph pattern, `DISTINCT`, `Istream(...)` and multi-triple
  aggregates. Six `RouteSelectionOracleTest` cells move from `X@exe:directional-shape-unsupported` to `DMP`
  (`{FUTURE,PASTFUTURE}|multi|{plain,orderlimit,bindfilter}|nostatic`); pinned by
  `CqelsQueryCompilerDirectionalMultiPatternSelectTest`'s `defaultRoute_*` tests. Multi-triple aggregates have
  since been admitted too (next entry).
- **Directional windows run multi-triple aggregates within one stream by default.** An aggregate over a
  `[FUTURE f]` or `[PAST p FUTURE f]` window whose single `STREAM` block holds two or more triple patterns, e.g.
  `SELECT ?s (COUNT(?w) AS ?n) … STREAM S { ?s ex:p ?v . ?s ex:q ?w . } GROUP BY ?s`, now executes instead of
  failing with "General directional query execution lands in a later milestone". It runs on the same per-anchor
  replay as the multi-triple `SELECT` above: each closed anchor joins its window's elements, runs `BIND` / `FILTER`
  over the joined rows, folds the aggregate over them, then applies `GROUP BY` / `HAVING` / `ORDER BY` / `LIMIT` to
  that anchor's groups. A join row counts in every anchor whose window holds all its triples (overlapping `STEP`
  anchors are not de-duplicated). An anchor whose join is empty emits no row, as on the single-triple directional
  aggregate. A global aggregate with a post-aggregate `SELECT` expression (`(COUNT(*) AS ?n) (?n + 1 AS ?m)`, no
  `GROUP BY`) now emits its row on both this route and the `[RANGE]` windowed hash join, where it emitted nothing.
  `-Dcqels.directional.multiPatternAggregate=false` restores the refusal; the flag is independent of
  `cqels.directional.multiPatternSelect`. Still refused: a directional join across two streams, a directional window
  beside a static graph pattern, `DISTINCT` and `Istream(...)`. Four `RouteSelectionOracleTest` cells move from
  `X@exe:directional-shape-unsupported` to `DMP` (`{FUTURE,PASTFUTURE}|multi|{agg,groupagg}|nostatic`); pinned by
  `CqelsQueryCompilerDirectionalMultiPatternAggregateTest`.

- **`WindowTripleIndex` object index and distinct-term set.** The per-window index now also keeps a predicate+object
  reverse lookup (`getByPredicateAndObject`) and `getDistinctTerms` (subjects and objects, literals included), built on demand
  on first call (the constructor is unchanged, so existing callers pay nothing); additive.
- **Opt-in protection of target-defining triples in the SHACL repair search.** Focus derives over the repaired
  graph, so deleting a node's last targeting triple un-targets it and vacuously "repairs" its violations.
  `ShaclStreamSolveConfig.Builder.protectTargetingTriples(true)` (default `false`, behaviour unchanged) adds integrity
  constraints to the repair program so `repair_del` may not remove `rdf:type C` for an `sh:targetClass C` shape, or any
  triple with predicate `P` for `sh:targetSubjectsOf P` / `sh:targetObjectsOf P`. The protection is per triple, not per
  node: a `P` triple is protected even when the node has another one, so a violation whose only repair is such a deletion
  now reports `SEARCH_BOUNDED`. `ShaclRepairProtectTargetingTriplesTest` pins the default and the opt-in for each target kind.

- **Opt-in bound on the CEP matches one input emits.** One input can decide a very large number of
  matches: the fan-out `SEQ(?h; ?b? AS o0; …; ?b? AS o254)` over 999 `H` decides 254 745 on the one `B`. The new
  system property `cqels.cep.maxMatchesPerArrival` (read at registration; `0` or less, the default, leaves it
  unbounded) and the constructor `NFAPatternProcessor(pattern, stateExplosionLimit, maxMatchesPerArrival)` bound
  the matches one input (a data arrival, a watermark, or end of stream) emits: the first `N` in the matcher's
  emission order are emitted, the rest are dropped before they are built, the last match emitted for that input
  reports how many (`PatternMatch.getDropped()`), and every dropped match is counted on the process-wide
  `cep.matches.dropped` meter (`NFAPatternProcessor.matchesDropped()`) with a rate-limited warning, like
  `cep.partials.evicted`. Dropped matches are lost matches. **The default is unchanged**: with the bound unset every
  match is emitted as before. Measured on the issue's compiled shape at 100 `H` (25 500 matches on the `B`): about
  316 bytes are allocated per match, about 236 of them by the matcher's runs (each match is also a live run, bounded
  by the state-explosion limit) and about 80 by the emitted `PatternMatch`/`CepMatch`; with a bound of 1 000 the
  `B` allocates about 5.7 MB instead of 8.1 MB, and the subscriber (for MCP, the rendered rows) receives 1 000
  matches instead of 25 500. Pinned by `NFAPatternProcessorMatchLimitTest`.
- **CypherQL cross-stream join, static-graph enrichment — third slice.** A two-stream join (below) may
  now carry `MATCH GRAPH "g" { … }` groups over a declared `FROM GRAPH` / `FROM STATIC` graph, e.g.
  `GRAPH "http://ex/users" { (user)-[:MEMBER_OF]->(team:Team) }` beside the `Clicks` / `Purchases` groups. The
  streams join exactly as on the non-aggregate join route; each joined pair is then ENRICHED, after the join, by
  the static-lookup operator the single-stream route uses (same graph scoping, `WITH DEPTH`, `CACHE`, and
  engine cache invalidation). The lookup runs when the pair is emitted, so it reads the static graph as of the
  join, and it is rooted at every ANCHOR — a variable the GRAPH groups share with either stream side. The GRAPH
  groups are matched starting from the joined binding, over both members' triples followed by the static facts:
  a joined variable they name constrains them, a new variable they bind (and the static properties of their
  nodes) is added to the row, and a key the joined row already has keeps the stream value. It is an inner join:
  a pair the GRAPH groups do not match emits no row, and N matches emit N rows. Still refused at registration,
  each with its own message: GRAPH groups sharing no variable with either side (a cartesian product), GRAPH
  groups without a declared static graph or without a static repository, a declared static graph with no GRAPH
  group, an aggregate over an enriched join, and — unchanged — an unsourced group beside the two streams (the
  static pattern must be a GRAPH group) and two streams that share a variable only through the static graph (the
  join key is what the two stream sides share). Pinned by `CypherQLCrossStreamStaticEnrichmentTest` and
  `CypherCrossStreamJoinLiveStreamTest`. Still to come: N-way joins, aggregates over an enriched join, and
  the same-window UNION rule.

- **CypherQL cross-stream join, aggregates — second slice.** An aggregate over the two-stream join of the
  first slice (below) now runs: JOIN FIRST, THEN BATCH THE JOINED ROWS. The sides meet in the same interval join
  (same keys, bounds and rows); each joined row is an event-time record at its LATER member's timestamp (the
  arrival that produced it), and the records are windowed by the **LEFT** stream's declared window with the
  operators the single-stream aggregate route uses — `[RANGE d]` tumbling, `[SLIDE d STEP s]` sliding — so
  `GROUP BY user RETURN user, COUNT(click) AS clicks` folds one row per group per closed window, and a pair whose
  click and purchase straddle a window boundary is counted in the window of the later one. `WHERE` filters each
  joined row; `GROUP BY`, `HAVING` and `ORDER BY` / `LIMIT` apply per evaluation, as on the single-stream windowed
  aggregate route. A window `[a, e)` closes on the JOIN's watermark — `min` of the two sides' — i.e. once
  BOTH streams have seen an element at or after `e` (the single-stream route closes on its one stream's first such
  element), so no pair can still arrive for it; source completion flushes the rest. The join's watermark reaches
  the window through a new `IntervalJoinOperator.eventTimeJoinWithWatermarks` overload that emits it, in order with
  the rows, each time it advances; the join itself is unchanged. CypherQL does not group implicitly, and over a join
  `COUNT(click)` counts joined PAIRS (`COUNT(DISTINCT click)` counts clicks). Still refused at registration: an
  aggregate over a join whose LEFT stream is `[NOW]` (no evaluation to fold within), `DISTINCT` on the aggregate
  route, `GROUP BY` / `HAVING` without an aggregate, and `DISTINCT` / `ORDER BY` / `LIMIT` on the non-aggregate join
  (it emits per arrival and has no evaluation boundary). Examples 6 and 20 of `example-continuous-cypher-queries.cql`
  now run and are no longer annotated as unsupported: example 6's `[:BY]` is renamed `[:BY_USER]` (`BY` is a
  CypherQL keyword) and both gain an explicit `GROUP BY`. Pinned by `CypherQLCrossStreamAggregateTest` and
  `IntervalJoinOperatorTest`. Still to come: N-way joins, static-graph enrichment of a join, and the
  same-window UNION rule.

- **CypherQL cross-stream join, without aggregates — first slice.** A `MATCH` whose groups read two
  declared streams and share at least one variable — `MATCH STREAM Clicks { (click:Click)-[:BY_USER]->(user:User) },
  STREAM Purchases { (purchase:Purchase)-[:BY_USER]->(user) }` — now joins them instead of being refused at
  registration. Each side's element is matched against that side's groups; every enumerated match is its own keyed
  row at the element's timestamp; the two row streams meet in the shared event-time interval join on the terms of
  ALL shared variables. A pair joins when the later member arrives while the earlier is still in its own stream's
  window (`tL - rightWindow <= tR <= tL + leftWindow`, inclusive; `[NOW]` is zero, `[RANGE d]` / `[SLIDE d STEP s]`
  is `d`, the bounds CQELS-QL's two-stream join derives), and is emitted once, then; a record behind both sides'
  watermark is late and dropped. `WHERE` applies to the joined row, which carries both sides' variables and node
  properties (a property both elements supply keeps the left side's value). Both named streams are subscribed as
  soon as the result is; a declared stream the `MATCH` does not name is still never subscribed, and the engine now
  pins both joined streams against `removeStream()` (`CompiledCypherQuery.matchedStreamName()` is replaced by
  `matchedStreamNames()`). Still refused at registration, each with its own message: two sources sharing no
  variable (a cartesian product), three or more sources, an unsourced group beside them, aggregates / `GROUP BY` /
  `HAVING`, `DISTINCT` / `ORDER BY` / `LIMIT`, a static graph (`FROM STATIC` / `FROM GRAPH` / `GRAPH` group), and a
  `[TRIPLES n]` window on a joined stream. Examples 6 and 20 of `example-continuous-cypher-queries.cql` aggregate,
  so they stay annotated as not supported (example 6 also uses `BY`, a CypherQL keyword, as a relationship type).
  Pinned by `CypherQLCrossStreamJoinTest`, `CypherCrossStreamJoinLiveStreamTest` and
  `CQELSEngineRemoveStreamTest`.
- **`LATENESS` on `[SLIDE d STEP s]` windows (arm (b)).** `WindowSpec.requireSparqlExecutable` no longer
  refuses `LATENESS` on `SLIDE` (or `[RANGE d STEP s]`, which normalizes to it). It is honoured on two routes. On a
  single-stream `[SLIDE d STEP s EMIT ON_CLOSE]` aggregate the held-back watermark built earlier for `[RANGE d]`
  applies. Each sliding window closes on its own once an element later than its end + `l` has been seen, so a late
  element lands in every one of its windows still open, and within the budget the rows equal the on-time run's. On a
  multi-stream join over `SLIDE` streams the budget becomes `maxOutOfOrderness`, as for `RANGE` in arm (a). Off those
  routes (a per-arrival `SLIDE` aggregate without `EMIT ON_CLOSE`, a non-aggregate query) `LATENESS` is still
  refused at execution assembly, and the refusal now names the window type. `[NOW]` and `[TRIPLES n]` stay outside
  `LATENESS` by design: they have no watermark to hold back. Without `LATENESS` nothing changes.
- **Static-side spatial lookup join (cqels/CQELS4J#97).** A registered CQELS-QL query on the per-arrival
  stream+static lookup route (`[NOW]` and the other shapes `executeWithLookupJoin` serves) with a top-level FILTER
  conjunct `geof:distance(?s, ?t, unit) <= r` (`<`, or mirrored; `uom:metre`, `uom:degree`, `uom:radian`) or a
  non-disjoint `geof:sf*` / `eh*` / `rcc8*` relation, where `?s` is stream-bound and `?t` is the object of a static
  pattern `?x <p> ?t`, now probes a `GeoSpatialIndex` over the static `?x <p> ?t` triples (inferred statements included, as in the forward lookup) with `?s`'s envelope
  (expanded by `r` on the sphere) and matches the static BGP once per candidate, instead of enumerating every
  static solution and filtering. The FILTER still runs exactly on every row, so the answer is unchanged. Case G6 of
  the issue at 10 000 POIs: about 3 ms per element instead of about 210 ms. The index is rebuilt on the first probe
  after any live commit the repository's `NotifyingSail` reports (not on the CACHE refresh TTL); a malformed static
  geometry is an always-candidate value, not a query failure; on a repository that does not notify, or with
  `-Dcqels.lookup.spatialIndex=false`, the static side is scanned as before. Not used with aggregates, `GROUP BY`,
  `UNION`, `OPTIONAL`, `NOT EXISTS`, `MINUS`, a variable static predicate, or on the composed (windowed) and
  temporal routes. Counters: `CompiledQuery.getStaticSpatialLookupStats()`; an INFO line logs the first build.
  Pinned by `StaticSpatialLookupJoinTest` and `GeodesicEnvelopeTest`.
- **CQELS-QL `FILTER` inside `FILTER NOT EXISTS`, and `FILTER EXISTS` (CQELS4J#92).** An inner `FILTER` in a
  `FILTER NOT EXISTS { … }` body was refused at registration. It is now evaluated per candidate solution of the
  block's patterns, on the outer row extended by that solution, so it reads variables bound outside the block, as
  SPARQL 1.1 §8.1 requires: `FILTER NOT EXISTS { ?zone ex:hostile ?h . ?h ex:threat ?lvl . FILTER(?lvl > ?hp) }` with
  `?hp` from the stream, or the same with `geof:distance(?pt, ?hp2, uom:metre) <= 25`. `FILTER EXISTS { … }`, which
  did not parse, is the same probe with the opposite polarity. The other constructs inside the block are still refused.
  On the stream-only routes a block's stream-bound join keys and the outer variables its inner FILTERs read are now
  carried into the row and stripped at the tail, so a block whose key is not projected answers instead of being
  refused at registration (`NotExistsStreamOnlyProjectionTest`, `AggregateFoldAfterReshaperTest`,
  `CqelsQLStaticBgpCompleteMatchTest` and `RouteSelectionOracleTest` updated); an unprojected `BIND` target is still
  refused. Pinned by `SubBlockFilterOuterScopeTest` (cqels-engine) and `DroppedNotExistsAndAggregateClauseTest`.

- **CQELS-QL per-aggregate `DISTINCT`.** `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` and `GROUP_CONCAT` now accept an
  optional `DISTINCT` after the opening parenthesis, as SPARQL 1.1 section 18.5.1 writes it: `COUNT(DISTINCT ?x)`,
  `SUM(DISTINCT ?x)`, `GROUP_CONCAT(DISTINCT ?x; SEPARATOR=",")`. Each distinct value of the argument within a
  group's window is folded once; `MIN` and `MAX` answer as they do without it. Before, the grammar rejected the keyword
  with a syntax error. The flag is carried on the aggregate itself (`AggregateSpec.isDistinct()`), so it is separate
  from the query-level `SELECT DISTINCT`, and a `HAVING` aggregate keeps its own flag, so `COUNT(?x)` in `SELECT` and
  `COUNT(DISTINCT ?x)` in `HAVING` are two aggregates. It works on every route a CQELS-QL aggregate runs on: the
  incremental windowed-aggregate fold dedupes the window's payloads before folding, and the batch fold (windowed hash
  join, `[TRIPLES n]`, `[NOW]` with `GROUP BY`) deduplicates the group's values through `AggregateComputation`.
  `COUNT(DISTINCT *)` parses only to be refused at registration with a message that names it, because the aggregate
  fold sees only the projected variables, not whole solutions; name the variable instead. CypherQL has its own grammar
  and is unchanged. Pinned by `PerAggregateDistinctTest`; the old pin that the grammar admits neither form is rewritten
  as `GlobalCountStarWindowMatrixTest.countDistinctStarIsRefusedAndCountDistinctVariableParses_closesPerAggregateDistinct`.
- **`EMIT ON_CLOSE` / `EMIT ON_UPDATE` on a legacy window choose an aggregate's emission model.**
  A single-stream aggregate over one stream triple pattern with no static graph pattern was emitted on close
  under `[RANGE d]` and per arrival under `[SLIDE d STEP s]`, `[RANGE d STEP s]` and `[TRIPLES n]`, and only the
  window keyword said which; `EMIT` on a legacy window parsed but was refused at registration. Now the clause
  states the model and the keyword's choice applies only when it is absent (ADR-0003 D7): `EMIT ON_CLOSE` runs
  the incremental windowed-aggregate path, one row per window (per group) when the window closes, and on a
  `SLIDE` it folds one window of `d` every `STEP` — the one legacy shape on which the `STEP` is read — and on a
  `[TRIPLES n]` one tumbling window of `n` ARRIVALS, closed by its `n`-th arrival (an element is one slot
  however many triples it carries, and one the pattern rejects takes a slot too; a window that never fills
  emits nothing, even at source completion); `EMIT ON_UPDATE` runs the windowed hash join, one evaluation of
  the window's current contents per arrival. Over arrivals at 0, 300, 1100, 1500 and 2600 ms, `COUNT(*)` emits
  3 rows under `[RANGE 1s]`, 5 under `[RANGE 1s EMIT ON_UPDATE]`, 5 under `[SLIDE 2s STEP 1s]`, 4 under
  `[SLIDE 2s STEP 1s EMIT ON_CLOSE]`, 5 under `[TRIPLES 2]` and 2 under `[TRIPLES 2 EMIT ON_CLOSE]`.
  Still refused at registration: `EMIT EARLY_AND_FINAL` on a legacy window (the directional windows' model of
  running rows plus a correcting final row, which a legacy window has no open-window phase for), and `EMIT` on
  any query outside that population (non-aggregate, multi-pattern, static-bearing, multi-stream); `[NOW]` takes
  no `EMIT` in the grammar. An
  `EMIT ON_CLOSE` whose aggregate shape the on-close path declines (`HAVING`, a reshaper, more than one
  aggregate, `-Dcqels.aggregates.incremental=false`) is refused when the query is routed instead of running per
  arrival. Pinned by 40 new `RouteSelectionOracleTest` cells (route and emission count per window, pattern,
  grouping and clause), its count-window tests (`triplesOnCloseIsATumblingCountWindowOfArrivals`,
  `triplesOnCloseCountsArrivalsThePatternRejects`) and its population-edge and opt-out tests; the 294 existing
  cells are unchanged.
  `CqelsQueryCompilerDirectionalOnUpdateTest.onUpdate_legacyRangeAcceptedAndRunsPerArrival_closes841` and
  `CqelsQueryCompilerDirectionalGroupedEmitTest.legacyRange_groupedPerUpdateAccepted_closes841` replace the two
  pins that asserted the refusal.
- **`EMIT ON_CLOSE` / `EMIT ON_UPDATE` also choose the emission model of multi-pattern and static-bearing
  aggregates on legacy windows (ADR-0003 D7 amended 2026-10-31).** A single-stream aggregate over several
  stream triple patterns, or beside a static graph pattern, runs on the windowed hash join and is evaluated per
  arrival under every legacy keyword, bare `[RANGE d]` included, so it differed in model from its single-pattern
  twin one pattern apart, and `EMIT` was refused on it at registration (which also kept a BI-2-style two-period
  comparison from adding its static tag-class lookup). `EMIT ON_UPDATE` now names that default and
  `EMIT ON_CLOSE` evaluates the same route once per closed window, over exactly that window's arrivals, on the
  windows the single-pattern on-close route closes (`RANGE d` tumbling, `SLIDE d STEP s` on the `STEP` grid with
  the last time window flushed by completion, `TRIPLES n` tumbling windows of `n` arrivals); the fold, static
  lookup, in-batch reshapers, `HAVING` and per-window `ORDER BY` / `LIMIT` are the per-arrival evaluation's.
  Without `EMIT` nothing moves. `EMIT EARLY_AND_FINAL` on a legacy window stays refused, and so does `EMIT` on a
  non-aggregate or multi-stream query. Pinned by 48 new `RouteSelectionOracleTest` cells
  (`emitClauseSelectsTheEmissionModelOfComposedAggregates`, with `composedDefaultIsPerArrivalUnderEveryKeyword`
  for the unchanged default), `RouteBehaviourMatrixTest.a6c_composedAggregateOnClose_oneFoldPerClosedWindow` /
  `a6d_composedAggregateOnUpdate_runningRowPerArrival`, and `TwoPeriodComparisonStaticLookupTest`; the four
  population-edge cells that asserted the refusal now assert `EARLY_AND_FINAL`, `LATENESS` and non-aggregate
  refusals instead.

- **Plugins can ask the host which statements a role may see.** `cqels-plugin-spi` gains
  `StatementVisibility` (`enforcementActive`, `labelsOf`, `visible`) and `PluginContext`, handed to plugins
  through a new default `CqelsPlugin.initialize(engine, config, context)` that delegates to the two-argument
  form, so existing plugins compile and run unchanged. The MCP host fills it with an adapter over its
  existing access-policy checks; the seam adds no governance semantics. `snapshot()` returns an immutable view
  of the policy and labels for callers that judge many statements (stale until re-taken). A context without a
  seam, or a host that calls only the two-argument `initialize`, means the plugin must deny while a policy may
  be active.

- **`cqels-triegs` is a released module (`cqels-triegs`, `cqels-engine`).** Factorized incremental
  view maintenance for windowed joins: a conjunctive query over edge relations is maintained under epochs of
  edge inserts and deletes by a maintainer picked at registration. The API of `org.cqels.triegs.model`,
  `.maintain`, `.window`, `.dictionary` and `.aggregate` is frozen from this release; every other package
  is internal, and the module README lists the internal members of the frozen types. `cqels-engine` now
  depends on the module but routes no query to it yet. The built-in value dictionaries are unbounded. `TrieGsWindowAdapter`
  assumes the maintainer accepts every batch it adapts: after a rejected batch, discard the adapter.

- **`TrieGsStatementAdapter` feeds RDF statement windows to a TrieGS maintainer (`cqels-triegs`).**
  One window delta becomes one epoch batch: an insert per statement of each added element, then a delete per
  statement of each removed element. A statement carried twice in the window counts twice, as alpha.22's
  join counts it, so a maintainer it feeds must use `MULTIGRAPH` (`semantics()`). RDF terms map to
  type-prefixed `ValueId`s: a literal's label is length-prefixed and its language tag replaced by a
  per-code-point case key, so two terms share an id exactly when RDF4J's `Value.equals` holds, as in
  alpha.22. An ASCII tag (every valid BCP 47 tag) is keyed by its ASCII lower case on every JDK; a non-ASCII
  tag follows the running JDK's case tables. Literals whose cached `CoreDatatype` disagrees with their
  datatype IRI are out of scope. `value(ValueId)` decodes an id back to an equal term and refuses any id
  `valueId` would not produce. The frozen signatures of `org.cqels.triegs.window` may now also name
  cqels-core's stream and windowing types and RDF4J's model types. No query is routed to TrieGS yet.

- **Dropped late elements are observable through the engine API
  (`cqels-engine`, `cqels-core`).** A new `CQELSEngine.registerCqelsQuery`
  overload takes `CqelsQueryOptions`; `CqelsQueryOptions.defaults().onLateDrop(l)`
  calls `l` once per element the query drops as late, with a `LateDrop` carrying
  the query id, the element and the watermark it was late against (without
  `LATENESS`, the largest event time seen). `CQELSEngine.lateDropCount(queryId)`
  returns the per-query count of every `registerCqelsQuery` registration, with or
  without a listener. Covered: the incremental `[RANGE]` aggregate (strictly
  earlier without `LATENESS`, beyond the budget with it), `[SLIDE]` and composed
  aggregates under `EMIT ON_CLOSE`, the two-stream join and, reported for the
  first time, the `FUTURE` / centered windows. A listener that throws is logged
  and the query continues. Which elements are dropped is unchanged, and so is the
  two-argument `registerCqelsQuery`. Embedders of the compiler get the watermark
  through `CompiledQuery.setLateDataDropObserver`. Not covered yet: CEP, CypherQL
  and the operators reached only through `registerQuery(ContinuousQuery)`.

- **A millisecond duration unit in CQELS-QL: `ms`.** `[RANGE 1500ms]`, `LATENESS 500ms`, `STEP 250ms` and
  the other duration sites (`SLIDE`, `PAST`, `FUTURE`, `CACHE`) now parse; `5m` is still five minutes and `5ms` five
  milliseconds, and names that start with `m` or `ms` (streams, prefixes, variables, functions) are unaffected.
  Before, the smallest expressible tolerance was `1s`. `WindowSpec.toString()` now prints a sub-second duration in
  `ms` instead of truncating it to whole seconds.

- **Advance a stream's event time without an element: `DataStream.advanceTo(long)` (`cqels-engine`,
  `cqels-core`).** The last window of a stream that goes quiet can now be closed without pushing a heartbeat
  element or completing the stream. `advanceTo(t)` closes every event-time window of every CQELS-QL query on
  the stream exactly as an element at `t` would — `[RANGE w]` window `[a, a+w)` at `t = a+w` (also with
  `LATENESS 0s`), with `LATENESS d`, `d > 0`, at `a+w+d+1` — on the incremental aggregate, directional
  windows, the `EMIT ON_CLOSE` composed aggregates, directional CEP, and the two-stream join, whose watermark
  is the minimum of both streams (an advance on one stream never moves it past the other's; on the lagging
  stream it can raise it up to the other's, making later arrivals behind it late on either stream); on
  event-time CEP it releases the reorder buffer and hands the matcher the reorder watermark, deciding a
  trailing `NOT` on a quiet stream — "as an element at `t`" there means the same watermark movement, but the
  matcher always hears of an advance (never of a data-derived watermark), so even an advance below the newest
  element's time can finalize an expired run earlier than an element would (safe: no admissible later record
  can extend it). Below the newest element's time it closes no window elsewhere. It is not an element: no row,
  not in the replay history a later query receives, not journaled, not counted by `[TRIPLES n]`, never matched
  by a pattern, invisible to per-arrival routes, to `DataStream.subscribe` and to
  `ReactiveStreamEngine.getStream`. It is monotone (a value at or below the highest value already claimed,
  delivered or not, is a no-op returning `false`; a refused value, e.g. into a full pre-start buffer, stays
  claimed) and ignored after `complete()`. It is ordered with pushes through the stream's own sink, and
  through stream processors as long as they forward it and keep order (the new `StreamProcessor` contract;
  every bundled processor does; one that moves elements to another thread keeps only the order it gives them).
  CypherQL and other non-CQELS-QL queries do not receive it (`ContinuousQuery.acceptsPunctuation()`, default
  `false`). `advanceCepWatermark` stays as the per-query form. The empty-element heartbeat keeps working.

### Fixed

- **`GraphPatternJoinOperator` emits each complete match once instead of once per edge order**
  (`cqels-core`). `findCompleteMatches` tried every unmatched pattern edge at every level, so a match
  was reached by each order its remaining edges could be matched in: a triangle over three distinct
  nodes gave 6 rows (each rotation twice), and a k-edge pattern could repeat a match up to (k-1)!
  times. It now extends only the lowest-index unmatched edge that has a bound endpoint, so each
  rotation is emitted once (a triangle gives 3 rows, a 4-cycle 4, a path 1). A pattern edge whose
  endpoints are both already bound now enumerates every matching stream edge between them, so
  parallel edges (several predicates, or a wildcard type) still each yield their own match.

- **`WindowDeltaOperator` no longer silently drops processing-time timer firings past 256 (`cqels-core`).**
  Firings went into a multicast sink with a 256-entry buffer. A synchronous input runs its whole push before the
  operator's merge subscribes the timer side, and a downstream that withholds demand backs firings up into the same
  buffer, so firings past 256 failed and `TimerService` discarded the `EmitResult`, losing the window emissions they
  trigger. The sink is now unicast with an unbounded buffer, bounded by the armed processing-time timers, and
  `TimerService` checks every emit: it retries `FAIL_NON_SERIALIZED` and errors the operator's output on any other
  failure before the subscription ends. Completing the timer sink when the input completes or errors is
  retried the same way, so an input that completes while the timer thread is mid-firing can no longer leave the
  output unterminated.
- **A declared `resultMapping` now decides the Java value of every value, literal or resource, at the row-value
  write point (slice 1).** Only the flat route and the static-solution writers applied a mapping to a
  literal; the composed, stream-only and directional projections, `BIND` targets (top-level and OPTIONAL-local),
  `GROUP BY` keys, expression-referenced and seeded bindings and the temporal-join matcher bindings applied it to an
  IRI or blank node only, so `"string"` over `"30"^^xsd:int` emitted `"30"` on one route and `Long 30` on another.
  `rowValueForVariable` and `bindSolutionValue` are now one rule. A `FILTER`/`BIND` reading such a variable on the
  composed route now reads the mapped value, as on the flat route since an earlier fix. Not in this slice, still tracked on
  the same issue: ORDER-BY-only sort keys, aggregate aliases, aggregate arguments (kept on the plain value on both folds),
  the incremental-aggregate route, the temporal-join dispatch rows,
  and which value the expression readers see. Pinned by
  `ResultMappingPresentationInventoryTest.aMappedValueIsTheSameJavaValueOnEveryRoute` and the `..._everyValue` cells.
- **A fully-open static pattern that shares no variable now answers every statement in scope.** On
  the lookup routes, a static pattern whose subject, predicate and object are all unbound variables,
  `?x ?pp ?o`, never asked the candidate fallback, so it answered only what a rooted forward view
  supplied: on the per-arrival route of a query with no join key, the facts of the element's first subject
  (so over a multi-statement element the rows followed the statement order), and on the composed windowed
  route nothing. SPARQL answers every seed with every matching static triple. When such a pattern shares
  no variable with any other static pattern (nor, being unbound, with the seed), its extent is every
  statement in the static scope, so the fallback now reads exactly that: a new bounded whole-scope read,
  holding at most `cqels.join.crossProductCap` x pattern count + 1 statements, refused over the bound like
  every other fetch, and memoised per arrival on the lookup routes. Over `b ex:label "Lb"`,
  `c ex:label "Lc"` and two `s0` labels, `STREAM S { ?s ex:p ?v . } ?x ?pp ?o` answers 4 rows for
  `(b ex:p 1)` and 8 for `(b ex:p 1) (c ex:p 2)` in either order, on both routes, where it answered
  `[1|b|Lb]`, `[1|b|Lb, 2|b|Lb]` / `[2|c|Lc, 1|c|Lc]` and `[]`. A fully-open pattern that shares a variable
  is not read that way: the pattern it shares the variable with anchors it in the promoted order. Still
  view-only: fully-open patterns that share variables only with one another (`?x ?p ?y . ?y ?q ?z`); the
  registration warning, the spec, the cheat sheet and the `register_stream_query` description now name
  that shape instead. A consequence for the root-heuristic question: a single fully-open pattern no longer discriminates whether
  the LDBC `personId` root heuristic is live (both roots answer the whole scope); the chain still does.
  Pinned by `PerArrivalElementAsWindowTest.lookupRoute_keylessFullyOpenPattern_answersEverySeedWithTheWholeScope`
  (formerly `lookupRoute_keylessViewOnlyPattern_isAnsweredFromTheFirstSubject_residual`, characterized),
  `ComposedUnjoinableStaticBgpTest.fullyVariableStaticPattern_answersTheProductOnBothRoutes` (formerly
  `..._answersOnlyWhatARootedViewSupplies`, characterized), `fullyVariableStaticPattern_overTheBound_isRefusedAtTheFetch`
  and the updated `fullyVariableStaticBgp_isRootedByTheLdbcHeuristic_onTheJsonPathOnly`, and
  `StaticBgpMatcherTest.disconnectedOpenPattern_isReadFromTheWholeScope_inEitherOrder`,
  `connectedOpenPattern_neverReadsTheWholeScope` and `wholeScopeRead_overTheBound_isRefusedAtTheFetch`.

- **`ORDER BY` / `LIMIT` on a single fixed-predicate stream triple under a declared window now rank each
  evaluation (ADR-0002 / ADR-0003 D3).** A single-stream query whose stream side is one fixed-predicate
  triple pattern with no aggregate (`STREAM S { ?s ex:p ?v . }`, with or without a static graph pattern) under
  `[RANGE d]`, `[RANGE d STEP s]`, `[SLIDE d STEP s]` or `[TRIPLES n]` was refused at registration with
  `ORDER BY … LIMIT k` ("would ignore the window it declares"), with a bare `ORDER BY` and with a bare `LIMIT`:
  without the modifiers it runs on a per-arrival route (`executeStreamOnlyWithExpressions` /
  `executeWithLookupJoin`) that keeps no window buffer, so there was no evaluation to rank within. D3 requires
  per-evaluation ranking on every single-stream non-`[NOW]` route, and the earlier change had done it only for the composed
  windowed route. Now the modifier alone sends this shape to the windowed hash join (`executeWindowedHashJoin`, or
  `executeWindowedHashJoinWithLookup` with the static pattern), which buffers and evicts by the declared window,
  and the existing stateless `applyBatchSolutionModifiers` ranks and truncates each evaluation, as it already did
  for the multi-triple shapes. The bare modifiers take the same per-evaluation reading: a bare `ORDER BY` orders
  each evaluation's rows, a bare `LIMIT k` keeps `k` rows of each. Over arrivals `s2000, s0, s1000` under
  `[TRIPLES 1]`, `ORDER BY DESC(?s) LIMIT 1` answers `s2000, s0, s1000`. The registration predicates and
  `execute()`'s dispatch read one predicate (`ranksWithinWindowEvaluations`, a disjunct of
  `needsWindowedJoinEvaluation`), so they cannot disagree. The same query without `ORDER BY` / `LIMIT` keeps its
  per-arrival route (the broader decision is still open), and `[NOW]` keeps the continuous top-K. Pinned by
  `RouteDivergenceProbeTest` P3d (now CONVERGED), `RouteBehaviourMatrixTest` C4,
  `OrderByLimitLivenessTest.windowedTopKOnAPerArrivalShapeRanksEachWindowEvaluation` and the eight
  `{RANGE,RANGE_STEP,SLIDE,TRIPLES}|single|orderlimit|*` cells of `RouteSelectionOracleTest` (`WHJ` / `WHJL`).
  See Upgrade notes.

- **A relative IRI in a triple pattern no longer matches every term.** The AST builder stripped
  `<rel>` to the bare token `rel`, which the matcher classifies as a variable, so `?x ex:p <rel>` matched any object.
  CQELS-QL has no `BASE` declaration, so a relative IRI (`<rel>`, `<>`, `<a/b#c>`) in a subject, predicate or object is
  now refused when the query is parsed, on the CQELS-QL text path and for an angle-bracketed term of a JSON definition,
  with a message naming the absolute-IRI and prefixed-name alternatives. Bare numeric constants (`?x ex:n 3` still
  matches by lexical form) and a literal's relative datatype are unchanged. Pinned by `RelativeIriConstantTest`.

- **On the composed windowed lookup route, the rows of one arrival now share one static work budget
  whatever the static BGP's shape.** The static-BGP work guard (`cqels.join.crossProductCap` x
  static pattern count) is documented as bounding the evaluation, `|window rows|` x `|static solutions|`,
  and the per-arrival lookup route has shared one budget per arrival for every query since the shared budget. On the
  composed route only a static BGP sharing no variable with the stream pattern did; one that shares any
  variable (the predicate position included) started a fresh budget for every window row, so one arrival's
  rows could each spend the whole bound. They now share one counter per arrival. Unchanged by design:
  exceeding the bound on this route still ends the query rather than dropping the arrival, because a row
  already joined against window state cannot be un-joined. The alpha.21 known issue naming the per-row
  budget no longer applies. Pinned by
  `PerArrivalElementAsWindowTest.composedRoute_sharedVariableStaticBudget_isTheArrivals` (formerly
  `composedRoute_keyedStaticBudget_staysPerRow`) and `composedRoute_staticWorkBudget_stillFailsTheQuery`.
- **A static pattern whose only constant is a literal object now gets candidates from the repository.**
  On the lookup routes, an unbound-subject static pattern whose predicate is an unbound variable
  and whose object is a literal-form constant, `?x ?px "Alice"` or `?x ?px 42`, never asked the
  candidate fallback: a literal form names no unique repository term to query by, so nothing in the
  pattern resolved. It answered only what a rooted forward view supplied, which on the composed windowed
  route of an unjoinable static BGP is nothing, so that query answered no rows instead of the cartesian
  product its registration warning describes. The fallback is now asked for the statements whose object
  has that lexical form, whatever its datatype or language tag, which is exactly what the static matcher
  compares a literal constant by, so no triple the matcher rejects is added and none it accepts is missed.
  The alpha.21 known issue naming this shape, and the matching exception in the unjoinable-BGP registration
  warning, the spec and the cheat sheet, no longer apply; an unbound-subject pattern whose predicate and
  object are both unbound variables (`?x ?px ?o`) still got none until the fix above. Pinned by
  `ComposedUnjoinableStaticBgpTest.unjoinablePatternWhoseOnlyConstantIsALiteralObject_answersTheProduct`
  (formerly `..._getsNoCandidates`, characterized), `literalObjectRead_answersExactlyWhatMatchTermAccepts`
  and `literalObjectRead_reachesBeyondTheRoot_onTheFlatAndKeyedRoutes`, and
  `StaticBgpMatcherTest.literalObjectOnlyAnchor_isServedByLexicalForm_inEitherOrder` and
  `literalObjectRead_overTheBound_isRefusedAtTheFetch`.
- **`NOW()` is one instant across the BIND and FILTER stages of one row on the flat static-lookup and composed routes.**
  Each row-at-a-time BIND or FILTER stage rebuilt its `SPARQLResult` from a plain row, so `BIND(NOW() AS ?a) BIND(NOW() AS ?b)`
  there could bind two different instants. A query whose top-level BIND or FILTER calls `NOW()` now carries the arrival's
  (flat route) or window batch's (composed route) instant on the row, across row copies and stage rebuilds, so every stage reads
  it; the next arrival or batch reads the clock again. The carrier is the existing compiler-internal `TermCarryingRow`, stripped
  before the subscriber sees the row, and `NOW()` outside a top-level BIND or FILTER (a `HAVING` already carried it) is unchanged.
  Pinned by `NowStablePerEvaluationTest.twoBindStagesOnTheFlatStaticLookupRouteAgree` and `...OnTheComposedRouteAgree`.
- `StreamElementToStreamEventAdapter.lift` no longer drains its source before the downstream asks, and a downstream
  cancel now reaches a source that is still emitting synchronously. `lift` subscribed the source inside its
  `Flux.create` callback and only wired cancellation once that subscribe returned, so a synchronous source was drained
  into the per-subscriber buffer during subscription, `take(n)` / `next()` / `blockFirst()` on a lifted endless
  synchronous source never returned, and the same on a large finite one pulled it to the end. The source is now
  subscribed on the first downstream request, with the cancel target wired before it. The demand is unchanged: once
  requested, the source is consumed with one unbounded request, because the temporal-join and CEP routes lift shared
  (`share()`d) streams and most of them consume the lift with bounded demand, which would otherwise throttle every
  other consumer of that stream. So the request that first signals demand still drains a synchronous
  source, and an endless one returns from it only if the downstream cancels in-line; the class Javadoc documents this.
  Pinned by the three `StreamElementToStreamEventAdapterTest` cases for that fix.
- **A watermark stream from another thread no longer holds the thread that delivered a data event
  to an `NFAPatternProcessor` subscription**. A subscription's inputs are applied by
  whichever producer finds its channel idle. While a data event was being applied, a second thread
  calling `advanceWatermark` faster than the watermarks were applied kept the data thread applying
  them for as long as they came. If the source was shared, its other subscribers waited with it.
  A thread now applies at most 256 inputs enqueued by other threads, then returns. The rest of the
  queue continues, in the same FIFO order, on `Schedulers.boundedElastic()`. The single bounded
  channel and its admission quotas are unchanged.
  - "Synchronous" now means synchronous for the calling thread's own inputs. A single-threaded
    caller and a synchronous replay still have every input applied, and every result delivered,
    before the emitting call returns. Under cross-thread contention beyond the 256-input budget,
    the remainder of the queue, with its results, is applied and delivered on a scheduler thread.
  - If the scheduler refuses the continuation twice, the subscription is terminated with an
    `IllegalStateException` naming the overload, and a warning is logged. Neither the queued inputs
    nor a producer's thread are left waiting.
  - The bound costs nothing on the single-producer path: one loop now polls the channel, applies
    each input and delivers its matches, replacing the `map` + `flatMapIterable` stages and their
    request traffic. The `NFAPatternBenchmark` rows stay within the benchmark gate of main (table on
    the PR).
  - Measured in `NFAPatternProcessorWatermarkTest.aWatermarkStreamFromAnotherThread_doesNotHoldTheDataProducersThread`:
    before, the data thread was still held after 5 s while about 894 000 watermarks were supplied.
    After, it returned after the budget.
- **A storage directory has one writer; a second backend on it fails to open instead of deleting records.**
  The file-backed journal keeps its committed length in memory, so a second backend on the same directory (two engines
  without a `path`, which all shared `java.io.tmpdir/cqels/<id>`; a restart overlapping the old process; a mirror pointed
  at the primary's path) cut the file back to its own length on its next append, deleting records the other had
  acknowledged. Every provider-built backend now holds an OS file lock on `<path>/backend.lock` from open to `close()`;
  a second open, in the same JVM or another process, fails with `IllegalStateException` naming the directory. A
  `PersistenceCoordinator.create` that fails after opening its primary now closes it, so a retry does not find its own
  directory locked (`BackendDirectoryLockTest`, `PersistenceCoordinatorTest`, the provider tests).
- **Java deserialization of journal payloads is filtered.** A `java-serialized` journal record went to an
  unfiltered `ObjectInputStream`, and the record's type name comes from disk, so anyone able to write the storage
  directory could plant a record that ran a gadget chain or a deserialization bomb at the next recovery (the per-record
  CRC has no key and is no protection). `BinaryCodecRegistry` now reads every `java-serialized` record through an
  `ObjectInputFilter`: JDK value and collection types, RDF4J model values and `org.cqels` types are admitted, plus the
  patterns of the new `cqels.storage.serialFilter` system property; every other class is rejected, the object graph is
  capped at depth 16, and no array length or reference count may exceed the record's byte length. A JVM-wide
  `jdk.serialFilter` can still reject further. Writing a payload whose class the filter rejects now fails the append
  instead of journaling a record a restart could not read (`BinaryCodecRegistryTest`,
  `JournalPayloadSerialFilterTest`).
- **A checkpoint whose window or state-value class the restore would reject now fails at snapshot.**
  `InMemoryStateBackend.snapshot` and `TimerService.snapshot` wrote any `Serializable` class, while the filtered
  restore rejects classes outside the defaults and `cqels.storage.serialFilter`; such a checkpoint succeeded
  and the recovery after a crash failed. The snapshot now checks every class the blob names (each class descriptor
  written, nested values and arrays included) with `BinaryCodecRegistry.serialClassCheck`, the restore filter's
  patterns and JVM-wide `jdk.serialFilter`, and fails with an `IllegalStateException` naming the class and the
  property. In the engine this takes the path a non-`Serializable` state value already took: every periodic
  checkpoint fails (metered, logged once per burst, retried on later elements, the journal still holds every element)
  and the forced checkpoint of `stop()` rethrows after teardown completes, until the class is admitted. Depth and size
  bounds, including the limits of a JVM-wide `jdk.serialFilter`, stay read-side only (`SnapshotSerialFilterTest`,
  `BinaryCodecRegistryTest`).

- **A `java-serialized` record of an incompatible class version no longer reports the deserialization limits.** The
  filtered read labelled every `InvalidClassException` that named no rejected class as "exceeds the
  deserialization limits", so a journal record or checkpoint blob written before a class changed its
  `serialVersionUID` failed with a message blaming depth and size bounds. The limits message now appears only when a
  limit rejected the read; any other `InvalidClassException` is reported with its own message (`local class
  incompatible: ...`) (`BinaryCodecRegistryTest`).

- **Java deserialization of checkpoint blobs is filtered.** `WindowDeltaOperator.restoreState`,
  `TimerService.restore` and `InMemoryStateBackend.restore` read their checkpoint bytes with an unfiltered
  `ObjectInputStream`, so anyone able to write the storage directory could plant a blob that ran a gadget chain or a
  deserialization bomb at restore (the checksum has no key and is no protection). They now read through
  `org.cqels.persistence.SnapshotSerialFilter`, which delegates to `BinaryCodecRegistry.deserializeJavaFiltered`: the
  very filter of the journal payloads (same class patterns including `java.util.concurrent.atomic`/`locks`,
  same `cqels.storage.serialFilter` property, object graph capped at depth 16, no array length or reference count
  above the blob's byte length, same message when a JVM-wide `jdk.serialFilter` rejects). `cqels-core` now depends on
  `cqels-storage-spi` for it. A rejected blob fails the restore with an `IllegalStateException` naming the class and the
  property; window or state-value classes of your own must be admitted through that property
  (`SnapshotSerialFilterTest`).

- **A beyond-budget drop under `LATENESS` is now reported (arm (b)).** On the single-stream on-close aggregate
  route, an element later than its `[RANGE d LATENESS l]` window's budget was dropped by the window operator without
  reaching `setLateDataDropListener` (the earlier change reported only the no-`LATENESS` source drop). The lift now reports an
  element once every window it belongs to has closed, for `RANGE` and `SLIDE` alike. An element still inside one open
  sliding window lands there and is not reported. The rows are unchanged. Pinned by
  `CqelsQueryCompilerLatenessOnRangeTest.latenessOnRange_beyondBudget_isReported` and
  `latenessOnSlide_beyondBudget_isDroppedFromEverySlideAndReportedOnce`.

- The durable stream catalog no longer needs the whole journal in memory.
  `FileBackedEventJournal.readFrom` decoded every committed record into a list before filtering and
  sorting it, so `PersistenceCoordinator.readStreamCatalog()` (`collectList()` then a fold over
  the control records) held every event payload at once, and the `block(timeout)` around it could
  not bound the read: an engine whose journal was larger than its heap could not seed its catalog,
  which it reads at the first cataloged creation, at start when it re-creates streams, and at start
  on an engine with a journal mirror. `readFrom` now streams records from the file on demand, from a
  file and committed length taken together at subscription, and `readStreamCatalog()` folds them
  as they arrive, so its memory is bounded by the catalog (`StreamCatalogBoundedReadTest` folds a
  journal holding more than twice a 64 MiB child JVM's heap; before the fix that child ran out of
  memory). The coordinator's two other whole-journal scans, the gap check for a checkpoint written
  by an older version and the check for an event below a truncation offset, are folds over the
  stream as well; checkpoint recovery's replay still collects its suffix. Corruption handling is
  unchanged: every record is still read, CRC32C-checked and decoded with the same messages, and a
  damaged record still fails the catalog read, the scans and any read consumed to the end; it now
  arrives as the flux's error after the records before it rather than as an exception from the
  `readFrom` call, so a subscriber that stops early no longer sees damage past what it read
  (`FileBackedEventJournalRecoveryIntegrityTest`). `readFrom` no longer sorts: records are written
  in ascending offset order, and a record whose offset is below the one before it now fails the
  read, and a `truncateBefore`, with `CorruptCommittedStateException`; `readFrom` used to reorder it
  silently.

- **A stream catalog read now honours its replay deadline.** `PersistenceCoordinator.readStreamCatalog` ran the
  journal scan synchronously on the caller's thread before `block(replayTimeout)` began counting, so a slow or blocked backend
  could hold a registration past the deadline. The scan now runs on `boundedElastic` and the caller waits with a bounded
  `get`; on expiry the read fails with an `IllegalStateException` carrying a `TimeoutException` cause and the subscription is
  cancelled, with no partial catalog returned (`StreamCatalogDeadlineTest`). The three sibling replay reads follow in a later entry.

- **The recovery replay, the checkpoint gap check and `hasEventBefore` now honour the replay deadline.**
  Like the catalog read, each subscribed to the journal synchronously before `block(replayTimeout)` began counting, so a slow
  or blocked backend could hold recovery, `writeCheckpoint` or `truncateJournalBefore` past the deadline. All four reads now
  share one helper that scans on `boundedElastic` and waits with a bounded `get`; on expiry the read fails with an
  `IllegalStateException` carrying a `TimeoutException` cause (`ReplayReadDeadlineTest`).

- **An `OPTIONAL`'s inner `FILTER` that reads a variable bound outside the block never matched (CQELS4J#91).** The
  outer variable was not in the row the block probed, so the FILTER read it unbound, rejected every candidate, and the
  block never bound: `OPTIONAL { … FILTER(?lvl > ?hp) } FILTER(!BOUND(?h))` admitted every row. Such variables are now
  materialised for the block (fetched on the static-lookup routes, carried on the stream-only ones) and stripped
  before emission. The same carriage fixes an `OPTIONAL` whose join key is bound only by the `STREAM` block and not
  projected on a stream-only `[NOW]` query, which never bound either. Pinned by `SubBlockFilterOuterScopeTest`.

- **A two-stream temporal join with static patterns rooted at different stream-bound variables now
  answers (CQELS4J#90).** `STREAM a { … ex:zone ?zr … } STREAM b { … ex:zone ?zo … }` beside
  `?zr ex:area ?polyR . ?zo ex:area ?polyO .` under `FROM <g>` registered and emitted no row: the
  route's static view is rooted at one join key (`?zr`), its matcher ran without the repository
  candidate fallback the other two lookup routes use, so `?zo`'s pattern never found its fact and every
  joined row was dropped. The temporal route's matcher now has that fallback, read in the same scope as
  its forward lookup (the FROM graphs for plain patterns), so the static patterns are evaluated per
  joined row as on the single-stream route, and a fact outside the FROM graph still matches nothing
  (`TemporalJoinStaticPatternsTest`).
- **`geof:distance` for any geometry pair, `geof:buffer` in metres (CQELS4J#94, CQELS4J#95).** `geof:distance`
  now answers for any two geometries (point, line, polygon, collection) as the minimum distance between their
  boundaries, 0 when they intersect, where RDF4J answered for points only. Distances are great-circle on a sphere of
  radius 6371008.7714 m (about 0.5% from WGS84), not ellipsoidal. `geof:buffer` with `uom:metre` now buffers in
  metres through a local equirectangular projection (accurate at city scale) instead of in planar degrees. Coordinates
  are range-checked (lon [-180, 180], lat [-90, 90], finite) and polar (|lat| above 85) or antimeridian-spanning
  geometries, and a buffer that would cross those limits, are a type error rather than a wrong number.

- `geof:ehEquals` is now the DE-9IM TFFFTFFFT test (JTS `equalsTopo`) instead of `ehInside && ehContains`, which was
  never true for equal regions. Identical points compare equal, and a geometry collection operand is a type error
  (CQELS4J#96).
- `geof:` functions now accept a CRS IRI prefix on `geo:wktLiteral` (GeoSPARQL Req 10/11). EPSG:4326 literals are
  read with lat/lon axis order and swapped to lon/lat; an unknown CRS IRI is a type error and the spatial index no
  longer indexes such literals. `geof:getSRID` still reports the original CRS of the literal.
- The CqelsQL aggregate-keyword names (`?count`, `?avg`, `?sum`, `?min`, `?max` as variables and as `AS` aliases)
  were admitted by the grammar already, but the tests that claimed to pin it parsed with the lenient
  `parseCqelsQL`, which prints ANTLR's "mismatched input" error and recovers, so they passed even with the
  grammar change reverted. They now parse with `parseCqelsQLStrict`, and new cases evaluate each alias on a
  `[RANGE 2s]` windowed aggregate, check all five aliases in one projection, check `(MAX(?max) AS ?min)`, and keep
  `?now` / `?range` as controls (`CqelsQlVariableNameKeywordTest`); with the five tokens removed from `idToken`
  five of them fail. No grammar or runtime change.
- **`cqels://engine/status` now says how many recovered events were delivered and how many were correctly skipped.**
  After a restart, `persistence.appliedPosition` advances through recovered records that were re-emitted to a
  stream and through ones correctly skipped (the stream is absent or being removed, or the record belongs to an
  older stream generation) alike, and the replay log counts every replayed record, so no figure told an operator
  how many recovered events reached a stream's subscribers. The recovery loop now counts each event record at the
  point it emits into the stream's sink (`delivered`) or skips it by rule (`skipped`); a record whose emission
  fails is neither, and control records are not events. `PersistenceHealth` carries the two numbers as
  `recovery().deliveredEvents()` / `skippedEvents()`, and the status resource shows them as
  `persistence.recovery.deliveredEvents` / `skippedEvents`, next to two new scrape counters
  `cqels_mcp_recovery_delivered_events_total` / `cqels_mcp_recovery_skipped_events_total`. Pinned by
  `CQELSEngineIngestBatchTest.restartRecovery_countsDeliveredAndSkippedEvents`,
  `ReactiveStreamEnginePersistenceTest.restartRecovery_countsDeliveredAndSkippedEvents` and
  `KnowledgeGraphResourceProviderTest.engineStatusReportsRecoveryDeliveredAndSkippedEvents`.
- **A CypherQL `FROM STATIC "g"` / `FROM GRAPH "g"` now scopes the static lookup to the declared graphs.** Both
  clauses only switched the static lookup on: the `EnrichedLookupJoinOperator` built for them carried no
  `lookupOverride`, so it read the WHOLE repository and a fact stored only in another graph, or in the null context,
  joined as if it were in `g`. The lookup now reads exactly the declared graphs: every `FROM STATIC` and `FROM GRAPH`
  graph, merged, plus the graph of an explicit `MATCH GRAPH "…" { }` block (the author named it there), for the forward
  lookup and every `WITH DEPTH` hop. An empty graph list never reaches `getStatements`, where an empty varargs means every
  context. A query with neither clause stays stream-only. The matcher runs over one pool of stream and static triples and
  never resolves a group's graph URI, so the declared graphs are one merged dataset, not one dataset per group
  (`CypherQLStaticGraphScopeTest`).

- **A plain static pattern on the per-arrival and composed lookup routes now reads only the graphs the
  query declares with `FROM`.** Both routes read every context of the static repository whatever
  the query declared: their lookup operators had no context-scoped lookup, and the candidate fallback
  (`staticCandidates`, behind the forward view and the `OPTIONAL` / `UNION` / `NOT EXISTS` probes) passed
  no context, so a fact stored only in an undeclared graph, or in the repository's null context, joined a
  query declaring `FROM <g>`. Only an explicit `GRAPH <iri> { ... }` block was scoped. A plain
  pattern now reads the default graph SPARQL 1.1 §13.2 defines: the merge of the `FROM` graphs, on the
  forward lookup, every `WITH DEPTH` hop, the candidate fallback and the reshaper probes alike. `FROM
  NAMED` graphs are not in it, so a query that declares only `FROM NAMED` has an empty default graph and
  its plain patterns match nothing. A query that declares no graph keeps reading the whole repository,
  which is the service's default dataset when none is declared. Explicit `GRAPH` blocks keep their scoping;
  the temporal route is covered by the next entry
  (`FromGraphScopedStaticLookupTest`; `ComposedLookupLiteralJoinKeyTest.factOutsideTheDeclaredGraph_isExcludedOnBothRoutes`
  and `PinnedGraphScopeRegressionTest.plainStaticLookupReadsOnlyTheFromGraph` replace the pins of the
  old behaviour). Test fixtures that declared `FROM` but stored their data in the null context now
  store it in the declared graph, as do the engine pipeline example, the cdsp fleet-enrichment demo and
  `PerArrivalRouteBenchmark`; the tutorial-03 use case, whose data is in the null context, no longer
  declares `FROM STATIC`. The SNB loader and update path store their data in `<http://snb/graph>`, the
  graph the SNB queries declare, and cqels-cdsp's `StaticGraphLoader` stores each configured graph under
  its own URI.

- **On the temporal join route a plain static pattern under `FROM NAMED` alone now matches nothing.**
  That route reads the static repository only through its forward lookup, which read the one
  declared graph, `FROM` or `FROM NAMED`, for every pattern, so `FROM NAMED <g>` with a plain pattern
  read `<g>`, a graph SPARQL 1.1 §13.2 leaves out of the default graph. Plain patterns now read the
  default graph, as on the other two lookup routes; a `GRAPH <g>` block still reads `<g>`, and
  `FROM <g>` is unchanged. The route has no candidate fallback and refuses `OPTIONAL` / `UNION` /
  `FILTER NOT EXISTS`, so it makes no other static read to scope; those facts, and the refusal of a
  static pattern with no declared graph, are pinned beside the fix
  (`FromGraphScopedTemporalLookupTest.fromNamedOnly_plainPatternMatchesNothing` and its siblings).

- **On the temporal join route a plain static pattern beside a `GRAPH <g>` block no longer reads `<g>`
  under `FROM NAMED <g>` alone.** The route flattens its static patterns into one list without
  their block, so its forward lookup read the block's graph for all of them, and the plain pattern
  matched a fact that is only in the named graph. The route admits exactly one declared graph, so the
  two scopes are either the same graph (`FROM <g>`: the default graph is `<g>`) or the default graph is
  empty (`FROM NAMED <g>`), in which case the plain pattern, and with it the conjunction, has no
  solution and nothing is read. A separate default graph beside the named one (`FROM <d> FROM NAMED
  <g>`) stays refused on this route
  (`FromGraphScopedTemporalLookupTest.plainPatternBesideGraphBlock_underFromNamedOnly_doesNotReadTheNamedGraph`,
  `...plainPatternBesideGraphBlock_underFrom_readsTheFromGraph`, `...separateDefaultAndNamedGraph_isRefused`).

- `NOW()` read the wall clock on every call, so two `NOW()` calls in one expression, or the rows of one
  window batch, could carry different timestamps, against SPARQL 1.1 section 17.4.5.1. It now returns one
  instant per evaluation, where an evaluation is one window batch or one arrival's solutions on a
  per-arrival route, and the next batch or arrival reads the clock again. The instant rides on the
  `SPARQLResult` being evaluated, not on the evaluator or a static, so concurrent queries keep independent
  timestamps. `NowStablePerEvaluationTest` pins both, with an injected clock that answers a new instant on
  every read.
- **A declared `resultMapping` now applies to a stream-bound literal on the flat static-lookup route.** A JSON
  query declaring an object-position stream variable as `"string"` still received the datatype-narrowed `Long` for an
  `xsd:integer` object, because the stream-side write took the mapping into account only for an IRI or blank node,
  while the static side (`typedSolutionValue`) applied it to every value; subject and predicate positions already
  agreed with the static side. The flat route (`mapToResultRows`) now writes a stream-bound value through
  `typedSolutionValue` too, so `"string"` over `"30"^^xsd:integer` emits `"30"`, and a query with no declared mapping
  keeps the plain datatype narrowing. Pinned by
  `CqelsQueryCompilerIriRowRepresentationTest.objectPositionNumericWithDeclaredStringMapping_yieldsAString_closesTheObjectPositionGap`
  (the former `..._keepsDatatypeNarrowing_pin`, rewritten), `declaredMapping_streamAndStaticObjectPositionLiteralsAgree`,
  `declaredMapping_streamSubjectAndPredicateAgreeWithTheStaticSide` and
  `ResultMappingCorrelationTermTest.aMappedStreamBoundLiteralPresentsItsMappingOnTheFlatRoute`. Not changed: the composed
  route (a stream window over a range) and the other stream-side writers (`BIND` targets, expression-referenced
  bindings, the temporal-join matcher bindings) still keep the lift for a literal, as characterized in
  `ResultMappingCorrelationTermTest.aMappedStreamBoundLiteralPresentsItsLift_characterizedNotEndorsed`.
- **A `BIND`-mediated correlation now constrains the static pattern instead of fanning it out.**
  `STREAM S { ?s ex:p ?v . } BIND(?s AS ?x) ?x ex:label ?lbl .` relates the two sides only through the `BIND`
  target `?x`, and SPARQL 1.1 §18.2's `Extend` binds it before the later patterns are joined. Join resolution
  scanned triple patterns only, so the compiler called the static pattern unjoinable (and warned), enumerated
  every `ex:label` statement in the graph for every stream row, and let the `BIND` overwrite `?x` afterwards: 15
  rows where 4 are right on the per-arrival route, and three where one is right on the composed windowed route
  (where had turned a silent empty answer into this wrong one). A top-level `BIND` whose expression reads
  only variables the STREAM block binds (or the targets of earlier such `BIND`s) now counts as a stream-side
  variable: its target is seeded into the static probe, and the unjoinable-BGP warning no longer fires for it.
  A `BIND` over a variable only the static side binds still does not correlate. The stage order is unchanged.
  The value seeded into the probe is also the value the row emits: a nondeterministic expression such as
  `BIND(IF(RAND() < 0.5, ex:a, ex:b) AS ?x)` is drawn once, so `?x` and the label probed with it always agree,
  and so do the earlier `BIND` targets it is built from
  (`ComposedUnjoinableStaticBgpTest`: `flatRoute_bindMediatedCorrelation_constrainsTheStaticSideOnTheBindTarget`,
  `bindMediatedCorrelation_correlatesOnTheBindTarget`, which replaces the characterization
  `bindMediatedCorrelation_answersWronglyWhereItUsedToAnswerNothing`,
  `bindOverAStaticSideVariable_doesNotCorrelateAndStaysUnjoinable`,
  `flatRoute_nondeterministicBindTarget_keepsTheProbedValueThroughTheBindStage`,
  `composedRoute_nondeterministicBindTarget_keepsTheProbedValueThroughTheBindStage` and
  `flatRoute_nondeterministicBindChain_keepsTheEarlierTargetsValueToo`).
- **A `BIND` of a mapped variable now correlates by that variable's term.** With a `resultMapping` on `?friend`
  (a `"string"`-mapped IRI), `BIND(?friend AS ?key)` evaluates over the row's presentation, so `?key` got the
  re-minted `xsd:string` literal and a later `OPTIONAL` / `FILTER NOT EXISTS` on `?key` seeded from it: the `OPTIONAL`
  null-filled and the `NOT EXISTS` kept the row, whether or not `?key` was mapped itself (both static-lookup routes;
  top-level on the composed aggregate topology, block-local on both). The `BIND` seam (`bindBindTarget`, at the
  top-level and the `OPTIONAL`-inner call sites) now records the source's term beside the target when the expression
  is a bare variable reference and the source carries one, so the block seeds from the IRI. The target's presentation
  and what any expression reads are unchanged, and `BIND` stays where it ran. Still true: a computed expression
  (`BIND(STR(?friend) AS ?key)`) has a new value of its own and reads the presentation, and whether the evaluator
  should read the term channel is a separate decision. Pinned by
  `ResultMappingCorrelationTermTest.aTopLevelBindTargetOfAMappedSourceCorrelatesByTheSourcesTerm`,
  `aBlockLocalBindTargetOfAMappedSourceCorrelatesByTheSourcesTerm` (both previously characterized as not endorsed)
  and `aComputedBindOverAMappedSourceStillReadsThePresentation`. Only a query with both a mapping and a reshaper is
  affected; every other query keeps byte-identical rows.
- **A row left empty after the WHERE-group helpers are stripped is now dropped on the static-lookup routes,
  with or without a `FILTER`.** A `SELECT` list naming only variables no pattern binds (`SELECT ?ghost`)
  emitted one empty row `[{}]` when a `FILTER` referenced a stream variable (the fetched `?p` made the row
  non-empty before the emptiness check, and `stripFilterOnlyStaticVars` then removed it) and no row `[]`
  without the `FILTER`; the same held for the join variable a `FILTER NOT EXISTS` makes the lookup fetch. Every
  other emission site already drops an empty record, so `stripFilterOnlyStaticVars` now drops a row that the
  strip itself emptied; the one carve-out stays,
  `COUNT(*)` over an empty batch keeps its row. SPARQL treats an all-unbound solution as a valid
  solution, so this is the engine's documented convention applied consistently, not a spec requirement; the
  convention is now decided and uniform. Pinned by
  `CqelsQueryCompilerPairDConvergenceTest.d_allUnbindableProjectionEmitsNoRowWithOrWithoutTheFilter` and
  `d_allUnbindableProjectionEmitsNoRowUnderNotExists`, with `d_countStarKeepsItsRowUnderTheSameFilter` as the
  control. The strip step is shared with the composed route, so it drops such a row there too (by construction;
  not separately measured).

- **A `BIND` target the `SELECT` list does not name no longer appears in the emitted row.** SPARQL
  projects the `SELECT` list, but the routes that defer `BIND` to the outer pipeline (the composed
  stream-plus-static windowed route, the flat per-arrival static-lookup route, and the composed route's
  in-batch `ORDER BY` / `LIMIT` and `ISTREAM` tails) wrote the target into the row and
  `stripFilterOnlyStaticVars` deliberately retained it, so `SELECT ?s ?lbl ... BIND(7 AS ?z)` emitted `s`,
  `lbl` and `z`. The stream-only windowed route, both directional routes, the temporal join and the
  incremental aggregate route already projected it away (the issue's follow-up comment had measured only the
  stream-only and composed routes; the rest were measured here). The target is now dropped at the final projection step,
  after `FILTER`, `ORDER BY`, `GROUP BY`, `HAVING` and `DISTINCT` have read it, so a `BIND` target that
  only feeds one of those still works. An empty `SELECT` list (`SELECT *`, programmatic only) is untouched.
  Pinned by `UnprojectedBindTargetTest` and `RouteDivergenceProbeTest.p3b_composedJoin_projectsAwayNonProjectedBindTargets_closes774`
  (the former `p3b_composedJoin_leaksNonProjectedBindTargetsIntoTheRow` pin, rewritten).
- **Every variable the `SELECT` list does not name is now projected away, and the composed `ISTREAM` delta is
  taken on the projected rows.** The strip above covered top-level `BIND` targets only; a variable
  bound by an `OPTIONAL` block (`?r`), by a `BIND` inside one (`?zz`), or a stream variable the `SELECT` list does
  not name also appeared in the rows of the composed route. The final projection now removes every in-scope
  variable (stream and static pattern, `OPTIONAL`, `UNION` and block-local `BIND` targets, top-level `BIND`
  targets) that is not in `SELECT` and is not an aggregate alias, still after every stage that reads it. The
  `ISTREAM` delta on the composed and stream-only windowed routes compared rows before that projection, so two
  arrivals whose rows differed only in an unprojected column both emitted although they projected identically;
  the projection now runs ahead of the delta, so `eval(t) \ eval(t-1)` is over projected solutions
  (`UnprojectedBindTargetTest`: the `?r` and `?zz` cells and the `ISTREAM` pair, 5 rows before, 2 now).

- **`SELECT DISTINCT` with `ORDER BY … LIMIT` under `[NOW]` no longer re-emits an unchanged projected snapshot**.
  The fused top-K kept one best-ranked representative row per projection and re-emitted whenever its
  snapshot of those full rows changed, so a better-ranked row of a key it already held swapped the hidden
  representative, left every projected row as it was, and emitted the projected row again — an element
  carrying `(a ex:p 1) (a ex:p 2)` emitted `[a, a]` under `SELECT DISTINCT ?s … ORDER BY DESC(?v) LIMIT 1`
  where `DISTINCT` over one arrival is `[a]`, and a lookup-route `BIND` that materialised the sort key
  emitted `[L, L]` the same way. It now compares the projected snapshot with the one it last emitted and
  suppresses an emission only when the two are identical, so that swap emits nothing, while a change in a
  projected column still re-emits (`PerArrivalElementAsWindowTest.nowDistinctTopK_aRepeatedProjectedSnapshotIsSuppressed_closes904`,
  `nowDistinctTopK_aChangeInAProjectedColumnStillReEmits`,
  `WindowScopedDistinctTest.theFusedTopKSuppressesAnUnchangedProjectedSnapshot_closes904`). Only consecutive
  identical projected snapshots are suppressed: an intermediate snapshot that differs still repeats its
  projected rows, so one arrival of `(a ex:p 1) (b ex:p 2) (a ex:p 3)` under the same query emits `a`, `b`,
  then `a`, each emission superseding the last
  (`PerArrivalElementAsWindowTest.anIntermediateSnapshotThatDiffersRepeatsAProjectedRowWithinOneArrival`). The plain `ORDER BY … LIMIT` top-K is unchanged: it can
  still emit an intermediate snapshot per row of one arrival, each superseding the last, the arrival's final
  snapshot being the answer; one snapshot per arrival would need the arrival boundary inside the top-K and is
  not part of this change.
- **A projected variable, or a `HAVING` variable, that is neither a `GROUP BY` variable nor an aggregate alias is now
  refused when a grouped query is registered.** After the fold only the `GROUP BY` variables and the aggregates
  are in scope (SPARQL 1.1 §11, §18.2.4), and the grouped row is rebuilt from exactly those. `SELECT DISTINCT ?v
  (COUNT(*) AS ?n) … GROUP BY ?sensor` registered and emitted rows with no `?v` column, so `DISTINCT` de-duplicated
  over `?n` alone, and `HAVING (?v > 0)` on an ungrouped `?v` registered and dropped every group, a permanently empty
  query with no error. The constructor now throws an `IllegalArgumentException` naming the variable and the legal
  forms. A variable inside an aggregate call (`HAVING (COUNT(?v) > 0)`) is that aggregate's argument and is not
  checked; a query with an aggregate and no `GROUP BY` is one implicit group and is covered too; the temporal-join
  route is left to its own refusal. Projecting or testing a `GROUP BY` variable (projected or not) or an aggregate
  alias is unchanged (`GroupedProjectionNonKeyTest`). Five existing tests and two fixtures relied on the gap; they were
  rewritten to group by or aggregate the variable, or to reach their evaluation-failure path another way.

- **An `ORDER BY` key that is neither a `GROUP BY` variable nor an aggregate alias is now refused when a grouped
  query is registered.** `SELECT ?v (COUNT(*) AS ?n) … GROUP BY ?e ORDER BY DESC(?v)` registered and ran, and
  its output was byte-identical to the same query without the `ORDER BY`: the fold rebuilds each row from the group
  keys and the aggregate aliases only (SPARQL 1.1 §11, §18.2.4), so the sort compared an absent key on every row. The
  constructor now throws an `IllegalArgumentException` naming the variable and saying which keys are legal. A query
  with an aggregate and no `GROUP BY` is one implicit group and is covered too; the temporal-join route is left to its
  own refusal. Ordering by a `GROUP BY` variable (projected or not) or an aggregate alias is unchanged
  (`GroupedOrderByNonKeyTest`).
- The windowed hash join's `cqels.join.crossProductCap` guard was checked after the join, against its final binding
  list, while the join builds its bindings level by level. An intermediate level far above the cap was
  therefore allocated in full before the check, and a join whose last pattern is selective or unsatisfiable never
  tripped it at all: measured at a cap of 10, a four-pattern star whose second level holds k^2 partial bindings and
  whose last two patterns prune them to one row allocated about 470 MB at k = 200 and 11.6 GB at k = 1000 on the
  serial join (1.6 GB and 40 GB over four subjects on the parallel one) and answered without consulting the cap.
  Both the serial and the parallel join now check the cap while they build the bindings, as the multi-stream
  route's side matcher does: at most `cap` result bindings, checked before the one past it is built, and at
  most `cap x patterns` partial bindings explored per arrival, so the same windows now fail in under 2 MB. Both
  failures are loud, name the same flag, and say the count is taken before FILTER; the result-cap message now says
  "produced more than N" instead of the count it used to reach after materializing it, and the parallel join's
  message is the serial one's rather than a re-wrap prefixed with the exception class
  (`CqelsQueryCompilerWindowedJoinCapTest`).
- **A literal, IRI or prefixed-name constant in a `STREAM` block's triple pattern is now matched as the RDF term it
  denotes, on the multi-stream route and on the single-stream route.** Both routes compared the constant as
  text against `Value.stringValue()`, so datatype, language tag and term kind were never checked. On the multi-stream
  temporal-join route a decorated literal (`"30"^^xsd:integer`, `"cat"@en`) never matched the literal it declared (it
  matched only a literal whose lexical form was the raw query token), and on both routes a plain or decorated literal
  matched every datatype and language with the same lexical form, and an IRI constant and a literal with the same text
  matched each other. The constant is now parsed once into an RDF4J `Value` (`RdfTermClassifier.constantTerm`: lexical
  form unescaped, datatype or language tag split off, a prefixed datatype name expanded through the query's `PREFIX`
  declarations with the built-in `xsd:` prefix as fallback) and compared with `Value.equals()`, the comparison
  variable bindings already used. This is term equality, not value equality: `"030"^^xsd:integer` does not match
  `"30"^^xsd:integer`. A plain literal constant is an `xsd:string` literal (RDF 1.1), so it matches a plain or
  `xsd:string` literal only. A bare numeric or boolean token (`30`, `TRUE`) keeps its datatype-agnostic lexical
  comparison. `StreamConstantTermEqualityTest` pins the five probe cells of the report plus the IRI/literal
  directions, term-equality, escapes and prefixed datatypes on the multi-stream route and on the single-stream route
  under a count window and a time window. The matcher that joins a stream against a static BGP is separate and is
  unchanged. The spec's section on the multi-stream route now describes the new rule.
- A variable repeated across positions of one triple pattern (`?s ?s ?o`, `?x ?p ?x`) was not checked for
  agreement on the serial windowed BGP join: the matcher bound the variable from its first position and never
  compared the later one, so a window of non-reflexive triples emitted rows. The parallel path (four or
  more distinct subjects) already rejected them, so the two paths disagreed on the same window. The serial
  candidate loop now requires the repeated positions of one pattern to carry the same term, so both paths
  return no rows for non-reflexive triples and exactly the reflexive one when present
  (`CqelsBgpJoinParityTest`, `repeatedVariableWithinOnePattern_*`).
- **A `resultMapping` conversion failure no longer ends the result stream.** On both static-lookup routes
  (the flat `executeWithLookupJoin` and the composed `executeWindowedHashJoinWithLookup`), a declared mapping whose
  converter throws on a fetched value (`"long"` over `"abc"` raises `NumberFormatException`; likewise `integer`,
  `double`, `boolean`) escaped the fetch, reached the subscriber as `onError` and killed the standing query, for any
  variable the row emits or an expression reads. The variable is now left unbound in that row, exactly as a
  converter that answers `null` already left it (an expression reading it errors: FILTER rejects, BIND leaves its
  target unbound), the row's other variables are kept, and a WARN names the query, variable, mapping and exception
  once per query and variable, with later failures at DEBUG. The RDF term is still recorded for correlation. The
  conversion is guarded in the shared `typedSolutionValue` seam and in the temporal route's residual filler, so every
  writer of a mapped static value is covered
  (`ResultMappingConversionFailureTest`). A flat-route row that is left with nothing fetched is still dropped, as
  before (`ResultMappingCorrelationTermTest.anEmptyPresentationRowIsDroppedOnTheFlatRoute_characterizedNotEndorsed`
  is unchanged).
- **`CONCAT()`, `IRI(?iri)` and the term tests over an unbound variable followed SPARQL 1.1 only partly.** Three
  built-in calls in the expression evaluator deviated from the spec. `CONCAT()` with no argument was a type error, but
  section 17.4.3.12 defines it from `fn:concat`, which gives the empty string; it now returns `""^^xsd:string`. The
  zero-argument case is an early return in the dispatch, separate from the concatenation itself. `IRI(?x)` (and
  `URI(?x)`) over an IRI was a type error, but section 17.4.2.8 returns the IRI unchanged; it now does, while an
  integer or unbound argument stays an error. `isIRI`, `isURI`, `isBLANK`, `isLITERAL` and `isNUMERIC` over an
  unbound variable answered `false`, but section 17.4.2 defines them on RDF terms and section 17.6 makes an argument
  error an error of the call; they now answer a type error, so `FILTER(!isIRI(?x))` drops a row where `?x` is
  unbound instead of keeping it, and `BIND(isIRI(?x) AS ?t)` leaves `?t` unbound. Pinned by
  `ExpressionEvaluatorTest` (`concatWithNoArgumentIsTheEmptyString_17_4_3_12`,
  `iriOverAnIriReturnsTheIriUnchanged_17_4_2_8`, `termTestsOverAnUnboundVariableAreATypeError_17_4_2`,
  `filterNegatedTermTestDropsAnUnboundRow_17_4_2`, `bindOfATermTestOverAnUnboundVariableLeavesTheTargetUnbound_17_4_2`)
  and `BuiltInFunctionsTest`. None of the three was a recorded deviation. `CONCAT` over a non-string argument and
  the other string and date accessors are not changed here.

- **MCP multi-session hardening: three latent races now real under the Streamable-HTTP transport.**
  (1) `register_reasoning` validated a candidate base and then committed it with nothing serializing the two
  steps, so two concurrent registrations could validate one base while the other committed; the profile default
  is now read, the candidate validated and the commit made under one lock
  (`RegisterReasoningSerializationTest`). (2) The shutdown loop took a snapshot of the stream-query registry and
  then cleared it, so a registration that attached its engine subscription between the two, or began after the
  snapshot, was dropped from the registry with its engine subscription still live; the registry now has
  `closeAndDrain()`, which refuses every later registration and removes each live one atomically with its
  engine-id attach, and the loop uses it (`CqelsMcpServerShutdownRaceTest`). (3) The first
  `enforcementActive()` probe cached its answer with a plain write, so a probe that had read "no policy" could
  overwrite the `true` a concurrent `set_access_policy` had just stored and switch enforcement off again; the
  first probe now commits with `compareAndSet(null, ...)` and can never regress a `true`
  (`EnforcingCacheFirstProbeRaceTest`).

- **SHACL repair search now finds repairs that create a focus node.** `ShaclCandidateGenerator`
  enumerated repair candidates in one pass over the original snapshot's focus nodes, while the ASP `focus/2`
  rules derive over the repaired graph, so a repair that adds a targeting triple (an `rdf:type` for
  `sh:targetClass`, any triple of the predicate for `sh:targetSubjectsOf` / `sh:targetObjectsOf`) made a focus
  node that had no candidates of its own: a two-step repair such as typing a value as the target class and then
  giving it the property that class's shape requires reported `SEARCH_BOUNDED`, although it exists. Generation
  now iterates to a fixpoint over the focus set implied by the snapshot plus the add candidates found so far,
  in at most `maxTotalEdits` rounds (a focus node first reached in round k needs a repair of at least k edits).
  Constraint checks still read the original snapshot, and the search stays sound because the solver validates the
  repaired graph. `ShaclRepairFocusClosureTest` pins the two-edit repair, a shape whose candidate set is
  unchanged by extra rounds, and the rounds bound on a growing chain. Deleting a node's last targeting triple
  remains an admissible repair unless `protectTargetingTriples` is set (see Added).

- `AspFactMapper.mapElement` returned no facts for a `GraphStreamElement` (what `DataStream.push(Collection<Statement>)`
  produces), because `GraphStreamElement.isRDFStatement()` is `false` by contract and the mapper treated every such
  element as non-RDF. `AspContinuousQuery` maps window contents from user streams through `mapWindowUpdate` and
  `mapElements`, so the facts of a graph element never reached the ASP session, with no error. `mapElement` now
  maps every statement of a graph element, and an empty one still maps to nothing
  (`AspFactMapperTest.mapsEveryStatementOfAGraphElement`, `emptyGraphElementMapsToNoFacts`,
  `mapsWindowUpdateWithGraphElementInAddsAndRemovals`); a non-RDF element is unchanged.
- The single-stream `[NOW]` per-arrival lookup route is pinned to validate the whole stream BGP before it reads the static side. The tracker reported that it seeded each variable from the first matching statement, so `?o :of ?x . ?o :v2 ?x` over an element with conflicting objects could still emit a row. That no longer reproduces: since the route admits an element only through its one-element-window stream solutions, a variable shared by two patterns must take one value. A two-pattern element with conflicting objects emits no row in either statement order, one with agreeing objects emits one row, and a single-pattern control still answers (`CqelsQueryCompilerFlatRouteBgpMatchTest.now_sharedVariableWithConflictingObjectsEmitsNoRow_agreeingObjectsEmitOne`). Test-only; no behaviour changes.
- The vendored Delta ASP `NaiveGrounder` kept grounding a rule instance whose negative body literal was
  already contradicted, so `c(X) :- a(X), not b(X).` was still instantiated for `X = 1` when `b(1)` was
  assigned TRUE. The grounder's instantiation strategy treated every negated ground literal as
  satisfied, producing superfluous nogoods that cost grounding time and memory without changing any answer
  set. It now stops on a negated literal whose atom is TRUE at decision level 0, where the assignment is
  permanent. A TRUE assigned above level 0 and an MBT atom still count as satisfiable, because backtracking
  can retract them and nothing would re-trigger grounding of the instance afterwards, and negative literals
  still consume no grounding tolerance, so the `strict` and `permissive` settings behave as before.
  `NaiveGrounderTest.testGroundingOfRuleSwitchedOffByTrueNegativeBody` is re-enabled, and two new tests pin
  the level-above-zero and MBT cases.
- `ReactiveReteAdapter.apply` forwarded an RDF element whose `asStatement()` is `null` (`isRDFStatement()` true, no
  statement; reachable through `new RDFStreamElement(null, ts)` or `push((Statement) null)`) instead of dropping it.
  It read "no statements" as "not RDF, pass through", although the pre-multi-statement code dropped such an
  element. The null-bearing element therefore reached downstream consumers that dereference the statement
  (`ShaclContinuousQuery`, `AspFactMapper`) and escaped `emitInputTriples=false`. The empty case now asks the element
  which it is, the same disambiguation `ReasoningService` already uses: a malformed RDF element is dropped under both
  `emitInputTriples` settings, while a non-RDF element and an empty `GraphStreamElement` still pass through and a
  multi-statement element behaves as before (`ReactiveReteAdapterMultiStatementTest`).
- `STRLEN` and `SUBSTR` counted UTF-16 code units, so a character outside the Basic Multilingual Plane counted as
  two and `SUBSTR` could return half of its surrogate pair. Both now count characters (Unicode code points),
  as SPARQL 1.1 §17.4.3.2 and §17.4.3.3 define them through XPath `fn:string-length` and `fn:substring`. Strings
  without such characters are unchanged, and so are `SUBSTR`'s 1-based and out-of-range rules. No normalisation is
  applied: a letter and a combining mark are two characters. Pinned by
  `Sparql11MissingBuiltInsTest.strlen_countsCodePointsNotUtf16Units` and
  `Sparql11MissingBuiltInsTest.substr_countsCodePointsNotUtf16Units`.
- **RETE truth maintenance no longer double-counts a re-recorded justification.** With
  `enableTruthMaintenance(true)`, the justification index stored each support as a multiplicity and
  incremented it whenever the same justification key (rule plus premise set, one logical support) was recorded
  again, for example when a premise was re-asserted. The distinct-support count hid this, but a later decrement
  left a phantom support of 1 behind, so a consequent could stay supported after its only logical justification
  was gone. Supports are now set-like: re-recording a key keeps one support and remembers its latest
  oldest-premise time, and window pruning drops the key only when that latest recording ages out, so a stale
  duplicate entry is a no-op while a refreshed premise still keeps the support live
  (`TruthMaintenanceSeamTest.reRecordingSameJustificationKeyStaysSetLike`,
  `distinctJustificationKeysForOneConsequentStillCountSeparately`,
  `windowPruneOfStaleRecordingKeepsKeyWhileRefreshedPremiseIsLive`). Observable inference output is unchanged.
- `StreamElementToStreamEventAdapter.lift`, and the two join operators it feeds, dropped the subscriber's Reactor
  context. `lift` subscribed its source with the lambda `subscribe(...)` overload, which carries an empty
  context, so a source composed on `deferContextual` (credentials, tenant, tracing, request metadata) saw nothing of
  what a downstream caller wrote with `contextWrite`. `IntervalJoinOperator` and `WindowedJoinOperator` subscribed
  their two inputs the same way, so on the multi-stream temporal-join route the context was lost at the join even
  with `lift` fixed. `lift` is now composed on `deferContextual` and subscribes the source with a subscriber that
  exposes that context, and both join operators subscribe each input with the context of their own subscriber; demand
  is unchanged (one unbounded request per input, so a shared source is still never backpressured by one consumer). A
  downstream `contextWrite` now reaches the sources of the multi-stream temporal-join route (the `temporalJoin` hint,
  and two `STREAM` blocks with or without a conjoined side), the CypherQL `RANGE` and `SLIDE` window routes,
  directional CEP (both inputs when it has an external watermark input) and event-time CEP. A `share()`d source is
  subscribed once for all its consumers, so it sees one subscriber's context (the first's), not each subscriber's.
  Pinned by `ReactorContextRouteTest`, `CepReactorContextRouteTest` and
  `StreamElementToStreamEventAdapterTest.lift_propagatesTheDownstreamContextToAContextualSource`.
- **A multi-triple `STREAM` block on the multi-stream interval-join route no longer holds dead solutions
  against `cqels.join.intervalBufferCap`.** A side solution spanning `[oldest, newest]` was
  bucketed and expired by its newest constituent, so a left solution stayed buffered until
  `newest + upperBound` although the join stops admitting partners for it at `oldest + upperBound` (and a
  right solution likewise past `oldest - lowerBound`). At a sustained rate those entries filled the
  buffer and could trip the cap on a valid query. The interval join now retires such a solution at its
  oldest constituent through a second index keyed by that timestamp; records that are single
  statements are never in the index and expire exactly as before, and no row changes.
  `TemporalJoinDispatchSideSolutionRetentionTest` pins that a multi-triple workload fits the cap an
  equivalent single-triple workload needs, and `IntervalJoinOperatorRetentionTest` pins both sides'
  boundaries and that seeded interleavings emit identical rows with and without early retirement.
- `MultiWayJoinState.deleteRange` removed the relation's elements in the time range from the join
  indexes but returned a zero delta and left the maintained aggregate unchanged, so after a range
  delete the aggregate still counted join results whose elements were gone. It now retracts
  each element in the range the way `delete` does (remove it from the indexes, then negate its
  recomputed delta), in timestamp order, and returns the summed negative delta; the result and the
  remaining state equal those of deleting the same elements one by one (`DeletionTest`, whose
  `testBinaryJoinRangeDeletion` previously pinned the zero delta). It costs one per-element delete,
  so it is no faster than calling `delete` per element. The issue also reported that
  `WindowedJoinAggregateOperator` never evicts on a slide and so reports a growing aggregate; it keeps
  one state per (key, window) and inserts each element into every window it belongs to, so a slide
  starts a new state and nothing needs evicting. `WindowedJoinAggregateOperatorSlidingTest` pins this
  against a brute-force oracle for COUNT and SUM across slides, in both `KEY_PARTITIONED_ONLY` and
  `BATCH_ON_CLOSE`, and the operator's Javadoc no longer says it evicts.
- `SPARQLExecutor` evaluated each window batch by adding only single-statement elements to its temporary
  repository, so a `GraphStreamElement` (what `DataStream.push(Collection<Statement>)` produces, and
  `GraphStreamElement.isRDFStatement()` is `false` by contract) was skipped and its triples never became queryable:
  a plain SPARQL query over such a stream returned no rows and raised no error. The batch loop now adds every
  statement of a graph element, as it does for a single statement, and the graph element's timestamp counts towards
  the result timestamp (`SPARQLExecutorTest.graphElementTriplesAreQueryable`, with
  `SPARQLExecutorTest.singleStatementIsQueryable` as the control).
- **The string built-ins were type-lax and dropped the language tag, and the date accessors returned non-spec
  datatypes.** `STRLEN("42"^^xsd:int)` answered 2, and `UCASE`, `LCASE` and `SUBSTR` read any literal through
  its lexical form, although SPARQL 1.1 §17.4.3.1.2 takes a string literal (simple, `xsd:string` or language-tagged)
  and makes anything else a type error; they now return an error for another datatype or a non-literal, as
  `CONTAINS` and `STRSTARTS` already did. `UCASE`, `LCASE` and `SUBSTR` also returned a plain literal, losing the
  first argument's language tag, where §17.4.3 has the result carry the first argument's kind; `UCASE("Hello"@en)` is
  now `"HELLO"@en`. `CONCAT("a"@en, "b"@en)` lost the shared tag too: the result now keeps it when every argument is
  tagged with that same tag and is a simple literal otherwise (§17.4.3.12). `SECONDS` returned an `xsd:double` and
  now returns the `xsd:decimal` §17.4.5.6 requires, so a fractional second is exact; `YEAR`, `MONTH`, `DAY`, `HOURS`
  and `MINUTES` returned `xsd:int` and now return `xsd:integer`. Pinned by the string-function and date-accessor section of
  `BuiltInFunctionsTest` (`testStringFunctionsRejectNonStringLiteral`, `testUcaseLcaseSubstrCarryLanguageTag`,
  `testConcatKeepsSharedLanguageTag`, `testConcatDropsLanguageTagWhenMixed`, `testSecondsIsXsdDecimal`,
  `testDateAccessorsReturnXsdInteger` and their evaluator-level companions). `CONCAT` of a non-string literal and
  `STRLEN`'s own `xsd:int` result datatype are unchanged.
- **`VariableLengthPathOperator` returns each path once, matches relationship types by IRI, and cannot match a longer
  IRI that contains the type.** A path of exactly `maxHops` edges was emitted twice, once on expansion and
  again when its state was polled, so a `min == max == N` query returned twice the real count. Paths are now emitted
  only on expansion. The relationship type was matched with `String.contains`, so `ex:Friend` also matched
  `ex:FriendOf`; it is now compared by equality. The start node is marked as visited by the path state's constructor
  as before, and a path that cycles back to the start is still not returned. Pinned by
  `VariableLengthPathOperatorTest` (`testPathAtMaxHopsIsReturnedOnce`, `testRelationshipTypeMatchesIriExactly`,
  `testPathDoesNotCycleBackToStart`, and exact counts in `testSimplePathFinding`). (Superseded: the class is
  removed in this release.)

- `MaxAggregateFunction` and `MinAggregateFunction` reported an infinite input as no value. Both used the
  infinity they start from as the "nothing seen yet" marker, so a group whose only value was `-INF` (MAX) or
  `+INF` (MIN) returned `null`, and an infinity that was the current extreme was dropped in merges. Both now
  return the infinity: the empty accumulator is a NaN bit pattern that no numeric literal can produce, and the
  public accumulator type stays `Double`. An empty input still returns `null`, and a NaN input still yields NaN
  (`MaxMinAggregateFunctionTest`, closing).
- `SimplePatternProcessor` now agrees with `NFAPatternProcessor` on unquantified linear `next` / `followedBy`
  patterns that the NFA accepts, states with no predicate and `notNext` / `notFollowedBy` before a positive
  state included. Its pattern build stripped a begin node with no predicate, so that state consumed
  nothing and every later `whereContext` guard saw a prefix one event short: `begin("baseline").followedBy("spike")`
  with a guard reading `prev.get(0)` threw `IndexOutOfBoundsException` over `(10, 20)`, and the defensive guard
  `prev.isEmpty() || ...` over `(10, 20, 30)` reported three single-event matches where the NFA reports the
  pairs `[10,20]` and `[20,30]`. The begin node is now kept (it matches any event and consumes one), and a
  negated state is no longer treated as unmatchable: it is never consumed, an event that satisfies it kills the
  partial match (`notNext` judges the adjacent event, even one the next positive state could take; `notFollowedBy`
  every event left in the gap before the next positive state, not the event that state consumes), so
  `A, notNext(B), followedBy(C)` over `(A, X, C)` now matches `[A,C]` instead of nothing, and
  `A, notFollowedBy(B), followedBy(B)` over `(A, B)` matches `[A,B]`. The decision
  lives in one method, `ProcessorState#advance`, and the matches' runs are labelled by state as the NFA labels
  them. `SimplePatternProcessorNfaParityTest` runs each shape through both processors and requires equal
  labelled matches (each state's name and the positions it consumed), over pinned cases and a sweep of the
  negation shapes. `notFollowedBy(...)` directly followed by `next(...)` is now refused at construction with
  `IllegalArgumentException`, as the NFA refuses it (design note SQ2). Still not modelled: `followedByAny`, taken
  as `followedBy` (one continuation per start where the NFA keeps every one); quantifiers and optional or looping
  states; and a trailing `notFollowedBy`, which the NFA completes at the end of the window or stream and this
  timer-less processor never completes.
- **CypherQL resolves a backquoted property the same way on every clause.** The lexer keeps a backquoted
  identifier's backquotes in the expression text while the matcher stores a property under its bare local name, so
  ``n.`value` `` matched nothing: not in `RETURN`, an aggregate argument, `ORDER BY`, `WHERE`, `HAVING`, `GROUP BY`,
  a pattern's property map, or the static-lookup resolver, which also split a joined path on every dot, so
  ``n.`value.unit` `` read `value`. One helper (`CypherIdentifiers`) now unquotes every property component, and a
  path is carried as components, never re-split on a bare dot. A quoted and an unquoted spelling of a property are one
  row key and one output column (``RETURN n.`value` `` is the column `n.value`). A property component written as a
  full IRI (``p.`http://example.org/age` ``) resolves to the IRI's local name, the key the matcher stores, so it is
  the column `p.age` on every clause, and a colon-bearing local name (`age:v2`) is not cut a second time by the
  static lookup. The `RETURN DISTINCT` refusal for a
  quoted property, which existed only because of this, is removed, and `ReturnExpression.hasQuotedProperty()` goes
  with it (`CypherQLQuotedIdentifierTest`; the `..._characterization` pins for a quoted property in
  `CypherQLReturnDistinctTest` and for a backquoted full IRI in `CypherQLExactResolutionTest` now assert the resolved
  behaviour).

- The legacy `CypherStreamQuery` matched node labels, the RDF `type` shortcut and relationship types by substring,
  so `:Person` accepted `:Personnel` and `:knows` accepted `:knowsOf`. A label or relationship type now matches only
  when the predicate IRI equals it or its local name equals it, the same rule `GraphPatternJoinOperator` applies.
  The `CONTAINS` operator in property filters is unchanged and still matches substrings by design. Pinned by
  `CypherStreamQueryExactMatchTest`.
- `ParallelWindowedAggregateOperator`'s four-argument constructor accepted a `null` configuration and
  replaced it with `ParallelExecutionConfig.defaultConfig()`, but then passed and dereferenced the
  original `null` parameter for the aggregation-backend selection, so a `null` configuration threw
  `NullPointerException` instead of constructing with the default. Both uses now read the
  normalised value, and the operator reports the default parallelism and the `LEGACY` backend
  (`ParallelWindowedAggregateOperatorNullConfigTest`). The other constructors already passed a
  non-null configuration and are unchanged.
- **Private tracker issue numbers no longer ship inside user-facing messages.** The tracker is private but
  the engine is published, so an exception message, MCP tool error or description, MCP resource payload or log
  line that cited an issue number named a reference no reader could resolve. A scan of every string literal and text
  block under `src/main/java` in all modules (comments and Javadoc stripped) found 66 lines of message text naming 39
  distinct issue numbers, in 26 files across `cqels-core`, `cqels-engine`, `cqels-mcp`, `cqels-shacl`,
  `cqels-storage-spi` and `cqels-benchmarks`; each now says the same thing without the number (the issue's first count
  of 16 covered `cqels-core` and `cqels-engine` only). The wording is otherwise unchanged, except that four messages
  that named an issue as the place the work is tracked now say the work is planned. Eight tests that pinned a number
  in the message text now pin the surrounding words instead (`ComposedUnjoinableStaticBgpTest`,
  `CypherQLStreamSourceTest`, `CypherQueryCompilerTest`, `CepMinusRejectionTest`, `IngestAckMatrixTest`,
  `FoldOnWritePaneStoreFormatMarkerTest`, `LmdbTemporalPaneStoreTest`, `RocksDbTemporalPaneStoreTest`).
  Two checks keep it from coming back, and only the second is authoritative.
  - A line-based check in `.github/docs-sanity-check.sh` fails CI on `#<digits>` in a code line of a tracked
    `src/main/**/*.java` file (it skips comment lines and anything after a `//` that is outside a string literal, so a URL inside a message does not end the search; it can still miss a text-block line with a URL before the reference); it was red on the 66 lines before the sweep. A second hint in the
    same script reads tracked source resources only (`scripts/ci/published-resources-hint.py`) and says so in its
    output; it cannot see generated, filtered or shaded content.
  - The existing "Verify no private-repo references inside staged jars" step of `publish.yml`, which also runs on a
    `workflow_dispatch` rehearsal that publishes nothing, now runs `scripts/ci/staged-jars-disclosure-gate.py` over
    every staged jar and applies `scripts/ci/disclosure.py`'s own policy (a bare `#<digits>` with only a fully
    qualified `CQELS4J#<digits>` allowed, HTML entities decoded, the private repository's name or URL, downstream
    names, e-mail addresses, Unicode-aware token boundaries). It reads bytes, so it needs no model of how a resource
    got into a jar, and no heuristic can hide a name from it: the complete raw bytes of every entry, every complete
    entry name, comment and extra field are always searched for the private names (and, in one linear pass and one unescape, for
    names written as HTML entities; a name encoded twice is a documented limit), and the structured readers below
    can only add hits: every entry is inflated by the gate itself under a byte budget and must match its declared size and CRC, archives nested
    in archives are opened (to depth four), a class file's constant pool is parsed and each string decoded as the JVM's
    modified UTF-8 (a `SourceDebugExtension` attribute is read too, and one that does not decode fails the gate), a declared text encoding (UTF-8, UTF-16, UTF-32 by
    BOM) that does not decode fails the gate, opaque binaries (images, native libraries) are held to the private-name rules over their
    printable runs, since a bare number or e-mail address is a text rule (a short name such as a downstream
    project's must stand alone there, because it occurs by chance in random data), zip comments, entry names and extra fields are held to the private-name rules, and an unreadable
    directory, jar, entry or malformed class fails it. In a `-shaded.jar`, an entry under a first-party root
    (`org/cqels/`, `docs/`, `META-INF/services/`, `META-INF/maven/org.cqels/`, the manifest) gets the full policy as in
    a thin jar, an entry byte-identical (name and SHA-256) to a thin jar's is skipped as already read, and every other
    entry, which is third-party by elimination, is held only to the private-name rules. The shared policy does not cover
    unhashed forms such as "issue 123" or "PR 123", and a letter followed by a combining mark counts as a non-word
    next to a reference. The gate stops at the first refusal, so it
    is a fail-closed check, not a complete inventory. The gate has 109
    self-test cases (`scripts/ci/staged-jars-disclosure-gate.test.sh`, run by the public-disclosure workflow).
  - The gate's first run on real jars found three e-mail-shaped sample literals in two example classes, now written
    with " at " in place of the at-sign. `docs/CQELSQL_CEP_SYNTAX.md` carried one tracker reference, now worded as a
    known limitation; the three `CQELS4J` references in `MCP_MEMORY_PROFILE.md` stay because that repository is
    public.
- **`FILTER` comparisons between duration-typed literals compare by value instead of by text.** Two
  `xsd:duration`, `xsd:dayTimeDuration` or `xsd:yearMonthDuration` literals compared their labels, so
  `"PT10S" > "PT9S"` and `"PT60S" = "PT1M"` were false; both are now true. SPARQL 1.1 §17.3 defines no comparison
  operator for durations, so these comparisons are a CQELS extension that adopts XPath 3.1 Functions and Operators
  where it applies. Equality follows `op:duration-equal` (§8.2.5), which takes any two durations: equal when the
  month components and the second components are both equal, so `P1Y = P12M`, `P1D = PT24H`, `PT1.0S = PT1S` and
  `"P0D"^^xsd:dayTimeDuration = "P0M"^^xsd:yearMonthDuration`, while `P1M != P30D` and `P400Y != P146097D`. Functions
  and Operators orders only two operands of the same subtype (§8.2.1, §8.2.3) and plain `xsd:duration` not at all;
  CQELS goes further and infers the family from the zero components: two durations whose second components are
  both zero compare by months, two whose month components are both zero compare by seconds, and any other pair is a
  type error. So plain `xsd:duration` operands order within one family, `"P0M"^^xsd:yearMonthDuration <
  "PT1S"^^xsd:dayTimeDuration` is true, and `P1M < P30D` is a type error, which is stricter than XSD 1.1's partial
  order (that would call `P1M > P27D` true). Seconds compare as exact decimals (`PT0.1S < PT0.1000000000000000001S`).
  An ordering comparison (`<`, `<=`, `>`, `>=`) to which neither family rule applies, a comparison against a
  non-duration operand (a plain or `xsd:string` literal, a number, an `xsd:dateTime`), or one with a malformed
  lexical is a type error and drops the row; `=` and `!=` never need a family, so `P1M != P30D` keeps the row. Lexicals are parsed against the
  regular expression XSD 1.1 displays for each datatype, which needs a digit on both sides of a decimal point, so
  `PT.5S` and `PT1.S` are malformed although the XSD grammar productions admit them. As an implementation
  restriction a lexical longer than 64 characters is malformed even when it is in the lexical space (`P`, 63 zeros
  and `D` denotes zero but is rejected), so the cost of a comparison is bounded by that length even for a field such
  as `P99999999999999999999D`. This holds on the stream-only route (the reported `[NOW]` query and the windowed join
  of a multi-pattern stream pattern), on the temporal join of two `FROM STREAM` sources, and on the routes that
  evaluate `FILTER` on the materialised row: stream+static lookups, `HAVING` and filters inside `OPTIONAL` (next
  entry). CypherQL `WHERE` still compares durations as text (Known issues). Public report: CQELS4J#82
  (`ExpressionEvaluatorDurationComparisonTest`, `CqelsQueryCompilerDurationFilterRouteTest`).
- **A row-bound duration keeps its datatype through the result row.** The row lift
  turned an `xsd:duration`, `xsd:dayTimeDuration` or `xsd:yearMonthDuration` binding into its lexical `String`, which
  the row-evaluated `FILTER`, `BIND` and `HAVING` re-read as `xsd:string`: two row-bound durations compared as text,
  and since the entry above, one compared with a duration constant such as `"PT9S"^^xsd:duration` dropped
  every row. The row now
  carries the duration as an RDF4J `Literal`, so those comparisons go by value like the stream-only route's, and the
  static probe behind `OPTIONAL`, `FILTER NOT EXISTS` and `UNION` correlates on the duration term instead of a
  re-minted `xsd:string` that matched nothing (so `FILTER NOT EXISTS { ?s ex:limit ?d }` with a matching static
  duration now drops the row). The literal's `toString()` is its lexical form, so the printed row, `GROUP_CONCAT`,
  the `GROUP BY` key, the `ORDER BY` / `MIN` / `MAX` text fallback and cqels-cdsp's write-back all still see `PT9S`.
  The `SELECT DISTINCT` key no longer reads a value's `toString()`: it is built from the RDF term (the label with its
  datatype or language tag, the IRI, the blank node), so `DISTINCT` keeps the IRI `<urn:x>` apart from
  `"urn:x"^^xsd:duration`, and `"PT1S"^^xsd:duration` apart from `"PT1S"^^xsd:dayTimeDuration`, which merged while
  both lifted to the same `String` (`distinctKeepsTermsThatShareOnlyTheirLabel`,
  `solutionKey_keysOnTermIdentityNotOnToString`); keys of values that print as RDF4J prints them are unchanged. A
  `GROUP BY` key tells a row-bound duration from a plain string and from an IRI, but not from the same label under
  another duration datatype (Known issues). Java only: cqels-rs still lifts durations to their label. The next entry
  extends this to every other non-core typed or language-tagged literal (`CqelsQueryCompilerDurationFilterRouteTest`).
- **Every row-bound language-tagged or non-core typed literal keeps its term through the result row.** The
  row lift turned a language-tagged literal, or one typed `xsd:dateTime`, `xsd:date`, a custom datatype or any datatype
  outside the XSD string, numeric and boolean families, into its lexical `String`, which the row-evaluated `FILTER`,
  `BIND` and `HAVING` (stream+static lookups, inside `OPTIONAL`) re-read as `xsd:string`. So `DATATYPE(?ts)` was
  `xsd:string` and `LANG(?label)` was `""`; a bare `FILTER(?ts)` over an `xsd:dateTime` kept the row, where SPARQL
  raises a type error; `sameTerm` could not tell `"cat"@en` from `"cat"@fr`; a `BIND` result lost its tag before the
  next expression on the stream-static join route; and the static probe behind `OPTIONAL`, `FILTER NOT EXISTS` and
  `UNION` seeded such a key as an `xsd:string`, so it never matched its own term and did match an untagged string of
  the same form. The row now keeps these literals the way it keeps durations, with `toString()` as the lexical form,
  so the printed row, `GROUP_CONCAT`, the `GROUP BY` key, the `ORDER BY` / `MIN` / `MAX` text fallback and cqels-cdsp's
  write-back still see the label. That includes a `geo:wktLiteral`, which an earlier change already kept as a term but which
  printed, and so concatenated in `GROUP_CONCAT`, in N-Triples form (`"POINT(1 2)"^^<...>`); it now prints its WKT.
  `SELECT DISTINCT` keeps `"cat"@en` and `"cat"@fr` apart. Numeric and boolean literals still narrow to Java numbers
  and booleans. Java only: cqels-rs still lifts these literals to their label, and the ASP result mapper
  (`AspResultMapper`) keeps its string forms (`CqelsQueryCompilerRowLiteralTest`,
  `CqelsQLStaticBgpCompleteMatchTest#aLanguageTaggedJoinKeyCorrelatesOnBothRoutes`, `ResultMappingCorrelationTermTest`).

- **A `FILTER(SEQ(...))` argument that is not the subject of a stream triple pattern is refused at
  registration.** Such a query registered before, and that step had no condition, so it matched any
  stream element (public report CQELS4J#74). Registration now fails with `IllegalArgumentException` naming the
  variable. A stream triple pattern is a triple in a `STREAM` block, or a bare triple when the query has exactly
  one `FROM STREAM` and no `STREAM` block, `FROM` graph, `FROM NAMED` graph or `FROM NAMED WINDOW`; nothing else can bind a CEP event,
  so a variable that appears nowhere or only as an object is refused, and so is a subject that appears only inside
  `OPTIONAL`, `UNION`, `GRAPH`, `WINDOW` or `FILTER NOT EXISTS` or in a bare triple the query keeps static
  (`CepSeqSubjectValidationTest`; `docs/CQELSQL_CEP_SYNTAX.md` now says which triples count).

- **An `OPTIONAL` that probes an unprojected prefix-`BIND` target now correlates**. This
  closes the matching known issue of `2.0.0-alpha.21`. A `BIND` declared above the `OPTIONAL` block whose
  target the `SELECT` list does not name was dropped when the solution became a row, so the block
  saw its key unbound. On the per-arrival `[NOW]` route every row null-filled — an outer row with
  two matches gave one unbound row instead of two bound ones (`OptionalBindProjectionRegressionTest`,
  `OptionalBindMappingRegressionTest`) — and the same held for a key read only by the block's inner
  `FILTER` or `BIND` expression (`OptionalBindExpressionRegressionTest`). On the windowed hash-join
  route (a multi-pattern stream BGP under `[TRIPLES n]` or `[RANGE …]`) the tested windows emitted
  no rows at all, including for a key read only by the inner `FILTER`
  (`OptionalBindWindowedRegressionTest`). The key is now carried through every binding-to-row
  conversion until the `OPTIONAL` blocks have run, and removed after the solution modifiers: rows
  carry only the `SELECT` variables, `DISTINCT` compares only the projected variables while
  `ORDER BY` / `LIMIT` still see the key (`OptionalBindModifierRegressionTest`,
  `OptionalBindWindowedRegressionTest`), and the per-arrival `ORDER BY … LIMIT` top-K without
  `DISTINCT` emits a new snapshot only when its projected rows change, not when a tie differs only
  in the key (`OptionalBindTopKRegressionTest`); the `SELECT DISTINCT` top-K compares its
  projected snapshot as well. On the windowed route that joins the stream BGP to a static
  pattern, rows used to expose the key column and an `Istream(...)` delta reported a second
  insertion when consecutive windows differed only in the key; rows now carry only the `SELECT`
  variables there too, and on both windowed routes the delta inserts the projected row once
  (`OptionalBindIstreamRegressionTest`). A `BIND` declared below the `OPTIONAL`
  block is unchanged and is still not a probe key.

- **The composed windowed lookup route now joins a row whose join key binds a literal**.
  A query joining the stream to the static graph on an ID value — the IS01 shape,
  `?person ex:id ?pid` beside a stream pattern binding `?pid` to `7` — answered nothing on
  `executeWindowedHashJoinWithLookup`: the route kept a join key only when it was an IRI or a
  blank node, because the lookup operator roots its view at a subject, so every row whose key
  bound a literal was dropped before the static side was evaluated, with no log line. The
  per-arrival lookup route answered the same query. Such a row is now enriched as the per-arrival
  route enriches it: with no root, the literal seeded into the static matcher, so the
  repository-backed candidate fallback asks for it as the object of the pattern that names
  it. For a literal join key whose static patterns the candidate fallback can serve, both routes
  now give the same rows for the same data (`ComposedLookupLiteralJoinKeyTest`). One divergence
  remains: the per-arrival route still reads a view rooted at the element's fallback subject while
  the composed route reads an empty one, so a static pattern the fallback cannot serve (an
  unconstrained `?x ?p ?o` beside `?person ex:id ?pid`) can answer under `[NOW]` and not under
  `[RANGE 10s]`. A declared `operators.lookupJoinKeyVariable` bound to a
  literal still drops its rows with its existing warning; this fix covers a key resolved from the
  patterns.

- **An explicit `GRAPH <iri> { … }` block on the ordinary single-stream lookup routes now reads
  only that graph, and a static constant on those routes now matches only the identical RDF
  term**. The `[NOW]` route and the windowed route looked a stream-bound key up across
  every context of the repository — in the forward read and in the repository candidate fallback
  alike — so a query scoped to `GRAPH <urn:g>` was widened to every graph: it returned a second
  `?hash` taken from another named graph, a reverse edge `?owner ex:owns ?item` stored only in
  another graph, and a `COUNT` that counted them, and a key with no binding in `<urn:g>` still
  answered from elsewhere. Both reads now pass the `GRAPH` block's graph as the context, including
  the read a `CACHE` TTL keeps; further graphs declared by `FROM` / `FROM NAMED` are not read. A
  query with no `FROM` / `FROM NAMED` at all treats the repository's graphs as its dataset, so
  `GRAPH <urn:g>` reads `<urn:g>`. As on the temporal route, `FROM <urn:g>` also exposes `<urn:g>`
  by name to a `GRAPH <urn:g>` block; this is a CQELS extension, since strict SPARQL puts a `FROM`
  graph only in the default graph and that block would have no solutions. The temporal route's
  dataset rule is unchanged: it still requires a dataset declaration, and aligning it is out of
  scope. The lookup may be
  anchored wherever the static matcher can fetch candidates by — a stream-bound variable in the
  subject, predicate or object position, a constant subject or predicate, or an IRI object — so
  `STREAM S { ?r ex:lookup ?item } GRAPH <urn:g> { ?owner ex:owns ?item }` and
  `STREAM S { ?r ex:lookup ?p } GRAPH <urn:g> { ?owner ?p "Alice" }` are still answered, now from
  `<urn:g>` only. That check only establishes that some triple can be fetched; the matcher still
  evaluates triples in declaration order, so an unconstrained triple written before the anchor sees
  only the forward view, as it did before. Separately, the two reads were inconsistent with RDF
  term equality on constants: they compared lexical forms, so a `GRAPH` constant
  `"7"^^xsd:integer` matched a stored plain `"7"`, `"chat"@en` matched `"chat"@fr`, and the
  stream pattern did the same. On these routes a constant now matches only an equal RDF term,
  datatype and language tag included, in the stream pattern, the forward read and the candidate
  fallback (`PinnedGraphScopeRegressionTest`). Some valid shapes are now refused at registration
  with an `UnsupportedOperationException` instead of being answered from every graph. These are
  conservative refusals, not ambiguities in the query: `GRAPH` blocks naming two different graphs
  and `GRAPH` blocks beside plain static patterns need a graph per pattern, which the flattened
  lookup does not keep, and a `GRAPH` IRI that the declared `FROM` / `FROM NAMED` graphs do not
  include would answer nothing under SPARQL. `OPTIONAL` / `UNION` / `FILTER NOT EXISTS` beside an
  explicit `GRAPH`, a second declared stream, and a `GRAPH` pattern with nothing to fetch by (no
  stream-bound variable, no constant subject or predicate and no IRI object) are refused as well.
  A refused registration registers nothing and calls no listener (`GraphScopeRegistrationTest`).
  Static patterns outside a `GRAPH` block, named windows and temporal joins keep their existing
  graph scope and constant matching; the default-prefix fix below applies to them as well.

- **A prefixed name with the default (empty) prefix now resolves on every compiled CQELS-QL
  route**. The shared prefix resolver expanded a name only when its colon came after the
  first character, so under `PREFIX : <urn:ex:>` the term `:p` stayed the verbatim string `:p` and
  never matched: a flat stream pattern `STREAM S { ?s :p ?o }` with no `GRAPH` block answered
  nothing for a streamed `<urn:ex:s> <urn:ex:p> "v"`. The default prefix now expands like any
  other wherever that resolver runs — predicates on every compiled route, the temporal join's
  constants, and the static matchers' subjects, objects and literal datatypes — so under
  `PREFIX : <http://www.w3.org/2001/XMLSchema#>` the constant `"7"^^:integer` is
  `"7"^^xsd:integer` and under `PREFIX : <urn:ex:>` a `GRAPH` subject or object `:o` is
  `<urn:ex:o>`. A query that declares no default prefix keeps the old behaviour: `:name` is left
  as written and matches nothing, without an error (`PinnedGraphScopeRegressionTest`).

- **CypherQL `RETURN DISTINCT` is now applied.** The keyword was parsed and
  `CypherQueryDefinition.isDistinct()` exposed it, but the compiler never read it — only the
  per-aggregate DISTINCT (`COUNT(DISTINCT x)`, `SUM(DISTINCT x)`, and the other aggregates, which
  already passed their own flag to the aggregate computation) was honoured — so duplicates were
  emitted, and under a `LIMIT` they took the slots the next distinct value needed (`LIMIT 2` over
  `30, 30, 20` answered `[30, 30]`). It now takes the scope CQELS-QL gives `SELECT DISTINCT`: on a
  `RANGE` / `SLIDE` / `TRIPLES` window it deduplicates each evaluation's rows between the
  `ORDER BY` and the `LIMIT`, so `k` bounds the deduplicated sequence; on `[NOW]` it deduplicates
  within one arrival and carries nothing across arrivals; an aggregate query deduplicates the
  folded rows of one evaluation; and the continuous `ORDER BY ... LIMIT` top-K deduplicates on
  insert, so arrivals `30, 30, 20, 40` at `LIMIT 2` answer `[30]`, nothing, `[30, 20]`, `[40, 30]`
  (`CypherQLReturnDistinctTest`; the two `CypherQLLivenessTest` cells that pinned the old behaviour
  now assert the new one). The deduplication key is the RETURN columns (the alias on a projecting
  route), not the whole row, and `1` and `1.0` are one value, as for `GROUP BY`. A query without
  `DISTINCT` is unchanged.

  The RETURN items that have a column, and so can be deduplicated, are decided from the parsed
  item and are exactly the ones the query projects without `DISTINCT`: a variable with a property
  path (`v`, `v.p`, `v.p.q`, with a quoted variable), an alias, and a recognised aggregate. The
  resolver reads the first two components of a path, so on the static-lookup route `v.p.q` carries
  `v.p`; on the stream-only route it is a column only when a predicate's local name is `p.q`. An
  item with no column — a `CASE`, boolean operator, scalar function, literal, parameter,
  parenthesised expression, or arithmetic written outside an aggregate — is refused at
  registration with "RETURN DISTINCT needs a projected column", because every row would otherwise
  carry `null` for it and collapse into one. A backquoted PROPERTY component (``v.`p` ``),
  including one inside an aggregate argument such as ``COUNT(v.`p`)``, which counts nothing, is
  refused under `RETURN DISTINCT` with "RETURN DISTINCT over a quoted property is not supported
  yet", because no route resolves it (the matcher stores the bare
  property, the expression text keeps the backquotes), so the key `DISTINCT` reads would disagree
  with the one `ORDER BY` reads; without `DISTINCT` a quoted property behaves exactly as before
  (`CypherQLReturnDistinctTest.aQuotedPropertyWithoutDistinctKeepsItsBehaviour_characterization`,
  `aQuotedPropertyInAnAggregateArgumentWithoutOuterDistinctKeepsItsBehaviour_characterization`).
  A programmatically built `ReturnExpression` is analysed by parsing its text with the grammar and
  gets its textual twin's answer when written as the parser stores it (no whitespace): a column
  item spelled with spaces (`metric . value`) is refused under `RETURN DISTINCT` as a programmatic
  item, because no row carries that key, while the textual query accepts it. Known limitation
  (not changed here): aggregate detection runs before this check and finds an aggregate inside an
  expression, so `RETURN DISTINCT COUNT(metric) + 1 AS cnt` is accepted and computes the `COUNT`
  without the addition, with or without `DISTINCT`
  (`CypherQLReturnDistinctTest.distinctAcceptsExactlyWhatTheNonDistinctPathProjectsAColumnFor`,
  `distinctAndPlainAgreeOnTheStaticLookupRoute`,
  `aProgrammaticDefinitionGetsTheSameAnswerAsItsTextualTwin`).

- **`COALESCE`, `REGEX`, `REPLACE`, `STRDT`, `MD5`, `ENCODE_FOR_URI`, `STRBEFORE` and
  `TIMEZONE` now evaluate**. The eight standard SPARQL 1.1 functions were not in the
  expression evaluator's dispatch, so a query using one compiled and registered, and then a
  `FILTER` over it dropped every row and a `BIND` left its target unbound. Each now follows
  SPARQL 1.1 section 17.4, with the errors it defines (`Sparql11MissingBuiltInsTest`; a
  compiled query keeps the rows each function says it keeps in
  `UnknownFunctionNameCharacterizationTest`):
  - `STRBEFORE`, `STRDT`, `MD5` and `TIMEZONE` are evaluated by RDF4J's own implementations of
    them. `TIMEZONE` is restricted to `xsd:dateTime`, and it reads the XSD lexical form, so a
    dateTime with no timezone, an offset with seconds, or an impossible date is an error.
  - `REGEX`, `REPLACE` and `ENCODE_FOR_URI` follow the XPath functions SPARQL defines them
    by. They are not delegated, because RDF4J's versions differ from those definitions.
    - The regular expression is parsed by the regular-expression grammar of XQuery 1.0 and
      XPath 2.0 Functions and Operators, the version SPARQL 1.1 references, and the Java
      pattern is generated from the parse (`XPathRegex`, `XPathRegexTest`). Where the 3.1
      version differs, the 1.0 meaning is kept: `(?:...)` and the flag `q` are errors, and
      `.` without `s` matches a carriage return. The flags are
      `s`, `m`, `i` and `x`. Character-class subtraction works, `\d` is any decimal digit, and `x` removes
      whitespace. Without `m`, `$` matches only at the very end; with `m`, `^` and `$` follow
      XPath's line rules at the ends of the string. Under `i`, a `\p{..}` category keeps its
      meaning, inside a class too. A back-reference to a group that took no part in the match
      matches the empty string. The parser refuses, as an invalid pattern, the forms
      `XPathRegexTest` lists. These include `(?` groups, possessive quantifiers, an unmatched
      `)`, `]` or `}`, foreign escapes, a block name that is not a canonical Unicode 13.0
      block name (case included), a back-reference to a missing or unclosed group, a
      quantity above 2147483647, and nesting deeper than 256 groups or subtractions. A match that overflows `java.util.regex`'s stack is an error
      rather than a crash; this includes `REPLACE`'s check for a pattern that matches the
      empty string. Compiled patterns are cached by pattern and canonical flags (the last
      256; `"im"` and `"mi"` share an entry, invalid flags are never cached); a pattern longer
      than 4096 characters is compiled each time and never cached.
    - `REPLACE` expands `$N` by XPath's rules, so `$09` and `$0000000001` take their decimal
      value. It raises XPath's errors for a pattern that matches the empty string and for a
      malformed replacement.
    - `ENCODE_FOR_URI` encodes a character outside the Basic Multilingual Plane as its four
      UTF-8 bytes.
  - `COALESCE` evaluates its arguments in order and stops at the first that succeeds. An
    argument that throws, such as a failing extension function, is skipped as an error.

  Known limits:
  - The XML name escapes `\i`, `\I`, `\c` and `\C` are refused, where XPath accepts them
    (pinned).
  - Under `i`, characters and ranges use Java's Unicode case-insensitive matching, which folds
    further than XPath's case variants: `REGEX("İ", "[i]", "i")` is true here and false in
    XPath (pinned).
  - `java.util.regex` does not always restore a capture when it backtracks out of a repeated
    group (JDK-8187082). A group value, and a back-reference's test of whether its group took
    part, can then see a capture left on an abandoned path. For example,
    `REPLACE("abcd", "^(([a-z]([a-z]))+?|[a-z]+)[a-z]$", "[$3]")` gives `"[d]"` where XPath
    gives `"[]"` (pinned, `replace_staleCaptureFromAnAbandonedPath_javaLimitation_pinned`).
  - `xsd:dayTimeDuration` compares by its lexical form.

- **A string literal in a `FILTER`, `BIND`, `HAVING` or `SELECT` expression, or a
  `GROUP_CONCAT` separator, now decodes the escapes SPARQL defines in a string literal**.
  Previously only `\"` and `\'` were decoded, so `REGEX(?x, "\\d")` passed two
  backslashes to the matcher. The `ECHAR` escapes `\t`, `\b`, `\n`, `\r`, `\f` and `\\`
  (SPARQL 1.1 §19.7) are now decoded too. So are the §19.2 code-point escapes `\u` + four
  hex digits and `\U` + eight, which must name a Unicode scalar value. Code-point escapes are
  decoded inside string literals only, not across the whole query text as SPARQL specifies
  (`ExpressionEvaluator.decodeStringLiteral`, `Sparql11MissingBuiltInsTest`).

- **A stream name is now bounded at 256 UTF-8 bytes where a stream is created.** Nothing bounded
  the length of a stream name: every refusal and log line of the form "Stream 'name' …" copied it, and
  `STREAM_CREATE` journal records carried it, so an embedder or an MCP client could make the engine allocate
  input-sized strings per call and write unbounded names into the journal. The bound is one constant,
  `org.cqels.stream.StreamNames.MAX_NAME_BYTES` (`StreamNamesTest`), and is enforced at `CQELSEngine.createStream`
  (both overloads) and `getOrCreateStream` (both overloads), at `ReactiveStreamEngine.registerStream`, and at the
  MCP `create_stream`, `push_stream_events` and short-term `store_memory` tools (`stream`), which refuse before
  they read the rest of the request. The refusal is an `IllegalArgumentException` (an error result at the MCP
  tools) that states the name's length and the bound and does not contain the name, and it comes before any log
  line, attribute copy, capacity slot or journal write (`StreamNameBoundTest`, `StreamNameBoundToolsTest`). The
  count is in place, with no copy of the name, and a tool's `stream` argument is checked by type and length
  before it is converted to text (`aNonStringStreamArgumentIsRefusedWithoutBeingConverted`,
  `anOverlongCharSequenceIsMeasuredByLengthAndNeverRead`), and the only argument method that runs
  before that check is the one on the `memory` discriminator of `store_memory`, read once by type and length (a
  value of the wrong type, or a lying `CharSequence`, cannot send the two decisions different ways). The
  `graph`, `private` and `events` arguments are not converted before it
  (`graphPrivateAndEventsAreNotConvertedWhenAnOverlongNameIsRefused`,
  `theMemoryTypeIsReadOnceAndDecidesBothTheOrderingAndTheDispatch`). A stream name that is already live in the
  durable catalog is not affected: a journal written by an earlier release may hold a longer name, and recovery
  (`recreateStreamsOnStart`), `createStream` / `getOrCreateStream` adopting that name, `push_stream_events`,
  `create_stream`, `store_memory` and `remove_stream` on it all keep working; only a new over-long name is
  refused (`aReplayedStreamCreateRecordOverTheBound_stillRecovers`,
  `aStreamAJournalHoldsUnderALongerName_staysUsableAndRemovable`). The messages other than creation (for
  example the "not found" text of an unknown stream) are unchanged. The decision whether a long name is a new
  generation is made under the engine monitor where the generation is minted
  (`aCatalogLiveLongNameRemovedAfterTheEarlyCheck_isRefusedWhereTheGenerationIsMinted`), so removing the stream
  between the early check and the creation cannot slip a new long name in.

- **An MCP client that stops reading its event stream no longer stalls other queries' result signals**.
  A result signal (`notifications/resources/updated`) was written on the engine's result thread:
  MCP SDK 1.0.0 writes a session's notification to its event stream synchronously, and its broadcast
  blocks its caller until every session's write returns (a `parallelStream().forEach(... .block())` that
  also occupies common-pool workers), so a client that opened its stream and stopped reading filled the socket
  buffer and held the thread, delaying the results of every query that shared it. The engine's thread now
  only queues a signal: each session has its own bounded queue, drained in order by a fixed pool of writer
  threads of the notifier's own, started with it, so the engine's thread never writes to a socket nor
  creates a thread (`McpNotifierHandOffTest.aStalledSessionDoesNotDelayAnotherSessionsResultOnTheSameResultThread`,
  `moreStalledSessionsThanWritersNeverBlockTheResultThreadAndAddNoThread`). Sessions with pending signals
  take turns, one signal each (`recipientsWithPendingSignalsTakeTurnsOneSignalEach`). There is at most one
  queued drain per session queue, a queue is created only while its session is in the directory, and
  completing a session's end callback discards its pending signals, withdraws its queued drain and removes
  its queue, under the same lock as every submission, so session churn leaves nothing behind once those
  callbacks have run (the session leaves the directory just before its callback runs, so the bound is not
  instantaneous; a closed notifier keeps its emptied queues; a session the garbage collector cleared is
  only noticed at the directory's next expunge)
  (`anEndedSessionsQueueAndQueuedDrainAreReleasedSoChurnLeavesNothingBehind`,
  `aSessionThatEndsBetweenLookupAndQueueGetsNoQueueAndNoQueuedDrain`,
  `aSessionThatEndsWhileItsDrainIsSubmittedLeavesNoQueuedDrain`). The
  broadcast of the shared inferred view (`cqels://memory/inferred`) goes through the same per-session
  queues on HTTP instead of the SDK's broadcast
  (`McpNotifierHandOffTest.theSharedViewBroadcastIsQueuedPerSessionSoAStalledOneHoldsNoOther`). Server
  shutdown discards pending signals and does not wait for a blocked write, and a signal queued while the
  server closes is not kept behind a writer that is still blocked
  (`McpNotifierHandOffTest.closeDiscardsPendingSignalsWithoutWaitingForABlockedWrite`,
  `anEnqueueRacingCloseRetainsNothingBehindAWriterThatIgnoresInterrupts`). A write has no deadline: a
  writer blocked on a stalled HTTP event stream is freed when the transport completes or fails that write;
  however many sessions stall, they hold at most the pool's threads; shutdown interrupts the writers
  without awaiting their termination.

- **Unchecked `long` arithmetic at extreme timestamps wrapped instead of saturating.** The derived
  watermark (`StreamElementToStreamEventAdapter.lift`) computed `maxSeen - maxOutOfOrdernessMs - 1`, so with
  `maxOutOfOrderness = Duration.ofMillis(Long.MAX_VALUE)` and a seen timestamp of `-1` the trailing `- 1`
  wrapped to `Long.MAX_VALUE`, the watermark jumped to the end of time, and every later record was dropped as
  late; it now saturates at `Long.MIN_VALUE` and emits no watermark
  (`StreamElementToStreamEventAdapterTest.unboundedOutOfOrderness_atNegativeTimestamp_doesNotJumpWatermarkToEndOfTime`).
  `WindowDeltaOperator.withinAllowedLateness` computed `watermark - window.maxTimestamp() <= lateness`, which
  wrapped negative for a watermark near `Long.MAX_VALUE` and a window ending in the past and admitted a record
  for an expired window; it now compares against the saturated deadline `maxTimestamp + lateness`, the same
  deadline the purge path uses (`TimestampOverflowTest.lateRecordBeyondLateness_atExtremeWatermark_doesNotRecreateThePurgedWindow`).
  The sliding, tumbling and session assigners built `start + size` with plain addition, so a window near
  `Long.MAX_VALUE` got a negative end and was closed immediately; the end is now clamped to `Long.MAX_VALUE`,
  and a record AT `Long.MAX_VALUE` (the end-of-stream watermark sentinel, which no half-open window can
  contain) is rejected with `IllegalArgumentException`, as the sliding and tumbling assigners already rejected a
  record at `Long.MIN_VALUE` (the two session assigners accept `Long.MIN_VALUE`). That is one rule for the whole
  change: a record at `Long.MAX_VALUE` is rejected wherever its timestamp places it in a half-open event-time
  window (the event-time assigners, the legacy snapshot `TumblingWindow` / `SlidingWindow`, the Cypher `[NOW]` and
  `[RANGE]` aggregate windows, and the SPARQL `[RANGE]` and `[SLIDE]` windows), and accepted where the timestamp does
  not place it in such a window (the processing-time assigners, `GlobalWindows`, the watermark adapter,
  `SessionWindow` batches). The rejection is the contract, not a late-record drop: the query fails, because a
  `Long.MAX_VALUE` timestamp is a producer bug, not a late record. The `IllegalArgumentException` terminates the
  subscription (`WindowDeltaOperator` wraps it in a `RuntimeException` and ends the stream; the Cypher and SPARQL
  paths surface it as `onError`), so for the records `[0, MAX, 1]` the final record is never processed. Near
  `Long.MIN_VALUE`, `SlidingEventTimeWindows` no longer assigns a record to no window (its `timestamp - size`
  bound wrapped), a window whose start would lie below `Long.MIN_VALUE` is not assigned, and
  `TimeWindow.getWindowStartWithOffset` throws `IllegalArgumentException` instead of returning a wrapped start
  when no representable window exists; the sliding assigners no longer saturate the exclusive lower bound
  `timestamp - size` to `Long.MIN_VALUE`, which dropped a window starting at exactly `Long.MIN_VALUE`. The directional
  and `RANGE ... LATENESS` replay (`DirectionalWindowReplay`) had the same `maxSeen - lateness - 1` shape as the
  adapter and is saturated too, and `DirectionalEventTimeWindows` computed its grid offset with
  `(raw + step) % step`, which overflowed for a step above `Long.MAX_VALUE / 2` and shifted the grid by one
  (`Math.floorMod` of each operand now). The same saturation is applied to the other window, trigger and join
  sites that did unchecked timestamp arithmetic: the legacy snapshot `SlidingWindow` / `TumblingWindow` /
  `SessionWindow`, `TimeEvictor`, `ContinuousEventTimeTrigger`, `ContinuousProcessingTimeTrigger`,
  `ProcessingTimeoutTrigger`, the lateness checks in `WindowedJoinOperator`, the SWAG interval-join probe
  ranges (`WindowJoinState`, `ParallelJoinAggregator`, `JoinGraph.Edge`, and the buffered probe in
  `IntervalJoinOperator`, which also no longer negates its bounds — `-Long.MIN_VALUE` wrapped to itself and made
  `subMap` throw — and expires buffered records against exact, un-negated thresholds), which are computed in exact arithmetic and
  then restricted to the representable timestamps: a low end that overflows makes the range empty rather than
  clamping it into a false match (`TimestampMath.offsetRange` / `mirroredRange`), the lookup-join cache expiry, and the
  window-scoped `DISTINCT` lower edge. `SPARQLExecutor`'s `[SLIDE ...]` window loop never terminated for a record within
  one step of `Long.MAX_VALUE` (`w += slideS` wrapped negative while still `<= ts`); it now terminates, assigns only
  the representable aligned windows, and rejects a record at `Long.MAX_VALUE` (`SPARQLExecutorTest.slideWindowNearLongMaxValue_terminatesAndAssignsTheRepresentableWindows`). Tests: `TimestampOverflowTest`, `LegacyWindowTimestampOverflowTest`,
  `WindowedJoinOperatorTimestampOverflowTest`, `SwagJoinTimestampOverflowTest`, `TimestampMathTest`, `IntervalJoinOperatorTimestampOverflowTest`,
  `TemporalJoinDispatchCoResidenceOverflowTest`. The multi-pattern co-residence check
  (`TemporalJoinDispatch.constituentsCoReside`) compares exact sums instead of clamping a sum and then comparing.

- **Cypher comparison operators and aggregate names, the shared and programmatic aggregate dispatch, and JSON
  window duration units no longer read the JVM default locale**. Five more conversions on the
  query-compilation and evaluation path used the no-argument `String.toLowerCase()` / `toUpperCase()`, which
  read the process-global base default locale. Under a Turkish (`tr`) or Azeri (`az`) default, lower-casing
  `I` gives a dotless `ı` and upper-casing `i` gives a dotted `İ`, so the converted text no longer matched
  the keyword it was compared with. Four sites upper-cased a lower-case spelling: the CypherQL comparison
  operators `contains` and `in` (`CypherQLASTBuilder.parseOperator`) became `CONTAİNS` and `İN` and fell
  through to the default `EQ` operator, so a filter silently compared for equality instead (the upper-case
  spellings `CONTAINS` and `IN` already worked); a Cypher aggregate such as `min(...)` in a `RETURN` clause
  became `MİN` and was not recognised as an aggregate; and `CqelsQueryCompiler`'s per-group aggregate step
  and `AggregateComputation.compute` threw `Unknown aggregate function` for a name such as `min`. The fifth
  site lower-cased an upper-case spelling: a JSON window duration unit
  such as `MIN`, `MINUTES` or `MILLIS` became `mın`, `mınutes` or `mıllıs` and threw `Unknown duration unit`
  in `CqelsQueryDefinition`. All five now convert with `Locale.ROOT`, so every spelling that worked under an
  English default behaves the same under any default; the `CypherQLASTBuilder.visitDuration` unit conversion
  is moved to `Locale.ROOT` as well, although the grammar admits only `ms`, `s`, `m`, `h` and `d` there
  today. This closes the known issue that the released 2.0.0-alpha.21 entry on built-in function
  dispatch lists as outstanding, including the two aggregate-name sites it names as throwing under such a
  default. The new test (`LocaleIndependentSyntaxTest`) parses a CQELS-QL aggregate query and executes both
  that parsed definition and a programmatic aggregate definition, and also runs `AggregateComputation.compute`,
  the Cypher operators and aggregate, JSON window units, and a hand-built
  `CypherQLParser.DurationContext` carrying the long unit `minutes`, each under `en`, `tr` and `az` defaults,
  and restores all three default-locale categories in a `finally` block.

- **A `StreamDefinition` built with a `null` `WindowSpec` is now `[NOW]`**. The typed
  `StreamDefinition(String, WindowSpec, Map)` constructor used to keep the `null`, for which
  `isNow()` was false, so through `CqelsQueryCompiler` a single-stream definition whose pattern needs
  the windowed join (or that carries aggregates) reached a join over a buffer that never evicted and
  paired statements from different arrivals; a fixed-predicate single-triple query without
  aggregates already took the per-arrival route. The constructor now stores `WindowSpec.now()`, as
  the string constructor already did, so evaluation uses only that arrival's statements
  (`NullWindowSpecRegressionTest`; `RouteDivergenceProbeTest` P3e): separate arrivals cannot complete
  a multi-triple pattern (also through a named-graph lookup, or a `FILTER(SEQ(...))` handed to
  `CqelsQueryCompiler`), a `GROUP BY` aggregate folds per arrival, and `SELECT DISTINCT`
  deduplicates within an arrival. Two consumers are unchanged: a definition with several
  `FROM STREAM` sources (temporal dispatch maps a null and a NOW spec to the same zero-length
  window) and the dedicated CEP compiler, which derives no `within` constraint from either
  (`TypedNullWindowCepTest`). These shapes registered with a typed null and are now refused, as under
  a written `[NOW]`: a stream-only join-shaped pattern with a bare `ORDER BY` or a bare `LIMIT`;
  `GROUP BY` with aggregates and a bare `ORDER BY`, a bare `LIMIT`, or both together (stream-only or
  composed); an aggregate without `GROUP BY`, also when only in `HAVING`; and `Istream(...)` over a
  join-shaped pattern (at `execute()`). A composed join-shaped query with a bare `ORDER BY` or
  `LIMIT` was already refused, and its `ORDER BY` plus `LIMIT` already used the continuous top-K.
  Embedders that need state across arrivals must pass an explicit window such as
  `WindowSpec.triples(10)`; see `docs/MIGRATION_GUIDE.md` for the table.

- **CypherQL `RANGE` and `SLIDE` windows now close on a live stream**. A query declaring
  `[RANGE d]` or `[SLIDE d STEP s]` emitted nothing on an engine-registered stream, because
  `TumblingWindow.apply` / `SlidingWindow.apply` and the aggregate `RANGE` branch collected the
  whole source and fired only when it completed. Both CypherQL routes, with and without
  aggregates, now run those windows through `TumblingWindow` / `SlidingWindow.applyEvents` (the
  event-time `WindowDeltaOperator`) with a watermark taken from the elements' own timestamps, so a
  window `[a, e)` closes on the first element whose timestamp is at or past `e` — the instant a
  CQELS-QL tumbling `RANGE` window closes on the incremental aggregate route without `LATENESS`
  (with `LATENESS d` that route holds its watermark back and closes later) — and the windows
  still open are flushed when the source completes. The wall clock plays no part: a quiet stream
  closes nothing (`CypherQLLivenessTest`, `CypherTimeWindowLiveStreamTest`). Two inputs the old completion-time
  batching accepted are now treated differently: an element that arrives after its window has
  closed is dropped as late (an out-of-order element that falls in a still-open window is kept,
  and under `SLIDE` it is counted in its open windows only), and window starts follow the floor of
  the timestamp — a `RANGE` window may start before epoch 0, so `[RANGE 1s]` now puts `-1 ms` and
  `0 ms` in separate windows where the old batching grouped them, while a `SLIDE` window starting
  before epoch 0 is not assigned (`SlidingWindow.applyEvents` skips negative starts), so a
  negative timestamp belongs to no `SLIDE` window. The late rule applies at any timestamp; the
  window-start change affects negative timestamps under both windows (the old batching truncated
  towards 0, so `[RANGE 1s]` grouped `-10001 ms` with `-10000 ms`), and under `SLIDE` also the
  non-negative timestamps within one window length of 0. `RETURN DISTINCT`, `ORDER BY` and
  `LIMIT` apply to each window as it closes, deduplicating and ranking that window's rows on its
  own, and a record at `Long.MAX_VALUE` fails the query with `IllegalArgumentException` on both
  routes of both windows, as on the `[NOW]` aggregate path (`CypherQLLivenessTest`).
  `[TRIPLES n]` is unchanged.

- **On the stream-only windowed route, `OPTIONAL` and `UNION` now reach the aggregate fold even when
  the block's join key is not projected**. `executeWindowedHashJoin` applies `UNION` / `OPTIONAL`
  inside its per-arrival batch before the aggregate fold (or the `ORDER BY` / `LIMIT` tail), but the
  batch's rows are `mapBindingsToRecord`'s projection — `SELECT` variables, aggregate arguments and
  `GROUP BY` keys — so a bare `COUNT(*)` left the block's subject out of the row it probed, the probe
  matched nothing, and an `OPTIONAL` null-filled every row (counts `1, 2, 3`, the bare window sizes)
  while a `UNION` dropped every row (no result at all). It is the unprojected-join-key mechanism
  fixed on the composed route; `COUNT(?s)` alone already answered the fan-out here. The batch now
  carries the `OPTIONAL` join keys and the `UNION` arm variables into its rows and strips them again before
  the fold, so the same query over the same data gives the same count on every route: the `OPTIONAL` above
  answers `2, 3, 4` and the `UNION` `2, 3, 3` (`RouteBehaviourMatrixTest` F1, F1c: the carried key
  does not leak into the output). Three pins of the same defect in `AggregateFoldAfterReshaperTest` flipped
  with it. `FILTER NOT EXISTS` keys are not carried here and are unchanged.

- The CypherQL refusal for `RETURN DISTINCT` over a backquoted (quoted) property cited a private tracker issue number in its exception message, a form the disclosure gates, which match only the `#<digits>` form, did not catch. The message now reads `RETURN DISTINCT over a quoted property is not supported yet: quoted property names are not resolved on any route`, followed by the item text and query id as before (`CypherQLReturnDistinctTest`).

- **On the lookup routes, a static BGP that writes a triple the candidate fallback cannot serve before the
  triple that anchors it now answers the same in every written order for the reported shapes: a leading open
  triple, a predicate-only anchor, two mutually anchoring triples, and cycles of up to four triples.**
  Other shapes can still depend on the written order, as listed below. The static
  matcher asks the repository for a triple's candidates only when something in it is resolved, and judged
  that in written order, so `?x ?px ?o . ?x ex:name "Alice"` answered nothing on the composed and the
  per-arrival lookup routes while `?x ex:name "Alice" . ?x ?px ?o` answered `Alice`'s facts; the same held
  for a BGP led by `?s ?p ?o`, and for an `OPTIONAL` / `UNION` / `FILTER NOT EXISTS` block that names a
  variable before the triple that binds it. When a candidate fallback is present the matcher now evaluates
  the written order exactly as before and then a promoted order (a resolved subject first, then a resolved
  object, then a resolved predicate; ties keep written order), and adds the promoted order's solutions
  that the written order did not find. A third, re-evaluated order then defers a triple that has nothing
  resolved when its turn comes and whose subject a later triple binds, so it is evaluated under that binding
  instead of against the forward view alone, and its new solutions are added the same way: two such triples
  behind an anchor, `?x ?p ?y . ?y ?q ?z`, answered in one written order and not in the other.
  Where no such triple can be deferred because each remaining one has its subject bound by another, a
  cycle such as `?x ?p ?y . ?y ?q ?x`, each written order answered a different solution; for a cycle of
  k such triples, k up to four, the re-evaluated order now runs once per member, each time with a
  different member first. Each re-evaluated order runs only where it differs from every other
  order. The written order's rows are kept unchanged and first, so each completed matcher invocation
  returns a superset of the rows it returned before. This is a guarantee per invocation, not per query:
  operators downstream of the matcher (`FILTER NOT EXISTS`, `OPTIONAL`, an ordered `LIMIT`) see the extra
  rows and can therefore answer differently, which is the intended correction. A reorder alone would not be monotone: a triple moved ahead
  can bind a subject from the forward view, after which the matcher no longer asks the repository for
  that subject, which is why the earlier reorder attempt was parked. The promoted and re-evaluated
  orders share one separate allowance the size of the existing work guard (`cqels.join.crossProductCap` x
  pattern count), or k times that size when a cycle of k triples is seeded, shared across the evaluation
  like the guard. A static BGP whose other orders differ does the written order's work as before, plus a
  separate allowance of one times the guard however many other orders run, or k times the guard for a seeded
  cycle of k triples (k at most four); the work is counted in successful binding extensions, not in candidate
  fetches or scans. The guard itself is charged exactly what the written order spends and fires as before. When the other
  orders exceed the allowance all their rows are dropped and the written order's answer stands. Still
  order-dependent: a cycle of more than four such triples, which is not seeded and keeps the written order
  (`StaticBgpMatcherTest.fiveMemberCycle_isOverTheSeedingLimit_andStaysDeclarationOrderDependent`), and the
  general interaction between the view and the fallback: a triple that binds a subject from the forward view
  in every order tried means a later triple on that subject never asks the repository, so a BGP such as
  `?s ?p ?o . ?s ex:outside ?v . ?s ex:anchor X` can answer differently from the same triples written
  `outside`, `anchor`, then the open triple, with no cycle involved (the interaction that parked the
  earlier reorder attempt, pinned by
  `StaticBgpMatcherTest.aSubjectBoundFromTheViewInEveryOrder_stillMakesTheAnswerOrderDependent`). The
  note in the `GRAPH` entry above, that an unconstrained triple written before the anchor sees only the
  forward view, therefore still holds for those shapes. Pinned by `StaticBgpMatcherTest`
  (`theAnswerIsTheDeclarationOrdersRowsPlusThePromotedOrdersNewOnes` runs every order on its own and
  asserts the union; `mutuallyAnchoringUnanchoredPatterns_areDeclarationOrderIndependent` and
  `mutuallyAnchoringUnanchoredPatterns_theZeroRowOrder_isAnsweredByTheReevaluation`;
  `mutuallyAnchoringCycle_isOrderIndependent_closes1131`,
  `threeMemberCycle_answersEveryRotation_inEveryWrittenOrder`,
  `aSeededCycleSpendsKTimesTheGuardFromTheAllowance` and `aCycleSeedOverTheSharedAllowance_isDropped`),
  `ComposedUnjoinableStaticBgpTest.staticBgpAnswer_isDeclarationOrderIndependent_onBothLookupRoutes`
  and `resolvedSubjectWithUnboundPredicateAndObject_stillGetsCandidates`, and
  `CqelsQLStaticBgpCompleteMatchTest.optionalAnswerDoesNotDependOnTheDeclarationOrderOfItsPatterns`.
- **`routeIgnoresDeclaredWindow` no longer reports `true` for the directional and temporal-join routes, and the
  refusal it leaked now names its real cause.** The predicate was computed from
  `needsWindowedJoinEvaluation` alone, which knows nothing about those two routes, so it was `true` for a
  `[FUTURE 1s]` single-pattern SELECT (`executeDirectionalSelect`, which assigns its LARS windows from that exact
  spec) and for a two-`STREAM` join (`executeTemporalJoin`, whose interval bounds are the two declared durations),
  although its documentation names only `executeStreamOnlyWithExpressions` and `executeWithLookupJoin`. It is now
  `false` for any query that declares a directional window or routes to the temporal join, so it is `true` only on
  the two per-arrival routes. The misreport was masked everywhere but one query: a directional window beside a static
  graph pattern with `ORDER BY` / `LIMIT` was refused at registration with "routes through a per-arrival evaluation
  that does not honour windows at all", which is false of any directional query. Registration now refuses it with a
  message that names the directional window beside a static graph pattern, which is what leaves the query with no
  execution route (`ORDER BY` alone and `LIMIT` alone get it too). The same query without the static graph pattern
  is still accepted. Pinned by `RouteSelectionOracleTest.routeIgnoresDeclaredWindowIsTrueOnlyOnThePerArrivalRoutes`
  (the former `...IsTrueOnTwoRoutesThatDoFormRowsFromTheirWindow`, flipped to `false` for both routes),
  `directionalWithStaticBgpAndOrderByLimitIsRefusedForTheStaticBgpNotForIgnoringWindows`, the
  `selectionPredicateVectorsPerRoute` `IW` column, and the `FUTURE` / `PASTFUTURE` `single|orderlimit|static` cells
  of `routeSelectionMatrix`.
- The temporal-join `FROM STATIC` guard that rejects zero or several declared graphs said the parser discarded each
  `GRAPH` block's IRI. The IRI is recorded and checked against the declared graphs; what flattening loses is the link
  from each triple pattern to its block, which is what blocks per-block lookup scoping. The message now says so
  (`CqelsQueryCompilerTemporalJoinStaticTest.temporalJoinGraphBlockWithZeroDeclaredGraphsNamesLostPatternBlockLink`).

- **A stream that no query reads no longer keeps every pushed element on the heap.** Each stream keeps a
  replay history for queries registered later, and it was bounded by element count only
  (`cqels.engine.streamReplayLimit`, 8192 by default, now within 64 MiB, which a query registered later receives,
  not a CEP query, which sees none of it), so 1,000 pushed elements of 500 KiB stayed reachable after GC (about 523 MB) with or
  without a memory store, and an MCP `push_stream_events` client could grow the heap with no consumer registered.
  The history is now also bounded by the estimated bytes of its elements: `cqels.engine.streamReplayMaxBytes`,
  64 MiB per stream by default (two bytes per char of each RDF term plus a fixed overhead per term and per
  element; any other payload counts as the overhead only). The history is trimmed to both bounds on every push,
  whoever is subscribed. A subscriber that has not yet requested an element still receives every element pushed
  since it subscribed: they are kept for that subscriber alone until it reads them or leaves. As with Reactor's
  replay sink, that backlog is bounded by neither limit, and each element it holds also costs a queue node of
  about 24 to 32 bytes for that subscriber. Pinned by `DataStreamNoSubscriberRetentionTest` and
  `BoundedReplaySinkTest`.
- **The outer group's `FILTER` now runs after its `OPTIONAL` / `UNION` blocks and its `BIND`s on every route
  (CQELS4J#85, CQELS4J#83).** SPARQL 1.1 §18.2.2 applies a group's filters to the whole
  translated group, its `LeftJoin`s and `Extend`s included, but where the `FILTER` ran depended on the route.
  On a stream-only query (`[NOW]`, `[TRIPLES n]`, `[RANGE]`, with or without `GROUP BY`) and on a windowed
  `GROUP BY` query with a static pattern, it ran before the `OPTIONAL`: over background `o1 ex:label 8` and one
  element `o1 ex:v 3 . o2 ex:v 4`, `OPTIONAL { ?o ex:label ?l } FILTER(!BOUND(?l))` answered o1 and o2 instead
  of o2, and `FILTER(BOUND(?l))` and `FILTER(?l = 8)` answered nothing instead of o1. On both stream+static
  routes it ran before `BIND`, so `BIND(7 AS ?z) FILTER(?z > 0)` answered 0 rows where the stream-only route
  answered 3, and the windowed `GROUP BY` topology answered no row at all — including for a `BIND` above the
  `OPTIONAL`, which the per-arrival routes already honoured. And on the stream-only routes and that
  `GROUP BY` topology a `BIND` written below an `OPTIONAL` ran before it: `OPTIONAL { ?s ex:label ?ol }
  BIND(BOUND(?ol) AS ?u)` gave `?u` false on rows where `?ol` was bound. On every route a `BIND` written
  between two positional blocks ran after both, so a `UNION` below it probing its target matched no arm and
  dropped every row, and the blocks applied every `UNION` before every `OPTIONAL` whatever their order in the
  text. Every route now runs the group left to right: the blocks in declaration order, each `BIND` after the
  blocks above it and before those below it, then the outer group's `FILTER`s, then `FILTER NOT EXISTS`; the
  definition records each `BIND`'s block position (`CqelsQueryDefinition.getBindingBlockPositions`,
  `getPositionalBlockOrder`), and `CqelsQueryDefinition.Builder.positionalLayout` carries both through a
  rebuild. A stream-only query with such a block carries
  the unprojected variables its `FILTER` and lower `BIND`s read past the blocks and strips them again, so its
  output keys are unchanged; those stages read the RDF term, language tag and datatype included, of every
  value the stream pattern, a `BIND` or an `OPTIONAL` / `UNION` block bound into the row, where they used to
  read its row lift (the static-lookup routes still read the row lift), and a row whose projection only
  such a `BIND` fills is kept until it has run. A
  `BIND` written between two blocks runs once, at its position, on every route. `FilterAfterOptionalRouteTest` pins the reported table on ten route variants
  (stream-only `[NOW]`, `[TRIPLES 2]` and `[RANGE 10s]`, each with and without `GROUP BY`; the windowed static
  `GROUP BY` route under `[TRIPLES 2]` and `[RANGE 10s]`; the two per-arrival static controls) with both
  controls (`FILTER(?x > 3)`, `FILTER NOT EXISTS`) and the two `BIND` positions; `RouteBehaviourMatrixTest`
  F9h pins a `BIND` before, between and after three blocks on the stream-only, windowed and composed routes;
  cells E2, E2b, E4, E4b, E4d, E5, E9, F1b, F3b, F3c, F6b, F9c and F9f, and
  `AggregateFoldAfterReshaperTest.composedFilterRunsAfterTheUnionMerge`, now assert the SPARQL answer
  where they pinned the old one; `DeferredGroupExpressionRouteTest` pins `LANG`, `DATATYPE`, `sameTerm` and
  `xsd:dayTimeDuration` filters over stream-, `BIND`-, `OPTIONAL`- and `UNION`-bound values on six stream-only
  branches, the `BIND`-filled projection, the between-block
  `BIND` on both static outer pipelines, and a rebuilt definition. Not changed here: the order inside an
  `OPTIONAL` block, whose own `FILTER`s still run before its own `BIND`s; a `BIND` whose target the block also
  binds (F9e); and an aggregate over a batch that a `FILTER` empties, which still emits no row instead of
  `COUNT = 0`.
- **The static-lookup operator's enrichment cache now has a maximum entry count.** It was bounded by its
  `CACHE` TTL only, and one arrival can insert up to one entry per statement (one forward view per distinct seed
  root), so a long TTL over a wide key space retained every subject it ever looked up. `EnrichedLookupJoinOperator`
  now holds at most 100,000 subjects by default (`-Dcqels.join.lookupCacheMaxEntries=<n>`, or
  `Builder.maxCacheEntries`): ten maximal 10,000-statement arrivals' roots within one TTL. Past the bound it evicts
  expired entries first, then the oldest-inserted, down to nine tenths of the bound; TTL expiry and
  repository-commit invalidation are unchanged (`EnrichedLookupJoinOperatorTest`
  `cacheHoldsAtMostItsBound_andStillServesARecentHitWithinTtl`, `cacheBoundDefaultsToTheSystemProperty`).
- **Repeated `execute()` calls on one compiled query no longer accumulate lookup operators.** Every
  `execute()` of a query with a static pattern (CQELS-QL or CypherQL) added an `EnrichedLookupJoinOperator` to
  `getLookupOperators()` and nothing removed it: 1,000 calls left 1,002 operators, each kept with its cache, and
  the per-call cost grew with the set (+70 % at 5,402). An operator is now in that set only while a subscription
  to the `Flux` `execute()` returned is live: it is added when the first subscriber arrives and removed when the
  last one completes, errors or is cancelled, so the engine's commit listener still reaches every live query
  (`LookupOperatorsCharacterizationTest` `repeatedExecuteSubscribeCancel_leavesLookupOperatorsAtPreCallSize`
  and its CypherQL twin; `StaticFactRefreshTest` `cachedLookupOfALiveSubscriptionIsInvalidatedOnRepoMutation`
  and its CypherQL twin pin invalidation with a `CACHE` TTL). The three route pins in
  `LookupOperatorsCharacterizationTest` now assert registration while subscribed and removal on cancel and on
  completion, where they asserted the operator stayed after completion. A normally registered engine query
  (one `execute()`, one subscription) behaves as before.
- **The static lookup's repository candidate fetch is now bounded by the work guard.** The
  candidate fallback that serves a static pattern the forward lookup view cannot reach (a reverse edge into the
  join key, an unbound-subject pattern, a constant subject outside the view) read the repository's whole answer
  into a list, deduplicated it, and only then matched it, while the work guard (`cqels.join.crossProductCap` x
  the static pattern count) counted successful extensions alone: `?g a ex:Field` over a million `ex:Field`
  instances read and deduplicated a million statements before the guard could fire. The fetch now streams the
  answer, deduplicates each statement as it reads it, and stops once it holds the work bound plus one; a pattern
  whose extent exceeds the bound fails with the work guard's own error, naming the pattern and the bound, after
  reading bound + 1 statements, and is never matched against a truncated extent. That bound is a constant of the
  evaluation, so the per-arrival candidate memo holds at most that many candidates per request. The refusal follows
  each route's existing rule: the composed windowed route fails the query, the per-arrival route drops the arrival
  with its once-per-query WARN. A pattern whose extent is within the bound answers as before on every route, and
  each arrival on the composed route still reads one extent per distinct request
  (`ComposedUnjoinableStaticBgpTest` `overBudgetCandidateExtent_isRefusedAtTheFetch_onEveryLookupRoute`,
  `candidateExtentAtTheBound_answersAsBefore_onEveryLookupRoute`, and
  `unjoinableStaticBgp_fetchesCandidatesOncePerArrival_notOncePerRow`, which now also counts statements read;
  `StaticBgpMatcherTest` `aFallbackExtentOverTheBound_isRefusedAtTheFetch_afterBoundPlusOneReads`,
  `theFetchLimitIsTheGuardsBound_andAnOverlongAnswerIsRefusedWhateverTheSource`,
  `aFallbackExtentAtTheBound_isFetchedWhole_andAnswered`). `unjoinableStaticBgp_isBoundedByTheWorkGuardAndFailsLoud`
  now asserts the refusal at the fetch and 6 statements read, where it asserted the extension guard's message
  after all 42 were read.
- **An unrecognized `format` is now an error instead of JSON.** `OutputFormat.fromString` returned JSON for
  any value outside its accepted set, although the tool schemas declare a strict `json` / `turtle` / `natural` enum,
  so a misspelled format silently changed the output shape. It now fails the tool call with a message naming the
  accepted values and echoing the bad one; `null`, absent and blank still mean the default JSON, and matching stays
  case-insensitive. `OutputFormatTest.fromStringUnknownIsRejectedNamingAcceptedValues` and
  `ReasoningToolHandlerTest.reasonWithUnknownFormatReturnsErrorNamingAcceptedValues` pin it.
- **ASP: two aggregates that each bind a variable the other one uses no longer drop the rule.** In a rule
  such as `q :- X = #count { Y : p(Y) }, Y = #count { X : p(X) }, r(X), s(Y).`, the aggregate rewriting of the
  vendored Delta solver looked for the binders of an aggregate's global variables in the whole rule body, so each
  aggregate became a dependency of the other; the rewritten encodings then formed a positive loop, neither aggregate
  result was ever derived, and the answer set silently lacked `q`. `AggregateRewritingRuleAnalysis` now searches only
  the body literals that become evaluable before the aggregate (the same rounds as a safety check), so the
  dependencies between the aggregates of one rule are acyclic, and the rule holds whenever its globals have a
  non-circular binding. The upstream test `AggregateRewritingTest.setComplexEqualityWithGlobals`, disabled upstream
  for this reason, is re-enabled and passes; `AggregateRewritingTest.mutualOutputVariablesOfTwoAggregates` and
  `AggregateRewritingRuleAnalysisTest.mutualOutputVariablesDoNotDependOnEachOther` pin the two-aggregate shape.
  A rule whose globals are bound only through each other (no other binder) is still rejected as unsafe
  (`AggregateRewritingRuleAnalysisTest.mutualOutputVariablesWithoutOtherBinderAreRejected`).

- **A path whose closure body is a sequence or holds a closure is no longer refused on windows it fits
  (`cqels-core`).** The pre-flight bound of `(ex:p/ex:q)+`, `(ex:p+/ex:q)+` and every other closure whose body is
  more than one edge step came from the window's index sizes alone, and refused, for `?x (ex:p/ex:q)+ ?y` with
  neither end bound, 5,000-triple stars and trees at 21,000,000 units or more when the work was under 125,000.
  When that size bound is over the budget, `PathPreflight` now measures the body's graph instead: the evaluator
  hops the body once from each term of the window (and from one term outside it, for a constant the window does
  not mention), and the walks are bounded over that graph as they already were for a one-step body, so the bound is
  the exact work on chains, stars and trees. A path the size bound admits costs the pre-flight nothing more than
  before. The hops of one closure in one direction are capped at `cap = 16 * (min(T, W) + 64)` units, for T triples
  in the window and W units of work admitted per partial binding, and the hops of every closure of one pattern
  together at `2 * cap`, where a unit is a unit of work, a look-up of the index (one per predicate of a step, whether
  it finds an edge or not), a term hopped from or a binding of a row the hops copy; past a cap the size bound
  applies. The query-shaped structures the hops read (each negated set's exclusion set, each predicate's IRI, the arms
  reversed for a backward walk and grouped by the steps they share) are built once, with the pattern, instead of on
  every hop, so the measurement visits at most 4 elements per unit it spends on the adversarial corpus its test runs
  (a negated set of 100,000 IRIs, 1,024 arms sharing prefixes, 1,000 closures, nested and 2,000-step bodies), and
  the hops are stopped, and the size bound applies, before they visit more than 4 elements per unit of their cap.
  A `PathPlan.PredicateSetStep` that is positive and empty, which no query yields, is refused when it is built. A
  path refused for its size quotes its first 200 characters, with its arm and step counts, instead of the whole
  path. A refusal says whether its bound was read from the index or measured. The bound still never undercounts
  (the randomized test runs every operator at a budget equal to its bound).

- **`forget_memory` refuses a raw SPARQL `LOAD`.** A `sparql` update
  containing `LOAD` made the server fetch a URL the client chose, including
  internal hosts and local `file:` paths; it is now rejected before it runs. When
  the server installs the query guard, each operation's WHERE evaluation in the
  raw update also runs under RDF4J's per-operation execution timeout (30 s by
  default); this is not a deadline for the whole call.

- **`SELECT Istream ?a ?b` and `SELECT DISTINCT Istream(...)` now say what to write** (`cqels-core`). The
  syntax error keeps its text and gains a suffix: `; Istream needs parentheses: SELECT Istream(?a ?b)` after an
  `Istream`/`ISTREAM`/`istream` keyword in the projection followed by a variable or `*`, and `; DISTINCT goes inside:
  Istream(DISTINCT ...)` after `SELECT DISTINCT Istream`. A query or stream that is merely named `Istream`, and every
  other syntax error, are unchanged.

- **Re-storing a fact no longer grows the semantic index.** Every document of a statement now carries
  a `key` field (the SHA-256 of its quad: subject, predicate, object with its datatype and language, and named
  graph), and `SemanticStatementIndexer.index` replaces whatever that key already holds. Storing a fact twice,
  or re-storing a chunked literal, no longer adds duplicate documents or skews BM25 statistics. Delete uses the
  same key. The key is injective over well-formed terms, with ASCII language tags compared case-insensitively as BCP 47
  requires, and a statement is hashed once however many
  passages its literal splits into. A subject, predicate, object, graph, literal datatype or language tag larger
  than Lucene's 32,766-byte term limit is refused with an `IllegalArgumentException` naming the field before any
  passage is embedded.
- **Search hits carry the literal's language, datatype and matched passage.** `SemanticSearchHit` gains
  `language()`, `datatype()` and `passage()`. `passage()` is the matched chunk (at most 512 characters) and is
  set only when the literal was split into several passages; otherwise it is null and `object()` is the full
  text. The 5-argument constructor is kept.

- **The semantic index refuses text with an unpaired UTF-16 surrogate instead of hashing it to another
  statement's key.** In unreleased snapshots only, UTF-8 encoding replaced an unpaired surrogate with
  `?`, so a literal holding a lone U+D800 and the literal `"?"` got the same `key`, and indexing or deleting
  one replaced or removed the other. `SemanticStatementIndexer.index` and `delete` and
  `RdfDocumentMapper.statementKey` now throw `IllegalArgumentException` naming the field when a subject,
  predicate, object, graph, literal datatype or language tag, or any text inside a quoted triple, holds an
  unpaired surrogate. Control characters such as U+0001 and valid surrogate pairs are still accepted, and the
  key of every other statement is unchanged.
- **`index` and `delete` throw `IllegalArgumentException` for a term with an unpaired surrogate, and `delete`
  also for a term over the size limit, input that alpha.21 accepted.** alpha.21 accepted lone
  surrogates, and its Lucene term-based delete replaced them with U+FFFD. Its `index` already refused a term
  over the 32,766-byte limit, through Lucene after chunking and embedding; `index` now refuses it before
  anything is hashed or embedded, and `delete` refuses it too. `deleteAll` skips a statement that `index` would refuse, since it
  can never have been indexed, and still deletes the rest of the batch, while a single `delete` of such a
  statement refuses. The size check counts UTF-8 bytes in one pass and stops at the limit instead of encoding
  the whole term. `HashingEmbeddingProvider` refuses a synonym map whose key or value holds an unpaired
  surrogate.

- **An RDF literal ending in a backslash no longer breaks ASP solving.**
  `AspFactMapper` writes such a label as `"...\\"`, and the Delta lexer read the
  final `\"` as an escaped quote, so the whole ASP program failed to parse: the
  query's subscription ended, or under `failClosedOnSolverError` every later solve
  was withheld. `QUOTED_STRING` now accepts `\\` as an escaped backslash.

- **`GeoSpatialIndex.queryMatches` no longer drops matches of a disjoint relation named by its full IRI, or of
  `geof:relate`.** The index prunes candidates by envelope intersection except for relations that can hold
  between geometries whose envelopes do not meet. It read a full IRI's local name after its last `:`, so
  `http://www.opengis.net/def/function/geosparql/sfDisjoint` was pruned like a non-disjoint relation, and it pruned
  `geof:relate`, whose DE-9IM pattern (for example `FF*FF****`) can require disjointness. In both cases a geometry
  whose envelope misses the query's was never checked and so never returned; geometries whose envelope meets the
  query's were still checked exactly. Both now scan every entry, and so does a name padded with whitespace
  (`" geof:sfDisjoint "`), which was likewise pruned; the unpadded `geof:sfDisjoint` prefixed form was already
  correct.

- **ASP solves no longer pay a fixed ~20 ms classpath scan each.** The
  standard-library externals are scanned once per JVM, and the aggregate
  encoders are built once per encoding choice instead of once per solve. A
  1k-fact program without aggregates now solves in ~8 ms instead of ~26 ms;
  large windows, dominated by stratified evaluation, are unchanged.

- **An EWKT `SRID=n;` prefix other than `SRID=4326;` is now a type error in every `geof:` function except `geof:getSRID` (which does not normalise its argument), and in the spatial index.** Before, any `SRID=n;` was stripped and the coordinates were read as CRS84 lon/lat. So `SRID=3857;POINT(1500000 6800000)` was indexed, and projected metres were compared as degrees with no error. Other SRIDs are now refused like an unknown CRS IRI: counted in `WktNormalizer.unknownCrsCount()` and logged at WARN once per value, up to 256 distinct values (the cap is shared with CRS IRIs). `SRID=0;` and the leading-zero form `SRID=04326;` are refused as well. A malformed prefix such as `SRID=abc;` is a type error too (it used to be stripped).

- **`geof:buffer` with a NaN, infinite or non-numeric radius is now a type error.**
  It returned `"POLYGON EMPTY"^^geo:wktLiteral` for a NaN or infinite radius in `uom:metre`,
  `uom:degree` and `uom:radian` alike. A plain string such as `"100"` was buffered as a number in
  metres and degrees. The radius must now be a well-formed numeric literal (`xsd:double`,
  `xsd:decimal`, `xsd:integer`, …, so `"1.5"^^xsd:integer` is refused too) with a finite value, in
  every unit. A negative radius still shrinks the geometry, as in JTS. A radius too large for
  `uom:radian` or `uom:degree` is a separate matter.
- **`GeoSparqlSimpleReasoner.infer` keeps the named graph.** An inferred statement now
  carries the context of the statement it is inferred from instead of always landing in the
  default graph.

- **A runtime exception thrown inside RDF4J's GeoSPARQL implementation is now a type error.**
  `GeoSparqlFunctions.evaluateFunction` caught only `ValueExprEvaluationException`, so for example a JTS
  `TopologyException` from `geof:buffer` with a degree radius of roughly 919 to 2340 degrees (intermittently within
  that range) escaped to the query evaluator instead of being a type error. Any `RuntimeException` from the RDF4J call
  is now a type error, logged at DEBUG (an `Error` still propagates). Radii above 180 degrees keep RDF4J's result.

- **Concurrent ASP solves no longer return wrong answer sets.** The Delta solver numbered the element tuples of `#count` and `#sum` aggregates, and user-level enumeration atoms, in one static table that every normalization cleared, so a solve running alongside another, or interleaved with another normalization, could give two tuples the same number and derive a smaller count or sum. Each compiled program, that is each solve, now numbers its enumeration atoms in a table of its own. The rule, nogood and choice ids that solves draw from shared static generators are now handed out atomically, so solves that reach the lazy grounder (choice rules, or with stratified evaluation turned off) no longer collide either.

- **A windowed aggregate on the incremental route no longer drops a group whose aggregate argument
  is unbound.** This affects a single-pattern `COUNT(?x)` / `SUM` / `AVG` / `MIN` / `MAX`
  over `[RANGE d]`, `[SLIDE d STEP s EMIT ON_CLOSE]`, `[TRIPLES n EMIT ON_CLOSE]`, a `LATENESS`
  window or a directional (`FUTURE`) window. The route discarded every solution whose argument was
  unbound or an expression error, such as `BIND(1 / 0 AS ?x)`. A group whose solutions all had such
  an argument vanished from that window's rows: `GROUP BY ?s` with `COUNT(?x)` answered nothing
  instead of `?n = 0` for each subject. Those solutions now keep their group in the window and
  contribute nothing to the aggregate. That matches the batch route, which already skipped an
  unbound argument: the group reports `COUNT` 0 and `SUM` 0, and `AVG` / `MIN` / `MAX` appear with
  a null value. A mixed group aggregates only its bound arguments. `AVG` of a group with no bound
  argument is null on both routes, where SPARQL 1.1 gives 0; that is a known deviation, unchanged
  here. Without `GROUP BY`, a window none of whose solutions binds the argument now reports its
  row (`COUNT` 0) on a directional window, as the directional replay does; on a legacy window it
  still emits no row, as the windowed hash join does. On a directional window under
  `EMIT ON_UPDATE` or `EMIT EARLY_AND_FINAL`, every arrival with an unbound argument now fires its
  running row too. On a five-arrival stream where two arrivals leave the argument unbound,
  `[FUTURE 2s EMIT ON_UPDATE]` without `GROUP BY` goes from 3 rows to 5, and `EARLY_AND_FINAL`
  goes from 5 rows to 8 without `GROUP BY` or to 9 with it. Under `ORDER BY … LIMIT`, a restored
  group competes for the window's top rows like any other, so it can displace a row the query used
  to report (`ORDER BY ?n LIMIT 1` over groups `a:1` and the restored `b:0` now reports `b:0`).

### Deprecated

- **Six unreferenced public `cqels-core` classes are marked `@Deprecated(forRemoval = true)`.** Nothing in
  the repo calls them outside their own file (a test-only reference to `CypherStreamQuery` aside, and javadoc).
  They stay in this release and are to be removed in a later one, as a separate `!` change (the earlier
  removal as precedent). Legacy query API: `org.cqels.query.cypher.CypherStreamQuery` and `CypherResult`,
  `org.cqels.query.cypher.SimpleCypherParser`, `org.cqels.lang.sparql.SPARQLQuery`,
  `org.cqels.query.sparql.StreamExtension`; use CypherQL (`CQELSEngine.registerCypherQuery`) or CQELS-QL
  (`CQELSEngine.registerCqelsQuery`). Experimental SWAG join: `TriangleJoinAggregateSpec` (the machinery is a
  research vehicle per its issue and the decision brief; `MultiWayAggregateSpec` generalises it).
  `ContinuousProcessingTimeTrigger`, `DeltaTrigger`, `DeltaEvictor`, `SlidingProcessingTimeWindows` and
  `DeltaFunction` are NOT deprecated: `WindowingExtensionPointsEndToEndTest` now drives them through
  `WindowDeltaOperator` (`DeltaEvictor` through `WindowedJoinOperator`, the only operator with an evictor hook).

### Changed

- **The property-path closure memo is on by default (`cqels-core`).** A CQELS-QL query now keeps each
  closure's frontiers across arrivals while the window's edges its body can match are unchanged, so an arrival that
  touches none of them (an `ex:age` triple under `?x ex:knows+ ?y`) no longer walks the closure again. The flip
  follows gate G-4: over 10,000 random arrivals per RANGE, SLIDE and TRIPLES window, random paths (nested closures
  included) give the same rows and the same `PathBudgetExceededException`s at every arrival with the memo on and
  off (`ClosureMemoGateTest`), and the compiled queries emit the same rows in the same order
  (`PathClosureMemoEndToEndTest`). Two defects of the memo, found before the flip, are fixed: a frontier whose
  body walks from every window term (`ex:a (^(ex:p*/ex:q*))+ ?y`) is no longer kept, since its work grew with
  terms the memo did not watch, and the memo is keyed on its body's edges in window order, since the order decides
  where a search from both ends of an inner closure stops; either let the memo answer a window the same budget
  refused without it. The frontiers kept per closure and direction now also stop at 2^18 nodes. One evaluator
  serves each compiled query and serialises its path evaluations. `-Dcqels.path.closureMemo=false` turns the memo
  off (read once, at class load).

- **BREAKING: `org.cqels.operator.join.VariableLengthPathOperator` is removed.** Property paths now execute
  through `PathExtension` on the windowed hash join, which evicts with the window; the operator's adjacency
  accumulated forever (its only removal was `clearIndex()`), and it enumerated node-unique simple paths, which is
  neither SPARQL's endpoint set nor openCypher's trail. Nothing in the engine called it; its test and the
  `VariableLengthPathBenchmark` JMH class are removed with it. CypherQL variable-length relationships never used
  it; they run on their own window-batch route. An external caller must move to a CQELS-QL property path
  (`?x ex:p+ ?y`, `?x ex:p*/ex:q ?y`) or a bounded CypherQL relationship (`-[r:T*1..3]->`). Neither is a drop-in for
  the operator's `Path` API: a CQELS-QL path returns endpoint pairs only, and CypherQL binds `r` to the path's
  relationships, not its nodes, and enumerates openCypher trails rather than node-unique simple paths
  (`shortestPath(...)` / `allShortestPaths(...)` are the closest match for `shortestPathOnly`). In CQELS-QL, hop bounds `{n,m}` are not in v1 (so `maxHops(n)` has none either; write
  `ex:p|ex:p/ex:p|...`), and a closure with neither end bound is subject to the path work budget
  (`-Dcqels.path.workBudget`) and fails loudly past it.

- **A `FILTER` comparison of a QUDT-typed literal no longer compares labels.** A literal whose datatype is a
  QUDT unit IRI (`http://qudt.org/vocab/unit/...`) used to be compared as a string by its label; it is now converted to
  a common unit first. Cases that flip: `"20"^^unit:KiloW < "3000"^^unit:W` was true (the labels `20` < `3000`) and is
  now false (20 kW is 20000 W); `"1"^^unit:KiloM = "1000"^^unit:M` was false and is now true; and a quantity against a
  plain number (`"5"^^unit:KiloW < 200`, was false), operands of different dimensions (`"5"^^unit:KiloW != "200"^^unit:SEC`,
  was true), a unit outside the built-in table (unless both sides use the same unit) and an ill-formed magnitude are now
  a type error, so the row is excluded. Details in the Added entry above and `docs/UPGRADING.md`.
- **Default parallel pool sizing is capped at cores-per-socket on multi-socket hosts.** The defaults of
  `ParallelExecutionConfig` `parallelism`, `SwagConfig` `maxTasks` (value only) and the no-pool overloads of
  `ParallelAggregator` / `ParallelJoinAggregator` (previously all cores / `ForkJoinPool.commonPool()`) are now
  `min(availableProcessors, physical cores of the largest socket)` when `/sys/devices/system/cpu/cpu*/topology`
  shows two or more packages; with no detectable topology (one socket, non-Linux, unreadable `/sys`) nothing
  changes. `-Dcqels.parallelism.maxThreads=N` replaces the probed cap on any machine. Explicitly configured values
  are never capped. What is capped is the `ForkJoinPool` sizing: `ParallelExecutionConfig` `parallelism` and the
  no-pool overloads of `ParallelAggregator` / `ParallelJoinAggregator`. Not capped: Reactor `Schedulers.parallel()`
  (the default `ParallelExecutionConfig.scheduler` and `SwagConfig.batchScheduler`), the `parallelStream` in
  `CqelsQueryCompiler`, and SWAG batch sizing, because `SwagConfig.maxTasks` has no consumer in main so its capped
  default changes nothing yet. See `docs/PERFORMANCE_TUNING.md`. Not measured on a 4-socket host (the B6 gauge needs one);
  pinned by `ParallelismDefaultsTest` against `/sys`-style fixtures.
- **The windowed `owl:TransitiveProperty` closure is maintained incrementally.** A window delta that flipped a
  transitive edge rebuilt the whole window's closure (adjacency rebuild plus a hop-bounded BFS from every node) and
  diffed the whole window; the earlier change only skipped deltas that flip no such edge. `ReasoningService.applyIncremental` now
  keeps each window's closure per transitive property and named graph, and on such a delta recomputes only the rows of
  the nodes within `maxTransitiveInferenceHops - 1` reverse hops of a flipped edge's source, with the same bounded
  BFS. Rows are rebuilt rather than counted or re-derived, so a deletion that pushes a surviving path past the hop
  bound is exact. The output diff visits only the statements the delta touched. Output is unchanged:
  `IncrementalTransitiveClosureDifferentialTest` compares every delta with the previous whole-window path, kept as a
  test-only oracle, on fixed-seed streams (inserts, retractions, duplicates, cycles, chains longer than the hop
  bound, bounds 2 to 5). `TransitiveClosureDeltaBenchmark` (cqels-benchmarks; one bridge edge toggled on a window of
  100 / 1 000 / 10 000 ten-node chains, bound 4, JDK 17): about 10 / 11 / 14 µs per delta, was about
  1.1 / 33 / 600 ms.
- **`maxTransitiveDerivedStatementsPerDelta` bounds what one delta derives.** It counts the statements a delta
  adds to the window's closure, across all transitive properties and graphs; statements the closure already holds
  and statements a delta retracts do not count. Before, it was checked against the whole window's recomputed closure,
  so once a window's closure outgrew the cap every transitive-relevant delta failed, however small. A delta that adds
  more than the cap still fails the stream with `IllegalStateException` instead of returning a truncated closure.
  Pinned by `TransitiveClosureCapTest`.
- **`TransitivePropertyInferenceRule` switches the transitive closure on.** The rule still derives nothing per
  statement; its presence among the resolved rules, rather than the `OWL2_RL` capability flag, now enables the
  windowed closure. `OWL2_RL` includes it, so nothing changes there; adding it as a custom rule now enables the
  closure under another profile, where it was a no-op before.
- **The static matcher filters an arrival's memoised fallback candidates once, not once per seed**.
  For a fallback-served static pattern, `StaticBgpMatcher.enumerateStaticPatternsFrom`
  re-scanned the pattern's memoised candidate list (its whole extent) against every seed's
  bindings, and the work budget charges only successful extensions. So two disconnected stream
  patterns giving s seeds beside `?x ex:p "hit"` over an `ex:p` extent of N examined s x N candidates
  to answer s x k rows. The candidates a pattern matches under a binding projection (its terms and
  the values the seed gives its variables) are now kept for the arrival
  (`StatementCandidateSource.matching`, memoised by `ArrivalCandidateMemo`), so each seed examines its
  matches only. Answers, their order, what the work budget charges and `cqels.join.crossProductCap`'s
  meaning are unchanged; a source without the memo (the temporal route, the row-scoped source)
  still examines every candidate on each call. Pinned by `StaticSeedProbeCountTest`: 900 seeds over an extent of 2,000 fall
  from 10,806,000 candidate term reads to 22,800 on both the per-arrival and the composed route.

- **The stdio outbound-buffer limit is documented, with its mitigation.** The MCP Java SDK
  stdio transport (`mcp-core` 1.0.0; no release up to 2.2.0-M1 changes it) completes a send when the message is in its
  unbounded outbound buffer, so a stdio client that stops reading stdout lets the server's heap grow and the
  notifier's queue and `droppedNotifications` do not see it. `cqels-mcp/README.md` (Transports), the
  `cqels://docs/memory-profile` resource and the stdio startup log line now say so and point long-lived agents at the
  HTTP transport, where the per-session bound holds. Documentation only; the SDK pin test
  `McpNotifierHandOffTest.onStdioTheSdkBuffersEverySignalWhileStdoutIsStalledSoNoneIsDropped` is unchanged.
- **The stream-side matcher takes a variable-predicate pattern's candidates from the index**.
  `indexedHashJoinBindings` and its parallel twin, used by the per-arrival routes and the
  windowed hash join, scanned every statement for each partial binding of such a pattern. So
  `?s ?p ?o . ?s ?p ?o` over one 10,000-statement element made about 10^8 candidate examinations to
  answer 10,000 rows, and `cqels.join.crossProductCap`, which bounds the bindings held, never saw
  them. The candidates now come from the subject, predicate or object the partial already binds
  (the subject and object indexes are built on first use). Answers, their order and the cap's
  meaning are unchanged. Pinned by `IndexedProbeCountTest`: term reads for that shape fall from
  about 4.0M (per-arrival) and 8.0M (windowed) to 28,000 and 30,000 at 2,000 statements, and now
  double when the element doubles.
- **The stream-side matcher narrows a constant-predicate pattern by its bound object**. A
  pattern whose object the partial binds but whose subject it does not (`?a ex:p ?o . ?b ex:p ?o`)
  still scanned the predicate's whole bucket for every partial binding, quadratic in that
  predicate's statements. Its candidates now come from the (predicate, object) index, and when the
  partial binds both subject and object the narrower of the (predicate, subject) and
  (predicate, object) buckets is taken, in the serial and the parallel join. Answers, their order
  and what the work budget charges are unchanged. Pinned by `IndexedProbeCountTest`: at 2,000
  statements term reads fall from 8,016,000 to 18,000 for the object-bound shape and from 510,000
  to 26,000 for `?s ex:p ?o . ?s ex:q ?o` over four subjects, on both the per-arrival and the
  windowed route.
- **The per-arrival lookup route joins an arrival's stream BGP once instead of twice**. Since
  the shared budget landed, the admission gate and `mapToResultRows` each ran the full one-element join; the solutions the
  gate finds now travel with the element across the lookup operator to seed the static side. The gate
  is unchanged: an element with no stream solution is still dropped before any repository read. Rows
  are unchanged. `PerArrivalRouteBenchmark` gains the multi-pattern shape the issue named
  (`lookupMultiPattern`, `[NOW]`, two stream patterns plus one static); short runs, before → after:
  about 110 → 160 ops/s with one temperature statement per element and 53 → 79 ops/s with three.
  Pinned by `PerArrivalElementAsWindowTest.lookupRoute_joinsEachArrivalOnce_919` (the element's
  statements are read twice per arrival, was three times).

- **The CEP matcher evaluates an argument's event condition once per arrival instead of once per
  open run**. The `where` condition of a `SEQ` argument (for a compiled query: its triple
  patterns and the `FILTER`s that read only its own variables) depends on the arriving event alone,
  yet every open run evaluated it again, and in the fan-out shape `SEQ(?h; ?b? AS o0; …; ?b? AS o254)`
  that re-evaluation was about three quarters of the matcher's short-lived allocation. The result is
  now shared by every run for the arrival. A cross-event `FILTER` (a `whereContext` guard), which
  reads the run's earlier events, is still evaluated per run. The matches are unchanged for
  conditions that are pure functions of the event, as `Pattern.where` now documents, and the
  reference-conformance gate still reports no divergence class. With 100 open runs and one `B`
  (25 500 matches), allocation on that arrival drops from about 1 330 to about 320 bytes per match.
  Pinned by `NFAPatternProcessorPredicateMemoTest` (one evaluation per slot on the 100 runs' `B`,
  a correlated guard still evaluated per run with each run's own answer, and the bytes per match).
- **`register_stream_query`'s `query` description is a short contract; the long-form reference moved to the
  `cqels://docs/cqelsql` resource**. The field had grown to a single ~4,000-word paragraph inside a JSON
  string. It is now under 2,000 characters (window forms, what a registration rejection means, where `STEP` differs per
  language, `LATENESS`, the Cypher and ownership notes) and points at the new "MCP `register_stream_query` Reference"
  section of the cqelsql cheatsheet, which carries every other sentence of the old description under headings (window
  forms, row counts and emission axes, `STEP` per language and `EMIT`, per-arrival routes, late events, the rejection
  surface). No tool parameter or enum changed. Pinned by `RegisterStreamQueryDescriptionTest` (length budget, section
  headings present in the resource) and `ToolSchemasJsonValidityTest`.
- **The static BGP matcher de-duplicates candidates on their RDF terms, and classifies terms without a
  regex** (cqels/CQELS4J#99, item 1). `StaticBgpMatcher` keyed every candidate statement of every static
  pattern by a string built from its subject, predicate and object, and both copies of `isPatternVariable`
  compiled a regex per call; in the issue's profile of a journal replay these were most of the static
  join's time. Candidates, including those of the repository fallback read, are now keyed on the
  subject/predicate/object `Value`s, and the bare-name check is a character loop. The answers are unchanged except for language tags: RDF4J compares them
  case-insensitively, as RDF 1.1 does and as the matcher's own join test already did, so `"x"@en` and
  `"x"@EN` on the same subject and predicate are now one candidate, not two. The per-element cost of an
  unbound-subject static pattern still grows with the store; pattern order (item 2) is not changed.
  Pinned by `StaticBgpMatcherTest` (datatype and term kind still distinguish candidates, language-tag case
  does not, the view/fallback merge, and `isBareName` against the old regex).
- **A static BGP whose every pattern is anchored is evaluated cheapest pattern first** (cqels/CQELS4J#99,
  item 2). On the lookup routes, `StaticBgpMatcher` evaluated the static patterns in written order and then
  again in promoted orders, so `?o a ex:Objective` written before `?o ex:kind ex:first` read and joined every
  objective in the store for every arrival, and a store a listener grows made each element cost more than the
  last. Now, when every pattern has its subject, object or predicate resolved when its turn comes in written
  order, the lookup view was read at fire time (no `CACHE` declared, the default), and the repository source
  answers unbound-subject reads in full, the matcher first tries one selective order: at each step the
  anchored pattern with the smallest extent (measured in the view or by a repository probe that reads at most
  256 candidates), ties going to written order. Every such order matches the same graph, so the rows are the
  same with the same multiplicity, though they can come in a different order, with one deliberate change: a
  constant IRI object (`?f ex:homepage <http://a>`) now matches the IRI only. The written order also matched a
  literal `"http://a"` whenever an earlier pattern had bound the subject (a lexical comparison against that
  subject's own statements), but not otherwise, so the rows depended on pattern order; an IRI and a literal
  are different RDF terms (SPARQL 1.1, section 18.3), and the selective order compares by term. A query that
  relied on the literal match gets one row fewer here; it keeps them when the selective order is not used.
  Pinned by `StaticBgpSelectiveOrderCompiledTest`, which also covers the wiring through a registered query. The selective order spends its
  own allowance (a third slot of the work counter, bounded by the work bound and shared across an arrival like
  the guard), never the guard (`cqels.join.crossProductCap`): if it exceeds that allowance, its rows are
  discarded, the previous evaluation runs and is charged to the guard exactly as before, and later rows of the
  arrival skip the selective order. So no arrival or row the previous evaluation answered is refused, and some
  it refused are answered. Probes, the selective order and that fallback read each repository request once.
  With a declared `CACHE`, on the per-row path of `OPTIONAL`, `NOT EXISTS` and `UNION` blocks (whose source
  answers resolved subjects only), for a BGP with a pattern that has nothing resolved in written order, or
  where the selective order is the written one, the evaluation is unchanged.
  `EnrichedTriples.subjectComplete()` reports whether a view was read at fire time, and
  `StatementCandidateSource.servesUnboundSubjects()` (default false) whether a source answers unbound-subject
  reads in full. Pinned by `StaticBgpMatcherTest` (the issue's shape at 2000 objectives reads under 1000
  candidates instead of all of them and answers under a work bound the written order exceeds; every written
  order of an anchored BGP answers as its declaration order, over an empty view and over one the lookup
  operator reads at fire time; a wide constant subject does not go first; extents over the probe limit tie to
  written order; a greedy order over its allowance hands over to the written one; rows sharing a counter are
  charged to the allowance, not the guard; a subject-only source keeps written order; each request is read
  once) and `EnrichedLookupJoinOperatorTest.viewIsSubjectCompleteOnlyWhenReadAtFireTime`.

- **The `Istream(...)` refusals and the named-window refusal name the construct, the shape and the alternative, and
  carry no internal plan labels** (`cqels-core`). Three `Istream(...)` refusals said "stage 2", "stage 2b",
  "stage 2c" and "stage 3", labels of an internal plan; the one that listed the supported shapes also still listed
  `UNION`, `OPTIONAL` and `NOT EXISTS` as unsupported although the join accepts them. Each now states what is supported
  (a windowed join of several patterns over one stream, with or without static patterns, and no aggregate, `GROUP BY`,
  `HAVING`, `DISTINCT`, `ORDER BY`, `LIMIT` or `FILTER(SEQ(...))`; past-only `RANGE`/`SLIDE`/`TRIPLES` incremental
  aggregates) and the alternative. The named-window refusal ended in "require Phase 3 compiler support"; it now tells
  the user to declare the window inline on each stream. The opening words of every message are unchanged
  (`Istream(...) delta emission is ...`, `Named window execution is not yet implemented.`), so a test asserting on
  those prefixes still matches. No message in `cqels-core` or `cqels-engine` carries a `#<number>` tracker reference.

- **Apache POI no longer reaches the classpath of modules that depend on `cqels-asp`.**
  `poi` and `poi-ooxml` are now optional dependencies of `cqels-asp`. Only `AnswerSetToXlsxWriter`, the xlsx
  output of the Delta CLI, uses them, but they used to put POI, `poi-ooxml-schemas` and `xmlbeans` (about 15 MB)
  on the runtime classpath of `cqels-shacl`, `cqels-engine` and every module above the engine.

- **The semantic index is stamped with its format and embedding identity, and an index with a missing or
  different stamp is refused when opened.** Every commit, including the one `close()` performs, records
  `s2.format=2`, `s2.embedding.id` and `s2.embedding.dimension`. Opening an existing index that has no stamp
  (one written before this version), or whose stamp names another provider or dimension, throws
  `IllegalStateException`. The message names the directory, both identities and the remedy. Previously such an
  index silently mixed vectors from two models. `EmbeddingProvider` gains an abstract `id()`: `disabled`,
  `hashing/<dim>`, or `hashing/<dim>/syn-<digest>` when a synonym map is set. The hashing provider now
  lower-cases with `Locale.ROOT` and splits on any character that is not a Unicode letter or digit, so
  non-Latin text contributes to its vectors (vectors change for non-ASCII text only). Searches reuse a
  `SearcherManager` instead of opening a reader per call.

- **The default output of a query without `Istream(...)` is documented per route, and a skipped named-window query in `cqels-cdsp` says what to do instead.** The specification (`CQELS-QL_SPEC.md` §8) and the syntax cheat sheet now tabulate what each route reports by default: a fixed-predicate single pattern reports each arriving solution once, a variable-predicate pattern or a one-block join under a `[RANGE]` / `[SLIDE]` / `[TRIPLES n]` window re-reports the window on every arrival, and a two-stream join reports each matched pair once; the per-arrival default differs from `Istream` of the window once the projection drops variables. No behaviour changes. The warning `QueryRegistry` logs when it skips a query that uses `FROM NAMED WINDOW` no longer ends in "requires Phase 3 compiler support": it keeps its opening words and says to declare one window per stream inline and use `STREAM` blocks, and that two different windows over one stream cannot be expressed yet.

- **The published jars no longer carry dependency versions with known High-severity vulnerabilities.**
  Jackson 2 moves from 2.17.0 to 2.18.12, Jackson 3 (which the MCP SDK brings into `cqels-mcp`) from 3.0.3 to
  3.1.8, Jetty from 12.0.16 to 12.0.40, Guava from 32.1.2-jre to 33.7.2-jre, commons-io from 2.11.0 to 2.22.0
  and the MCP SDK from 1.0.0 to 1.0.2. A grype scan of the staged release jars, the `cqels-mcp` shaded jar
  included, reports no High findings, down from 23. `cqels-core` now also declares commons-io and Guava;
  `cqels-geo`, `cqels-s2` and `cqels-functions-ext` also Jackson 2 `jackson-databind`; `cqels-mcp` commons-io,
  Jackson 2 and Jackson 3 (it declares no Guava). They reached these only through RDF4J or the MCP SDK, so the
  fixed versions also reach a project that depends on a CQELS module.

### Upgrade notes
- **A static pattern such as `?x ?pp ?o` that shares no variable now reads the whole static scope, and can
  fail the budget.** Such a pattern used to answer silently from a forward view alone, a subset of
  its SPARQL answer (nothing on the composed windowed route). It now answers the product of every seed with
  every statement in the static scope (the declared `FROM` graphs, or the whole repository when none is
  declared), which the read walks until it holds `cqels.join.crossProductCap` x pattern count + 1
  statements, once per arrival on the lookup routes. A scope larger than that bound is refused with the work
  guard's error: the composed and temporal routes fail the query, the per-arrival route drops the arrival
  with its once-per-query WARN. A query that used to answer such a subset can therefore fail where it
  silently answered. Constrain the pattern (a constant predicate, or a variable it shares with another
  pattern) or raise `-Dcqels.join.crossProductCap=<n>`.
- **A single fixed-predicate stream triple with `ORDER BY` / `LIMIT` under a `RANGE`, `TRIPLES` or `SLIDE`
  window now registers and ranks per evaluation (ADR-0002 D3).** These queries were refused at
  registration, so no running query changes. A query that now registers runs on the windowed hash join, not
  on the per-arrival route its modifier-free twin takes, and that changes its emission model, not only its
  ranking: every arrival re-evaluates the whole current window (RSTREAM-cumulative), so a bare `ORDER BY`
  re-emits each row for as long as it stays in the window, where the modifier-free query emits each row once,
  on its arrival. `SELECT DISTINCT` beside the modifier deduplicates within each evaluation instead of
  across the declared window's arrivals. The cost is the windowed join's buffer and index maintenance per
  arrival. If you relied on the refusal, or want a ranking over everything ever seen, declare `[NOW]` with
  `ORDER BY … LIMIT k` for the continuous top-K.
- **`cqels.directional.multiPatternSelect` is now an opt-out.** A directional `SELECT` over a multi-triple
  stream BGP that used to be refused at execution now registers and returns rows. Setting the property to `true` changes
  nothing; set it to `false` to keep the old refusal. A volatile expression (`RAND()`, `STRUUID()`, `NOW()`) in
  `BIND` or `FILTER` on this route is evaluated once per closed anchor, so overlapping `STEP` anchors can give one join
  solution different values; ADR-0003 D8's per-occurrence cache is not implemented yet.
- **A composed windowed lookup query whose static BGP shares a variable with the stream pattern can now
  fail on its static work bound where it used to answer.** The bound
  (`cqels.join.crossProductCap` x static pattern count) now applies to an arrival's window rows together,
  not to each row alone, so an arrival whose rows each fit it but together exceed it ends the query with the
  work guard's error, as an unjoinable static BGP's arrival already did. Raise
  `-Dcqels.join.crossProductCap=<n>` if such a query's window legitimately needs more.
- **A static pattern such as `?x ?px "Alice"` can now answer more rows, and costs a scan.** On the
  per-arrival lookup route and on the composed route with a join key, such a pattern used to answer only the
  subjects the root's forward view reaches; it now answers every subject in the static scope whose
  statement carries an object of that lexical form. On the composed route of an unjoinable static BGP it
  used to answer nothing and now answers the product. No index answers a lexical form, so the read walks
  the static scope (the declared graphs, or the whole repository) until it holds
  `cqels.join.crossProductCap` x pattern count + 1 matches, on the lookup routes once per arrival per
  distinct literal; a match
  count over that bound fails with the work guard's error as any other fetch does. Write the predicate as a
  constant to read by the predicate instead.
- **Storage backends require an explicit `path`.** The `lmdb`, `rocksdb`, `iotdb-session` and `iotdb-tsfile`
  providers no longer default to `java.io.tmpdir/cqels/<id>`: that directory does not survive a reboot on many hosts,
  and every engine without a `path` shared it. A backend without `path` now fails to open, which fails `start()` under
  the default `failIfPrimaryUnavailable(true)`. Set `BackendConfig.builder(id).property("path", dir)` for the primary
  and every mirror, one directory each; the MCP server needs its storage-backend path configured likewise. A
  directory already held by another backend also fails to open (see Fixed).
- **`java-serialized` payloads are limited to admitted classes.** A stream payload without a registered
  `BinaryCodec` is journaled with Java serialization only if its class is a JDK value or collection type, an RDF4J
  model value or an `org.cqels` type; anything else now fails its append, and a journal record naming such a class
  fails recovery. Admit your own payload classes with `-Dcqels.storage.serialFilter=com.acme.sensors.**`
  (`jdk.serialFilter` pattern syntax, `;`-separated), or register a `BinaryCodec` for them. Do this before upgrading
  an engine whose journal already holds such payloads.
- **`cqels://engine/status` gains `persistence.recovery`.** Two additive fields,
  `deliveredEvents` and `skippedEvents`, count the recovered events re-emitted to a stream and the ones
  correctly skipped; both read `0` when nothing was replayed or persistence is off. Nothing existing
  changes. `PersistenceHealth` gains a trailing `recovery` component (the old six-argument constructor remains).
- **An aggregate whose legacy window states no `EMIT` now logs one WARN per query** naming the
  emission model its window keyword chose — on-close for `[RANGE d]`, per-arrival for `[SLIDE d STEP s]`,
  `[RANGE d STEP s]` and `[TRIPLES n]` (and for `[RANGE d]` under `-Dcqels.aggregates.incremental=false`) — and
  the `EMIT` clause that would state it. Nothing else changes for a query without `EMIT`: same route, same rows.
  Add the named clause to silence the warning (`LegacyEmitDefaultWarningTest`).
- **A CypherQL query with `FROM STATIC` / `FROM GRAPH` now sees only those graphs.** Before, the clauses
  switched the static lookup on and the query read every graph of the repository. A fact stored in the null context or in
  an undeclared graph no longer joins. Store the data in the declared graph, declare the graph it lives in, or name it in
  a `MATCH GRAPH` block. A query with no clause is unchanged: it reads no static data.
- **A query that declares `FROM` now sees only its `FROM` graphs on every route.** On the
  per-arrival and composed lookup routes a plain static pattern used to read the whole repository even
  under `FROM <g>`; it now reads the triples stored in context `<g>` (the merge, with several `FROM`s), as
  the temporal route already did for `FROM`. If your data is in the repository's null context or in another graph,
  either store it in the declared graph (`conn.add(s, p, o, graphIri)`, or load the file with that
  context), or drop the `FROM` clause, so that the query reads the whole repository again. A query that
  declares only `FROM NAMED <g>` now matches nothing with a plain pattern, on the temporal route too:
  write the pattern inside
  `GRAPH <g> { ... }`, or declare `FROM <g>`. The same holds for a JSON definition's `staticGraphs` /
  `namedGraphs`. CypherQL queries go through their own compiler and are unchanged: `FROM STATIC` /
  `FROM GRAPH` there still read the whole repository. A `WITH DEPTH` / `CACHE` clause stays
  attached to its `FROM`, so dropping the clause drops those hints too. cqels-cdsp now stores each
  `cdsp.static.graphs` document under its configured URI: a query that names it with `FROM STATIC` reads
  it, and one with no `FROM` reads every loaded graph.
- **Output changes for a JSON query that declares a `resultMapping` on a stream-bound variable whose value is a
  literal, on the flat static-lookup route.** `"string"` (or `"long"`, `"double"`, ...) over an
  object-position literal now yields the declared Java type, as the static side always did: a declared `"string"`
  over `"30"^^xsd:integer` emits `"30"` where it used to emit `Long 30`. An expression that reads the variable
  (`FILTER`, `BIND`) now sees the mapped presentation, so `FILTER(?v > 29)` over a `"string"`-mapped `?v` no
  longer compares numerically and drops the row. Remove the declaration to keep the datatype-narrowed value. A
  query with no declared mapping is unchanged. The composed, stream-only and directional routes follow in the next
  entry.
- **A declared `resultMapping` now decides the Java value of a literal on every route, not only the flat one.**
  On the composed stream+static route, both stream-only routes and the directional routes, a `"string"` (or
  `"long"`, `"double"`, ...) mapping over a stream-bound literal now yields the declared type: `"string"` over
  `"30"^^xsd:int` emits `"30"` where it used to emit `Long 30`. The same holds for a `BIND` target, a `GROUP BY`
  key and an expression-referenced binding. A `FILTER` or `BIND` on the composed route now reads the mapped value, as
  on the flat route since an earlier fix, so `FILTER(?v > 29)` over a `"string"`-mapped `?v` no longer compares numerically
  and drops the row. Remove the declaration to keep the datatype-narrowed value. A query with no declared mapping
  is unchanged. Not in this change, still tracked on a follow-up: an aggregate argument (both folds keep the plain value),
  aggregate aliases, `ORDER BY`-only sort keys, the temporal-join dispatch rows, and the value the stream-only
  `[NOW]` route's `FILTER` reads.

- **A query that joins its static pattern to the stream through a `BIND` target answers fewer rows.**
  On both static-lookup routes, `BIND(?s AS ?x)` followed by `?x ex:label ?lbl` used to fan the static side out
  over every `ex:label` statement and relabel each row with the `BIND`'s value; it now returns only the rows
  for the `BIND`'s value, so the extra rows are gone and the unjoinable-pattern warning no longer fires for that
  query. The same holds when the `BIND` target merely adds a constraint beside a shared variable. Nothing to
  change in the query; code that depended on the fan-out was reading rows the query never asked for.
- **A static-lookup query whose `SELECT` list names only variables nothing binds no longer emits one empty
  row**. `SELECT ?ghost` over a stream pattern joined with a static pattern used to answer `[{}]` when a
  `FILTER` referenced a stream variable, or a `FILTER NOT EXISTS` made the lookup fetch its join variable
  into the row, and `[]` otherwise; it now answers `[]` in every case. A consumer that counted those empty
  rows will see none. `COUNT(*)` is unaffected.
- **Result rows no longer carry a `BIND` column the `SELECT` list does not name.** A query such as
  `SELECT ?s ?lbl ... BIND(?v + 1 AS ?z)` on a stream-plus-static route used to emit `z` as well. A consumer that
  read such an unprojected column now sees it absent: add the variable to the `SELECT` list to keep it
  (`SELECT ?s ?lbl ?z`). The shipped `cqels-cdsp` queries already project every `BIND` target they declare.
  The same holds for any other variable the `SELECT` list does not name (an `OPTIONAL`-bound or `UNION`-bound
  variable, a block-local `BIND` target, a stream pattern variable) on the composed route.
- **A `[NOW]` `SELECT DISTINCT … ORDER BY … LIMIT` query no longer re-emits an unchanged projected snapshot**.
  An arrival whose later rows only improved a hidden (unprojected) column of an already-held row used to re-emit
  that projected snapshot; it is now suppressed. Only a re-emission of the snapshot the query just emitted is
  suppressed: an intermediate snapshot that differs still repeats its rows, so a projected row can still recur
  within one arrival. A consumer that counted the suppressed repeats sees fewer rows, and each emission still
  supersedes the previous one.
- **A grouped query that projects, or tests in `HAVING`, a variable that is neither a `GROUP BY` variable nor an
  aggregate alias now fails at registration with an `IllegalArgumentException`**. Before, it registered:
  the projected column was simply absent from every row (so `SELECT DISTINCT` de-duplicated over what remained), and
  a bare ungrouped variable in `HAVING` dropped every group. This covers `GROUP BY` and any aggregate, an aggregate
  with no `GROUP BY` being one implicit group. Rewrite it one of three ways: add the variable to `GROUP BY`
  (`… GROUP BY ?sensor ?v`), aggregate it (`(MAX(?v) AS ?top)`, then project or test `?top`), or drop it from the
  projection. In `HAVING`, an aggregate call over the variable (`HAVING (COUNT(?v) > 0)`) is fine; only the bare
  variable is refused.
- **A grouped query that orders by a variable that is neither a `GROUP BY` variable nor an aggregate alias now fails at
  registration with an `IllegalArgumentException`**. Before, it registered and the `ORDER BY` did nothing. Order
  by a `GROUP BY` variable or by an aggregate alias instead: `… GROUP BY ?e ORDER BY DESC(?n)` for
  `(COUNT(*) AS ?n)`, or `ORDER BY ?e`. If you meant to order by the value inside each group, aggregate it first
  (`(MAX(?v) AS ?top)`, then `ORDER BY DESC(?top)`).
- **A query whose top-level `BIND` targets a variable already bound above it is now refused when it is registered.**
  A `BIND(expr AS ?v)` where `?v` is a variable of a stream or static pattern of the group, the target of
  an earlier `BIND`, or a variable an `OPTIONAL` / `UNION` block written above the `BIND` binds used to register,
  and answered with a route-dependent value for `?v`. It now throws `IllegalArgumentException` naming `?v`. Rename
  the `BIND` target to a fresh variable and select that one. A `BIND` written above a block that binds the same
  name is unaffected.
- **A windowed stream BGP whose join explores more than `cqels.join.crossProductCap` x its pattern count partial
  bindings for one arrival now fails, even when its result is within the cap.** Such a join used to build
  every intermediate binding and answer; it now stops with the cap's loud error before allocating them. A query
  whose intermediate fan-out is legitimate can raise the cap with `-Dcqels.join.crossProductCap=<n>`, or make its
  stream triple patterns more selective. A result above the cap fails as before, with "produced more than N
  intermediate bindings" in place of the exact count.
- **A quoted literal, IRI or prefixed-name constant in a `STREAM` block's triple pattern now matches by RDF term
  equality, so a query can answer differently.** A plain literal constant (`"30"`) used to match a statement
  of any datatype or language with that lexical form and now matches only a plain or `xsd:string` literal: to match
  `"30"^^xsd:integer` write the decorated constant. A decorated literal constant on the multi-stream route used to
  match nothing and now matches its own term. An IRI constant no longer matches a literal with the same text, nor a
  literal constant an IRI. Comparison is exact term equality (no XSD canonicalisation), so a constant with a
  different lexical form or datatype than the stream's literal (`"030"^^xsd:integer` against `"30"^^xsd:integer`,
  or `"30"^^xsd:integer` against an `xsd:int` literal) does not match. A bare numeric or boolean constant (`30`,
  `true`) is unchanged and still ignores the datatype.

- **A windowed stream pattern that repeats a variable within one triple (`?s ?s ?o`, `?x ?p ?x`) now returns only
  triples whose repeated positions are equal.** On a window with fewer than four distinct subjects the
  serial join used to return the non-reflexive triples as well; a query relying on that now sees fewer rows, the
  same rows the parallel path already returned.
- **A directional-window query with a static graph pattern and `ORDER BY` / `LIMIT` is still refused at
  registration, with a different message.** The old text blamed a per-arrival evaluation that does not
  honour windows; the new one says a directional window is executable only on a query with no static graph pattern.
  Code that matched the old wording needs updating; the failure itself, and its timing, are unchanged.
- **A `FILTER` or `BIND` that tests an unbound variable with `isIRI`, `isURI`, `isBLANK`, `isLITERAL` or
  `isNUMERIC` now sees an error instead of `false`; `IRI` over an IRI and `CONCAT()` now return a value**.
  `FILTER(!isIRI(?x))` used to keep a row where `?x` is unbound and now drops it, as SPARQL 1.1 section 17.4.2
  specifies; `BIND(isIRI(?x) AS ?t)` used to bind `?t` to `false` and now leaves it unbound. To keep unbound
  rows, test `BOUND(?x)` first (`FILTER(!BOUND(?x) || !isIRI(?x))`). `IRI(?iri)` used to be an error and now
  returns the IRI, and `CONCAT()` used to be an error and now returns the empty string, so a `BIND` or `FILTER`
  using either now produces a row it used to drop.

- **An unrecognized `format` on a `cqels-mcp` tool call now fails with an error.** Clients that sent a value
  outside the declared enum (for example `xml` or `csv`) used to receive JSON; they now receive an error naming
  `json`, `turtle` and `natural`. Send one of those values, or omit `format` for JSON.
- **A context-local hook such as `onErrorContinue` written downstream of `lift` now also applies to the source
  chain upstream of it.** `lift`, and the temporal-join and CEP routes built on it, now carry the subscriber's
  Reactor context to their sources, and Reactor keeps `onErrorContinue` in that context. So a source operator that
  throws on one element, for example a `map`, used to end the stream with that error and now has the element skipped
  while the stream goes on, as in any plain Reactor chain. To keep the error terminal for the source, end the source
  chain with `onErrorStop()` (`StreamElementToStreamEventAdapterTest.lift_downstreamOnErrorContinue_nowAppliesToTheSourceChain`).

- **Closing a persistent engine can now take up to 30 seconds.** `close()` waits for lenient journal-mirror writes
  that were in flight when it was called, so a mirror blocked inside a write delays the close rather than being closed
  under that write. A mirror write that finishes or is cancelled promptly adds no wait.
- **`MultiWayJoinState.deleteRange` now updates the aggregate and returns the negative delta.**
  A caller that corrected the aggregate itself after a range delete now has those results removed
  twice; drop that correction.

- **String built-ins, `CONCAT` and the date accessors changed their results.** `STRLEN`, `UCASE`, `LCASE` and
  `SUBSTR` of a non-string literal (for example `"42"^^xsd:int`) now leave the expression unbound or drop the row
  in a `FILTER`, where they used to return a value from the lexical form; wrap the argument in `STR(...)` to keep the
  old reading. `UCASE`, `LCASE`, `SUBSTR` and a `CONCAT` whose arguments all share one language tag now return a
  language-tagged literal, so `UCASE(?label) = "ABC"` no longer matches a tagged `?label`: compare against
  `"ABC"@en` or apply `STR`. `SECONDS` is now `xsd:decimal` (was `xsd:double`) and `YEAR`, `MONTH`, `DAY`, `HOURS`
  and `MINUTES` are `xsd:integer` (were `xsd:int`); numeric comparison and arithmetic are unaffected, a lexical or
  datatype-sensitive comparison of the result is not.
- **`MaxAggregateFunction` and `MinAggregateFunction` now return `-INF` and `+INF` for an infinite input instead of
  `null`.** A group whose only value is an infinity now yields that infinity, so code that read `null` as "no
  value" for such a group must read the result as the infinity. The engine's query path does not use these
  classes, so SPARQL and Cypher results are unchanged.
- **`SimplePatternProcessor` output changes for patterns whose begin state has no predicate and for patterns
  with a negated state**. A begin node with no `where` now consumes the first event, so a later
  `whereContext` guard sees it in its prefix (it used to see an empty list), and a pattern with `notNext` or
  `notFollowedBy` can now match (it used to never match). A pattern with `next(...)` directly after
  `notFollowedBy(...)` is now refused by the constructor with `IllegalArgumentException`; write `followedBy`
  there. No documented demo or tutorial shape is affected:
  each of them gives its begin state a predicate and uses no negation. Code that wrote
  `where(e -> true)` on the begin node to get this behaviour is unchanged.

- **`RETURN DISTINCT` over a backquoted property now registers.** A query such as
  ``RETURN DISTINCT n.`value` ``, or one with ``COUNT(n.`value`)`` under `DISTINCT`, used to be refused at
  registration with "RETURN DISTINCT over a quoted property is not supported yet"; it now registers and
  deduplicates on the property. Without `DISTINCT`, a quoted property that used to match nothing (an empty column,
  a `COUNT` of 0, a `WHERE` that dropped every row) now matches the property it names. The unaliased output column
  of a quoted property is its unquoted row key (`n.value`), and `ReturnExpression.hasQuotedProperty()` is removed.
- **A static BGP written with a triple the candidate fallback cannot serve ahead of the triple that
  binds its subject can now answer more rows.** Each completed matcher invocation
  keeps its declaration-order rows, first and unchanged, so a query's answer can still change where an
  operator downstream of the matcher (`FILTER NOT EXISTS`, `OPTIONAL`, an ordered `LIMIT`) sees the added rows; the added rows are the ones the same BGP answered when written in another order. Such a query
  does the written order's static-enumeration work as before, plus a separate allowance of one times the
  `cqels.join.crossProductCap` guard, or k times it for a cycle of k such triples (k at most four), counted
  in successful binding extensions rather than candidate fetches; the guard itself still bounds the written
  order alone. Downstream operators (`FILTER NOT EXISTS`, `OPTIONAL`, an ordered `LIMIT`) can answer
  differently because of the added rows, which is the intended correction.

- **A query whose static pattern's repository extent exceeds the work bound now fails at that pattern instead
  of fetching everything.** The bound is `cqels.join.crossProductCap` (default 100,000) times the static
  pattern count, and it now limits each candidate fetch as well as the binding extensions. A query that used to
  answer although one fallback-served pattern matched more repository triples than that — because a later
  pattern, or the pattern's own literal object, discarded most of them — now gets the work guard's error naming
  that pattern: the composed windowed route fails the query, the per-arrival route drops the arrival with its
  WARN. A pattern whose object is a literal-form constant (`?x ex:name "Alice"`) is fetched by its predicate
  alone, so the predicate's whole extent counts. Constrain the pattern (bind its subject or object from the
  stream, or put a constant IRI there), or raise `-Dcqels.join.crossProductCap=<n>`.

- **A query whose expression literal or `GROUP_CONCAT` separator holds a backslash escape
  SPARQL does not define is now refused when it is parsed**. This covers `"\d"`, a
  code-point escape with too few hex digits, and a surrogate or out-of-range code point.
  Before, such an escape was kept as written. Write a backslash that the function should see
  as `\\`: `REGEX(?x, "\\d")` (`CqelsQueryParser`, `Sparql11MissingBuiltInsTest`).

- **A caller that creates a stream named with more than 256 UTF-8 bytes now gets an
  `IllegalArgumentException` (an MCP error result) instead of a stream.** Shorten the name. A stream
  that has a `STREAM_CREATE` record in the journal under a longer name needs no change. A stream that has none
  is not covered: one registered directly through `ReactiveStreamEngine.registerStream` before the stream
  catalog existed lives in the journal only as generation-0 event records, so after the upgrade it cannot be
  registered again under that name: the call is refused, and replay skips its records as those of an absent
  stream. Registering it under a shorter name does not migrate those historical events: they stay in the journal
  under the original name and are not replayed into the new one, so preserving them means exporting or migrating
  them before the upgrade. The engine does not add a compatibility scan to tell such a name from a new one: the
  only journal read a creation makes is the existing initial read of the stream catalog (once per engine, on its
  first catalog lookup), which reads the journal to fold its `STREAM_CREATE` and `STREAM_REMOVE` records and
  looks for no event records. A tool's `stream` argument must now be a
  string: a number or another non-string value is refused where it used to be converted to text.

- **The bound covers engine admission, not every writer.** `PersistenceCoordinator.appendControlRecord` and raw
  journal appends can still write a `STREAM_CREATE` record with a longer name, and the engine adopts such a
  record's stream later; the bound is checked where `CQELSEngine`, `ReactiveStreamEngine.registerStream` and the
  MCP tools admit a new stream.

- **A result signal can now be dropped when a session's signal queue is full**. A session holds at
  most `notificationQueueCapacity` pending signals (`CqelsMcpServerConfig.Builder`; by default the
  configured `streamQueryBufferSize`, itself 100 by default); past it the session's oldest pending signal is dropped and counted in
  `notifications.droppedNotifications` on `cqels://engine/status`, which reports the reading session's own
  count beside `notifications.queueCapacity`
  (`McpNotifierHandOffTest.aFullQueueDropsTheSessionsOldestPendingSignalAndCountsIt`). A signal carries no
  payload, so dropping one does not remove a row: the row stays in the query's poll buffer (under that
  buffer's own bound), readable with `recall_memory(queryId)` or a read of
  `cqels://queries/{queryId}/results`. The queue bounds what the notifier holds. On HTTP that is the whole
  backlog. On stdio it is not: MCP SDK 1.0.0's stdio transport completes a send as soon as the message is in
  its own unbounded outbound buffer, so stdout backpressure does not reach the notifier — while stdout is
  stalled the SDK buffers every signal handed to it and no drop is counted for that
  (`McpNotifierHandOffTest.onStdioTheSdkBuffersEverySignalWhileStdoutIsStalledSoNoneIsDropped` pins this).
  A burst produced faster than the writer hands signals over still overflows the queue and is counted
  (`onStdioABurstFasterThanTheWriterOverflowsTheQueueAndIsCounted`).

- **Result signals are written by a fixed pool of `notificationWriterThreads` threads**. The pool
  size is set with `CqelsMcpServerConfig.Builder.notificationWriterThreads`: by default twice the available
  processors, between 2 and 16, and at most 64.
  While fewer sessions than writers are stalled, every other session is served in turn; once as many
  sessions as writers stop reading at the same time, every writer is held and signal delivery to all
  sessions pauses until one of those writes ends, while signals keep queueing per session (drop-oldest,
  counted) and the engine is unaffected (`moreStalledSessionsThanWritersNeverBlockTheResultThreadAndAddNoThread`
  pins the pause). Raise the setting for a deployment that expects many slow readers at once.

- A `FILTER` comparing duration-typed literals can keep rows it used to drop and drop rows it used to keep, because
  it no longer compares text. Comparing a duration with a non-duration operand now drops the row for every
  operator; before, it compared the labels against a plain or `xsd:string` literal, and `!=` against a number was
  true. This now also applies to a row-bound duration, so on a stream+static query, in `HAVING` or inside
  `OPTIONAL`, `FILTER(?d = "PT9S")` against a plain string drops the row where it used to keep it.

- A CQELS-QL result row's value for a language-tagged variable, or one typed with a duration, `xsd:dateTime`,
  `xsd:date` or any datatype outside the XSD string, numeric and boolean families, is now an RDF4J `Literal` on every
  route, stream-only included (its `toString()` and `stringValue()` are the lexical form, as the `String` was), not a
  `java.lang.String`. Code
  that casts the value to `String`, or compares it with `equals` against a `String`, should read `stringValue()`
  instead, as for IRIs. On the stream+static routes such a key now correlates in `OPTIONAL`,
  `FILTER NOT EXISTS` and `UNION` by its own term, with or without a `resultMapping`, and no longer with an untagged
  `xsd:string` of the same lexical form; `SUM` / `AVG` over such literals no longer parse their labels as numbers.
- A caller of `VariableLengthPathOperator` must pass the full predicate IRI to `relationshipType(...)`, not a local
  name. A short name such as `"KNOWS"` used to match `http://example.org/KNOWS` by substring and now matches nothing.
  Paths of exactly `maxHops` are no longer returned twice, so result counts drop by that duplicate.
  (Superseded: the class is removed in this release.)
- `org.cqels.operator.join.VariableLengthPathOperator` (with its `Path` and builder types) is removed, a
  compile-time break for callers; move to a CQELS-QL property path as described under `### Changed`.

- **CEP queries whose `SEQ` arguments are not subjects of stream triple patterns now fail at
  registration.** They used to register and match any stream element at those steps. This includes every
  CEP query whose events are written in a `WINDOW :w { … }` block over a `FROM NAMED WINDOW` (those triples are not
  stream triple patterns, so named-window CEP is unsupported; a directional named window was already refused) and a
  query whose bare triples sit beside a `FROM <graph>`, a `FROM NAMED <graph>` or a `STREAM` block (they are then static). The fix is to
  make every `SEQ` argument the subject of a triple in a `STREAM` block.
- **`STRLEN` and `SUBSTR` on a string with a character outside the Basic Multilingual Plane return code-point
  counts and positions.** `STRLEN("😀")` is now 1 (it was 2), and `SUBSTR("a😀b", 2, 1)` is now `"😀"`
  (it was half of the surrogate pair). A string with only BMP characters gives the same answer as before.

- **A stream-only windowed query that aggregates (or sorts / limits) over an `OPTIONAL` or `UNION` now
  counts what the block binds, where its join key is not projected.** `SELECT (COUNT(*) AS ?n)`
  over `STREAM S { ?s ex:p ?v . } OPTIONAL { ?s ex:label ?ol . }` used to answer the bare window sizes
  and the same query over a `UNION` emitted nothing; they now count the block's solutions, as the
  composed route does. A consumer that depended on the old counts will see larger ones (and a `UNION`
  query that was silent will now emit).
- **A stream's replay history is now bounded in bytes as well as in elements.** A query registered later
  (not a CEP query, which sees none of it) receives the newest elements within both `cqels.engine.streamReplayLimit`
  (8192 elements) and `cqels.engine.streamReplayMaxBytes` (64 MiB of estimated element size by default), so when
  the elements average more than about 8 KiB it receives fewer than 8192 of them. Raise
  `cqels.engine.streamReplayMaxBytes` (a JVM system property, read when a stream is created) to keep more, or
  register the query first so that it sees every element.
- **Queries that combine a `FILTER` with an `OPTIONAL`, a `UNION` or a `BIND` can return different rows
  (CQELS4J#85, CQELS4J#83).** The outer group's `FILTER` now sees what those bound, on every
  route. On a stream-only query and on a windowed `GROUP BY` query with a static pattern, rows that
  `FILTER(!BOUND(?v))` after an `OPTIONAL` used to admit although the block bound `?v` are now dropped, and
  rows that `FILTER(BOUND(?v))` or a comparison on `?v` used to drop are now kept. On a query with a static
  pattern, a `FILTER` over a `BIND` target used to drop every row and now keeps the rows it accepts, and a
  windowed `GROUP BY` aggregate that answered nothing for such a query now emits its groups. A `BIND` written
  below an `OPTIONAL` or `UNION` now reads what the block bound on every route, so its value, and any group it
  keys, can change; one written between two blocks now runs there, so the block below it sees its target. An
  `OPTIONAL` written above a `UNION` now applies before it, as written, where every `UNION` used to apply
  first. A query that relied on the old order to admit every row should drop the `FILTER`; one that needs a
  `BIND`'s value before a block should declare the `BIND` above that block. Code that rebuilds a
  `CqelsQueryDefinition` from its getters must pass `getBindingBlockPositions()` and
  `getPositionalBlockOrder()` to `Builder.positionalLayout`: with only `bindingsDeclaredBeforeReshapers`,
  `build()` now refuses a definition that has both an `OPTIONAL` and a `UNION`, or a `BIND` below the first
  of two or more such blocks.

- **`CompiledQuery.getLookupOperators()` and `CompiledCypherQuery.getLookupOperators()` now list only the
  operators of live subscriptions.** The set is empty after `execute()` until its result is subscribed, and
  an operator leaves it when that subscription completes, errors or is cancelled; code that read the set after a
  result completed, or before subscribing, now sees it empty. Re-subscribing a result `Flux` whose earlier
  subscriptions have all ended starts its lookup operator from an empty cache, because no commit invalidation
  reached it in between. The lookup cache now evicts past 100,000 subjects; a query whose `CACHE` working set is
  larger can raise it with `-Dcqels.join.lookupCacheMaxEntries=<n>`.

- **Stream processors now receive `StreamPunctuation`.** A processor added with
  `CQELSEngine.Builder.addStreamProcessor` sees every `DataStream.advanceTo` as a `StreamPunctuation` in the
  sequence of elements, and must forward it unchanged. Its payload is an empty statement list, so a processor
  that passes through elements it does not recognise needs no change; one that casts the payload, rebuilds
  every element or counts arrivals must skip it (`element instanceof StreamPunctuation`). Dropping it makes
  `advanceTo` a no-op on that stream; turning it into a new element makes it an empty data element. Nothing
  reaches a processor unless the application calls `advanceTo`.

- **The text of three `Istream(...)` refusal messages and of the named-window refusal changed.** The opening
  words are unchanged; the rest no longer names internal stages or phases. A caller that matches on the full text of
  an `UnsupportedOperationException` from these queries must update; matching on the opening words keeps working.
  Syntax errors for `SELECT Istream ?a` gain a `; Istream needs parentheses: ...` suffix, so a test that compares the
  whole message for equality must update too.

- A project that calls `AnswerSetToXlsxWriter`, or runs the Delta CLI with xlsx output from its own classpath,
  declares `org.apache.poi:poi-ooxml` itself.

- An index directory created with `SemanticStatementIndexer.fileBased` before this version carries no identity
  stamp and is refused when opened. Nothing rebuilds the index for you: delete the directory and index the
  statements again, or keep using the earlier version for it. Custom `EmbeddingProvider` implementations must
  implement `id()`, returning a value that changes whenever a setting that changes `embed`'s output changes.
- `SemanticStatementIndexer.createSearcher()` is removed. Use `acquireSearcher()` and give each searcher it
  returns back with `releaseSearcher(searcher)` when done (for example in a `finally` block); a searcher that is
  not released keeps its index segments open.

- `geo:wktLiteral` data written as EWKT with an SRID other than 4326 (for example `SRID=3857;`, or `SRID=0;`, PostGIS's "unspecified" SRID) no longer matches any `geof:` filter (except `geof:getSRID`, which still reports CRS84 for these literals) and is not indexed. Reproject it to lon/lat and write it as plain WKT, `SRID=4326;` or the CRS84 IRI prefix.

- A `geof:buffer` radius given as a plain or `xsd:string` literal (for example `"100"`) now
  yields an unbound result instead of a buffer. Use a typed numeric literal; CQELS-QL numeric
  constants are already typed. This also applies to a radius bound from a variable: values that
  come from stream or input data as plain or `xsd:string` literals are refused, so type them
  (for example `xsd:double`) where the data is mapped to RDF.

- A project that depends on any published CQELS module and manages none of these artifacts itself now resolves
  commons-io 2.22.0, Guava 33.7.2-jre and Jackson 2.18.12 (`jackson-annotations` 2.21 through `cqels-mcp`);
  through `cqels-mcp` also Jackson 3 at 3.1.8, Jetty 12.0.40 and the MCP SDK 1.0.2. Before this release it
  resolved the versions RDF4J and the MCP SDK declare: commons-io 2.8.0 or 2.11.0, Guava 32.1.1-jre or
  32.1.2-jre, Jackson 2.13.5 or 2.17.0, Jackson 3 at 3.0.3 and Jetty 12.0.16. A project keeps an older version
  when it declares the artifact itself, manages it (its own `dependencyManagement`, an inherited parent, or an
  imported BOM such as `rdf4j-bom` 4.3.8, which manages `jackson-bom` 2.13.5, commons-io 2.11.0 and Guava
  32.1.2-jre), or reaches it through another dependency at equal or nearer depth. Raise those to at least the versions above.

### Known issues / follow-ups

- **`[NOW]` with a `GROUP BY` aggregate and `ORDER BY … LIMIT` stays refused at registration, by decision**.
  `[NOW]` is a continuous top-K over the whole stream, so a per-arrival fold cannot be ranked
  under it: each fold describes one arrival, and a group row from an arrival long gone would keep
  winning. Per-evaluation ranking of such a fold needs a declared window other than `[NOW]` (ADR-0002
  D3). The refusal message now says so: it names a declared window other than `[NOW]` as the way to rank per
  evaluation; no query that compiled before is refused now, or the reverse. The refusal is
  `rejectUnsupportedOrderByLimitShape`, pinned by
  `NowGroupByPerArrivalTest.nowGroupByOrderByLimitStillRejectedAtRegistration`.
- **The window-scoped `SELECT DISTINCT` time scope has a stated limit when event time regresses**.
  Under `RANGE` / `SLIDE` the scope prunes an occurrence against the running maximum of arrival timestamps.
  When event time goes backwards by more than the declared extent after a higher-timestamp arrival, that
  edge can evict an occurrence the window still holds, so the key is emitted once more: measured under
  `[RANGE 5s]`, 2 rows where the relation's `Istream(DISTINCT)` has 1, and 3 against 2. A backwards step of
  up to the extent agrees, so the scope is exact unless an arrival is more than the declared extent ahead of the
  ones after it. Exactness beyond that would need a second eviction index, so the limit is stated rather than fixed.
  Pinned by `WindowScopedDistinctTest.eventTimeRegressingAfterAHigherTimestampArrival_isOutsideTheScope`.
  The count scope (`[TRIPLES n]`, `[NOW]`) is indexed by arrival ordinal and is unaffected.
- **The per-arrival lookup route joins at arrival, so a mid-window static mutation is not seen by elements
  already in the window**. The route joins only the arriving element against the static side, as of
  its arrival, and never re-joins the elements already in the window. The composed windowed body re-derives the
  window on every evaluation and does see the change, so the two differ: measured under `[RANGE 10s]`, 2
  rows against 4. That is the route's contract, by design (ADR-0003 D6 keeps the two routes apart), and the same
  as-of-arrival reading `FROM <iri> CACHE <duration>` already advertises, since re-joining
  every resident element on each arrival is the quadratic re-evaluation the per-arrival contract does not
  take on. Pinned by `PerArrivalIstreamContractTest.aMidWindowStaticMutation_isOutsideTheContract`.
- `ORDER BY`, `MIN`, `MAX` and top-K compare durations as text, so `"PT10S"` sorts before `"PT9S"`. CypherQL
  lifts a duration to its label, so its `WHERE` comparisons and its ordering compare durations as text too.
- A `GROUP BY` key holds a row-bound typed or language-tagged literal's label without its datatype or tag, so
  `"PT1S"^^xsd:duration` and `"PT1S"^^xsd:dayTimeDuration`, or `"cat"@en` and `"cat"@fr`, fall into one group, as they
  did while both lifted to the same `String`, on the incremental `RANGE` route and the batch route alike
  (`groupByKeysARowBoundDurationByItsLabel`, `groupByKeyMergesLanguageTagsOnTheIncrementalAndBatchRoutes`). Neither `GROUP BY` nor `DISTINCT` compares durations by value:
  `"PT60S"` and `"PT1M"` stay two groups and two `DISTINCT` rows.

- **`store_memory` can leave part of a call unindexed while reporting success.** When a fact of the
  call has ill-formed text, the RDF write is kept, but indexing refuses that fact and stops there: the facts
  after it are not indexed, and the failure is only logged as a warning. A boundary check in `cqels-mcp` is
  proposed.
- **A static `UNION` joined to a stream pattern drops every row on the flat paths when the `UNION`'s
  join variable is not projected.** This affects queries with no aggregate and no `Istream(...)`. The
  flat paths are the windowed hash join of such a query with no `ORDER BY` or `LIMIT` (`DISTINCT` alone
  does not avoid it), the per-arrival route, and `[NOW]`.
  Workaround: project the join variable (for example `?c`); on a window other than `[NOW]`, adding
  `ORDER BY` or `LIMIT` also works, because it applies the `UNION` before projection.
- **On the windowed hash join, a solution whose projection is empty is dropped before aggregation.**
  This holds for per-arrival and closed-window evaluation alike, when the only variables the solution
  would project are an aggregate argument or a group key and they are unbound. So a global aggregate
  over a window where no solution binds the argument reports no row instead of `COUNT` 0, `COUNT(*)`
  beside an aggregate whose argument can be unbound undercounts, `HAVING (COUNT(?x) = 0)` without
  `GROUP BY` reports nothing, and an all-unbound group key reports no group. The missing global row
  also affects the incremental aggregate route on legacy windows (`RANGE`, `SLIDE` / `TRIPLES` with
  `EMIT ON_CLOSE`, and `LATENESS` windows) when no solution binds the argument; that route keeps
  all-unbound groups. Workaround: bind the
  argument (for example with `COALESCE`), or count with `COUNT(*)` only in queries whose aggregate
  arguments are always bound.
- **`AVG` of a group with no bound argument is null, where SPARQL 1.1 gives 0.** This holds on both the
  incremental and the batch aggregate route.


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
  <version>2.0.0-alpha.22</version>
</dependency>
```

## Verifying this release

The signed manifest and its signature are served from the artifact repository;
the public key is served from a different origin, so verifying with it is not
circular. Take the key from the commit pinned below.

```bash
set -euo pipefail
BASE=https://maven.cqels.org/releases/supply-chain/2.0.0-alpha.22

curl -fsSLO https://raw.githubusercontent.com/cqels/CQELS4J/d682924b4237f6fcf1f7795e3dab43fac286d2ea/cosign.pub
echo "36dd8daa9988f23eb40c4f3550fa7bdfa3796e5e58cce8d23b9cc6a99f47f30b  cosign.pub" | sha256sum --check --strict - || exit 1

curl -fsSLO "$BASE/SHA256SUMS" -O "$BASE/SHA256SUMS.bundle"
cosign verify-blob --key cosign.pub --bundle SHA256SUMS.bundle \
  --new-bundle-format=false --insecure-ignore-tlog SHA256SUMS
```

The URL is pinned to a commit rather than to `master`, so it keeps working
after a key rotation. `--insecure-ignore-tlog` is expected: these signatures
carry no transparency-log entry, which is precisely why the key's origin
matters. Full walkthrough: https://github.com/cqels/CQELS4J/blob/master/SUPPLY_CHAIN.md
