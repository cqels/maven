# CQELS 2.0.0-alpha.21

A **semantics release**: sixty fixes and twenty-one changes, with four further behaviour changes
marked under Added and Fixed, and the theme across the engine
work is one thing — a route now answers what the language promises rather than what its matcher
happened to walk first. The per-arrival routes gained a *stated* window contract — an arrival is
evaluated as the one-element window it is, so the route emits that window's `Istream`, with two
lookup shapes named under Known issues that stay outside the equality — and then grew into it. A
batch-shaped element — `DataStream.push(Collection, …)`, the MCP stream-push tool, the CDSP geo
taggers — now yields one row per solution instead of its first match, and a multi-pattern `[NOW]`
BGP whose patterns need two different statements of the same element answers for the first time.
The two static-lookup routes stopped both inventing and dropping rows, which answers the public
downstream report that enumerated the divergence between them (CQELS4J#67). A top-level `BIND` now
runs where SPARQL puts it relative to the `OPTIONAL` / `UNION` / `FILTER NOT EXISTS` blocks it
precedes, and a `GROUP BY` aggregate under `[NOW]` folds once per arrival on a live stream instead
of once at end-of-stream. CypherQL caught up on three fronts in one wave: exact matching and
resolution, `ORDER BY` / `LIMIT` that rank per evaluation or as an explicit global top-K instead
of hanging or silently truncating the query, and a `MATCH` that reads only the stream it names. Away
from the query engine, statement metadata finally works on the persistent MCP store, where every
annotated write had failed outright and taken valid-time recall and labelled governance with it.

More rows mean more work per arrival, so the row-count changes arrive with bounds — and on the
per-arrival routes two of those bounds **drop the arrival and warn once** rather than failing the
query. On such a route the element *is* the evaluation, so dropping it is well defined, whereas a
throw ends the subscription: without this, any writer of a stream could stop another session's
standing query with one oversized element. The windowed, composed and temporal routes are
unchanged and still fail loudly, because there an arrival is joined against window state and
dropping it is not a well-defined remedy. Four defects of that kind — three ways one client's push
could end another session's standing query, and a partial-result blow-up that arrived before
containment did — were found by reviewing the last change of the cycle and fixed before it
merged.

Away from the query engine, the release carries a cross-repository audit's fixes and the first two
implementation slices of ADR-0004. **Durability and recovery:** damage inside a journal's committed
prefix now fails the open instead of being trimmed away as a torn tail, every record of a new
journal carries a CRC32C, a journal's format is decided from a header in the file and never from
the records, offsets no longer restart at 1 after a `truncateBefore` that empties the
journal, a failed periodic checkpoint write on the plain `push` path no longer kills the stream and
every query on it, and a failed journal append leaves the stream honestly terminated. **MCP:** a
client's SPARQL can no longer make the server send a request to a URL the client chose; on the
multi-client HTTP transport a session sees and controls only the standing queries it registered,
and its registrations are released when the session closes or idles out. **Elsewhere:** a SHACL
shape that is not typed `sh:NodeShape` is no longer silently skipped, RETE `PRIORITY` no longer
loses an inference to a higher-priority rule whose filter rejects it, and windowed recursive
inference without truth maintenance no longer keeps a derived fact past its oldest premise. **CEP**
match semantics are now written down as ADR-0004 and enforced: a match is a labelled complex event
emitted exactly once, five pattern shapes are refused at registration, and a match is bounded in
arguments, argument-name length and events.

The release also brings client identity to the MCP HTTP transport, built on stream attributes
in the engine's durable catalog (under `failIfPrimaryUnavailable=false`, stream creation now fails
when a configured primary backend does not open): opt-in per-client bearer tokens that put the
caller's identity on every request and bind each session to its client; private streams, opt-in per
stream, owned by that identity with per-name leases and a per-client cap; a `notify: true` result
signal that reaches only the session that registered the query; and review follow-ups to them
(per-client tokens are limited to RFC 6750 `b64token` characters). Every HTTP session is now bound
(without per-client tokens, to one shared identity internal to the session filter, so stream
ownership is unchanged) under a total cap answered with `429`; without per-client tokens there is no
per-client limit, so one agent reconnecting without `DELETE` can hold every slot and lock out the
others until its sessions expire. The tool descriptions and docs now state, as a
wording-only correction, what a late registration receives: the stream's recent history,
though a CEP query gets none of it and a rule program at most 1024 pending observations plus 256 in
flight; live pushes during that backlog can be dropped, and on a shared stream a later subscriber
receives other clients' recent pushes. Stream-query results are byte-bounded, plain queries
included, and CEP match rows are rendered only when drained, and the MCP push tool bounds
each event before converting it, with stricter input types. A journal refuses to open on an
impossible durable value, a dangling sidecar link or an oversized durable entry, reports a read
fault on its contents as corruption, at open and on an open journal's later reads, no longer
lets a damaged record length swallow the records after it, and never assigns an offset past
`Long.MAX_VALUE`. Recovery likewise fails closed when an I/O fault keeps it from finding or
reading a committed checkpoint. A CEP `advanceWatermark` call waiting for room no longer
fails when its subscription is cancelled, an idle directional CEP registration no longer
reserves its full input channel, and the OCI smoke gate runs two clients and a private stream
across a restart. In the query engine, window-scoped `SELECT DISTINCT` on the per-arrival routes now
prunes its scope at every arrival, including one that yields no row, with no emitted row
changed; `HAVING` over an aggregate call answers what the alias form answers; and the
operator-level windowed join on dynamic-gap session windows no longer drops its buffered rows when
an element's session window equals an already-merged session (CQELS-QL session windows were not
affected).

### Added

- **Stream attributes in the durable catalog**. `CQELSEngine.createStream(name, attributes)`
  creates a stream carrying a small map of strings the engine does not interpret, returned by
  `DataStream.attributes()`; `createStream(name)` is unchanged. At most 16 attributes, keys of 1 to 128
  and values of up to 1 024 UTF-8 bytes; a null or a string that is not well-formed UTF-16 is refused.
  On a persistence-enabled engine the attributes are the payload of the generation's `STREAM_CREATE`
  record, written with envelope schema version 3 (only when attributes are present), and come back
  with the stream after a restart:
  - Creating a name whose catalog generation is live adopts it and returns its attributes; asking for
    different attributes is refused with both sets of keys, never values.
  - `CQELSEngine.liveStreamCatalog()` names every live stream with its attributes, whether or not it
    is registered in memory (`recreateStreamsOnStart` or replay off); empty when persistence is
    disabled, where the stream's own attributes are the only record.
  - Refused on a legacy format-1 journal, and on an engine with a journal mirror configured; such an
    engine also refuses to start while a live stream has attributes.
  - A creation with attributes whose append ends in `IndeterminateCommitException` fences the name on
    that engine until it restarts and recovery reads the journal. After an attribute-free one, only
    creations of that name with attributes are refused; an attribute-free retry works and lifts the
    fence. At most 1 024 fences: once the table is full, every stream creation on that engine is
    refused until it restarts.
  - **Behaviour change for `failIfPrimaryUnavailable=false`:** when persistence is configured but its
    primary backend fails to open, every stream creation (with or without attributes) and
    `liveStreamCatalog()` now throw `IllegalStateException` naming the backend, where stream creation
    used to succeed with persistence silently off. The catalog cannot be read, so whether a name
    already belongs to a generation with attributes is unknown. Persistence disabled by configuration
    is unchanged.
  - `EventJournal.checksumsEachRecord()` (default `false`; `true` for a format-2
    `FileBackedEventJournal`) and `StreamEnvelope.SCHEMA_VERSION_V3` / `schemaVersion()` are new in
    the storage SPI. A journal holding a v3 record cannot be read by an earlier release: see
    `docs/UPGRADING.md`.
- **Per-client bearer tokens on the MCP HTTP transport, with the caller's identity on every request and
  sessions bound to their client**, opt-in. `CQELS_MCP_HTTP_CLIENTS_FILE` (or
  `-Dcqels.mcp.http.clientsFile`) names a file of `clientId = token` lines, read at start (blank lines,
  lines starting with `#` and a byte order mark at the start of the file are ignored); an embedder passes the parsed map to
  `CqelsMcpServerConfig.Builder.httpClients(Map)`.
  - Validation at start: client ids match `[A-Za-z0-9._-]{1,64}` and are unique; tokens are unique, use
    only the characters `A-Z a-z 0-9 - . _ ~ + /` with optional trailing `=` (RFC 6750 `b64token`) and
    have at least 32 characters. An unreadable, empty or malformed file, or an
    invalid entry, refuses the start; a file error names the line, and no message contains a token.
    Setting it together with `CQELS_MCP_HTTP_AUTH_TOKEN` refuses the start.
  - Each request's token is compared in constant time with every configured token; a missing or
    unknown one gets `401`. The matching client id reaches tool and resource handlers through the MCP
    transport context and is read with `org.cqels.mcp.internal.CallerIdentity.of(exchange)`
    (authenticated client, anonymous client, or internal caller with no exchange). Private streams
    (next entry) are what acts on it.
  - A session id is bound to the client whose `initialize` the server issued it for, never to an id a
    client presents. A request with another client's session id gets `403`; an id that is not bound
    (never issued, closed or expired) gets `404`, the status MCP SDK 1.0.0 returns for an unknown
    session; both are refused before the request reaches the transport, and the binding is checked
    again in the same locked step that admits the request, so an idle expiry cannot slip between the
    two. A POST is classified from the same decoded text, with the same deserializer, as the SDK reads
    it; a body that is not a JSON-RPC message gets `400` and is not forwarded. An `initialize` that
    carries a session id gets `400` (the SDK would create a new, unbound session for it). Before a slot
    is reserved, an `initialize` passes the checks the SDK makes: its Origin/Host validator (when an
    allow-list is configured; refused with the validator's status), an `Accept` header with both
    `application/json` and `text/event-stream`, and the typed conversion of `params` to
    `InitializeRequest` with `protocolVersion`, `capabilities` and `clientInfo` present (otherwise
    `400`). An `initialize` the SDK still refuses with a 4xx returns its slot; one that ends with a 5xx
    and no session id keeps its slot until restart, since the SDK may hold a session whose id was never
    sent. The filter reads a POST body only up to `CQELS_MCP_HTTP_MAX_BODY_BYTES` (`413` beyond it). A binding ends after a successful `DELETE`, once the session's last request
    (including an open event stream) has ended, and when the idle sweep
    (`CQELS_MCP_HTTP_SESSION_IDLE_TIMEOUT`) releases the session, which covers every bound session
    (every HTTP session: see the Fixed entry on the total session cap), not only those holding registrations; the sweep first ends the SDK's transport session
    with an internal `DELETE`, and keeps the binding and its slot (retrying on the next sweep) if that
    fails. Later in this release the client's own `DELETE` that the transport answers `404`, for a
    session it dropped after a failed write, also ends the binding: see the co-review follow-ups
    below. At most 32 bound sessions per client and `CQELS_MCP_HTTP_MAX_SESSIONS` (default 1 024) in
    total; an `initialize` beyond either gets `429` and creates no session. Tokens are compared as SHA-256 digests, so the work per request does
    not depend on the presented or the configured tokens' lengths.
  - With the single shared token, no authentication and stdio every caller is still anonymous. On HTTP
    their sessions are bound too, to one shared identity held only to the total cap (see the
    Fixed entry on the total session cap).
- **Private streams owned by the client identity, with leases and a per-client cap (MCP)**,
  opt-in per stream. `create_stream` takes an optional boolean `private`. Omitted, a new stream is
  created shared and an existing stream of either mode (the caller's or shared) is adopted unchanged;
  `false` creates a shared stream and is refused on an existing private one. Every stream is shared
  unless created with `private: true`, so no deployment changes behaviour.
  - A private stream carries the stream attribute `cqels.mcp.owner = <clientId>`, written with its
    generation in the durable catalog. Its owner is read from the live stream, and from
    `CQELSEngine.liveStreamAttributes(name)` when the stream is not in memory. Only that client, in any session,
    may push to it, store into it (short-term `store_memory`), register `register_stream_query`, a
    one-shot CQELS-QL/Cypher `query`, `watch_invariant` or `register_rules` over it, create it again,
    remove it or see it in `cqels://streams`. Anyone else gets one error, `Stream '<name>' belongs to
    another MCP client and is not shared; the request was refused and nothing was changed`, counted as
    `cqels_mcp_stream_ownership_refused_total` on `/metrics`. Internal callers (the embedding application)
    are not restricted; streams it creates in code without ownership attributes are shared, except that
    a creation adopting a live private generation of the name keeps its owner, and one that writes
    `cqels.mcp.owner` itself is private to that client id.
  - `private: true` needs a client identity (`CQELS_MCP_HTTP_CLIENTS_FILE`); on stdio, HTTP without
    authentication and with the single shared token it is refused, naming that configuration. Such
    callers cannot use a private stream either.
  - The mode is fixed at creation: `create_stream` on an existing stream is idempotent when `private` is
    absent or equal to the stream's mode, and refused when it differs. The default short-term stream
    `shortterm` cannot be private.
  - At most 64 private streams per client by default: `CQELS_MCP_HTTP_MAX_PRIVATE_STREAMS_PER_CLIENT`
    (or `-Dcqels.mcp.http.maxPrivateStreamsPerClient`, or
    `CqelsMcpServerConfig.Builder.httpMaxPrivateStreamsPerClient(int)`), 1 to 1 024, checked at start.
    The count covers the client's live private streams in memory and in the durable catalog, so streams
    restored at start count, and a private creation whose journal commit was indeterminate, which counts
    until recovery resolves it; a creation beyond the cap is refused naming it. `remove_stream` frees a place,
    also for a private stream that exists only in the durable catalog (stream reasoning enabled): the
    stream is adopted for the moment and removed, which needs one free stream slot; with every slot in use
    the removal is refused naming that cause.
  - Nothing happens to a private stream when a session ends or the server restarts. With a storage
    backend it comes back with its owner; with stream reasoning enabled it is not re-created, and only
    its client can create the name again or remove it; without a storage backend it is gone, like every
    stream.
  - Per-name leases bind the ownership check to the stream that is used: a push, a short-term store, a
    registration or a one-shot continuous query holds the names it was checked against until the
    engine call has returned, and a creation or removal under such a name waits for it
    (`stream '<name>' is busy — retry` after 30 s).
  - `create_stream` reports an engine refusal as `create_stream error: stream '<name>' was not created:
    <reason>` (for example an unavailable primary, or a journal mirror refusing the owner attribute).
  - Stream names are not secret: "belongs to another client" and "does not exist" are distinct answers.
  - Engine: `CQELSEngine.liveStreamAttributes(name)` answers one name's live catalog generation, and
    `liveStreamCatalog()` copies a live-only index, so neither the ownership check nor the cap scans every
    name the journal has ever seen.

### Fixed
- **The publish pipeline now derives a cumulative `maven-metadata.xml` for the anonymous Maven
  mirror, which previously carried none at all**. Each publish
  run's staging repo generates its own `maven-metadata.xml`, but that per-run file lists only
  the one version staged that run; copying it over the mirror as-is would have replaced the
  accumulated version list with a single entry while every older version's directory stayed on
  disk, unreported — so the mirror deliberately never served this file at all. The cost: no
  version range (`[2.0.0-alpha.18,)`), `LATEST`/`RELEASE`, or Dependabot-style tooling could
  see that a release existed; only a consumer who already knew an exact version string could
  resolve one. `scripts/ci/mirror-maven-metadata.py` now regenerates `maven-metadata.xml`
  (plus its `.sha1`/`.md5` companions) from the union of the versions already on the mirror
  and the versions this run staged, so `<latest>`/`<release>` and version-range resolution
  work against the real, accumulated release history. The withdrawn `2.0.0-alpha.17` cannot
  reappear by accident — the generator has no directory to enumerate for it; only an explicit
  republish of that exact tag would stage one and see it included, the same as any other
  version.

- **`COUNT(*)` with an unprojected `GROUP BY` key now emits, and a `FILTER NOT EXISTS` block
  correlated on a variable a route cannot resolve is now refused at registration instead of
  silently answering wrong**. `mapBindingsToRecord` used to carry only `SELECT`
  and aggregate-argument variables, so an unprojected `GROUP BY` key produced an empty record
  and the `!record.isEmpty()` filter discarded the whole batch — `COUNT(*)` answered nothing,
  and a named-argument aggregate collapsed every group into one null-keyed total. The fetch
  now carries every `GROUP BY` key and strips the unprojected ones only after the fold,
  `ORDER BY`, `HAVING` and the aggregate tail have run, matching the answer the
  incremental-aggregate route already gave. Separately, on the stream-only routes
  (`executeStreamOnlyWithExpressions` and `executeWindowedHashJoin`, plus the `executeStreamOnly`
  body that **Changed** below retires into the first of those) a
  `FILTER NOT EXISTS` block correlated on a variable the row cannot supply — in any triple
  position, or as a `BIND` target — answered unpredictably: the block could match nothing, so
  the anti-join kept an outer row it should have excluded, or it could match a value the
  repository happened to hold in that variable's place, so the anti-join excluded an outer row
  it should have kept. Registration now refuses that shape outright, naming the unresolvable
  variable; projecting it, making it an aggregate argument, a `GROUP BY` key, or — on a query
  with no aggregate and no `GROUP BY` — an `ORDER BY` key all resolve it. The composed-aggregate
  route is unaffected: its fold already runs after the reshapers and already excludes
  correctly.

- **On the composed stream+static route, an `OPTIONAL` block's fan-out now reaches the
  aggregate fold even when the block's join key is absent from the aggregate record**.
  `executeWindowedHashJoinWithLookup`'s aggregate topology evaluates `OPTIONAL`
  before the fold, but `mapBindingsToRecord` carries only `SELECT` variables, aggregate
  arguments and `GROUP BY` keys — so a bare `COUNT(*)` left the block's join key out of the
  record entirely, the enrichment fetch had nothing to seed the block's probe from, and the
  `OPTIONAL` null-filled every row regardless of what the repository actually held. The same
  fetch-and-strip mechanism `FILTER NOT EXISTS` has used since alpha.20 now also
  carries an `OPTIONAL`'s join key, on both static-lookup routes — the flat route
  (`executeWithLookupJoin`) had the identical defect and is fixed in the same change. One
  registration side effect matches the `FILTER NOT EXISTS` fetch's own: `ORDER BY`
  may now sort by an `OPTIONAL`-only join key that was previously refused at registration
  because the key could not reach the rows.

- **The composed stream+static route now fetches a `UNION` arm's referenced variables into
  the row, so a `UNION` whose arms correlate on an unprojected variable emits the rows it used
  to drop**. `staticEnrichmentRowVariables()` — the fetch behind
  `executeWindowedHashJoinWithLookup`'s static enrichment — did not include the UNION-arm
  variable set, so an arm's subject that was neither projected nor referenced by an expression
  was absent from the row, `applyUnionBlocks` could never match it, and because no arm
  matching means the `UNION` contributes nothing, every row of the query was silently dropped.
  The flat static-lookup route has fetched this set since alpha.4 — that entry calls the
  route it fixed "the composed stream+static path", the name this file now reserves for the
  windowed-join route; the composed route
  now matches it. Third and last of the unprojected-join-key family on this route, after
  `FILTER NOT EXISTS` and `OPTIONAL` — all three shared the same root cause.

- **A declared `resultMapping` no longer breaks the correlation of an `OPTIONAL`, `FILTER NOT
  EXISTS` or `UNION` arm on the mapped variable**. A mapping declared through
  `CqelsQueryDefinition.Builder.addResultMapping` (or the JSON dialect's `resultMapping`) on a
  variable a reshaper correlates on — e.g. an IRI mapped to `"string"` — was materialised into the
  row in its mapped form before the reshaper ran, and the reshaper's seed re-minted that String as
  an `xsd:string` literal that matched nothing: the `OPTIONAL` null-filled, the `FILTER NOT
  EXISTS` excluded nothing and a `UNION` whose arms correlate on it dropped the row (no arm
  matched), on both static-lookup routes — for NOT EXISTS ever since alpha.20's join-key
  fetch, for OPTIONAL since that fetch's mirror. The row now carries the
  RDF term beside a mapped variable's presentation (`TermCarryingRow`, compiler-internal, never
  emitted) and the reshaper seed reads the term; everything emitted or evaluated — the row,
  FILTER / BIND / HAVING, DISTINCT, ORDER BY, GROUP BY, the aggregates — still reads what it
  read before. Where the engine applies a declared mapping to a row value is inconsistent across
  write sites and readers; that is pre-existing, neither created nor fixed here, and tracked
  separately. The guarantee covers a mapped variable bound directly by a graph pattern:
  a variable that receives a mapped variable's value through a `BIND` still correlates by the
  BIND's own result — the re-minted presentation, not the original term — so a block
  correlating on such a target still null-fills or stops excluding, top-level and block-local
  alike; pinned as a characterized limitation and tracked separately. Only a query
  with both a mapping and a reshaper is affected; every other query keeps byte-identical rows.
  Reachable only through the Builder (CQELS-QL text has no mapping syntax; the JSON dialect
  cannot express a reshaper).

- **`ORDER BY … LIMIT` on the composed stream+static route whose stream side needs the windowed join
  (a multi-triple stream BGP or a variable predicate; no aggregates, a declared window other than
  `[NOW]`) ranked cumulatively across the whole stream instead of within each evaluation, fixed by
  implementing ADR-0002 D3 for that route**. Two independently-discovered symptoms of the
  same root cause: (1) a row already admitted to the top-K could be evicted by a later, freshly
  re-derived duplicate of a DIFFERENT row, silently replacing a still-current, distinct answer with
  a second copy of its peer; (2) once a row won a slot, it kept winning it forever, even
  after its underlying data left the declared window — when the highest-ranked element arrives
  first (a descending sort over a descending stream, say) nothing could ever displace it, so a
  `LIMIT 1` query fell permanently silent after its very first evaluation. A descending
  sort over an ASCENDING stream did not go silent: every arrival is a new maximum, so the top-K
  re-emitted.
  The composed route's non-aggregate topology now materialises a per-evaluation row batch, as
  its stream-only sibling (`executeWindowedHashJoin`) and its own in-batch-aggregate topology
  already do — both correct on this point — and ranks it with the existing, stateless
  `applyBatchSolutionModifiers` (`List#sort` + slice, freshly computed every evaluation — no state
  carried across evaluations for a re-derived row to corrupt). An earlier attempt added
  content-keyed admission to `TopKOperator.offer()` directly; code review found that fix itself
  wrong — it could not tell a re-derived duplicate of one solution apart from two INDEPENDENT
  solutions that happen to project an equal row, and so silently violated SPARQL bag semantics for
  non-`DISTINCT` `SELECT`. That attempt was withdrawn; `TopKOperator`'s ranking is unchanged from its
  prior release behaviour (only its initial heap sizing changed, in the CypherQL `ORDER BY` /
  `LIMIT` entry below) and it stays in use on the boundary-less (`[NOW]`) routes — D3's documented
  exclusion — which keep no window buffer, so each arrival's rows reach it once and there is no
  re-derived duplicate for it to mis-rank. D3's remaining scope — a single fixed-predicate stream
  triple under a non-`[NOW]` window, with or without a static lookup, which
  `executeStreamOnlyWithExpressions` / `executeWithLookupJoin` still refuse at registration with
  `ORDER BY`, `LIMIT` or both, exactly as before this release — is not changed here and is tracked
  separately.

- **Built-in function dispatch, `UCASE`/`LCASE` and a declared `resultMapping` type no longer read
  the JVM default locale**. Four sites on the query-evaluation path canonicalised text with
  the no-argument `String.toLowerCase()` / `toUpperCase()`, which read `Locale.getDefault()` on
  every call — process-global state an application can change at any time, and Turkish or Azeri
  from the start under `-Duser.language=tr`. "The default locale" is not one value: a JVM keeps a
  base default and two `Locale.Category` defaults (`DISPLAY` and `FORMAT`), `Locale.setDefault(Locale)`
  overwrites all three, and `Locale.setDefault(Category, Locale)` overwrites one. The no-argument
  `toLowerCase()` / `toUpperCase()` read the BASE default, so these four sites follow that one; an
  application that sets only a category default does not move them, and a library that sets all
  three and restores only the base leaves `DISPLAY` and `FORMAT` changed — `String.format`,
  `NumberFormat`, `DateFormat`, `getDisplayName` — while case conversion is back where it started.
  The tests here restore all three for that reason. Under such a default a bare built-in name spelled with
  an upper-case `I` (`STRUUID`, `ISIRI`, `IRI`, `CONTAINS`, `MINUTES`, `CEIL`, … — 12 of the
  dispatch switch's 34 labels) canonicalised to a dotless i — `"IRI".toLowerCase()` was
  `U+0131 U+0072 U+0131` under such a default against `U+0069 U+0072 U+0069` under `Locale.ROOT` —
  missed the switch and fell to a
  registry miss, so `BIND(STRUUID() AS ?u)` bound nothing and `FILTER(ISIRI(?s))` dropped every
  row; `UCASE("i")` returned `U+0130` where `Locale.ROOT` returns `U+0049`, and `LCASE("I")` returned
  `U+0131` where `Locale.ROOT` returns `U+0069`, and SPARQL 1.1
  §17.4.3.4 / §17.4.3.5 define both as the locale-insensitive XPath `fn:upper-case` /
  `fn:lower-case`; and a declared `resultMapping` type spelled `"IRI"`, `"Integer"` or `"Instant"`
  fell through to the string extractor, so the same query projected a `String` under one locale
  and an `IRI` under another. Wrong answers, not errors. All four now canonicalise with
  `Locale.ROOT`, as the prefixed-name and `<iri>` dispatch branches and the aggregate-name
  normalisation already did; every spelling that worked under an English default is unchanged. One population sees a CHANGED
  result rather than a fixed one: a Lithuanian (`lt`) default, whose case mappings move the
  combining dot in BOTH directions, and in opposite ways. Measured on JDK 24 — `UCASE("i̇")` was
  `U+0049` under `lt`, dropping the dot, and is now `U+0049 U+0307` under `Locale.ROOT`, keeping it;
  `LCASE("Ì")` was `U+0069 U+0307 U+0300` under `lt`, adding a dot, and is now `U+0069 U+0300`. The
  change is toward the XPath contract SPARQL 1.1 §17.4.3.4/§17.4.3.5 require, but it is a change.
  Found by code review of the ADR-0003 draft. Same-class sites elsewhere in the codebase are
  not changed here and stay tracked on the issue — including two that are on the evaluation
  path and were left out by SYMPTOM rather than location: aggregate-name normalisation in
  `CqelsQueryCompiler.computeAggregate` and `AggregateComputation.compute` runs while producing each
  group's result, and under such a default throws `Unknown aggregate function: MİN` instead of
  answering wrongly.

- **The flat static-lookup route emitted rows no stream element justified, and dropped valid
  ones** (CQELS4J#67). The route — `executeWithLookupJoin`, taken by every single-stream query
  with a static graph pattern under `[NOW]`, and by a single fixed-predicate NON-AGGREGATE stream
  triple with a static pattern under any window (a grouped aggregate over the same shape takes the
  composed route) — never matched the stream BGP against the arriving element.
  It seeded each stream variable one at a time by first match (by predicate for an object
  position; the first statement unconditionally for a subject or variable-predicate position),
  force-bound the first static pattern's subject variable to the join key, and let the static
  matcher bind whatever was still free from the repository. Wrong in both directions:
  - **Fabricated rows (soundness).** An element matching NO stream pattern still produced a row
    whenever a static pattern shared a variable with the unmatched pattern: `STREAM S { ?o ex:of ?f }`
    beside the guard `ex:coll ex:member ?f` answered `{f=ex:allowed}` to a pushed
    `ex:unrelatedObs ex:unrelated ex:x`, under `[NOW]`, `[TRIPLES 1]` and `[RANGE 3s]` alike; a
    constant in the stream pattern (`?o ex:of ex:allowed`) was never checked against the element.
    Under `[NOW]`, a two-pattern stream BGP beside ANY satisfied static pattern stopped enforcing
    its conjunction: an element carrying only the first pattern's statement emitted a row with the
    second pattern's variable simply absent, and when the two statements arrived in separate
    elements the second emitted a complete-looking `{f, val}` row whose `?f` no stream statement
    had bound — the guard supplied it. A consumer cannot tell such a row from a real one.
  - **Dropped rows (completeness).** The root-subject seed overwrote a real binding. With a
    variable-predicate stream pattern (`?o ?p ?f`, which the key extractor does not read) the
    forward guard `?f a ex:Field` was evaluated with `?f` force-bound to the element's SUBJECT,
    so the reporter's correct reverse-guard query answered nothing once rewritten as a forward
    edge, or once the satisfied `?f a ex:Field` was added beside the guard; and a static BGP
    whose FIRST pattern's subject variable is not the join-key variable (`?g a ex:Field` alone,
    or after a reverse guard) had that variable force-bound to the key and lost every row but
    the accidental one.
  The route now applies the stream-only route's constant-aware BGP gate
  (`streamPatternFullyMatches`) before the lookup, seeds the static matcher from that match's
  bindings (`bindStreamPatternVariables`) and pre-binds nothing else, and builds each row's
  stream side from the same match: no row unless every stream pattern matched within the
  element, every stream variable carrying the value the match bound, and a static pattern
  constraining a shared variable instead of assigning it — SPARQL 1.1 §18.3.1, and what
  `CQELS-QL_SPEC.md` §5.2 called an INNER join all along; §5.2's sentence that "only the FIRST
  statement of a multi-statement payload binds each stream variable" described the defect and
  is replaced. Affected releases: the fabricated static-side bindings and the dropped rows date
  from 2.0.0-alpha.19, when this route's static side became matcher-authoritative with a
  repository fallback, and are in 2.0.0-alpha.20; the partial `[NOW]` rows are older. Pinned by
  `CqelsQueryCompilerFlatRouteBgpMatchTest` — the reporter's measured cases, each observed
  failing before the change — and the matrix cells that characterised the partial and stitched
  rows now pin the drop.
  **What to re-check** — any query on this route whose rows you have been consuming:
  (1) a `[NOW]` query with two or more stream triple patterns beside a static pattern, fed
  one-statement-per-element input: it emitted a fragment per element and now emits nothing unless
  ONE element satisfies every pattern (a `RANGE` / `TRIPLES` window joins across elements) — and,
  as this fix first shipped, satisfying them was necessary, not sufficient: the walk took the
  first consistent match per pattern and did not backtrack, so an element that admits a solution
  only by re-matching an earlier pattern contributed nothing (one element carrying
  `(o2 ex:of allowed)`, `(o1 ex:of allowed)`, `(o1 ex:v 1)`, whose one solution needs pattern one
  re-matched, yielded no row, though before the fix it answered correctly by the accident that
  made the fabricated rows possible). The one-element-window entry at the end of `### Changed`
  finds every solution, so in this release that element answers `ex:o1`'s row again;
  (2) a reverse-edge guard such as `ex:coll ex:member ?f`, or any static pattern
  sharing a variable with the stream: rows for elements that did not match the stream pattern
  are gone; (3) a constant in a stream pattern: it is now enforced; (3b) a projected SUBJECT
  variable over a multi-statement element: the old extractor took the FIRST statement's subject
  unconditionally, so `SELECT ?o` could answer a subject from an unrelated statement — a
  fabricated binding on the MATCHING path, not only the unmatched one. It now answers the subject
  the match bound; (4) the forward rewrites and
  `?g a ex:Field`-style shapes above: rows the seed dropped now appear — and a static pattern
  sharing NO variable with the stream now yields the cartesian product where it yielded the
  key subject's own facts. That open question — whether the shape should instead be refused at
  registration on both routes — is now settled the other way, and the composed route was
  brought to this route's answer rather than this route to a refusal; see the entry further below
  on a static graph pattern sharing no variable with the stream BGP.
  **Not changed here**, and left open on CQELS4J#67 when this fix merged: a multi-statement element
  contributed one match — the first consistent one in statement order, as on the stream-only
  route, with no backtracking — so a fully-variable pattern whose relevant statement is not first
  missed, and so did a fixed-predicate multi-subject payload whose only solution needs an earlier
  pattern re-matched. Both are answered by the one-element-window entry at the end of
  `### Changed`. The other two shapes this entry first listed as untouched — the composed route's
  view-scope blindness and this route's projection sensitivity — are fixed by the entry below.

- **Each static-lookup route dropped rows the other answered** (CQELS4J#67). Two fixes had been
  applied to one of the pair and never to its twin, so the same guard, or the same `FILTER`,
  behaved differently depending on which route the query's shape selected. Both are completeness
  defects — no row is fabricated by either — and both are now applied on both routes
  (`ADR-0002` D6's pair-D convergence).
  - **The composed route's static side saw only forward edges.**
    `executeWindowedHashJoinWithLookup` → `enrichBoundRowWithStatic` passed no repository
    fallback to the static-BGP matcher, so the static side was served solely from the forward,
    subject-rooted BFS view rooted at the join key. A pattern that view is structurally blind to
    — a reverse edge into the key, or a constant-subject triple outside the frontier —
    contributed no assignment even though the triple was stored, and the row was dropped: the
    guard `ex:coll ex:member ?f` beside a two-pattern stream BGP answered nothing under
    `[TRIPLES n]`, `[RANGE d]`, `[SLIDE d STEP s]` and `[PAST d]`, and neither did the
    single-pattern form once a grouped `COUNT` sent it to the same route. It now passes the same
    candidate source the flat route has passed since 2.0.0-alpha.19, under the work bound
    it already applied — so a BGP is matched against the graph on both routes, and a reverse edge
    is a triple like any other. Honestly stated, that work bound covers only successful
    extensions, not the candidate fetch itself: an unconstrained pattern's repository read
    materialises its full matching extent before the bound can fire, a pre-existing characteristic
    on the flat route that this fix now also reaches on the composed route — where
    it runs once per arrival per distinct `(subject, predicate, object)` request; it ran once per
    windowed-join result row until a within-arrival memo landed, and it bypasses the
    lookup operator's own `CACHE` TTL by design. The unbounded extent of the read itself is
    tracked separately and not changed here.
  - **The flat route did not carry a FILTER's unprojected stream-side inputs.**
    `executeWithLookupJoin` → `mapToResultRows` fetched the expression-named STATIC-BGP variables
    only, so a `FILTER` naming a stream variable the `SELECT` list omits evaluated it as absent —
    an error, hence `false` — and dropped every row: `SELECT ?f … STREAM S { ?o ?p ?f . } …
    FILTER(?p = ex:of)` answered nothing while the same query projecting `?p` answered correctly.
    The fetch now also carries the expression-named STREAM variables (intersected with the ones
    the stream pattern can bind), which is what the composed route's adapters have copied since
    2.0.0-alpha.13; the non-projected ones are stripped again after post-processing, as
    they always were. SPARQL 1.1 §18.5: `FILTER` applies to the group's solutions before
    projection. **Known, undecided edge**: a projection naming only variables no side binds (e.g.
    `SELECT ?ghost`) now emits one empty row where a `FILTER` references a stream variable; without
    the `FILTER` it emits nothing — both pre-existing shapes, neither decided.
  Affected releases: both defects are in 2.0.0-alpha.19 and 2.0.0-alpha.20; the composed route's
  view scope is older, and the flat route's fetch has had this gap since the expression fetch was
  introduced.
  **What to re-check** — queries whose rows you have been consuming: (1) any static pattern whose
  join-key variable is in OBJECT position (`ex:coll ex:member ?f`), or whose subject is a constant
  outside the key's forward neighbourhood, beside a multi-triple or variable-predicate stream BGP
  under a declared non-`[NOW]` window, or beside a grouped aggregate: rows that were dropped now
  appear, and a multi-valued match fans out per assignment as it already did on the flat route;
  (2) any `[NOW]` query, or single fixed-predicate stream triple, with a static pattern and a
  `FILTER` / `BIND` / `HAVING` naming a stream variable the `SELECT` list omits: rows that were
  dropped now appear. Both directions ADD rows; neither removes any.
  **One registration change**: on the flat route `ORDER BY` may sort by exactly what the
  projection fetches, so a sort key derived from an unprojected stream variable —
  `BIND(?streamVar AS ?rank) … ORDER BY DESC(?rank)` — is now ACCEPTED and sorts for real, where
  it was refused at registration because the key could not reach the rows. A key that NEITHER side
  binds is still refused. Pinned by `CqelsQueryCompilerPairDConvergenceTest` (each cell naming its
  route, with a mutation proof per mechanism) and by the two relicensed
  `OrderByLimitLivenessTest` cells.

- **`GraphPatternJoinOperator` binds a repeated pattern variable consistently and matches
  relationship types exactly instead of by substring**. A pattern edge that reuses the
  same variable at both endpoints, e.g. `(x)-[:KNOWS]->(x)`, now matches only a self-loop edge
  (subject equal to object); previously the second endpoint's binding silently overwrote the
  first, so an edge whose subject and object differ could produce a spurious match with a
  discarded, wrong binding — no match, partial or otherwise, is emitted for such an edge now.
  Relationship-type matching is now exact: a pattern type must equal a candidate predicate's
  full IRI or its local name (the segment after the last `/` or `#`; only when the predicate
  has neither is the segment after its last `:` used instead, so a URN-style predicate like
  `urn:example:KNOWS` still matches by local name without a `:` inside an ordinary path
  segment — e.g. a versioned type `http://ex/KNOWS:v2` — being mistaken for that fallback
  delimiter), instead of accepting any predicate that merely *contains* the pattern type as a
  substring. A pattern that relied on the old substring behavior — e.g. type `KNOWS`
  incidentally matching a predicate like `http://ex/NOT_KNOWS` or `http://ex/KNOWS_WELL` —
  now correctly gets no match for those edges; legitimate matches by full IRI or local name
  (as defined above) are unaffected.

- **A `BIND` target is now a usable probe subject for an `OPTIONAL`, `FILTER NOT EXISTS` or
  `UNION` block on the two stream+static (static-lookup) routes**. Every top-level
  `BIND` ran AFTER the graph-pattern reshapers on these routes, so a block probing a variable a
  `BIND` produces probed an UNBOUND variable. **Which pipeline ran it differs by topology, and so
  did the blast radius:** the per-arrival lookup join and the composed route's FLAT pipeline defer
  `BIND` to `execute()`'s outer pipeline, which runs after the outer reshapers; the composed
  route's per-evaluation `ORDER BY` / `LIMIT` batch and its `Istream(...)` branch have the same
  ordering INSIDE their own batches. The composed route's AGGREGATE topology was never affected —
  it runs its whole `BIND` stage ahead of its in-batch reshapers. Symptoms on the affected
  branches:
  `BIND(?s AS ?key) FILTER NOT EXISTS { ?key ex:role ?e }` excluded nothing (the anti-join was
  inert), `OPTIONAL { ?key ex:tier ?t }` left `?t` null in every row, and
  `{ ?key ex:role ?r } UNION { ?key ex:badge ?r }` emitted no row at all, since a `UNION` row
  survives only through an arm that matched. Projecting the target did not help. The compiled
  query now splits the top-level `BIND` list at the first POSITIONAL block, in declaration order
  (SPARQL §18.2.1 translates a group left to right): a `BIND` written ABOVE the first `OPTIONAL` /
  `UNION` block is evaluated before it, and on these routes a `BIND` written BELOW one still runs
  after it and keeps reading what the block produced. **`FILTER NOT EXISTS` is treated separately
  and is not positional at all** — it is a FILTER, and SPARQL §18.2.2.6 applies a group's filters to
  the whole translated group, `Extend` included, so it now runs after the WHOLE top-level `BIND`
  list on these routes and sees a `BIND` written below it as well as above it. That was the same
  8-rows-instead-of-2 defect with the `BIND` and the block swapped, and the two routes that already
  ran their `BIND` stage first (the stream-only windowed hash join, the composed aggregate topology)
  answered it correctly all along; all routes now agree. (Where a below-block `BIND` runs relative
  to a POSITIONAL block stays route-dependent and is not changed here: the per-arrival stream-only
  route, the stream-only windowed hash join and this same composed route's aggregate topology
  evaluate their whole `BIND` stage ahead of their reshapers, so a below-block `BIND` runs before
  the block there. Newly pinned on all three, pre-existing on all three, and tracked separately.)
  **Rows change** for a composed or per-arrival stream+static query whose reshaper
  probes a `BIND` target — a `FILTER NOT EXISTS` now excludes, an `OPTIONAL` now binds, a
  `UNION` now emits — on the per-arrival lookup join and on all three branches of the composed
  windowed route (its flat pipeline, its per-evaluation `ORDER BY` / `LIMIT` batch, and
  `Istream(...)`), which answer identically. A `[NOW]` + `GROUP BY` query on the per-arrival lookup
  join, which folds inside each arrival, runs the same order inside that fold, so its groups change
  the same way. **Rows can also DECREASE**: a pre-block `BIND` whose
  target the block itself binds now CORRELATES that block instead of being overwritten by it — e.g.
  `BIND("X" AS ?t)` above `OPTIONAL { ?s ex:tier ?t }` goes from 14 rows carrying the repository's
  tiers to 8 carrying `"X"` (measured on the composed route), which is the SPARQL left-join on a
  shared bound variable. A `BIND` that shadows a variable an outer pattern already bound is
  undefined per SPARQL §18.2.2.6 (the target must not already be in scope); this engine neither
  rejects nor defines it, and such a query's rows can move either way. A second, narrower change follows from the same
  ordering: on those routes the `FILTER` stage runs after the reshapers, so a `BIND` declared
  above a POSITIONAL block is now also in scope for the group's `FILTER` (SPARQL §18.2.2.6) — a
  query with NO reshaper block keeps those routes' documented `FILTER`-before-`BIND` order
  unchanged, as does every other route. **Which queries are untouched:** one with no `OPTIONAL` /
  `UNION` / `FILTER NOT EXISTS` block at all, and one whose only blocks are `OPTIONAL` / `UNION`
  with no top-level `BIND` above the first of them — for those the split is empty and the whole
  `BIND` list runs where it always ran. A query with a `FILTER NOT EXISTS` is NOT in that set even
  when no `BIND` sits above it: the anti-join moved from before the `BIND` stage to after it, which
  is the change described just above. The split is at the FIRST positional block: a `BIND` written
  BETWEEN two of them still runs after all of them, since every route applies its positional
  reshapers as one stage.

- **A static graph pattern sharing no variable with the stream BGP no longer makes a windowed
  composed query silently answer nothing**. Such a pattern gives the compiler no
  join key to resolve, and the composed stream+static route (taken when the stream BGP needs
  cross-element join under a `RANGE` / `TRIPLES` window) dropped every row for having none. The
  query then ran forever emitting nothing — no exception, no warning, no empty-result signal —
  on all four of that route's shapes: the plain projection, `GROUP BY` with aggregates (not even
  a zero-count row), `Istream(...)` delta emission, and `ORDER BY` / `LIMIT`. Those rows now
  APPEAR: the static BGP is evaluated with no root, conjunctively, and each stream-side row is
  joined with every solution it has — the cartesian product SPARQL 1.1 §18.5 defines for two
  BGPs with disjoint variables, and the answer the per-arrival lookup route has given since
  alpha.20's static-side convergence (CQELS4J#67). **What to re-check**: any windowed query whose
  static pattern shares no variable with its `STREAM` block. It answered nothing and now answers
  `|window rows|` x `|static solutions|` PER EVALUATION, and the composed route re-evaluates its
  whole window on each arrival, so a forgotten shared variable is now expensive instead of
  invisible. Three things bound and announce that: the EVALUATION's enumeration dies loudly on the
  existing `cqels.join.crossProductCap` work guard (times the static pattern count) rather than
  truncating — the rows of one arrival share one budget, so the guard bounds
  `|window rows|` x `|static solutions|` and not merely each row's share of it; registration logs
  one warning naming the query, the condition and that flag; and
  the repository read behind it is now performed once per arrival per distinct `(subject,
  predicate, object)` request instead of once per result row. **A second shape answers
  too**: a static pattern whose only shared variable sits in PREDICATE position (`?x ?p ex:target`
  beside `STREAM S { ?s ?p ?v . ?s ex:q ?w . }`). Join-key resolution scans subject then object
  position and never the predicate — deliberately, since its job is to find a lookup ROOT and the
  operator is subject-rooted — so that query had no key either and dropped every row; it now
  answers, and because the matcher is seeded with every stream binding the stream's `?p`
  constrains the static side, making it a real join rather than a product (measured: one row, not
  two). The candidate fetch behind it is built from that SEED-BOUND predicate, so the shape answers
  even when nothing else in the pattern resolves: `?x ?p "Alice"`, whose literal-form object has no
  unique repository Value to query by, resolved neither position and never consulted the fallback,
  so on the composed route — whose view is empty for a keyless query — the row was dropped in
  silence, while on the per-arrival route the shape answered only what its rooted view happened to
  carry. Both now fetch candidates from the seed-bound predicate. It is not warned about: the two
  BGPs do share a variable. Unchanged: a static pattern that DOES share a variable in SUBJECT or
  OBJECT position, which still joins on it; and the multi-stream temporal-join route, which still
  refuses this shape at `execute()` — a refusal conditioned on the join key being null, so it fires
  for every keyless static BGP, predicate-only sharing included, rather than on the variable sets
  being disjoint. An algorithm-specific asymmetry, now written down in `CQELS-QL_SPEC.md` §10.7
  rather than left open. Also still unchanged, on both lookup routes: an unbound-subject static
  pattern whose predicate and object are both unbound variables (`?x ?px ?o`), or whose predicate
  is an unbound variable and whose object is a literal-form constant (`?x ?px "Alice"`,
  `?x ?px 42`), gets no CANDIDATES — an unconstrained repository read is refused by design, and a
  literal-form object is not used to constrain one — so it answers only what a rooted
  forward view can supply, which on the composed route is nothing at all (an unjoinable BGP with
  no honoured `operators.lookup.joinKeyVariable` gives it no root, hence no view), while the per-arrival route
  always roots at something — the join key, else the object of a `personId` / `personIri`
  statement, else the payload's first subject — and answers that subject's own facts; and a keyless
  static pattern that lives ONLY inside `UNION` arms, which is not part of the top-level static BGP
  this fix enumerates, so it stays silently empty and unwarned (pinned as a residual, not a
  regression).

- **`operators.lookup.joinKeyVariable` naming something no `STREAM` pattern binds is now IGNORED
  rather than honoured**. Honouring it resolved a non-null join key that no row could bind,
  so every row was dropped by the static-lookup route's key extraction and the unjoinable-BGP
  warning was suppressed — the same defect this entry's sibling fixes, reachable from a config
  field. The hint is
  now validated by POSITION, so a constant term of the `STREAM` BGP whose text matches the name no
  longer qualifies (nor does a bare boolean or numeral, which the pattern classifier would
  otherwise call a variable); the query lands on the answer it would have had with no hint, which
  is the documented one; and the ignored hint logs its own warning naming the query and the name.
  An honoured hint no longer silences the unjoinable-BGP warning either — that warning now tests
  the variable-set intersection it describes, not the join key. One case validation cannot reach,
  because it is data rather than shape: a hint naming an OBJECT-position variable the `STREAM`
  binds to a LITERAL on every arrival. That still drops every row, but no longer in silence — it
  logs once per query, naming the hint and what the row bound it to.

- **CypherQL `ORDER BY` / `LIMIT` now emit on a live stream, and the shapes that cannot
  be answered continuously are refused at registration instead of silently hanging or
  terminating the query**. The tail collected the WHOLE result flux and then sorted
  and sliced it, so on an engine-registered stream — which never completes — an `ORDER BY`
  query emitted nothing at all while buffering without bound, and a bare `LIMIT k`
  compiled to a truncation that cancelled upstream after k rows, silently ending a
  continuous query. Mirrors the CQELS-QL decision; both languages now
  answer the same contract.
  - *Windowed* (`RANGE` / `SLIDE` / `TRIPLES`): the window is the evaluation boundary, so
    `ORDER BY` / `LIMIT` rank and truncate **per evaluation** — a stateless sort-and-slice
    over that window's rows, recomputed every evaluation and carrying no state across
    them. Both the non-aggregate and the aggregate pipeline do this. Of the three, only
    `TRIPLES` closes on a LIVE stream today: `TumblingWindow` / `SlidingWindow` collect
    their source and fire on completion, so a `RANGE` / `SLIDE` query ranks correctly but
    stays silent until the stream ends. That is a pre-existing property of those window
    operators — the same query with no modifiers at all is equally silent — and is not
    changed here; it is pinned by a characterization test and left to a follow-up.
  - *Boundary-less* (`[NOW]`, or no declared stream) with `ORDER BY ... LIMIT k`: an
    explicitly GLOBAL continuous top-K. The whole current snapshot is re-emitted whenever
    it changes, and an arrival that changes nothing emits nothing.
  - *Boundary-less with a bare `ORDER BY`, or a bare `LIMIT`*: **rejected at registration**
    with an `UnsupportedOperationException` naming the two supported shapes. There is no
    bounded row set to order, and a global `LIMIT` on a continuous query is never what the
    author meant. (Ordering by an ascending event-time attribute is a sound shape tracked
    separately; it is not implemented here.)
  - *An `ORDER BY` that names a `RETURN` alias now ranks by that alias's value.* On the
    stream-only route the row is keyed by `variable.property` and an alias is never
    materialised into it, so `RETURN metric.value AS score ORDER BY score` compared a key
    that did not exist — every comparison a null tie, leaving the order to arrival. This was
    already true of the batch sort before this change; it is fixed for both it and the new
    continuous top-K. An `ORDER BY` naming an aggregate's alias is unchanged, because the
    fold writes that alias as the row key.
  - *Boundary-less **aggregates** with `ORDER BY ... LIMIT`*: also **rejected**, with or
    without `GROUP BY`. Each arrival now folds to a complete snapshot that the next arrival
    supersedes, while the continuous top-K ranks across arrivals — so the ranking's scope is
    undefined, and an obsolete value keeps winning (an ungrouped `MAX(...) ORDER BY ... LIMIT
    1` over an arrival of 100 followed by one of 1 would keep answering 100). The same
    refusal CQELS-QL kept for this shape.
  - *A huge `k` no longer fails or pre-allocates the continuous top-K.* `TopKOperator` (shared by
    the CQELS-QL and CypherQL top-K) sized its heap's backing array to `k + 1` at construction, so
    `k = Integer.MAX_VALUE` threw `IllegalArgumentException` and `Integer.MAX_VALUE - 1` an
    `OutOfMemoryError` before a row was ranked, and any large `LIMIT` paid for rows that never
    arrived; the heap now starts at `min(k, 64) + 1` and grows on demand, so `k` is a bound, not an
    allocation (the class Javadoc on `INITIAL_HEAP_CAPACITY`; pinned by
    `TopKOperatorTest#testAHugeKDoesNotEagerlyAllocateTheHeap`).

- **A CypherQL `[NOW]` `GROUP BY` aggregate now emits one evaluation per arrival**.
  `[NOW]` is a zero-length window holding only the current arrival, but the
  `GROUP BY` branch collected the entire stream into a single batch: on a live stream the
  query emitted NOTHING while the list grew without bound, and on a finite source it
  emitted one fold at completion instead of one per element. The arrival is now the
  evaluation boundary, with or without `GROUP BY`. **This changes what such a query
  answers**: a count under `[NOW]` can no longer exceed the rows a single arrival
  produces. To aggregate ACROSS elements, declare a window that accumulates
  (`RANGE` / `SLIDE` / `TRIPLES`). Mirrors the CQELS-QL decision.

- **CypherQL now emits every match and resolves RETURN, property names and literal values exactly**.
  Affects every CypherQL query; a query that relied on
  the previous behaviour will see different rows.
  - *Every match is emitted.* The stream matcher returned at the first matching subject and the
    first successful relationship-chain orientation, so a stream element containing two `:Sensor`
    nodes, or two valid paths for one chain, produced ONE row instead of two. Matching now
    enumerates every complete binding — every candidate subject, every chain orientation with
    backtracking, and the Cartesian product across independent patterns — and projects ONE row per
    binding. An undirected pattern therefore matches an edge in both directions, as Cypher's
    `(a)-[r]-(b)` does. Identical bindings are collapsed (RDF triples are a set).
  - *Separate pattern GROUPS are now a conjunction.* An unsatisfiable group empties the
    binding set for that element. A query whose groups name different streams — e.g.
    `MATCH STREAM A { (xx:Sensor) }, STREAM B { (yy:Gauge) }` — emitted one partial row per
    element before; this change alone made it emit none, because each element is matched
    against every group and no single element satisfies both. In this release that shape is
    refused at registration instead — see the `### Changed` entry on a `MATCH` reading only the
    stream it names.
  - *A repeated variable is checked, not overwritten.* `MATCH (n:Sensor), (n:Actuator)` over two
    DIFFERENT subjects rebound `n` and emitted an invalid row; it now yields no row, and one
    subject carrying both labels yields exactly one. The same rule applies to a relationship's
    endpoints, so `(x)-[:R]->(x)` now matches only a genuine self-loop.
  - *`RETURN` resolves from the bindings.* On the static-lookup route a simple `RETURN a` picked
    any subject whose IRI merely CONTAINED `a` — inventing a value for an unbound variable and
    missing a genuinely bound one. It now resolves by exact variable name from the binding map,
    and an unbound variable projects no key. `RETURN v.prop` parsed the variable and then ignored
    it, so it could read another node's value; it now reads only triples whose subject is `v`'s
    bound resource, and the pattern must actually match for the row to exist.
  - *Property names match exactly.* `v.age` was resolved with `predicate.endsWith("age")`, so a
    `.../voltage` predicate answered it. Resolution is now local-name equality, the same rule the
    node-property, relationship-type and label matchers already use, applied identically to stream
    and static triples.
  - *Literal values keep their datatype.* Every literal was run through `Integer.parseInt` then
    `Double.parseDouble` regardless of datatype, so a plain `"42"` surfaced as the number 42
    (and `"001"` as 1, `"1e3"` as 1000.0). Conversion is now datatype-driven and matches the
    CQELS-QL row representation: the XSD integer hierarchy to `Long`, `xsd:double`/`xsd:float`/
    `xsd:decimal` to `Double`, `xsd:boolean` to `Boolean`, everything else — plain, `xsd:string`
    and language-tagged literals included — to `String`. **This CHANGES comparisons a query could
    have relied on:** a plain numeric-looking literal now compares and sorts as a string, and
    equality against a quoted value in `WHERE` now matches where it previously did not. The
    sentence above describes the STREAM route; on the static-lookup route `RETURN v.prop` used
    to return every value as a `String` (it went through a string extractor), so the direction
    there is the opposite one — `"42"^^xsd:int` was the string `"42"` and is now the number
    `42`. Both routes now answer the same rule. Consumers that serialise these rows see the
    change: `cqels-mcp`'s query tool emits JSON `42` where it emitted `"42"`.
  - *A relational comparison across value categories no longer matches.* `WHERE item.code > 10`
    over a plain `"7"` compared the two LEXICALLY and admitted the row; a string and a number
    have no ordering (this engine's own expression evaluator holds such operands incomparable,
    and openCypher yields null), so the row is now dropped — and, because an incomparable pair
    is a type error rather than a false boolean, `NOT` does not turn it into a match. Equality
    follows the same category rule — cross-category `=` is false and `<>` is true —
    which now also covers the boolean/string coercion that used to make a plain `"true"`
    equal to the boolean `true`, and `IN` membership with it (a plain `"true"` is not a
    member of `[true]`; numeric types share one category, so a `Long` row value still matches
    `[45.0]`). The rule holds for **every** clause that compares: `WHERE`, `HAVING` and the
    inline node-property constraint `MATCH (n {code: 42})` all go through one comparison now.
    They did not before — `HAVING` compared without the rule, and the inline form compared raw
    string forms — so the same constraint on the same value answered differently depending on
    where it was written. A type error —
    this one, or a `WHERE` over a property the row does not have — is now UNKNOWN in a
    three-valued sense and propagates through `AND` / `OR` / `NOT` instead of collapsing to
    false at the leaf, so `NOT (code > 10)` does not readmit the row. That also makes the
    two `WHERE` evaluation paths agree on a missing property, which the parenthesised path
    previously negated into a match.
  - *Relationship matching is isomorphic and edge-aware.* No statement may satisfy two
    relationships of the same match, so `(a)-[:R]-(b)-[:R]-(c)` over a single edge is no longer
    two rows by walking back over it. Two PARALLEL edges between the same pair of nodes are two
    solutions again — the identical-binding collapse is now keyed on the matched statements as
    well as the named bindings, so a genuinely duplicated statement in one element still
    collapses while parallel edges do not.
  - *An anonymous node inside a relationship chain links its hops.* `(a)-[:R]->()-[:R]->(c)`
    gave both relationships a null endpoint, so the hops shared no join variable and matched
    independently — a Cartesian product of unconnected hops, including pairs with no path
    between them (9 rows where the pattern describes 2) — and any label on that node was
    dropped. The compiler now names such a node internally; the synthetic name never reaches a
    result row.
  - *An inline property on a node inside a relationship pattern is enforced.* The
    relationship matcher checked its endpoints' labels but never their properties, so
    `MATCH (a {code: "nope"})-[:R]->(b)` matched a node whose `code` is something else.
  - *A node is any resource, not only a subject.* A node that appears only in OBJECT position
    can now bind, so `MATCH (a)-[:R]->(b), (b)` matches instead of yielding nothing. An
    unconstrained `(n)` therefore also binds resources that only appear as objects, including
    the class IRI of an `rdf:type` triple.
  - *One local-name rule.* Row keys were built by one resolver (`#` then `/`) and matching by
    another (`#`, `/` or `:`), which disagreed on URNs: over `urn:example:temp` the property
    was matchable as `temp` but its row key was `sensor.urn:example:temp`, so `WHERE
    sensor.temp = …` found nothing. Both now use one resolver, with the colon as a FALLBACK
    delimiter only — used when the IRI has neither `/` nor `#`, so `http://ex/KNOWS:v2` keeps
    the local name `KNOWS:v2`.
  - *A `WITH DEPTH n` lookup's later hops are visible to the pattern.* The static route read
    only the statements stored under the ROOT subject, so a depth lookup that followed an edge
    to a second resource fetched its type and properties and then matched against neither:
    `(a)-[:KNOWS]->(b:Person)` emitted nothing. Matching and `RETURN` now read every fetched
    hop.
  - *An unbounded cross product fails loudly.* Independent patterns multiply, so one element
    could materialise an enormous product silently. The matcher now throws once an element's
    intermediate bindings exceed `cqels.join.crossProductCap` (default 100000), in the same
    idiom — and for the same reason — as the CQELS-QL join's work bound: truncating would
    silently drop valid rows and make any downstream aggregate wrong with no signal.
  - *Composite `GROUP BY` keys no longer collide.* The key joined the group values with a literal
    `||`, so `("a||b", "c")` and `("a", "b||c")` merged into one group and combined their
    COUNT/SUM/AVG. The key is now a structured immutable list compared element-wise by typed Java
    value, so nulls and same-`toString()`/different-type values stay distinct; groups are emitted
    in first-seen order. Numeric keys are canonicalised first, so `"1"^^xsd:int` and
    `"1.0"^^xsd:decimal` are ONE group — every other comparison here treats all numbers as one
    category, and the key now does too, while a number and the string `"1"` stay two groups.

- **`store_memory` metadata (`meta`) now works on the persistent RDF store, which previously
  rejected it outright**. Statement metadata is stored as RDF-star annotations whose subject
  is the quoted base triple, but the on-disk RDF4J NativeStore behind `rdfStorePath` /
  `CQELS_MCP_RDF_STORE_PATH` cannot hold quoted triples, so any `store_memory` call carrying `meta`
  failed with `value parameter should be a URI, BNode or Literal` and stored nothing — leaving
  valid-time (`asOf`) recall, `includeMeta`, the `source`/`minConfidence` filters and labelled
  governance (`accessLabel` + `set_access_policy`) unusable on a persistent deployment. On a
  NativeStore the same metadata is now attached to a deterministic reification node
  (`rdf:subject`/`rdf:predicate`/`rdf:object`) in the annotation graph, and every writer and
  reader — store, metadata recall, `forget_memory`'s cascade, and the governance label lookup —
  derives the encoding from the repository, so they always agree and recall behaves identically;
  metadata, access labels and policies survive a restart. The in-memory store keeps the RDF-star
  encoding unchanged.

- **A `FILTER NOT EXISTS { … }` body no longer drops the clauses it cannot evaluate**.
  The CQELS-QL builder kept only the triple patterns and a `STREAM` block of the body and skipped
  everything else, and the anti-join probes only the static triple patterns, so it ran without the
  rest. With an inner `FILTER` dropped it removed rows it should have kept:
  `FILTER NOT EXISTS { ?person ex:email ?email . FILTER(CONTAINS(?email, "carol")) }` excluded
  every person with an email, not only Carol. A `STREAM` block was kept but never read: a body of
  only a `STREAM` block excluded no row, and a `STREAM` block beside triple patterns answered the
  same as the triple patterns alone. A `STREAM`, `FILTER`, `BIND`, `GRAPH`, `OPTIONAL`, `UNION`,
  `MINUS`, `WINDOW` or nested `FILTER NOT EXISTS` inside the body is now refused at parse time,
  naming the construct and its line and column. This refuses the shapes rather than evaluating them.
- **A SELECT expression that wraps an aggregate no longer answers the bare aggregate**.
  The builder registered the first aggregate it reached under the alias and discarded the rest of
  the expression. Measured over two events in one `GROUP BY` group: `(COUNT(*) * 10 AS ?c)` answered
  the count, `(SUM(?v) + COUNT(*) AS ?c)` answered the sum, `(COUNT(*) > 1 AS ?c)` answered the count,
  and `(-SUM(?v) AS ?c)`, `((COUNT(*)) AS ?c)` and `(ABS(SUM(?v)) AS ?c)` registered no aggregate at
  all, so the rows carried no `?c` and were not grouped. A SELECT expression that
  contains an aggregate call but is not a bare `AGG(...)` — arithmetic around it, two aggregates, a
  comparison, a sign, extra parentheses, a function call over it — is now refused at parse time.
  `HAVING` expressions are not affected.

- **A client's SPARQL can no longer make the MCP server send a request to a URL the client chose**.
  The `query` tool, `recall_memory`'s `sparql` mode and stored `sparql` procedures passed the
  client's text to the repository unchanged, and RDF4J's default federated-service resolver is on the
  `cqels-mcp` classpath, so a `SERVICE <url> { … }` clause made the server fetch that URL — any host
  the server could reach, including ones the client could not — and return the response as rows. The
  server now installs, once, on the repository its tools read, a resolver that refuses every `SERVICE`
  clause before any request is sent (`SERVICE <url> refused: this server does not evaluate federated
  SERVICE clauses in client queries`), and those three paths now prepare their queries with a maximum
  execution time (new `clientQueryTimeoutSeconds`, default 30 s). RDF4J checks that limit when the next
  result row is requested, so a query that computes for a long time before its next row (a large
  aggregate, for example) fails only once that row is ready. An operator allow-list for `SERVICE` hosts
  is not part of this change.

- **On the multi-client HTTP transport, a session sees and controls only the standing queries it
  registered**. Session ids were only a quota bucket, so any session could list another
  session's queries (`cqels://queries`, `cqels://kg/stats`), read and drain its results
  (`cqels://queries/{queryId}/results`, `recall_memory` with `queryId`) so the owner never saw them,
  and `forget_stream_query` it. Each of those paths is now owner-checked, and another session's query
  reads exactly like an unknown one. The `resources/updated` signal for a `notify: true` registration
  reached every session at that point (MCP SDK 1.0.0 tracks no subscriptions); it carries only the
  resource URI, and a non-owner's read of that URI returns an empty result list and leaves the owner's
  rows in place. Later in this release the signal reaches only the session that registered the
  query: see the entry on the `notify: true` result signal below. stdio is a single session and is
  unchanged.

- **On the HTTP transport, a session's registrations are released when the session closes or idles
  out, and one client can no longer exhaust the registration caps for everyone**. Registrations
  outlived their session, and the per-session cap of 100 was keyed by session id, so one client
  rotating sessions filled the server-wide cap of 1000; after that no session could register a stream
  query, and a one-shot CQELS-QL or Cypher `query` failed too, because it took a registration slot. Now
  an HTTP `DELETE` that closes a session unregisters every registration it made (engine subscription
  and buffer), and a session with no request and no open event stream for
  `httpSessionIdleTimeoutSeconds` (default 1800) is released the same way (later in this release the
  idle sweep also ends the session in the transport, so its id gets `404`: see the entry on every
  session bound to an identity below). MCP SDK 1.0.0 has no
  session-close callback, so the server observes the session's requests on the MCP endpoint. The
  one-shot `query`'s temporary registration no longer counts against, or is refused by, the standing
  caps; at most 10 per HTTP session and 100 in all run at once. The `register_rules` (8) and
  `watch_invariant` (16) caps are now per session on the HTTP transport; on stdio they stay global.
- **A SHACL shape that is not typed `sh:NodeShape` is no longer silently skipped**. The
  `cqels-shacl` parser visited only subjects typed `sh:NodeShape`, so a top-level
  `sh:PropertyShape` carrying its own target, or an untyped subject carrying a target, enforced
  nothing and the window reported `CONFORMS`. Shapes are now discovered the way SHACL §2.1 names
  them: a typed node or property shape, a subject with a target, or a subject with `sh:path` or a
  constraint parameter. A targeted property shape is validated over its own targets (reported
  under its own id), an untyped targeted subject is validated as a node shape, and a property or untyped shape
  with no target that nothing references is ignored with one INFO log line, since SHACL gives it no
  focus nodes (a typed `sh:NodeShape` with no target is still a parse error, as before). Every discovered shape passes the supported-constraint allow-list. A shape that is also
  an `rdfs:Class` now validates its implicit class target as well as any explicit target; before,
  the implicit one was dropped when an explicit one was present, and a shape with only the
  implicit target was refused. A graph whose only shape is a targeted property shape now parses.
- **Damage inside a journal's committed prefix now fails the open instead of being trimmed away
  as a torn tail, and every record of a new journal carries a CRC32C**. Under
  `SYNC_ON_COMMIT` (the provider default), recovery used to stop at the first unreadable record
  even when it lay below the commit pointer, truncate the file there, lower the pointer and open
  normally: acknowledged records after the damage were deleted and their offsets handed out again
  to new events. A pointer naming more bytes than the file held was lowered the same way, and with
  no checksum a flipped payload byte replayed as different data. Now a record below the pointer
  that cannot be read in full or fails its checksum, or a pointer beyond the end of the file,
  makes the open throw `CorruptCommittedStateException` naming the file, the byte offset and the
  pointer, and leaves the journal and its commit pointer untouched; `PersistenceCoordinator` rethrows it from primary-backend
  startup even under `failIfPrimaryUnavailable=false`, as it already did for a corrupt checkpoint.
  Damage at or after the pointer is still a torn tail and is still trimmed. New journals are
  written in on-disk format 2 — the file starts with the 8-byte header `0x89 "CQELSJ" 0x02` and
  every record is `[offset][len][payload][crc32c]`; recovery and every `readFrom` verify the
  checksum, and a committed record that fails it on a later read throws the same exception. The
  format is decided from the header alone, never from the records: a record's bytes are
  chosen by whoever produced the event, and a CRC can be forced to any value, so an earlier
  pre-release build that took a matching checksum as proof of format 2 let an ingest client make
  a format-1 journal refuse to open.

- **Journal offsets no longer restart at 1 after a `truncateBefore` that empties the journal and a
  restart**. The next offset was recovered from the records still on disk, so truncating
  everything a checkpoint covered and restarting assigned offset 1 again, and
  `readFrom(checkpointOffset)` then skipped every event appended after the restart. `truncateBefore`
  now durably records the committed tail offset in an `event-journal.offset` sidecar before
  removing records, and the open resumes from the larger of that value and the highest offset on
  disk, under both durability policies and across a switch between them. The sidecar carries its
  own CRC32C and records the format as a cross-check of the header; it is written at the first
  open, so a damaged sidecar, a missing one beside a format-2 header or a `journal.format` marker,
  or one that disagrees with either fails the open with `CorruptCommittedStateException`. So does a
  missing `event-journal.bin` in a directory that already holds `journal.id`,
  `event-journal.commit`, `event-journal.offset` or `journal.format`, which used to open as an
  empty journal under `NONE`.

- **RETE `PRIORITY` conflict resolution no longer loses an inference to a higher-priority rule whose
  filter rejects it**. Two rules with the same head — say "too hot" (priority 10,
  `FILTER(?v > 100)`) and "too cold" (priority 1, `FILTER(?v < -20)`) — used to let the higher one
  claim the head before any filter ran, so a reading of −30 raised no alarm. An activation whose
  filters fail now claims nothing, and the next activation for that head in priority order fires;
  among equal priorities the first *passing* rule in declaration order wins. With truth maintenance
  on, the supported-facts view and the emitted inferences now agree on this shape (the view used to
  report the alarm that was never emitted). `PRIORITY` is the builder
  default and the `cqels-cdsp` reasoning bridge's fixed setting; `ALL` resolution is unchanged.
- **Windowed recursive inference without truth maintenance no longer keeps a derived fact past its
  oldest premise, nor lets it expire while a newer derivation still holds**.
  This is the configuration of every shipped reasoning profile (RDFS, RDFS-Full, OWL 2 RL/QL/EL,
  OWL-Lite) and of `cqels-cdsp` with chaining on. A derived fact used to be re-fed stamped with the
  event that completed it, so it outlived its oldest premise and kept producing consequences the
  window no longer entailed; and a second derivation of a fact already derived was dropped, so the
  fact expired with its *first* derivation even while a newer one still held. A derived fact now
  carries the earliest timestamp among its premises and leaves the window with that premise; a later
  derivation whose oldest premise is newer refreshes it and its consequences the same way, and
  re-emits it when the refreshed fact would outlive its last appearance in the stream (an
  assertion or an earlier emission), so a downstream window keeps holding it. With truth maintenance on, the regression shapes were already answered this way, and
  the two configurations now report the same supported facts after every event of those shapes.

- **A failed periodic checkpoint write on the plain `push` path no longer kills the stream and
  every query on it**. The write runs inside the stream's ingest path, so its exception
  terminated the stream while `isRunning()` stayed `true` and a later `tryPush` returned `true`
  for an element that was neither journaled nor delivered. It now follows the policy `ingestBatch`
  already applies: logged, counted on the same checkpoint-bookkeeping failure meter, retried on
  later elements, and the element is journaled and delivered.
- **A failed journal append on the `push` path now leaves the stream honestly terminated**.
  The append stays fatal for that stream — an element that is not journaled is not
  delivered — and every query on the stream still receives `onError`; the `DataStream` handle is
  now terminated too, so `tryPush` returns `false` and `push` drops, as for any terminated stream.
  The engine keeps running.
- **`close()` completes its teardown when the forced final checkpoint write fails**. It
  completes the stream sinks, releases the persistence backend, detaches the repository listener
  and shuts down an owned repository, and then rethrows the checkpoint failure; before, the
  throw skipped all of those steps.
- **A restart with the default `RecoveryPolicy` replays the journal when the checkpoints carry no
  query state**. A clean `stop()` writes a final checkpoint stamped at the last offset, and
  recovery skipped every journal record that stamp covered — but engine-built CQELS-QL and Cypher
  queries contribute no state to checkpoints, so a restart after a clean stop replayed nothing and
  their windows restarted empty. A checkpoint now covers its journal prefix only when it carries
  a state blob for every registered contributor and every registered query is one; otherwise
  recovery restores no checkpoint blob and replays the journal from its start (or from the last
  truncation point).
  `truncateJournalAfterCheckpoint` likewise refuses, with a WARN, to delete a prefix such a
  checkpoint is stamped over.

- **MCP `record_event` no longer merges two events recorded at the same time in different runs on a
  persistent RDF store**. The event IRI was the time plus a counter that restarted at 1 with
  every process, so after a restart an event recorded at an already-used time reused an earlier
  event's IRI: the two events became one node, and `recall_episodes` returned four events for two,
  two of them crossed (Alice did Bob's thing). The suffix is now a random
  64-bit value, so the IRI is `cqels://memory/event/<time>-<base-36 suffix>`. Decision ids minted by
  `run_procedure` (`cqels://memory/decision/<time>-<suffix>`) had the same counter and get the same
  suffix.
- **MCP `forget_memory` with `facts` reports the facts it actually removed**. It reported
  every well-formed entry as removed, so a fact with an IRI object (the schema had no way to say
  so), a fact stored in a user graph, and a fact that never existed all came back as
  `Removed 1 facts` while the store was unchanged. The count is now the number of statements
  removed, and each entry that matched nothing is named in the result, with a hint when an
  IRI-like object was matched as a literal. The `facts` items now declare `objectType`
  (`uri` | `literal`), `datatype` and `language`, so IRI objects and typed or language-tagged
  literals can be named. An entry that matches nothing no longer deletes the metadata of the same
  triple elsewhere, and a removed fact's metadata now goes only with its last copy: metadata is
  stored per triple, not per graph, so forgetting a fact from one graph keeps the metadata of a copy
  in another. While an access policy is active the result withholds the per-fact count and list,
  which would otherwise tell a caller whether a guessed restricted fact existed.
- **MCP `forget_memory` with `facts` and `graph` removes only those facts instead of clearing the
  whole graph**. The `graph` argument was handled first and the call returned after
  clearing it, so a request to forget one fact from a user's graph erased the user's whole memory
  graph. `{facts, graph}` now removes exactly those facts from that graph, de-indexing them and
  cascading their metadata as the longterm path does; `sparql` combined with `graph` or `facts` is
  rejected as ambiguous.

- **MCP HTTP transport: every session bound to an identity in every mode, under a total session cap**. With the single shared token or no
  authentication, an `initialize` was never refused, and the idle sweep (added earlier in this
  release) released an idle session's registrations but kept its MCP SDK transport session, so a client that initialized repeatedly
  without a `DELETE` grew the SDK's session table without limit (the per-client caps applied
  only with per-client tokens). Every HTTP session is now bound: with per-client tokens to its client,
  as before, and otherwise to one shared identity, internal to the session filter (handlers still see
  an anonymous caller, so stream ownership is unchanged).
  - **Behaviour change: new `429`.** The total number of sessions is capped in every HTTP mode by
    `CQELS_MCP_HTTP_MAX_SESSIONS` / `-Dcqels.mcp.http.maxSessions` /
    `CqelsMcpServerConfig.Builder.httpMaxSessions(int)` (default 1 024 — the previous fixed total of the
    per-client mode; 1 to 65 536, another value refuses the start naming the setting). The shared
    identity is held to this total only, not to the per-client 32, because one shared token serves
    many agents. An `initialize` beyond the cap gets `429` and creates no session.
  - **Lockout risk.** A slot is freed only by the session's `DELETE`, by the idle sweep ending the
    session, or by an `initialize` the SDK refuses with a 4xx. A session abandoned without `DELETE`
    holds its slot until the idle timeout — and until restart with `CQELS_MCP_HTTP_SESSION_IDLE_TIMEOUT=0`.
    With the shared token or no authentication there is no per-client limit, so one agent reconnecting
    without `DELETE` can take every slot and lock out every other agent until its sessions expire. An
    `initialize` that ends with a 5xx or another non-4xx status, without a session id, keeps its slot until restart, now in every
    mode (the SDK may hold a session whose id was never sent).
  - **Behaviour change: idle expiry ends the transport session.** With the shared token or no
    authentication, a session idle past `CQELS_MCP_HTTP_SESSION_IDLE_TIMEOUT` is now ended in the
    transport as well and its slot freed; a request with its id gets `404`, to which an MCP client
    answers with a new `initialize`.
  - The request checks of the per-client mode now apply in every HTTP mode: an id the server never
    issued (or closed or expired) gets `404` before the transport; a POST that is not a JSON-RPC message
    gets `400`; an `initialize` carrying a session id, or without an `Accept` listing both
    `application/json` and `text/event-stream`, or whose `params` the SDK cannot convert, gets `400`
    and creates no session; an `initialize` missing `capabilities` or `clientInfo.name`/`version`, or with no `protocolVersion` — all required by the MCP specification — gets `400` and creates no session (before, without per-client tokens, the transport accepted the first two and created a session before failing on the missing `protocolVersion`); one refused by the Origin/Host allow-list gets its
    status (`403` for the Origin, `421` for the Host) before a slot is taken. Any holder of the shared token may still use or `DELETE` any session, and a
    `DELETE` of an issued id that the SDK answers `404` (it dropped the session after a failed write)
    now releases that session in every mode.
- **MCP tool descriptions and docs: what a late registration receives**. The
  `push_stream_events`, `create_stream`, `watch_invariant` and `register_rules` tool and `stream`
  parameter descriptions, the server instructions, the `memory_routing` prompt, the
  `cqels://docs/memory-profile` resource (including its note on a one-shot CQELS-QL/Cypher `query`) and
  the README told agents that a query registered after the pushes receives none of them. A stream keeps
  its last `cqels.engine.streamReplayLimit` elements (default 8192) for a continuous query, invariant or
  one-shot `query` registered later (not for a CEP query, `cep: true`, which subscribes to the live
  stream and sees only events pushed after it is registered); older elements are gone. The texts now
  say so, with two further limits. A rule program (`register_rules`) receives the replay one solve per
  observation through its pending-solve buffer of 1024 observations plus 256 in flight (about 1280):
  newer observations of a longer replay are dropped and logged, and observations pushed while that
  backlog is being solved can be dropped too. A one-shot `query` returns at most `limit` rows (default
  100); the oldest replayed rows beyond it are counted as dropped. The ownership docs add that on a
  shared stream a late subscriber receives that history (data meant for one client belongs in a private
  stream), and the restart-replay notes in the README, the memory profile and `docs/UPGRADING.md` now
  exclude CEP queries too. No behaviour change except the wording of the `register_rules` overflow
  warning row (it now also names a late registration's replay as a cause); tests pin each statement.
- **Client identity and private streams on the MCP HTTP transport: co-review follow-ups**.
  - A clients file saved with a UTF-8 byte order mark failed the start with a misleading message; the
    mark is now ignored. A token with a non-whitespace control character or a non-ASCII character (a
    no-break space, for example) was accepted at start and could never match. Tokens are now restricted
    to RFC 6750 `b64token`, which also refuses some printable ASCII that was accepted before (`!`, `#`,
    `$`, `%`, `:`, `@`, `*` and the like); a refused token stops the start, naming the line (or the
    client id, for `Builder.httpClients`) and never the token.
  - The per-client cap on private streams now counts a private creation whose `STREAM_CREATE` append
    ended in an indeterminate commit: the engine fences the name and publishes no stream, but the next
    recovery may restore it with its owner, so before this a client at the cap could create one more
    and own one beyond the cap after a restart. Engine: `CQELSEngine.uncertainStreamCreations()`
    returns the fenced creations with the attributes they asked for.
  - A one-shot CQELS-QL/Cypher `query` read its temporary registration by id with no caller, so if
    that registration was forgotten while the query polled and another session registered a query
    under the same id, the poll returned that session's rows. It now reads only its own registration
    generation and stops when that is gone.
  - With per-client tokens and the session idle sweep off (`CQELS_MCP_HTTP_SESSION_IDLE_TIMEOUT=0`), a
    session the transport had dropped after a failed write to its event stream was never released: its
    client's `DELETE` got `404` and nothing else ended it, so its registrations kept pinning the streams
    they read and its session slot stayed taken. That `DELETE` (still answered `404`) now releases the
    session like a closing one, in every HTTP mode since every session is bound; with per-client
    tokens another client's `DELETE` of the id is still refused with `403`.
  - The MCP HTTP endpoint now serves only its exact raw path, in every HTTP transport mode (per-client
    tokens, the single shared token, no authentication). A request to an encoded or parameterised form
    of it (`/mc%70`, `/mcp;x`) gets `404` with `{"error":"not found"}` instead of the server's default
    error page, and a raw path that only decodes to the endpoint (`/a/../mcp`), which the transport
    used to serve, now gets `404` too. This is what keeps such a `DELETE`, which the transport answers
    `404` without looking at its sessions, from releasing a live session, in every HTTP mode since every
    session is bound. MCP
    clients send the endpoint path as configured and are not affected. The configured path
    (`CQELS_MCP_HTTP_PATH` / `Builder.httpEndpoint`) must use unencoded URI path characters (unreserved,
    sub-delimiters, `:`, `@`, `/`); `;`, `%`, a space, `?`, `#`, `*`, non-ASCII characters and an empty,
    `.` or `..` segment now refuse the start. Such a path never worked: it never matched the raw request
    path (the transport compared that too), or Jetty canonicalised it away or rejected it before routing.
  - A request naming another client's private stream is refused before it waits for any per-name lease;
    before this, the refused caller could hold a lease on another name it named while queued for up
    to 30 s, and a refused removal waited for the owner's operation on the name first.
- **A `notify: true` result signal on the MCP HTTP transport reaches only the session that registered
  the query**. `register_stream_query`, `watch_invariant` and `register_rules` with
  `notify: true` sent `notifications/resources/updated` for `cqels://queries/{queryId}/results` to every
  connected session, so every other session learned the query id and when each result arrived (the
  read the signal prompts was already owner-checked, so no row leaked). The signal now goes only to the
  event stream of the session that owns the registration, as the same JSON-RPC notification as before;
  if that session has ended, the signal is dropped, not broadcast. The server holds each session only
  weakly, so it never keeps a session the transport has ended — whether its client closed it
  (`DELETE`), the idle sweep ended it, the server shut down, or the transport dropped it after a failed
  write to its event stream (MCP SDK 1.0.0 does that without telling anyone). The idle sweep ends the
  transport session in every HTTP mode, so a request with an expired id gets `404` and the client
  initializes again; only when that end fails does the transport keep the session, which then stays in
  the directory, and a registration the client makes in it afterwards is still signalled.
  stdio is unchanged (one session). The `cqels://memory/inferred` signal of `register_reasoning` stays a
  broadcast: that view is shared by every session. Tests: `CqelsMcpHttpSessionIsolationTest`
  (`resultNotificationReachesOnlyTheOwningSession`, which replaces
  `resultNotificationReachesEverySessionButOnlyTheOwnerReadsTheRow`,
  `reasoningToolResultSignalsReachOnlyTheOwningSession`,
  `anIdleReleasedSessionIsEndedAndARegistrationInTheClientsNewSessionIsNotified`,
  `anIdleReleasedSessionWhoseTransportEndFailsStaysInTheDirectoryAndIsStillNotified`),
  `CqelsMcpHttpClientIdentityTest` (`aClientsNotifyRegistrationNeverNotifiesAnotherClient`,
  `aRegistrationNotifiesOnlyItsOwnSessionEvenForTheSameClient`, `aClosedSessionLeavesTheSessionDirectory`,
  `anExpiredSessionLeavesTheSessionDirectoryAndShutdownEmptiesIt`,
  `aResultSignalForAClosedSessionIsDroppedNotBroadcast`,
  `aSessionTheTransportDropsAfterAFailedWriteLeavesTheDirectory`), `McpNotifierTest` and
  `SessionDirectoryTest`.
- **A `watch_invariant` or `register_rules` registration no longer blocks the removal of other
  streams**. Both tools registered through `CQELSEngine.registerQuery(query, listener)`, which
  pins every data stream that exists at registration time, so `removeStream` refused every stream
  while one such registration was live — including streams the registration never reads. They now
  use the new `registerQuery(query, listener, inputStreams)`, which pins exactly the declared streams
  and gives the query only those streams as inputs; a query that looks up any other stream finds it
  absent. A declared stream that does not exist when the registration is made fails it with
  `IllegalArgumentException` naming the stream, and nothing is pinned. The two-argument overload is
  unchanged. Tests: `CQELSEngineRemoveStreamTest`
  (`declaredRegistrationPinsOnlyItsDeclaredStreams`, `declaredRegistrationIsWiredOnlyToItsDeclaredStreams`,
  `undeclaredRegistrationStillPinsAndWiresEveryStream`,
  `declaredRegistrationOfAMissingStreamFailsAndPinsNothing`) and `ContinuousReasoningToolHandlerTest`
  (`registerRulesPinsOnlyTheStreamItReads`, `watchInvariantPinsOnlyTheStreamItReads`).
- **`NFAPatternProcessor.advanceWatermark` no longer fails for a subscription cancelled while the
  call waits for room**. A call waiting on a full watermark share kept waiting after that
  subscription was cancelled, then threw `IllegalStateException` (`FAIL_OVERFLOW`) after the 1 s stall
  timeout for a subscription that no longer existed. It now returns as soon as the subscription has
  ended, with no failure, as it already did for a subscription that ended before the call. Measured
  in `NFAPatternProcessorWatermarkTest.cancellationDuringPacing_returnsTheProducerPromptly_withoutAFailure`:
  800 ms from the cancellation to the failure before, 0.2 ms to the return after.
- **An idle directional (FUTURE/centered) CEP registration no longer reserves its full input channel**.
  `DirectionalCepAdapter` allocated its 65 536-slot input channel as an array when the
  registration activated, before any input arrived. The channel keeps the same bound and the same
  split between the lifted stream and the external watermarks, but its storage now grows per queued
  input, as `NFAPatternProcessor`'s channel does. Measured in
  `DirectionalCepQuietStreamWatermarkTest.idleDirectionalRegistrations_doNotRetainTheChannelCapacity`:
  64 idle registrations retained 17 467 352 bytes of heap before, 698 584 after.
- **Stream-query byte budgets: review residue (MCP)**. Eight defects left by the review of the
  byte-budget change:
  - Short-term `store_memory` bounded only its `turtle`. Its `facts` are now counted (100 000) and
    charged to the same 8 000 000-character budget as the turtle, before any fact is parsed. A
    short-term fact's `subject`, `predicate` and `object`, and the `turtle`, must be strings, and
    `objectType` must be `uri` or `literal`; anything else is refused without being rendered.
  - A short-term request refused for its size had already created its stream; refused requests with
    distinct names could fill the stream cap with empty streams. Every size check now precedes stream
    resolution, so a refused request creates nothing.
  - `recall_memory` (with `queryId`) and the one-shot `query` tool checked that the response framing
    fits only after draining, so a recall that could not answer had already cleared the `dropped` /
    `oversized` report. Both check first; a refused recall drains nothing.
  - `CqelsMcpServerConfig.Builder.streamQueryMaxResponseBytes` accepted budgets too small to admit any
    row, or to frame an empty response. It now refuses a budget under 4 096 bytes
    (`StreamQueryRegistry.MIN_RESPONSE_BYTES`).
  - Rows cut by the final encoded-size check were lost while the response said `truncated` with a
    `remaining` that did not count them. They are now put back in the buffer at their arrival
    position, ahead of the rows that arrived since, and counted in `remaining`. A cut row that cannot be put back is reported as `dropped`, never as
    `truncated`; the one-shot `query` note counts the cut rows among those it discards.
  - A `CharSequence` that is not a `String` was sized through `length()` / `charAt()` and written
    through `toString()`. It is now copied into a `String` when its row is buffered, after its size admitted it (and when a CEP
    payload is rendered), so what is sized is what is written. A map whose distinct keys become equal
    strings is refused (`dropped` and `oversized`) rather than losing an entry.
  - The `dropped` warning always blamed a full buffer and recommended a larger `bufferSize`. It now
    names each cause — row count, byte budget, or a row too large on its own — with its remedy. The
    rows' text and every warning together stay within the response budget.
  - A literal's datatype IRI and language tag were scanned once per literal at admission; they are
    now sized once per shared instance.

- **An impossible value in a journal's durable state now fails the open instead of being read as a
  legitimate one**. `FileBackedEventJournal` decides what to do from its format header, the
  `journal.format` marker, the `event-journal.offset` floor and the `event-journal.commit` pointer,
  and a value none of them could have been written with is damage. Several such values were being
  reinterpreted, each of which disabled the committed-prefix check this journal gained earlier in
  this release (the entry on damage inside a journal's committed prefix, above) and let recovery trim acknowledged records or reissue their offsets: a pointer file shorter than its 8 bytes, a
  negative committed length, and a stored `Long.MAX_VALUE` were all read as "no pointer yet" (the
  last because `Long.MAX_VALUE` doubled as the scan's "uncapped" sentinel); and an offset floor of
  `-1` was accepted because its CRC32C verified, so the next append reset the counter and reissued
  offsets already acknowledged. All of these now throw `CorruptCommittedStateException` before
  recovery touches a file. An ordinary pointer past the end of the journal was **not** among them:
  the scan fell short of it and the committed-prefix guard then refused the open under both
  policies, so what changes for it is only that the refusal now happens before any record is walked.
  `Long.MAX_VALUE` is the exception that makes the distinction worth drawing — it is itself a
  past-EOF pointer, and it was the one value that switched that guard off instead of tripping it. A committed record whose payload
  does not decode as a `StreamEnvelope`, and a malformed `journal.id`, fail closed too. They used to
  escape untyped — the payload as the decoder's own `IllegalArgumentException`, the sidecar as an
  `IllegalStateException` wrapping an `IOException` — and `PersistenceCoordinator` catches either,
  so under `failIfPrimaryUnavailable=false` both degraded to "persistence disabled" with the
  committed records left on disk and out of recovery. The payload check is the only one that catches
  payload damage in on-disk format 1, which has no per-record checksum. Enabling
  `SYNC_ON_COMMIT` over a journal written under `NONE` now also forces the journal before publishing
  the recovered pointer, so the pointer never claims bytes still only in the OS cache. The full
  table of durable datum × damage mode → required outcome is recorded in `recoverValidPrefix`'s
  source, including the one row that is deliberately not enforced *under `NONE`*: a pointer that is
  not a record boundary is a state an earlier release's compaction legitimately leaves, and under
  `NONE` the pointer is only a floor, so refusing it would refuse an upgrade for an untampered
  journal. Under `SYNC_ON_COMMIT` the pointer is a cap that must be landed on exactly, and the
  committed-prefix check refuses such a value there as it always did.
- **A damaged record length can no longer swallow the records after it**. Recovery frames
  records without parsing payloads, so an oversized length looked like one whole record; the decoder
  then returned the first envelope inside it and ignored the rest, so the swallowed records
  disappeared from `readFrom` and their acknowledged offsets were handed out again. A
  `StreamEnvelope` whose wire form leaves trailing bytes is now rejected — the form is
  self-delimiting, so trailing bytes mean the framing is wrong.
- **An offset can no longer be assigned past `Long.MAX_VALUE`**. An offset floor just below
  the maximum is a *possible* value, so validating the floor cannot prevent this: the next append
  wrapped to `Long.MIN_VALUE`, and since recovery re-seeds the counter from the floor, every restart
  reissued it. Both append paths now refuse up front, and a batch that would cross the ceiling is
  refused whole before anything is written.
- **A journal sidecar that is a directory entry without a file behind it now fails the open**.
  Presence was tested with a check that follows symlinks, so a dangling link reported
  "absent": the commit pointer read as "no pointer yet", and under `SYNC_ON_COMMIT` recovery then
  trusted the uncommitted tail and replaced the link with a pointer committing it. Presence is now
  tested without following links at every site, a present entry that does not resolve is reported as
  corruption by each reader, and a `journal.id` that is not valid UTF-8 — which fails before the
  malformed-UUID check — is reported as corruption too rather than degrading to "persistence
  disabled".
- **A damaged durable entry can no longer size the read that loads it**. The commit pointer,
  the offset floor, the format marker and the `journal.id` sidecar are each a handful of bytes, but
  three of the four were loaded with `Files.readAllBytes` / `Files.readString`, which size their
  allocation by the entry's own length. An oversized or sparse entry therefore raised
  `OutOfMemoryError` — an `Error`, so `PersistenceCoordinator`'s `catch (Exception)` does not see it
  either — and aborted engine startup instead of producing the typed refusal; for `journal.id` it
  did so *after* recovery had already deleted or republished the commit pointer. All four now read
  one byte past the largest legitimate value and refuse anything longer, with no `size()` call to
  race against.
- **A read fault on the journal file's contents is now reported as corruption**, like a read
  fault on any other durable entry. Only the file's `size()` call was classified; a permission or
  media error on the *content* read surfaced as a plain `IllegalStateException`, which is the type
  that degrades to "persistence disabled" under `failIfPrimaryUnavailable=false`. The reads are
  wrapped and the writes deliberately are not: a failure to truncate or fsync is not evidence that
  what is on disk is damaged, and must not send an operator to a backup for a full filesystem.
- **`Long.MAX_VALUE` is reserved as an offset and never assigned**, so the highest assignable
  offset is `Long.MAX_VALUE - 1`. Without the reservation two of these checks contradicted each
  other: an offset floor of `Long.MAX_VALUE - 1` is accepted, an append from it could assign
  `Long.MAX_VALUE`, and a following `truncateBefore` wrote that as the floor — a sidecar the journal
  produced itself and then refused to reopen. A floor that already holds `Long.MAX_VALUE` is still
  accepted and simply means "exhausted": the open succeeds and the records stay readable, while the
  append paths refuse to assign. Refusing the open would deny access to records that are safe to
  read, and failing closed is meant to prevent corruption, not access.
- **A read fault on an already-open journal now fails closed on later reads too**. The
  open-time reads already reported a read fault as corruption, but `FileBackedEventJournal.readFrom`
  and `truncateBefore` (through `readAll`) still wrapped one as a plain `IllegalStateException`,
  which recovery under `failIfPrimaryUnavailable=false` answers by skipping the replay (continuing
  from the checkpoint, if any), so the committed records stayed on disk and out of recovery. They now throw
  `CorruptCommittedStateException`, which `PersistenceCoordinator` rethrows before its availability
  fallback. The message says the committed records cannot be read now and asks for the I/O fault to
  be fixed rather than for a restore: the file was readable when it was opened, so the cause may be
  transient. The constructor's second `Files.size` of the journal file, after the recovery scan,
  refuses as the other open-time reads do (by code inspection only: no test can make that one call fail). Writes
  (append, truncate, fsync, pointer publication) are unchanged, since a failed write is not evidence
  of damage. Test:
  `FileBackedEventJournalRecoveryIntegrityTest.unreadableJournalFileAfterOpen_readFailsClosed`.
- **A CEP `WITHIN` that overflows a run's deadline no longer costs an exception per live
  run**. `NFAPatternProcessor` computed a run's deadline with `Math.addExact` and
  caught the overflow, returning `Long.MAX_VALUE`; for a `WITHIN` near `Long.MAX_VALUE` that threw
  and swallowed one exception per live run on every input and watermark scan. The deadline now
  saturates with a comparison instead. Matching is unchanged, since the old code already saturated:
  `NFAPatternProcessorWatermarkTest.withinOfLongMaxValue_saturatesTheDeadline_noErrorNoRelease` pins
  the semantics and passes on the old code too, and `saturatingDeadline_boundaries` pins the
  arithmetic; what changed is the exception cost only.
- **A stream-query row whose nested map has a key that is not a string is refused
  (MCP)**. The row detach this release added for the byte budgets copied a map
  by re-hashing its keys, so an embedder's row with thousands of container keys sharing one hash
  code made the copy quadratic while still inside the byte budget. An MCP client cannot build such a
  row (JSON keys are strings); only an embedder pushing Java objects can. Every nested map in a row
  must now be string-keyed, whether or not it is copied: a `CharSequence` key becomes a `String` and
  a null key is allowed, and any other key refuses the row, counted in `dropped` and `oversized` and
  logged (`a map key is not a String`). Tests: `StreamQueryRegistryTest`
  (`detachingHashCollidingContainerKeysNeverRehashesThem`,
  `aNonStringKeyedMapIsRefusedWhetherOrNotASiblingValueForcesACopy`).
- **An I/O fault while finding or reading a committed checkpoint now fails closed**.
  `FileBackedCheckpointStore.latest()` read several such faults as absence, which
  `PersistenceCoordinator`'s availability fallback could then swallow: a fault while iterating a
  committed checkpoint's `states/` escaped as `UncheckedIOException` (`Files.list` opens eagerly,
  but a fault during iteration surfaces from the stream), a fault listing `checkpoints/` became an
  `IllegalStateException`, and a stat fault on a `cp-N` entry or its `manifest.bin`, `checksums.bin`
  or `states/` read as "not there". Under `failIfPrimaryUnavailable=false` recovery could then
  proceed as if there were no checkpoint, or an older one. Each of these now throws
  `CorruptCommittedStateException`, the classification the journal's read-fault entries above use
  for committed state, and a dangling link in place of any of these entries is a present, damaged
  entry, not an absent one. A checkpoint established to be absent or uncommitted is still skipped,
  and retention (`deleteOlderThan`) stays tolerant: it skips, and never deletes, an entry it cannot
  stat. Tests: `FileBackedCheckpointStoreListFaultTest`, `FileBackedCheckpointStoreLinkTest`,
  `FileBackedCheckpointStoreLegacyStatFaultTest` and
  `PersistenceCoordinatorTest.recovery_checkpointStatesIterationFault_failsClosedDespiteAvailabilityFallback`.
- **MCP `push_stream_events` bounds each event's payload before converting it**. The
  `nquads` body and a fact's `subject`, `predicate` and `object` were converted with `toString()`
  before they were measured, so an oversized value was materialised as a string before it was
  refused, and a whitespace-only `nquads` body sent alongside `facts` was not charged to the call's
  character budget. Each event is now copied once, within the per-event (2 000 000 characters) and
  per-call (8 000 000 characters) budgets, before anything is parsed, and everything after that
  reads only the copy; a whitespace-only `nquads` body is charged to the call and also counts toward
  the event's facts budget.
  - **Behaviour change: stricter input types.** `nquads` and a fact's `subject`, `predicate` and
    `object` must be strings, numbers or booleans (the tool schema declares strings); a JSON object
    or array there is refused instead of being converted to its text. `objectType` must still be
    `uri` or `literal`, and the error no longer echoes the value. A text `eventTime` is at most 64
    characters. A numeric `eventTime` must be a JDK integer, floating-point, `BigInteger` or
    `BigDecimal` value; a zero `BigDecimal` is accepted at any scale, a non-zero one with more than
    1024 decimal places is refused, and one with a scale below -12, so at least 10^13, is refused as
    out of range.
  - For applications that call the handler with their own objects: each `events` and `facts` list is
    asked for its size once and its elements are read exactly that many times (a list that runs out
    early is refused as changed while it was read); a character sequence's length is read once (a
    negative length is refused) and its characters copied, never through its `toString()`; a number
    is rendered only when an upper bound on its length is within 32 characters of the room left in
    the budgets; other `Number` classes, including subclasses of `BigInteger` and `BigDecimal`, are
    refused.
  - At the time of that fix the N-Quads statement and observation limits still applied only after
    the whole body was parsed, within the character budgets; later in this release
    `push_stream_events` stops parsing at the message that passes a limit, as described below.

- **A windowed join on dynamic-gap session windows no longer drops its buffered rows when an
  element's session window equals an already-merged session**. `MergingWindowSet` replaced
  the merged window's link to the state holding its rows with a fresh, empty one whenever a newly
  assigned window was exactly equal to that merged window, so `WindowedJoinOperator.join` lost the
  rows both sides had buffered before it; the existing link is now kept. For example, left `L1` at 0
  with gap 5, right `R` at 3 with gap 7 and left `L2` at 0 with gap 10, with both watermarks advanced
  to 9, now emit `L1:R` and `L2:R` where nothing was emitted. Session windows with a fixed gap cannot
  produce an assigned window equal to a merged one and are unchanged; CQELS-QL session windows run on
  a different operator and were not affected.

- **Window-scoped `SELECT DISTINCT` on the per-arrival routes now prunes its scope at every arrival,
  including one that yields no row**. Eviction used to run only when a row reached the
  `DISTINCT` stage, so an arrival that the stream pattern, the static lookup, a `FILTER`, a `UNION`
  with no matching arm, `FILTER NOT EXISTS` or `HAVING` left without a row pruned nothing, and the
  scope kept the last admitted window's keys indefinitely on a stream that produced no further
  matching row; that retention was stale, not growing, and no answer was affected. Every arrival on
  those routes now ends with an in-band boundary row that travels the same path as its rows; the
  `DISTINCT` stage evicts on it, once per arrival, and emits nothing for it. Evicting when the
  arrival is minted is still avoided, because the prefetching operators in between could run the
  source ahead of rows that still need the keys. No emitted row changes. `UNION` blocks now run
  under `concatMap` instead of `flatMap`, so under bounded subscriber demand their rows keep
  upstream order; the rows themselves are the same. Pinned by `WindowScopedDistinctTest`: nine route
  shapes × `[RANGE 5s]` / `[TRIPLES 1]` × bounded / unbounded demand, each asserting the emitted
  rows and the scope's tracked-key count.

- **A CEP match is no longer refused on a stale remembered event size (MCP)**. For an event an
  embedder pushes (not over MCP), the size of a recent large event is remembered from its first match; an
  embedder-owned mutable payload that shrank afterwards kept that size, so a later match that now fit — in
  `SEQ(?a; ?b?)`, a 2 MiB payload repeated in `events`, `named` and `run`, shrunk before `b` arrived — was
  refused as oversized before drain-time rendering. A match whose estimate exceeds the registry's admission
  limit (`StreamQueryRegistry.admissionLimit()`, now public: the smaller of the buffer budget and the
  response budget less its framing reserve) is now sized again from its live payloads before it is refused,
  and the remembered size is replaced (or forgotten once the event is no longer large), so a refusal on
  arrival never rests on a remembered size, and a fan-out over an event that is still large walks its
  payload once, not once per match (an event shrunk below 64 KiB is no longer remembered and is measured
  per match, as small events always were). Events pushed with `push_stream_events` or stored with `store_memory` are unchanged (sized
  once at ingestion). Two limits remain: a remembered size that is stale but still admissible is charged to
  the buffer as it is, so an over-estimated row can evict other rows (counted in `dropped`) until it is
  drained; and a refused match over an embedder's large event now costs a walk of its live payloads, each
  bounded by the budget, where it was a lookup (`CepResultByteBudgetTest`).

- **`HAVING` over an aggregate call answers what the alias form answers**.
  `HAVING (COUNT(?v) > 1)` evaluated the aggregate call to null and silently dropped every group, on
  the windowed hash join and on the `[NOW]` per-arrival fold, where `HAVING (?n > 1)` with
  `(COUNT(?v) AS ?n)` answered. An aggregate call in `HAVING` now reads the value the fold computed
  for the same aggregate in the `SELECT` list. Two calls count as the same aggregate when the
  function, the argument variable (`?v` and `$v` alike) and the `GROUP_CONCAT` separator match, so
  `COUNT(*)` and `COUNT(?v)` stay distinct. An aggregate the `SELECT` list does not compute is
  computed over the same group as a hidden aggregate: `HAVING` reads it and it is removed from the
  row before emission, under a name that cannot collide with a query variable. `COUNT` over a
  variable no solution binds is 0. As a consequence, a `GROUP BY` query whose only aggregate is in
  `HAVING` now folds like any other aggregate query, and a `[NOW]` query with no `GROUP BY` whose
  only aggregate is in `HAVING` is refused at registration, as a global aggregate over `[NOW]`
  already was. A `HAVING` expression with more than 1 000 aggregate calls is refused at parse time.
  A `GROUP_CONCAT` separator's quote escapes are now decoded, in `SELECT` as in `HAVING`, so
  `SEPARATOR='\''` joins with `'` and matches `SEPARATOR="'"`; other backslash escapes in string
  literals are kept as written, as before.

- **The multi-stream temporal join now joins the statements of a `List<Statement>` payload, where it
  dropped such an element silently** — a `GraphStreamElement` from `DataStream.push(Collection, …)`,
  the MCP stream-push tool or the RDF Messages codec, or any `StreamElement` whose payload is a
  `List<Statement>`. The same triples wrapped as a list now yield the same result bag as
  single-statement elements. Each statement is joined as if it had arrived alone at the element's
  timestamp, in list order: duplicates are kept, each statement keys on its own bindings, and a
  multi-triple side conjoins them as it conjoins single-statement arrivals. On this route a batch is
  therefore not one atomic observation, as it is on the per-arrival routes: its statements co-reside
  with any other statement at that timestamp, so under `[NOW]` they can conjoin with another element
  pushed at the same millisecond. Its statements are matched one by one: each side solution they
  complete takes its own slot against `cqels.join.intervalBufferCap` (several for one statement that
  completes several on a multi-triple side), a statement that matches a side pattern but completes
  no solution takes matcher state only, and a statement matching no side pattern takes neither. A
  late element is reported to the late-data listener once, as the element that arrived, not once per
  statement or per side solution its statements complete; when the joint watermark advances while
  its statements are being admitted, some of them may already have joined by the time it is
  reported. An empty list still advances its side's watermark and joins nothing. A list holding
  anything other than an RDF statement now fails the query with `IllegalArgumentException` before
  any of its statements is joined, where it was dropped silently (reachable through a custom
  `StreamElement` or a custom journal codec, so a bad journaled record fails the query again on
  every recovery); so does a list of more than `cqels.join.maxStatementsPerArrival` statements
  (system property, default 10 000, the MCP stream-push tool's per-message limit), counted while the
  list is copied, whatever its `size()` reports, so an oversized payload is refused before the copy
  grows past the bound — MCP input is already bounded upstream, so this guards embedders supplying
  their own `StreamElement`; a payload that is a collection but not a `List` is still dropped
  silently. A temporal-join query over batch-shaped elements that answered nothing now answers rows,
  and at sustained rates it can now reach the buffer cap.

- **`push_stream_events` stops parsing an N-Quads body at the message that passes a limit**. The
  limits on statements per message (10 000), observations per call (10 000 non-empty messages) and
  statements per call (100 000) were checked only after the whole body had been parsed, so one event's
  body within the 2 000 000-character limit could have every statement it held parsed before it was
  refused; a body of short lines such as `_:0<a:>"".` holds more than 150 000. The new overload
  `RdfMessageCodec.parseNQuads(document, maxStatementsPerMessage, maxMessages, maxStatements)` checks
  each message's own limit while it is parsed, statement by statement, and the document's message and
  statement limits once a message is complete, and throws `RdfMessageCodec.LimitExceededException`,
  naming the limit, at the first one passed; the handler passes it the observations and statements left
  in the call. `parseNQuads(document)` is unchanged. Some refusals read differently: a limit passed in an
  earlier message, or a message's own limit passed before a malformed line in it, is now reported instead
  of the parse error (a malformed line in the message that passes the call's observation or statement
  limit is still reported as the parse error); a limit passed anywhere in a body is reported instead of a
  reserved-context statement anywhere in it (that check still runs on the parsed body); and an oversized
  message says "more than 10000 statements" instead of its full count.

### Changed
- **A `STREAM` block inside `OPTIONAL` is now rejected at parse time (and at
  `CqelsQueryDefinition.OptionalBlock`'s constructor, for a query built programmatically),
  where a single-source `STREAM` used to compile and then silently null-fill at every match**.
  `CqelsQLASTBuilder#visitOptionalPattern` already rejected a *multi*-source `STREAM`
  inside `OPTIONAL` at parse time; the guard is now unconditional, and the shape it used to
  admit, which fell through at execution to `CqelsQueryCompiler`'s `tryMatchStreamOptional`
  stub — a method that unconditionally returned `false` — is deleted with it. **Named windows (`FROM NAMED WINDOW` / `WINDOW` blocks) are now rejected
  inside the `CompiledQuery` constructor instead of inside `execute()`.** For
  `CQELSEngine.registerCqelsQuery` callers this is not a timing change: registration already
  calls `execute()` synchronously, so the old check already threw the same
  `UnsupportedOperationException` during registration too, just after every other field had
  already been built. What actually changes is the still-public `new CompiledQuery(...)`
  construction path — it previously carried no such guard unless a caller went on to call
  `execute()` itself, and now rejects immediately, matching what `registerCqelsQuery` already
  guaranteed. The check still lives in the constructor rather than the parser, because
  `cqels-cdsp`'s `QueryRegistry` parses first, inspects `hasNamedWindows()` on the successful
  parse, and skips such a query gracefully with its own warning before ever reaching
  `compile()` — rejecting in the parser would have broken that already-tested graceful skip for
  no benefit. **Two separate dead-code-from-dispatch paths are removed — one of them a public
  API.** `IntervalJoinOperator.processingTimeJoin` (four `public static` overloads) is deleted:
  `TemporalJoinDispatch.assemble` only ever called the event-time join, never this one, so no
  registered query reaches it — but the method WAS public, callable directly by a library
  embedder, so this is source- and binary-incompatible for any such direct caller, with no
  drop-in replacement (see Upgrade notes). Independently, a dead branch inside
  `TemporalJoinDispatch.dispatch` is deleted too:
  `TemporalJoinSpec.Kind` has only ever declared one constant (`INTERVAL`), so its
  `!= Kind.INTERVAL` guard could never be taken. Both were confirmed unreachable from any
  registered query before deletion, so no registered query's output changes because of either
  removal.

- **`executeStreamOnly` — always a private method, never public API — is deleted; every query
  that used to route there now runs through `executeStreamOnlyWithExpressions` instead, an
  internal route consolidation implementing decision D4 of the route-consolidation epic**
  — nine execution routes become eight.
  Measurement found the two route bodies equal across the retired route's whole domain (window
  family × stream-pattern shape, 12 of 12 combinations): the 294-cell route-selection oracle
  confirms only the intended reroute (23 cells move from `SO` to `SOX`, exactly as expected —
  that oracle asserts which method dispatch selects, not row content), and the 39-cell
  behaviour matrix, which does check row content, shows zero differences. The sole observable
  effect: the `[NOW]`-multi-pattern-BGP WARN this route used to log is carried over to
  the surviving route with its text corrected — the old wording called the routing a bug and
  said the route emitted partial rows, and neither was true — and because BIND/FILTER no
  longer select a different route body, it now
  also logs for `[NOW]` multi-pattern queries that use BIND/FILTER — a population that never
  logged it before. Row output for that population is unchanged; only the log line is new.

- **A `GROUP BY` aggregate under a `[NOW]` window is now evaluated PER ARRIVAL and emits on a live
  stream; it used to buffer the whole stream and emit only if the source completed**. `[NOW]`
  is a zero-length window holding only the current arrival, so the arrival IS the evaluation
  boundary: each arriving element's rows are grouped and folded on their own, and that evaluation's
  group rows are emitted — the same per-evaluation snapshot the windowed routes already produce.
  Before this, the fold ran on the flat pipeline through a `collectList()` that
  completes only when the SOURCE does, so on an engine-registered stream — which never completes —
  such a query emitted NOTHING at all while the collected list grew without bound. Affects every
  `[NOW]` query with `GROUP BY` and aggregates, on both boundary-less routes: stream-only, and the
  flat stream+static lookup.
  **Who has to re-check what.** A consumer running such a query against a live stream saw no rows
  and now sees one fold per arrival — pure gain. A consumer running one against a FINITE source
  (a test `Flux`, a replay) received ONE fold over the whole source at completion and now receives
  one fold per element instead; the totals it used to read are not produced by `[NOW]` at all.
  Declare a window that accumulates — `[RANGE d]` or `[TRIPLES n]` — to keep aggregating across
  elements, as the spec's window table now states. No count under `[NOW]` can exceed the rows ONE
  ARRIVAL's graph pattern produces — one on the stream-only route, one per satisfying static-BGP
  assignment on the lookup route, and more when a `UNION` or `OPTIONAL` in the same query
  multiplies them (those reshapers run before the fold, so a UNION whose arms both match doubles
  the arrival's input). Nothing accumulates ACROSS arrivals, so a `HAVING` on a count that only an
  accumulated window could satisfy now selects nothing there. `SELECT DISTINCT` beside such a fold now runs INSIDE the arrival: it
  collapses rows that project equally within one evaluation and no longer suppresses a group row
  merely because an earlier arrival produced the same one. On these routes it previously carried a
  single unbounded seen-set across the whole stream, which downstream of a per-arrival fold would
  have emitted one row per distinct projected row for the stream's life and then nothing at all —
  the very symptom this entry removes. `HAVING` moves into the same evaluation, ahead of that dedup (SPARQL
  §18.2.5 order — the two do not commute). All three are pinned cell by cell. One consequence
  travels with that move, on this shape only: the in-batch `HAVING` every other
  evaluation-boundary route uses does not wrap the evaluation in a `catch` that drops the row, so
  a `RuntimeException` escaping the expression evaluator now SURFACES here instead of silently
  removing a group. That is intended — every EXPECTED failure (unknown prefix, type error, unbound
  variable) is signalled by a null return and still drops the row per SPARQL §17.2.2, so anything
  thrown is an engine defect — and it matches what the other routes have always done.
  Unchanged: a GLOBAL (no-`GROUP BY`) aggregate under `[NOW]` is still refused at registration,
  and `[NOW]` + `GROUP BY` + `ORDER BY` + `LIMIT` is still refused at registration — its
  REASON is restated, since the fold no longer buffers: a continuous top-K ranks ACROSS arrivals
  while each fold now describes ONE, so a group row from an arrival long gone would keep winning.

- **The per-arrival query routes now state, and honour, a window contract: they emit the
  `Istream` of the window they declare — and `SELECT DISTINCT` on them is scoped by that
  window instead of by the whole stream**. A single-triple-pattern, fixed-predicate,
  non-aggregate `SELECT`, with or without a static graph pattern, is evaluated per arriving
  element and reads no window spec. The documentation called that "inert" and said a declared
  `[RANGE 10s]` there "does nothing whatsoever"; measured against the windowed hash join's own
  body over the same elements, what those routes emit is exactly `eval(t) \ eval(t-1)` of the
  declared window's relation — RSP-QL `Istream`, over every state that relation passes through
  (the windowed bodies evict by the arriving element and THEN insert it, so a row whose only
  earlier occurrence its own arrival evicts leaves the relation and re-enters) — under bag
  semantics, across duplicate rows,
  expiry followed by re-entry, out-of-order arrival, `[TRIPLES n]`, `[SLIDE d STEP s]`,
  `[RANGE d STEP s]` and the composed stream+static route. The window names the base of that
  delta; it bounds nothing else there, because nothing else on those routes is window-sensitive.
  **This entry changes no emitted row for a query without `SELECT DISTINCT`** — the scope object
  below is built only when the query carries one, so every other query keeps the pipeline it had
  (the one-element-window entry at the end of this section does change such a query's rows when
  an element carries several matching statements): the contract is
  now stated (`CQELS-QL_SPEC.md` §4, the cheatsheet, the grammar comment, the `Istream(...)`
  refusal message and the `cqels-mcp` agent guidance, all of which previously said the opposite
  or disowned their own canonical example) and pinned by `PerArrivalIstreamContractTest`.
  **One behaviour change, for `SELECT DISTINCT` on these routes**: the dedup was a global `Flux#distinct` over the whole stream, with an unbounded
  seen-key set — the same hazard `ORDER BY + LIMIT` is refused for there, left unguarded. A key
  is now emitted when it ENTERS the window relation and again once every occurrence of it has
  left, so `SELECT DISTINCT ?s FROM STREAM S [RANGE 5s]` over three arrivals of one subject at
  `t = 0 s, 1 s, 30 s` answers 2 rows where it answered 1. Memory no longer grows with the
  stream's length. Eviction runs at every arrival: each arrival ends with an in-band boundary row
  that the `DISTINCT` stage evicts on, so an arrival that yields no row — or whose rows a later
  stage drops — prunes the scope too. The
  answer does not depend on the subscriber's demand:
  the arrival a row came from travels WITH the row, not on a clock the pipeline could run ahead
  of. Affected: a `SELECT DISTINCT` on one of those shapes whose declared window is `RANGE`,
  `SLIDE` or `RANGE … STEP` — a key that returns after its window expired is reported again
  instead of being suppressed forever. **`[NOW]` changes too, in the other direction**: its
  window holds only the element being evaluated, so its `DISTINCT` now collapses duplicates
  WITHIN one arrival and carries nothing across them — three arrivals of one subject answer 3
  rows where they answered 1, and on the static-lookup route an arrival's own duplicate rows
  still collapse. `[TRIPLES n]` is scoped by the last *n* ARRIVALS, counted by ordinal — elements
  the pattern rejects included, as the windowed bodies' buffer counts them — so `a, b, a` under
  `[TRIPLES 2]` answers 3 rows where it answered 2, and under `[TRIPLES 3]` still answers 2. The
  arrival a row came from is minted at the element boundary and travels with it, so none of this
  depends on the subscriber's demand, and an arrival that produces no row still counts: its
  trailing boundary row evicts against the ordinal and high-water mark it advanced, as the windowed
  bodies' buffer evicted at that arrival. Unaffected, and
  pinned: `[NOW]` beside a
  `GROUP BY` aggregate, whose fold rebuilds every row and so leaves no arrival on it to
  scope by; every windowed route; and every query without `DISTINCT`. Pinned by
  `WindowScopedDistinctTest`, whose cells answer each shape under bounded AND unbounded
  subscriber demand. The `Istream` equality was first measured for ATOMIC arrivals only;
  it holds for elements carrying several statements that match one triple pattern since the
  one-element-window entry at the end of this section. Two lookup shapes stay outside the
  `Istream` equality, for any arrival: a query with no join key and a static pattern the candidate
  fallback cannot serve, and a stream solution whose join key binds a literal; that entry states
  both. It is measured against a static side
  that does not change while elements are in the window too: the per-arrival lookup route joins
  only the arriving element and never re-joins one already in the window, so a mid-window
  mutation is invisible for it — likewise a property of the route rather than of the declared
  window, stated in the spec and tracked separately. And it is a claim about a PURE
  query, `CQELS-QL_SPEC.md` §10.8's own precondition: a volatile `RAND()` / `NOW()` / `UUID()` /
  `STRUUID()` / `BNODE()` in `BIND` or `FILTER` is re-evaluated per buffered element by the
  windowed body and once per arrival by these routes, so neither side is reproducible.
  One limit is specific to the new time scope: it prunes against the arrival's high-water
  mark, which is the edge the windowed body last evicted against unless event time REGRESSES
  after an arrival MORE THAN THE DECLARED EXTENT ahead of it — then the scope's edge is ahead
  of the body's buffer and a key can be re-emitted once too often (measured on both reachings,
  the earlier arrival forming no row and forming a row for another key, and at the threshold:
  under a 5s range a gap of exactly 5s still agrees and only a larger one diverges, so
  out-of-order event time alone does not reach it; exactness needs a maximum over a RANGE of
  arrivals, i.e. the buffer a dedup scope deliberately does not keep). Characterised
  in a cell, stated in the spec, tracked separately; the count scope is indexed by arrival
  ordinal, which never regresses, and is unaffected. And one shape never reaches the scope at
  all: `SELECT DISTINCT` beside `ORDER BY ... LIMIT` fuses its dedup into the continuous top-K,
  which keys on the projection but re-emits whenever its snapshot changes — and that snapshot
  holds the full representative rows, so an arrival fanning out to rows that agree on the
  projection and improve on the sort swaps the representative, changes the snapshot, and emits
  the projected row again — once for each row that improves it. Pre-existing and unchanged here (the same query
  answers the same with the scoping disabled), measured, and tracked separately.

- **A CypherQL `MATCH` now reads only the stream it names, and one that reads TWO declared
  streams is rejected at registration instead of answering partially**. Every declared
  input used to be merged into a single flux before matching, which discarded stream identity.
  Two consequences, both silently wrong rather than silent:
  - A `MATCH` whose groups named DIFFERENT streams emitted one **partial** row per element —
    each satisfying only its own group, none satisfying the conjunction the `RETURN` asks for —
    so a caller that does not null-check every column read plausible-looking invalid output.
    Combining groups from different sources into one row is a join this compiler does not
    implement, so the shape now raises `UnsupportedOperationException` at registration, naming
    both sources. Implementing the join for real is tracked separately; two examples in
    `example-continuous-cypher-queries.cql` are written in this shape and are annotated as
    not-yet-supported.
  - Declaring a second stream **at all** contaminated a single-source `MATCH`: an element
    arriving on a stream the `MATCH` never named could still bind its variables. A `MATCH` now
    reads only its own declared stream.
  - Relatedly, the stream the `MATCH` reads is windowed by **its own** declaration — only one
    stream is ever read. Both windowing paths read "the first declared stream", and
    `getStreams()` is a `Map.copyOf` of a `HashMap` whose iteration order is unspecified — so
    the window applied was not even the declared-first one, and a second declared stream could
    silently rewindow the first.
  - A declared stream no `MATCH` names is no longer **subscribed** by the query at all, so the
    engine now pins only the stream the `MATCH` reads: `removeStream()` accepts a stream a
    running Cypher query declares but never reads, where it used to refuse one that could not
    affect any result. (A `Sinks.many().multicast()` source retains what it is handed until its
    first subscriber, so such a stream's elements accumulate in the sink rather than being
    dropped.)
  Three shapes are refused alongside those: a `MATCH` made only of `GRAPH` groups when two or
  more streams are declared (it names no stream, yet still needs one to TRIGGER it); a
  `MATCH` naming a stream **no `FROM` clause declares** — the engine supplies every registered
  stream to a query, so that would silently consume an unrelated one and carry no window;
  and, with two or more streams declared, a `MATCH` group naming **no source** — and a
  definition with **no `MATCH` at all**, which the grammar cannot express but the public
  builder can — are refused too: which stream they read has no answer. An unsourced group
  previously worked as a deterministic stream UNION whenever both streams declared the same
  window, so a same-window union of two streams must now be expressed as **one** stream.
  Accepting an unsourced group when every declared stream carries an identical `WindowSpec`
  is a narrower rule, and a decision for the cross-stream join.
  A query declaring **one** stream keeps its old registration behaviour wherever its `MATCH` is
  unsourced or names that stream — the group resolves to it, exactly as before. The
  undeclared-name refusal is the exception, and it applies at **any** stream count:
  `FROM STREAM Stream1 … MATCH STREAM Other { … }` is now refused even with one stream declared,
  where it used to consume the engine's unrelated `Other` with no window at all.

- **The per-arrival query routes now answer every solution of an arriving element, not its first
  match**. The two routes that evaluate each element by itself — a single
  fixed-predicate, non-aggregate pattern under any legacy window, and any single-stream `[NOW]`
  query, with or without a static graph pattern — bound each triple pattern to the FIRST
  statement of the element that matched it, in query order, and never went back. For an element
  carrying several statements
  (a `DataStream.push(Collection, …)` payload, a multi-statement `push_stream_events` event,
  a `cqels-cdsp` geo tagger's output) that gave two wrong answers:
  - an element whose statements satisfy one pattern k times emitted ONE row where its own graph has
    k solutions — a reading inside two overlapping zones, which `SpatialZoneTagger` tags with both,
    was reported in one of them, whichever the tagger's set iteration put first;
  - an element whose only solution needs an earlier pattern re-matched emitted NOTHING: under
    `[NOW]`, `{ ?s ex:p ?v . ?s ex:q ?w }` over `a ex:p 1 . b ex:p 2 . b ex:q 9` answered no row,
    while the same element without `a ex:p 1`, or the same query under `[RANGE 10s]`, answered
    `b`'s.
  Each arriving element is now evaluated as a one-element window — its statements indexed and the
  stream BGP joined over them, the matcher the windowed routes already run — so it contributes the
  rows of every solution of its own statements, and the routes' `Istream` contract holds for
  multi-statement arrivals, where it was scoped to atomic ones. On the static-lookup route every one
  of the element's solutions seeds the static join, at its own join key when that key binds an IRI
  or blank node: 3 matching statements for
  a subject with 2 static labels are 6 rows where they were 2. What rides on those rows follows
  them. An inline `BIND` / `FILTER` applies to each solution. Under `[NOW]` a `GROUP BY` aggregate
  folds the arrival's solutions together: an arrival of 3 matching statements is one group of 3,
  where it counted 1. A window-scoped `SELECT DISTINCT` collapses an arrival's duplicate
  solutions. Under `[NOW]`, `ORDER BY … LIMIT`'s continuous top-K is offered each of those
  rows, so one such arrival can emit an intermediate snapshot per row, its last one the answer, and
  with `SELECT DISTINCT` the same projected row again for each row that improves it — an element
  carrying `(a ex:p 1) (a ex:p 2) (a ex:p 3)` is three copies of the one row. Two lookup
  shapes stay outside the `Istream` equality, for any arrival: a query with no join key and a
  static pattern the candidate fallback cannot serve, and a stream solution whose join key binds a
  literal. A query has no join key when the static and stream patterns share no variable and no
  `operators.lookup.joinKeyVariable` hint — a JSON-dialect option — names one; a static pattern
  whose subject and predicate are unbound variables and whose object is an unbound variable or a
  literal is then answered from the facts of one root, the element's first subject on the
  CQELS-QL text path, where the composed windowed route answers it with no row at all, and over a
  multi-statement element those rows follow the element's statement order. A solution whose key
  binds a literal has no root either: it is matched through the fallback against the element's one
  view, where the composed route drops its row — an element carrying `r1 ex:pid 7` beside
  `?person ex:id ?pid . ?person ex:name ?n` answers `[7|Ann]`, SPARQL's answer, where the composed
  route answers `[]`.
  With two or more stream patterns the element's own join is bounded by the partial bindings it
  HOLDS AT ONCE — the level being extended plus the level being built — at
  `cqels.join.crossProductCap` (default 100,000) itself, a ceiling that deliberately does not
  scale with the pattern count, because the memory alive at one instant must not and the pattern
  count is chosen by whoever registers the query: 20 disconnected patterns matching twice each
  hold about 1.5M bindings inside a `cap x patterns` budget, so the heap is gone before such a
  budget contains anything. An arrival's TOTAL work follows from the same ceiling rather than from
  a second guard — every level is at most the cap and there is one level per pattern, so at most
  `cap x patterns` extensions. The ceiling bounds the NUMBER of partial bindings held, whatever
  the query, not the bytes they occupy: each carries one binding per pattern matched so far, so
  the entries alive are that number times the binding width, which the query's pattern count sets.
  An arrival over the ceiling is DROPPED: it contributes no row, the query keeps running, and the
  reason is logged once per query (WARN, then DEBUG). The static side's work bound on the lookup
  route, the same flag x its pattern count, now behaves the same way and for the same reason, and
  is spent per ARRIVAL by all of an element's solutions together — the bound one element had when
  it was one seed — which also caps the rows one arrival can materialise there. The
  windowed routes, the composed lookup and the temporal route still fail the query for their own
  guards; on a per-arrival route the element IS the evaluation, so dropping it is well
  defined and costs exactly that arrival's rows — the emission is the declared window's `Istream`
  minus them — while a throw there reaches the subscriber as
  `onError` and would end a standing query that any writer of the stream could then stop (two
  findings of a codex security review of this change). An element carrying ONE statement takes a
  fast path through the same per-statement tests. An element in which each pattern matches at
  most one of its statements has at most one solution, found as before however many statements it
  carries: on the stream-only route it answers as it did, while on the static-lookup route that
  solution's static side is now rooted at its own join key, when that key binds an IRI or blank
  node, rather than at the element's first —
  `?s ex:p 2` beside `?s ex:label ?l . ?x ?pp ?o` over `(a ex:p 1) (b ex:p 2)` answered
  `[b|a|La]` and now answers `[b|b|Lb]`. A variable repeated WITHIN one triple pattern is still answered wrongly by
  the shared matcher — now the same way on the per-arrival routes as on the windowed hash join.

- **`push_stream_events` refuses an `eventTime` more than a day ahead of the server clock** (ADR-0004).
  One accepted far-future `eventTime` (year 2200, say) on a shared stream could push that stream's
  event-time progress out of reach of every later honest event. The MCP tool now refuses the whole call,
  before anything is pushed, when any event's `eventTime` is greater than server time plus the bound,
  and the error names the event index, its `eventTime`, the bound and the property to change. The
  engine is unchanged and still reads no wall clock.

- **CEP matches are labelled complex events, each emitted exactly once**. The CEP
  matcher (`NFAPatternProcessor`) is rebuilt as a labelled automaton (ADR-0004). It no longer
  emits the same complex event twice: `SEQ(?e1; ?e2+; ?e3)` over drop, weave, weave, spike answered
  `[drop weave weave spike]` twice (public report CQELS4J#75). It no longer collapses two different
  complex events with the same event list into one: `a+` then `b?` over two events that match both
  answers `{a:[e1, e2]}` and `{a:[e1], b:[e2]}`, which the new `PatternMatch` run accessors tell
  apart. A trailing optional argument matches when it is absent: `SEQ(?a; ?b?)` over `a` answers
  `[a]` as soon as `a` arrives. Other shapes that were never answered are now answered: an
  event that satisfies both an optional argument and the argument after it, the branch where an
  optional that occurred is left empty, a consumption two or more empty or negated arguments after
  the previous one, and a greedy run that ends before its successor or at the window close. Shapes that were answered but are not matches no
  longer are: a `[0,0]` argument consuming an event, a successor entered after one event whatever
  its quantifier, a `next` successor entered at a non-adjacent event, and a repeat that skipped an
  event its successor could take. Negations are judged on their scope, the open interval between
  the neighbouring consumptions: a negation after a repeat no longer sees the gaps inside the repeat,
  consecutive negations share their scope, a `notNext` judges the one adjacent event, and the event
  that ends the scope is not inside it. The conformance gate added with ADR-0004 compares the
  matcher with a reference enumerator of the specification and finds no difference on 5 000 seeded,
  500 clock-seeded and 10 375 204 exhaustively enumerated cases.

- **A repeated CEP argument consumes every relevant event**. Under skip-till-next-match a
  repeat can no longer end before an event it could take, so `SEQ(?a; ?b+; ?c)` over `a b1 b2 c`
  answers one match, `[a b1 b2 c]`, not one per number of `b` events; a match starting at each
  later `b` is still answered.
- **A trailing `NOT` is accepted when its window closes.** `SEQ(?a; NOT ?b)` with
  `[RANGE 10s]` emits `[a]` at the first event or watermark past `a` + 10 s if no `?b` came; a
  later event used to drop it there, so it was emitted only when the stream ended first. On `processWithTimeouts` the same partial now
  surfaces as a match, no longer as a timeout.
- **Events below the last watermark are dropped and counted.** A data event whose timestamp is
  below the last watermark the query has been given (`advanceWatermark`) is no longer matched; it is
  counted on the new meter `NFAPatternProcessor.lateDropped()` (`cep.late.dropped`). Data timestamps
  never make an event late: on a stream that is never given a watermark nothing is dropped, and an
  out-of-order event is matched where it arrives (positions are arrival order). Use
  `CepExecutionOptions.eventTime(...)` to reorder out-of-order input before the matcher.
- **A run is over once an event or watermark past its window arrives.** When an event or watermark
  whose time is past a run's first event + `RANGE` arrives after the run started, the run takes no
  further event: a far-future timestamp expires the runs that are open when it arrives, as before,
  and an in-window event arriving after it starts runs of its own rather than extending them. Runs
  that start after it are unaffected by it, so one far-future timestamp cannot silence later
  matches on a shared stream.
- **Five CEP pattern shapes are refused at registration** with `IllegalArgumentException`: two
  arguments with the same name; a negated argument with a quantifier or `greedy`; a negation
  directly after an argument that may be empty (`SEQ(?a; ?b?; NOT ?c; ?d)`); a trailing
  `notFollowedBy` — or one followed only by optional arguments — without a window
  (`SEQ(?a; NOT ?b)` with no `RANGE`); and `next(...)` directly after `notFollowedBy(...)`.
  A quantifier with min > max and a negative window are refused too. `CQELSEngine.registerCepPattern`
  and `registerCepQuery` refuse them when called, before anything is queued.
- **CEP patterns are limited to 256 arguments.** A pattern with more SEQ arguments than the system
  property `cqels.cep.maxPatternArguments` (default 256, read at registration) is refused with an
  `IllegalArgumentException` naming the count, the limit and the property; a value of `0` or less
  disables the limit. It bounds the work one registration can cost per event, and changes no
  admitted pattern's matches. A CQELS-QL `FILTER(SEQ(...))` is counted before any other compilation
  work.
- **CEP argument names are limited to 256 characters.** A pattern whose SEQ argument name (a
  variable, or its `AS` alias) is longer than the system property `cqels.cep.maxArgumentNameLength`
  (default 256, read at registration) allows is refused with an `IllegalArgumentException` naming the
  length, the limit and the property; a value of `0` or less disables the limit. The name is repeated
  in every MCP result row (`named`, `run`).
- **A CEP match is limited to 10 000 events.** A partial match that would consume more events than
  the system property `cqels.cep.maxEventsPerMatch` (default 10 000, read at registration) allows is
  dropped and counted on `NFAPatternProcessor.partialsEvicted()` (`cep.partials.evicted`), like the
  partials the state-explosion limit drops; a value of `0` or less disables it. A repeated argument
  fed events with one timestamp was otherwise bounded by neither the window nor the state-explosion
  limit, and so was the match it finally emitted, including its MCP row.
- **`NFAPatternProcessor.process()` receives watermarks.** `advanceWatermark` now reaches every
  live `process()` subscription as well as `processWithTimeouts()` ones, so a held complex event is
  decided on a quiet stream. Watermark saturation never terminates a data subscription: watermarks
  may fill only their own share of a subscription's bounded input channel, the rest stays reserved
  for the data the subscription has requested, and a caller whose watermarks find that share full
  waits while the subscription keeps consuming, failing loudly only after it has consumed nothing
  for a second. A caller never waits for another subscription's listener, so two listeners that call
  `advanceWatermark` from their callbacks cannot deadlock.
- **A full stream-query buffer no longer drops rows silently (MCP).** When a registration's
  `bufferSize` is reached, the oldest row is still evicted, but the eviction is now counted: the next
  `recall_memory` with that `queryId` (and a transient `stream_query`) reports how many rows were
  dropped since the previous drain, and the `cqels://queries/{queryId}/results` resource adds a
  `dropped` field. The process-wide meter `StreamQueryRegistry.resultsDropped()`
  (`stream.results.dropped`) counts every eviction. This applies to every stream-query registration,
  CEP or not, including `watch_invariant`.
- **A CEP input's matches are built as they are consumed.** `NFAPatternProcessor` no longer copies an
  input's matches into per-input lists before emitting them; each match is built when the subscriber
  takes it, so an input that decides many matches holds only the decided runs.

- **Stream-query results are byte-bounded, and CEP match rows are rendered only when drained (MCP)**.
  A CEP match was rendered into its result row on arrival — each event, a graph payload
  included, rendered into `events`, `named` and `run` — whether or not the row was ever drained, and
  the buffer and the drain were bounded by row count only. A CEP registration now buffers the compact
  match and renders it at drain time, rendering each event once per drain response and sharing that
  rendering across the response's rows. Every registration's buffer is also bounded by an estimate of
  its rows' rendered size (`StreamQueryRegistry.DEFAULT_MAX_BUFFER_BYTES`, 16 MiB): past it the oldest
  rows are evicted and counted exactly like a full buffer's (`dropped`, `stream.results.dropped`), and
  a row larger than either budget on its own is not buffered: it counts in `dropped` and is also
  reported separately as `oversized` (meter `StreamQueryRegistry.resultsOversized()`,
  `stream.results.oversized`). This changes plain (non-CEP) queries too: a row larger than the
  response budget used to be delivered and is now refused and reported. Each drain
  response is bounded the same way (`DEFAULT_MAX_RESPONSE_BYTES`, 4 MiB): it stops before the row that
  would cross the budget and says so. Sizes are bytes on the wire: the UTF-8 of every string after
  BOTH JSON escapings it goes through — the result serializer's and the MCP transport's encoding of
  the tool result's text — plus fixed per-field overheads. The bound has two layers. On push, an
  estimate computed without rendering refuses early a row that cannot fit; it charges a number by
  its magnitude, a literal its datatype and language, a statement its context, and a string its
  escaping, is bounded in depth, stops as soon as it passes the budget and sizes a shared value
  once. A value of any other type is never `toString()`-ed: its row is refused (counted
  `oversized`, logged with the type) — this also applies to plain queries. At drain, each row is
  sized complete from its live values before it is rendered and measured again before it is
  returned — that is the bound: a row that outgrew its estimate (a value mutated after it was
  buffered, or anything the estimate misses) is refused without being rendered, counted in
  `dropped` and `oversized`, and the drain continues with the next row. A refusal on push that
  makes a loss report pending notifies a `notify: true` registration (once until the next drain).
  A row is charged the larger of its JSON form and its text-format form (`natural` and `turtle` pad
  every row to the first row's columns), and the caller then measures the encoded text it actually
  returns, cutting any row past the budget (put back in the buffer at its arrival position and counted in
  `remaining`; the response is marked `truncated`) without ever building the whole oversized text; in the one-shot `query` tool the
  first row's columns carry across its polls. RDF-star
  triples are sized structurally, never through their expanded string. An event pushed with
  `push_stream_events` or stored with `store_memory` (`memory: "shortterm"`) is sized once, when it
  is ingested, and a CEP match over it is sized by
  lookup; for an event an embedder pushes, the finite sizes of recent large events (up to 64, held
  weakly) are remembered, a size past the budget is never remembered, and each match's sizing is
  bounded by the budget itself. `cqels://queries/{queryId}/results` refuses a query id longer than
  256 characters, and a blank or invalid one, without echoing the URI; it refuses a read whose
  framing cannot fit the budget before draining anything, and every registration refuses an id
  longer than 256 characters. Short-term `store_memory` refuses Turtle longer than 8 000 000
  characters before parsing it, and more than 100 000 statements once parsed and before any is
  pushed; `push_stream_events` applies the same 8 000 000-character limit to a call, each event
  counted before it is parsed, and its 100 000-statement limit message by message while the body is
  parsed. Both are configurable through
  `CqelsMcpServerConfig.Builder.streamQueryMaxBufferBytes` / `streamQueryMaxResponseBytes`. In the
  issue's scenario — 999 `H` events and one graph with a 512 KiB literal deciding 254 745 two-event
  matches into a `bufferSize=100000` registration — an 8 MiB buffer budget held 5 rows, the heap
  grew by 2.4 MB across the run (the test bounds it at 256 MiB), and five drains under a 2 MiB budget
  returned one row each, each marked truncated until the last (`CepResultByteBudgetTest`).
- **The OCI smoke gate runs two clients with per-client tokens and a private stream across the
  restart; private streams are documented for operators**. `scripts/oci/smoke.sh` (the PR image
  job and the candidate-promote job of `publish.yml`) starts the image with a two-client
  `CQELS_MCP_HTTP_CLIENTS_FILE` and the metrics endpoint, keeps every earlier assertion, and also
  requires: a request without a token gets `401`; client A's private stream takes a durable push;
  client B is refused on push and on create with the ownership error, does not see the stream in
  `cqels://streams`, and `cqels_mcp_stream_ownership_refused_total` counts both refusals; after the
  restart A's previous session id gets `404`, a query A registers over the stream receives its
  pre-restart observation, the stream is back with its owner and A's push continues the same journal,
  B is still refused, and once A removes the stream B can create the name.
  `docs/UPGRADING.md` describes private streams (no downgrade once a journal holds one, the cap, the
  lifetime per configuration, a removed client's streams) and the notification change;
  `cqels-mcp/README.md` lists the per-client cap; the tool descriptions say that another client's
  private stream is refused.

### Upgrade notes

**Row-count changes for `SELECT DISTINCT` on the per-arrival routes, under EVERY legacy
window**. The shape is a single fixed-predicate, non-aggregate `SELECT DISTINCT`, with or without a
static graph pattern — the two routes that read no window spec. Its dedup was global over the whole
stream; it is now scoped by the declared window, so a key suppressed forever is now re-reported
once it has left. Measured, and in every family the count goes UP: `[RANGE 5s]` / `[SLIDE]` over one
subject at `t = 0 s, 1 s, 30 s` answers 2 rows where it answered 1; `[TRIPLES 2]` over `a, b, a`
answers 3 where it answered 2 (and `[TRIPLES 3]` still answers 2); `[NOW]` over three arrivals of
one subject answers 3 where it answered 1, while an arrival's own duplicate rows still collapse.
Three in-tree cells pinned the old answer and are relicensed rather than weakened, which is the
evidence that real queries could have leaned on it. Unaffected by THIS change: any query without
`SELECT DISTINCT`, every windowed route, and `[NOW]` beside a `GROUP BY` aggregate. Other
changes in these notes do alter rows for a query without `SELECT DISTINCT` on these same routes —
an element carrying several matching statements now yields one row per solution, below.
The `### Changed` entry has the full contract and the five scope limits.

**Row-count changes on the flat static-lookup route** (CQELS4J#67): rows that no stream element
justified are gone, and rows the root-subject seed dropped now appear. The `### Fixed` entry
lists the four shapes to re-check; the one most likely to be noticed is a `[NOW]` query with two
or more stream triple patterns beside a static pattern over one-statement-per-element input,
which answered a fragment per element and now answers only from an element that satisfies every
pattern — from every solution of that element, since the one-element-window change further down
these notes. The fix first shipped with a matcher that took the first consistent statement per
pattern without backtracking, which dropped a multi-subject payload whose only solution needs an
earlier pattern re-matched (the `### Fixed` entry gives the measured case); that payload answers
again, as it did before this release.

**Rows APPEAR on both static-lookup routes** (CQELS4J#67, completeness half): a static pattern
whose join-key variable sits in object position, or whose subject is a constant outside the key's
forward neighbourhood, now matches on the composed route as it already did on the flat one; and a
`FILTER` / `BIND` / `HAVING` naming an unprojected stream variable now sees its input on the flat
route as it already did on the composed one. Both directions only add rows.

**On the composed route, that addition is not free.** For a composed query — a static pattern
beside a multi-triple or variable-predicate stream BGP — under an accumulating window
(`[RANGE d]`,
`[TRIPLES n]`, `[SLIDE d STEP s]`, or `[PAST d]`), where the static pattern is one the forward view
cannot reach (its join-key variable in object position, e.g. `?collection ex:member ?f` with a
stream-bound `?f`, or its subject a constant outside the key's forward neighbourhood): every
arrival now performs a repository candidate scan per distinct `(subject, predicate, object)`
request, memoized across that arrival's windowed-join result rows. It was one scan per row
until then. That scan bypasses `CACHE`, and it materialises the static pattern's
full matching extent before the work bound can apply; the cost is new only because these
rows used to be silently dropped rather than computed. Watch per-arrival latency on such queries —
it scales with the number of DISTINCT requests the window's rows make, times the static pattern's
candidate extent — and until the bound also
covers the scan itself, tracked separately, mitigate today by preferring a guard whose
subject is the join-key variable where your data allows it, keeping the window narrow, or, if you
control the static data, putting the membership fact on the key's forward edge instead of the
reverse edge.

One registration
change comes with the second: on the flat route `ORDER BY DESC(?rank)` over
`BIND(?streamVar AS ?rank)` is now accepted and sorts, where it was refused because the sort key
could not reach the rows — a query that used to fail at registration now runs.

**Two shapes that used to register now fail loudly.** A `FILTER NOT EXISTS` block correlated on a
variable the stream-only routes cannot put in the row is refused at registration, naming the
variable — project it, make it an aggregate argument or a `GROUP BY` key, or, on a
query with no aggregate and no `GROUP BY`, an `ORDER BY` key. A `STREAM` block inside `OPTIONAL` is
rejected at parse time and by `CqelsQueryDefinition.OptionalBlock`'s constructor; it
previously compiled and null-filled every match, and `OptionalBlock`'s `getStreamPattern()` /
`hasStreamPattern()` accessors are deprecated with it. `IntervalJoinOperator.processingTimeJoin`'s
four `public static` overloads are deleted too — no registered query reaches them, but a
library embedder calling them directly will no longer compile or link; there is no drop-in
replacement, only the event-time join that `TemporalJoinDispatch` has always used internally.

**One behaviour change on one route shape: composed stream+static queries (a static graph pattern
alongside the `STREAM` pattern) whose stream side needs the windowed join (a multi-triple stream
BGP or a variable predicate), no aggregates, a declared window other than `[NOW]`, with `ORDER BY`
and/or `LIMIT`.** `[NOW]`-windowed queries and every other route are unaffected — a single
fixed-predicate stream triple under a non-`[NOW]` window, with or without a static lookup, is still
refused at registration with `ORDER BY`, `LIMIT` or both, as before: ADR-0002 D3's remaining scope,
tracked separately; an `Istream(...)` projection over this same shape stays rejected with a
modifier, and so is the shape under a directional (`[FUTURE …]`, `[PAST … FUTURE …]`) or `LATENESS`
window — but for a **bare** `ORDER BY` or **bare** `LIMIT` those three refusals now come from
`execute()` with the directional / `LATENESS` / `Istream` message instead of from
`rejectUnsupportedOrderByLimitShape`'s "ORDER BY on non-time attributes" / "LIMIT without ORDER BY"
text. The query is refused either way, and still synchronously inside `registerCqelsQuery`, so the
only user-visible change is the exception's wording — which now names the actual reason. A query
definition built programmatically with no window spec at all (the
typed `StreamDefinition(String, WindowSpec, Map)` constructor with a `null` `WindowSpec` — a shape
query text cannot express) declares no window, so it is not this route shape and keeps its previous
behaviour: `ORDER BY … LIMIT` still rides the continuous top-K and a bare modifier is still refused
at registration.

- **Ranking is now scoped to each evaluation, not cumulative across the whole stream.** A row that
  wins the top-K keeps winning only while its underlying data is still inside the declared window;
  once evicted from the window, it stops competing, exactly as `executeWindowedHashJoin` (the
  stream-only sibling) already behaved. If your application relied on a rediscovered-and-never-
  reconsidered "best ever seen" answer from this specific route shape, it will now see the answer
  the current window actually supports instead.
- **Emission shape changed from a sparse "re-emit only when the top-K changes" event stream to one
  complete, freshly-ranked batch per evaluation** (per arriving stream element) — matching the other
  windowed-batch routes. A consumer counting emissions, rather than reading each snapshot's content,
  will see a different (typically higher, and now regular) emission rate.
- **`DISTINCT`, when combined with `ORDER BY`/`LIMIT` on this same route shape, now dedups within
  each evaluation** — against that evaluation's own row set, where the continuous top-K deduped
  against its cross-evaluation representatives (`applyDistinctTopK`). `HAVING` is a per-row test
  ahead of the truncation on both the old tail and the new one; `DISTINCT`/`HAVING` alone (no
  `ORDER BY` or `LIMIT`) are unchanged.
- **A bare `ORDER BY` (no `LIMIT`) or a bare `LIMIT` (no `ORDER BY`) on this same route shape now
  registers and executes**, ranked or truncated per evaluation like the paired-modifier case above.
  Previously registration rejected either shape outright on this specific route, matching the
  documented refusal for a route with no evaluation boundary to rank inside — a refusal that never
  applied here in the first place, since this route always had a window to bound each evaluation's
  batch; it only shared the rejection because it was, until now, misrouted through the boundary-less
  continuous top-K tail. The other routes that materialise evaluation batches
  (`executeWindowedHashJoin`, `executeIncrementalAggregate`, the two directional `SELECT` routes,
  and this route's own aggregate topology) already accepted a bare modifier — registration admits
  any such route — and this brings the composed non-aggregate topology in line with them. A bare
  modifier on the single fixed-predicate shape named above is still refused.

**One merged change alters results for existing Builder configurations**. It replaces a
silently uncorrelated answer with the SPARQL one, and it makes correlation depend on whether a
variable is mapped. **The surprising half is the asymmetry:** a *mapped* variable now correlates
by its exact RDF term — same datatype AND same lexical form — while an *unmapped* one keeps
correlating through the row's Java lift, so declaring a mapping on a key changes how that key
correlates, in two directions:

1. **A mapped numeric or boolean key matches by exact term — datatype and lexical form — not by
   value.** Unmapped, a numeric/boolean key correlates by VALUE, because the lift lost the term and
   a value-compatible hedge compensates: `"30"^^xsd:int` in the row matches
   `"30"^^xsd:integer` in the repository, `"030"^^xsd:int` matches `"30"^^xsd:int`, and
   `"1"^^xsd:boolean` matches `"true"^^xsd:boolean`. Mapped (`"long"`, `"string"`, `"boolean"`,
   …), the term is carried and compared with RDF term equality, so none of those pairs matches
   any more — an `OPTIONAL` on it null-fills, a `FILTER NOT EXISTS` on it stops excluding, a `UNION`
   arm on it stops matching — which
   is what SPARQL BGP matching says, but it is a **silent row change** for a configuration that
   worked before. Both mismatches are common: RDF4J's `createLiteral(int)` mints `xsd:int` while
   Turtle's bare `30` mints `xsd:integer`, and a lexical form is whatever its producer wrote
   (`"030"`, `"+30"`, `"1"` for a boolean). Aligning the datatypes alone does NOT prevent the
   change; the lexical forms must match too (canonical forms on both sides). Otherwise drop the
   mapping on that key — an unmapped numeric or boolean is emitted as its `Long` / `Double` /
   `Boolean` lift, and whether a mapped one is emitted any differently depends on the write
   site, which is tracked separately, so on some routes the mapping changes nothing but
   the correlation.
2. **A mapped language-tagged or non-core typed key now correlates by its own term — and only by
   it.** Unmapped, `"alice"@en` or `"2020-01-01"^^xsd:date` lifts to a plain `String` that
   re-mints as `xsd:string`, so a reshaper on it never matched the key's own term — only an
   untagged `xsd:string` literal of the same lexical form, where the repository held one —
   unchanged. Mapped, the term is carried and the block matches it: an `OPTIONAL` that used to
   null-fill now binds, a `FILTER NOT EXISTS` that used to be inert now excludes, a `UNION` arm
   that used to match nothing now matches. The same exactness takes the old match away: where the
   repository holds the untagged `xsd:string` form and not the key's own term, an `OPTIONAL` that
   used to bind now null-fills, a `FILTER NOT EXISTS` that used to exclude now keeps the row, and a
   `UNION` arm that used to match now drops it — a silent row change of the same kind as
   direction 1, for a configuration that worked before. Aligning the repository's literal with the
   key's term (the same language tag or datatype) restores the match; so does dropping the
   mapping on that key.

Neither direction touches an unmapped key — carrying the term for every variable stays a separate
item — nor a query with no reshaper. A mapping whose converter cannot present a value stays
open. A converter that throws still terminates the stream at the fetch on both routes — by
reading: neither fetch site catches. What was measured: on the flat route a row whose only fetched
presentation is unbound is still dropped before any reshaper runs, unchanged; on the composed
route the same row — stored with an explicit null — now correlates, one row where there were two
uncorrelated ones. The limitation above — a BIND target of a mapped source — is unchanged: it
still seeds from the re-minted presentation.

**A windowed query whose static pattern shares no variable with its `STREAM` block now answers,
where it used to emit nothing at all**. On the composed route — a static graph pattern
beside a stream BGP that needs the cross-element join, under `[RANGE d]`, `[TRIPLES n]`,
`[SLIDE d STEP s]` or `[PAST d]` — such a query dropped every row for having no join key, on all
four of that route's shapes. It now answers `|window rows|` x `|static solutions|` PER EVALUATION,
and this route re-evaluates its whole window on every arrival, so re-check any query where the
shared variable was forgotten rather than omitted on purpose: the cost is now visible instead of
the answer being invisible. The enumeration is bounded PER EVALUATION by the existing
`-Dcqels.join.crossProductCap=<n>` (times the static pattern count) and THROWS mid-stream when
that bound is exceeded rather than truncating — the rows of one arrival share one budget, so the
cap bounds `|window rows|` x `|static solutions|` rather than each row's share of it — and
registration logs one warning naming the query and the condition. A static pattern whose only
shared variable sits in PREDICATE position also answers now, and as a real join — the stream's
binding for that variable constrains the static side — so it is not warned about.

**Before relying on the new rows, one disclosure: on the composed route a `BIND`-mediated
correlation is still not honoured as a join**. `BIND(?s AS ?x)` followed by `?x ex:label
?lbl` relates the two sides for a reader but not for join-key resolution, so the query counts as
unjoinable (it warns), the static side is enumerated freely, and the `BIND` overwrites the variable
afterwards: measured on this route, one row per `ex:label` statement in the graph — three, of
which one is the answer — where the same query answered `[]` before the fix (measured both ways).
The unjoinable-BGP fix therefore
turns a silent empty answer into a wrong one for that one shape — the flat route has answered it
this way since alpha.20, and converging both is tracked separately. The unjoinable-BGP
warning says so rather than advising you to introduce a shared variable you appear to have
already.

**One static enrichment per DISTINCT join key of an arrival**. A multi-statement arrival
whose statements bind the join key to k different IRIs now roots k forward views — one per distinct
key — where it rooted exactly one before, because each stream solution's static side must be rooted
at ITS OWN key. The count is bounded by the arrival's statement count and de-duplicated within the
arrival (a repeated root is looked up once, and the candidate memo serves repeated candidate
requests once), but it is NOT bounded across arrivals by anything except the lookup cache: with no
`CACHE` clause the operator's TTL is zero, so every distinct key is a fresh repository traversal.
If your elements carry many distinct keys, declare a `CACHE` duration — that is the remedy — and
size it against how often those keys repeat. Tracked as a cost property, with the measurements, in
the per-arrival growth issue rather than bounded by a guard: capping it would refuse rows the
corrected semantics require.

**Not covered**: a keyless static pattern that lives only inside `UNION` arms. It is not
part of the top-level static BGP the fix enumerates, so it still answers nothing and still without
a warning — unchanged by this release, and pinned as a characterized residual rather than assumed.

**More rows from batch-shaped elements on the per-arrival routes, and two bounds that drop an
arrival instead of the query**. Who sees more rows: a query on the per-arrival routes (a single fixed-predicate,
non-aggregate pattern under a legacy window, or any single-stream `[NOW]` query) whose elements
carry several statements matching one of its triple patterns — `DataStream.push(Collection, …)`,
a multi-statement `push_stream_events` event, or the batches the `cqels-cdsp` geo taggers emit
(`SpatialZoneTagger`, `ChargingStationProximityTagger` and `ZoneSpeedViolationTagger` add one
statement per zone or station found). Such an element now yields one row per solution instead of one, and a `[NOW]`
multi-pattern query answers elements it used to drop. Both bounds are `cqels.join.crossProductCap`
(default 100,000, the flag that already bounds the windowed joins); raise the flag, make the
patterns more selective, or push smaller elements. The first bounds the element's own join, with
two or more stream patterns: an arrival whose join builds more than that cap x the number of
stream patterns partial bindings — where a join that really grows that large, such as two patterns
sharing no variable over 448 matching statements, is DROPPED with no row from it and one WARN per
query, where it used to contribute at most its first match. The guard that enforces this is on what
the join HOLDS AT ONCE (the level being extended plus the level being built), at the cap itself
rather than the cap times the pattern count: a wide level can exhaust the heap inside a cumulative
budget that has not been exceeded, and the pattern count is an input whoever registers the query
chooses. The `cap x patterns` figure above follows from it — one level per pattern, each at most
the cap — rather than being checked separately. It bounds the NUMBER of partial bindings held, not
their bytes: each carries one binding per pattern matched so far, so a wider BGP holds wider
bindings and the entries alive scale with the query's pattern count even though the object count
does not. Relative to the release before this change the ceiling refuses nothing — an
element whose live set exceeds the cap was answering one first-match row then — but relative to
the behaviour this fix first merged with it drops some wide arrivals that would otherwise have
answered; raise `cqels.join.crossProductCap` if you have one. The
second is the static side's work bound on the per-arrival lookup route, whatever the shape of the
static BGP: the cap x its pattern count per ARRIVAL, spent by all of an arrival's stream solutions
together — which is what one element spent before this release, when it had exactly one solution,
so a single-solution arrival is bounded exactly as it was and an element with k of them no longer
spends k times that. Exceeding it drops that arrival the same way, with its own WARN. The same
budget bounds what an arrival can MATERIALISE on that route: every static solution is reached
through one counted extension, and at most one row is built per solution.
**Neither fails the query**, because on a per-arrival route a throw reaches the subscriber as
`onError` and ends the subscription, so any writer of the stream could stop another session's
query with one oversized element (both found by a codex security review of this change). The
composed windowed route, the windowed hash join and the temporal route are unchanged and still
fail loudly: there an arrival is joined against window state, so dropping it is not a well-defined
remedy. A dropped arrival contributes no row, so a route's emission is the declared window's
`Istream` minus the rows those arrivals would have produced. And
`BatchUnrollerProcessor` is no longer needed for a single-stream CQELS-QL query to see every
statement of a batch; it remains what `registerAspQuery` needs, and for a CQELS-QL query it now
costs something: unrolling destroys the joins WITHIN an element, so over an unrolled stream a
`[NOW]` BGP whose patterns need two different statements can never match.

**A `[NOW]` + `GROUP BY` aggregate now folds once per arrival**. A CQELS-QL `GROUP BY`
aggregate under `[NOW]`, stream-only or beside a static graph pattern, emits one fold per arriving
element. On a live stream it used to emit nothing at all; on a FINITE source (a test `Flux`, a
replay) it emitted one fold over the whole source at completion, and those totals are no longer
produced by `[NOW]` — declare `[RANGE d]` or `[TRIPLES n]` to keep aggregating across elements.
Three consequences travel with the move: `SELECT DISTINCT` beside such a fold now dedups within
the arrival instead of across the stream; a `HAVING` on a count only an accumulated window could
reach now selects nothing; and a `RuntimeException` escaping the `HAVING` expression evaluator now
surfaces instead of silently removing the group (an expected failure — unknown prefix, type error,
unbound variable — still drops the row). A CypherQL `[NOW]` `GROUP BY` aggregate changed the same
way.

**A top-level `BIND` above the first `OPTIONAL` / `UNION` now runs before it, and `FILTER NOT
EXISTS` now runs after every top-level `BIND`**. On the two static-lookup routes a block
that probes a `BIND` target now sees it: a `FILTER NOT EXISTS` now excludes, an `OPTIONAL` now
binds, a `UNION` now emits. Rows can also DECREASE: a `BIND` above a block that binds the same
variable now correlates that block instead of being overwritten by it (measured on the composed
route: 14 rows became 8). A `BIND` above a positional block is now also in scope for the group's
`FILTER`. Untouched: a query with no `OPTIONAL` / `UNION` / `FILTER NOT EXISTS` block, and one
whose only blocks are `OPTIONAL` / `UNION` with no top-level `BIND` above the first of them. A
query with a `FILTER NOT EXISTS` is not in that set even with no `BIND` above it, because the
anti-join itself moved after the `BIND` stage. The `### Fixed` entry has the per-route detail.

**`OPTIONAL` and `UNION` on the static-lookup routes now answer rows they dropped or null-filled,
and one sort key now registers**. An `OPTIONAL` whose join key the aggregate record
did not carry (a bare `COUNT(*)`, for instance) now binds from the repository instead of
null-filling every row, on both static-lookup routes, so an aggregate over it can change. On the
composed route a `UNION` whose arms correlate on a variable that is neither projected nor named by
an expression now emits the rows it silently dropped. `ORDER BY` on an `OPTIONAL`-only join key,
refused at registration before, now registers and sorts.

**CypherQL: rows may change for every query, several shapes are now refused at registration, and
the matcher has a new loud failure**. Rows: every match is now emitted instead
of the first, separate pattern groups are a conjunction, a repeated variable is checked instead of
overwritten, and `RETURN`, property names and literal values resolve exactly — a query that relied
on the previous behaviour will see different rows. Literal values keep their datatype, so
consumers that serialise rows see it: `cqels-mcp`'s query tool emits JSON `42` where it emitted
`"42"`. Newly refused at registration:

- on a boundary-less query (`[NOW]`, or no declared stream), a bare `ORDER BY` or a bare `LIMIT`;
- on a boundary-less query, an aggregate with `ORDER BY … LIMIT`, with or without `GROUP BY`;
- a `MATCH` whose groups read two different declared streams;
- a `MATCH` naming a stream no `FROM` clause declares — at any stream count, one included;
- with two or more streams declared: a `MATCH` made only of `GRAPH` groups, a `MATCH` group naming
  no source, and a definition with no `MATCH` at all.

The matcher now throws once one element's intermediate bindings exceed
`cqels.join.crossProductCap` (default 100,000) instead of materialising the product silently. And
`removeStream()` now accepts a stream a running Cypher query declares but never reads, where it
used to refuse it.

**Case conversion no longer reads the JVM default locale, which changes one population's
`UCASE` / `LCASE` results**. Built-in dispatch, `UCASE` / `LCASE` and a declared
`resultMapping` type name now canonicalise with `Locale.ROOT`. Under a Turkish or Azeri default
that is a fix; under a Lithuanian (`lt`) default it is a CHANGED result: `UCASE("i̇")` now keeps
the combining dot it used to drop, and `LCASE("Ì")` no longer adds one — toward the XPath contract
SPARQL 1.1 §17.4.3.4 / §17.4.3.5 require. Every spelling that worked under an English default is
unchanged.

**`GraphPatternJoinOperator` matches relationship types exactly and binds a repeated variable
consistently**. A pattern type now matches only a predicate's full IRI or its local name,
so `KNOWS` no longer matches `http://ex/NOT_KNOWS` or `http://ex/KNOWS_WELL`; and an edge that
reuses one variable at both endpoints, `(x)-[:KNOWS]->(x)`, matches only a self-loop. A caller
that relied on the substring match loses those edges.

**Dropped stream-query rows are reported.** When nothing was dropped, every response is unchanged.
When rows were evicted from a full buffer since the previous drain, `recall_memory` (with `queryId`)
and `stream_query` add a second text content (`warning: N result row(s) were dropped …`) and
structured content `{"dropped": N}`, and the results resource adds `"dropped": N` beside `results`.
A client that reads only the first text content sees the same rows as before. Each drop is reported
once, on the next drain.

**Two new registration refusals**. Both are thrown as `IllegalArgumentException` by the
CQELS-QL builder, which the strict parser that engine registration uses and the lenient
`CqelsQueryParser.parseCqelsQL` share; the JSON query format has no `FILTER NOT EXISTS` and no
aggregates, so it cannot reach either. A refused shape was dropping part of the query without an
error, so the refusal replaces an answer to a different query.

- A `FILTER NOT EXISTS` body with anything other than triple patterns fails with a message
  starting `CqelsQL parse error: '<construct>' is not supported inside a FILTER NOT EXISTS block
  (line L:C)`. The body may hold triple patterns over the static graph only. A refused construct is
  unsupported there and has no general equivalent rewrite: moving an inner `FILTER` to the top
  level, for example, changes which rows survive.
- A SELECT expression over an aggregate fails with a message starting `CqelsQL parse error: SELECT
  expression for ?<alias> (line L:C) wraps an aggregate in a larger expression`. Select the bare
  aggregate, `(COUNT(*) AS ?c)`, and compute the expression downstream of the query; expressions over
  aggregate aliases in SELECT are not supported yet.

**`SERVICE` is refused in client SPARQL**. A `query`, `recall_memory` or stored `sparql`
procedure that uses `SERVICE` now fails with a tool error naming the clause (`SERVICE <url> refused:
…`); under `SERVICE SILENT` the refused service contributes one empty solution, as SPARQL specifies for a
failed silent service. No request is sent in either case.
There is no switch to re-enable it. Client SPARQL on those paths also now fails with `Query evaluation
took too long` past `clientQueryTimeoutSeconds` (default 30; `CQELS_MCP_QUERY_TIMEOUT` /
`-Dcqels.mcp.queryTimeout`; `0` = no limit), on both transports.

**Per-session visibility on HTTP**. A client that registered a query in one HTTP session and
read, drained or forgot it from another — including a new session after reconnecting — now gets `No
such registered stream query` / `Unknown query ID`, or an empty `results` list from the resource, and
`cqels://queries` and `cqels://kg/stats` list only the caller's own registrations
(`cqels://engine/status` still counts every registration). Query ids remain unique server-wide, so
registering an id another session holds still fails with `already registered`.

**Session cleanup on HTTP**. Closing an HTTP session now unregisters its stream queries,
invariants and rule programs, and so does leaving it idle for 30 minutes (no request, no open event
stream); re-register after reconnecting. Tune or disable the idle release with
`httpSessionIdleTimeoutSeconds` / `CQELS_MCP_HTTP_SESSION_IDLE_TIMEOUT` /
`-Dcqels.mcp.http.sessionIdleTimeout` (`0` disables it; a close still releases). On HTTP, the 8
`register_rules` and 16 `watch_invariant` registrations are now allowed per session rather than per
server.
- **Reasoning output changes in two cases, mostly with more rows.** Under `PRIORITY`, a head shared by
  filtered rules is now inferred whenever any of those rules passes its filter, not only when the
  highest-priority one does. With recursive inference under a window, a re-derivation from
  newer premises keeps an inferred triple — and the facts built on it — alive, and re-emits it when
  it would otherwise outlive what the stream already carries (this includes a triple that is also
  asserted); the one case with fewer rows is a consequence that needed an
  inferred triple whose oldest premise had already left the window, which is no longer derived.
  Emitted inferences still carry the current event time. Measured on `cqels-cdsp`: none of
  its shipped rule sets (the four SHACL rule files and the four example rule sets) has two filtered
  rules with the same head, and the two chaining examples print the same output before and after.

**SHACL shapes that were silently ignored now validate**. A shapes file holding a
top-level `sh:PropertyShape` with a target, an untyped subject with a target, or an
`rdfs:Class` shape may start producing violations where its windows reported `CONFORMS`, and a
shapes file may now be rejected at parse time if one of those shapes — or a shape with no
target that nothing references — uses a constraint outside the supported list. A property shape
reached through `sh:property` that also carries a target is now accepted (it was rejected), and
validated both for its parent and over its own targets. Two malformed shapes that SHACL rules out are
now refused at parse time on every shape kind: a blank-node `sh:targetNode` value, and a blank-node
shape that is also an `rdfs:Class`. An `sh:minCount` / `sh:maxCount` value must now be an integer
literal (`xsd:integer` or an integer-derived XSD type); a string such as `"1"` is refused. The parse error for a graph with no
targeted shape now reads `No SHACL shape with a target found …` instead of
`No sh:NodeShape found`.

**New on-disk journal format for new journals; new failure mode at open**. A
journal created by this version starts with the 8-byte header `0x89 "CQELSJ" 0x02` and writes a
4-byte CRC32C after every record; do not reopen it with an earlier version, which knows neither.
An existing journal (no header) stays format 1: it keeps format 1 even after a truncation empties
it, and it is read and appended in format 1, never converted, with a `WARN` at open that its
payload integrity is unchecked. The first open adds an `event-journal.offset` sidecar beside the
journal, in either format. A journal written by a pre-release build of this version has the CRC
trailers but no header, and is refused at open with a message saying so: recreate the backend
directory (or restore it from a backup); no released version wrote such a journal. Under
`SYNC_ON_COMMIT` an open over a damaged committed prefix now throws
`CorruptCommittedStateException` where it used to succeed after silently deleting acknowledged
records. The recommended response is to restore the backend directory from a backup; without one,
move the whole backend directory aside (journal and checkpoints together) and start empty,
accepting the loss of its persisted state. Do not delete only the journal files beside a surviving
checkpoint: offsets would restart below the checkpoint's offset and replay would skip new events.
One false positive is possible on upgrade: a pre-upgrade build that crashed inside
`truncateBefore`, after replacing the journal file and before rewriting the commit pointer, leaves
a pointer larger than the file, which this version reports as damage. If that is the known cause,
deleting `event-journal.commit` lets the next open trust the file's records as they stand.

**Push no longer stops on a checkpoint failure; a restart replays more**. A stream
fed by `push` keeps journaling and delivering through failed periodic checkpoint writes; the
failure shows as an ERROR log and as a `persistenceHealth()` whose last checkpoint stops
advancing, instead of as a terminated stream. A journal-append failure now also terminates the
`DataStream` handle (`tryPush` returns `false`).
`close()` still throws when the final checkpoint write fails, but only after its teardown has
completed. On restart with the default `RecoveryPolicy`, an engine with any engine-built
CQELS-QL or Cypher query registered, or whose checkpoint carries no contributor state, now replays
the whole retained journal instead of nothing, so queries registered before `start()` receive the
pre-restart elements again — a windowed query can therefore emit rows after a restart that it did
not emit before. If such an engine also registers a stateful custom query, that query's checkpoint
blob is no longer restored; it rebuilds from the replay instead. With `replayEnabled(false)`,
nothing replays, so every registered contributor's blob is still restored. Engines whose registered queries
all contribute state keep today's behaviour. A `restoreLatestCheckpoint(false)` workaround is no
longer needed for this, and behaves as before. With `truncateJournalAfterCheckpoint(true)`, a
checkpoint that does not cover every query no longer truncates, so the journal grows until one
does. A restart whose queries the restored checkpoint does not cover, over a journal already
truncated behind it, now fails `start()` with `UnrecoverablePrefixException` instead of starting
with incomplete history. This includes a journal that an earlier version truncated behind a
checkpoint without state, which it did with `truncateJournalAfterCheckpoint(true)` and only
engine-built queries. To start anyway on the retained suffix, set
`truncateJournalAfterCheckpoint(false)` and `restoreLatestCheckpoint(false)`.

**`forget_memory` result text and argument shapes changed**. The `facts` result reads
`Removed <n> of <total> facts from <graph>`, followed by `. No match for: …` when an entry matched
nothing; it was `Removed <total> facts from longterm memory`. While an access policy is active it
reads `Processed <total> facts for <graph> (per-fact results are withheld …)` whatever matched. A client that parsed the old text
must read the new form, and one that relied on the count as confirmation now sees `0` for facts
that were never removed. `{facts, graph}` now means "remove these facts from this graph"; to clear
a graph, send `graph` alone. `{sparql, graph}`, `{sparql, facts}` and a `facts` value that is not
an array are now errors, as are an argument or fact key the tool does not take and an explicit
`null` or wrongly typed value for one it does (a fact may still carry `store_memory`'s `meta`,
which is ignored). A fact whose `object` is an IRI must carry `objectType: "uri"`, as on
`store_memory`; without it the object is matched as a plain literal and reported as no match.

**Event and decision IRIs have a new suffix**. New events are
`cqels://memory/event/<time>-<base-36 suffix>` and new decisions
`cqels://memory/decision/<time>-<base-36 suffix>`, with a random suffix; before this release
the suffix was a decimal counter. Ids already in a store keep that old form. Events that already merged in an existing persistent store
cannot be separated automatically: the merged node holds both events' subjects and objects with
nothing recording which belonged together.

**`push_stream_events` has a future-time bound** (ADR-0004). A call whose events include an `eventTime`
greater than server time plus `cqels.mcp.push.maxFutureSkewMs` is refused whole with an error result
(`event[<i>] eventTime <t> is beyond the plausibility bound ...`); the default is 86400000 (one day).
Set the JVM system property to another number of milliseconds to move the bound, or to `-1` to disable
it, for example for a replay of future-dated data. Past timestamps are not bounded, so historical replays
are unaffected, and an event with no `eventTime` is stamped with server time and passes. The same handler
serves stdio and Streamable-HTTP.

**CEP output changes, from the ADR-0004 migration ledger**. Each change below is a
class of `docs/design-notes/cep-run-identity-ledger.md` (as generated at `aed1e88b`) with its
witness, the old output and the new; `CepLedgerClassTest` pins every class.

- One match per complex event: `SEQ(?a; ?b+; ?c)` over `a b b c` answered `[a b b c]` twice and
  `[a b c]` once; it answers `[a b b c]` once (repeats consume every relevant event).
- `a+` then `b?` (overlapping predicates) over `x1 x2` answered `[x1 x2]` once; it answers it
  twice, as `{a:[x1, x2]}` and `{a:[x1], b:[x2]}`, plus `[x1]` and `[x2]`.
- Trailing optional `SEQ(?a; ?b?)` over `a` answered nothing; it answers `[a]` at `a`.
- `SEQ(?a; ?b?; ?c)` over `a c c` answered `[a c1]` and `[a c2]`; it answers `[a c1]` only, as
  `SEQ(?a; ?c)` does. Over `a b c` it answered `[a b c]`; it also answers `[a c]`.
- `SEQ(?a; NOT ?b)` within 10 over `a@0 x@20` answered nothing; it answers `[a]` at `x@20`.
- An event below the last watermark (`advanceWatermark`) was matched; it is dropped and counted.
  Without watermarks nothing changes here: an out-of-order event is still matched where it arrives,
  as before.
- A greedy argument is a selector over complex events with the same start: over `a@0 a@2`,
  `begin(a).followedByAny(b).optional().greedy()` answered `[a0 a2]`, `[a0]` and `[a2]`; it answers
  `[a0 a2]` and `[a2]`. A trailing greedy run is decided by its window close or, for `next`, by the
  adjacent non-matching event, not only at end of stream; one may end before its successor.
- `begin(a).times(1, 2).notNext(b)` over `a x x` answered `[a]` twice, at each `x`; it answers it
  once, judged on the adjacent `x`.
- `begin(a).notFollowedBy(b).followedBy(c)` over `a c1 c2` answered `[a c1]` and `[a c2]`; it
  answers `[a c1]`, as without the negation.
- `begin(a).followedBy(b).optional().notFollowedBy(...)`, `SEQ(?a; NOT ?b)` without a window,
  `next(...)` directly after `notFollowedBy(...)`, duplicate argument names and quantified negations
  were accepted; registration now throws `IllegalArgumentException` (see Changed).

**CEP argument limit.** A CEP pattern (direct `Pattern` API, `registerCepPattern`, or a CQELS-QL
`FILTER(SEQ(...))` registered with `registerCepQuery`) with more than 256 SEQ arguments is now
refused at registration with `IllegalArgumentException`. To admit larger patterns, set
`-Dcqels.cep.maxPatternArguments=<n>`; `0` (or any non-positive value) disables the limit.

**CEP argument-name limit.** A CEP pattern whose SEQ argument name is longer than 256 characters is
now refused at registration with `IllegalArgumentException`. To admit longer names, set
`-Dcqels.cep.maxArgumentNameLength=<n>`; `0` (or any non-positive value) disables the limit.

**CEP per-match event budget.** A partial match that would hold more than 10 000 events is now
dropped and counted on `cep.partials.evicted` instead of growing; the match it would have completed
is not emitted. To allow longer matches, set `-Dcqels.cep.maxEventsPerMatch=<n>`; `0` (or any
non-positive value) disables the budget.

**Compiled guards bind every SEQ argument from its own run.** A cross-event `FILTER` that reads an
earlier argument now sees that argument's first event for every pattern. In a negation-free
pattern with a quantified argument, a run's later events used to spill into the next argument's
variables: in `SEQ(?e1+; ?e3; ?e4)` with `FILTER(STR(?e3) = STR(?e4))` over HS-A, HS-B, Acc-C,
Coll-B the filter compared `?e3` with HS-B and matched; it compares the acceleration C and does not.

**`PatternMatch` carries the run.** New `getRunPositions()` (per argument, each consumed event's
`Position(arrival, timestamp)`, `arrival` = the index of the event on the subscription; positions
are ordered by arrival),
`getRun()` (the events at those positions), `getStateNames()`, and the sparse
`getRunByName()` / `getRunPositionsByName()` (only the arguments that consumed events, in argument
order); `getNamedEvents()` and
`getEvent(name)` now return each argument's first event (they were empty). A new constructor
`PatternMatch(stateNames, run, runPositions)` builds one; the existing constructors give a match
without a run. `equals`/`hashCode` are unchanged. `getEvents()` of a match the engine produced is
unmodifiable; it already was for patterns with a negation, and was a mutable list otherwise. A
whereContext guard now receives a `MatchContext` (still a `List` of the events so far) that also
reports each event's argument.

**MCP CEP rows gain `named` and `run`.** A `register_stream_query {cep: true}` buffer row now has
`named` (argument name → first event) populated and a new `run` map
(argument name → `[{event, timestamp, arrival}…]`, for the arguments that consumed events), so two
matches over equal events stay distinguishable.

**`process()` and a cold source.** `process()` now reads its input through the same bounded
serialized channel as `processWithTimeouts()`, so a synchronous cold source (`Flux.just`,
`Flux.fromIterable`) is pulled up to half the channel ahead of the matcher, and the source's own
side effects run before the matches for those events are emitted; the matches themselves are the
same. A hot source subscribed before it emits, such as an engine stream, still has each event
matched before the next is taken.

**Stream-query drains are byte-bounded**. A drain whose rows would exceed the response budget
(4 MiB by default, estimated from the rendered rows) returns the rows that fit and is marked
truncated: `recall_memory` (with `queryId`) adds a text content (`truncated: … N result row(s)
remain buffered — drain again to continue`) and structured content `"truncated": true,
"remaining": N`; the results resource adds the same two fields. Drain again for the rest — a
truncated drain loses nothing. The one-shot `query` tool (CQELS-QL or Cypher) carries the same
structured content, but its temporary query is torn down when it returns, so its note says the
remaining rows are discarded and points to `register_stream_query` for a result that large. The buffer is bounded at 16 MiB of estimated rows per
registration as well as by `bufferSize`; rows evicted for bytes are reported in `dropped` like any
other eviction. A single row larger than either budget — including a plain-query row, which used to
be delivered — is never buffered: it counts in `dropped` and is also reported as `"oversized": N`
(structured content plus a warning on `recall_memory`, a field on the results resource), only when
N > 0. The budgets bound bytes on the wire: the UTF-8 of each string after the serializer's JSON
escaping and the MCP transport's (the tool result's text is JSON-escaped again), plus a fixed
overhead per field; every row a drain returns is sized that way against the response budget, so a
row refused at drain (it grew after it was buffered) is reported the same way. A row holding a value
of a type outside strings, numbers, booleans, RDF terms and statements, and collections, maps and
arrays of these is refused rather than rendered with `toString()` (later in this release a nested
map must also be string-keyed: see the note on nested map keys below). A refusal on push now also notifies a `notify: true` registration, once until the next drain.
A CEP row's event maps are now shared, read-only objects within one drain response.

- A journal directory holding any of the impossible or damaged durable values listed under
  Fixed now refuses to open where it previously opened and silently discarded records. The
  recommended response is unchanged: restore the backend directory from a backup, or move it aside
  and start empty, accepting the loss of its persisted state. See
  `docs/PERSISTENT_STORAGE_USAGE.md`, journal integrity.
- With `failIfPrimaryUnavailable=false`, a read fault on the journal after it was opened (a
  permission or media error during recovery's replay) now stops recovery with
  `CorruptCommittedStateException` instead of skipping the replay (continuing from the checkpoint,
  if any). The message asks for the I/O fault to be fixed; restore from a backup only if the
  file itself is damaged. Code calling `FileBackedEventJournal.readFrom` or `truncateBefore`
  directly receives that exception, which is not an `IllegalStateException`, for a read fault. The
  same holds for an I/O fault while finding or reading a committed checkpoint (listing
  `checkpoints/`, a stat of a checkpoint or of its files, or iterating its `states/`): recovery now
  stops with `CorruptCommittedStateException` where it used to continue as if there were no
  checkpoint, or an older one, and `FileBackedCheckpointStore.latest()` throws that exception for
  such a fault.
- An embedder whose stream-query rows hold a nested map keyed by anything but strings (after a
  `CharSequence` key becomes a `String`; a null key is allowed) now has those rows refused, counted
  in `dropped` and `oversized` and logged. Rows from MCP clients are unaffected, their JSON
  keys being strings.
- `push_stream_events`: a call in which an `nquads` body or a fact's `subject`, `predicate`
  or `object` is a JSON object or array, or a text `eventTime` is longer than 64 characters, is now
  refused with an error naming the event; before, such a value was converted with `toString()` and
  used as text (a fact's `object` became a literal of it), and a long text `eventTime` was parsed.
  As before, one refused event refuses the whole call before any event is pushed, and an
  `objectType` other than `uri` or `literal` is refused, now without echoing the value. A client
  that sends what the tool schema declares (strings; numbers and booleans are also accepted) is
  unchanged. The numeric `eventTime` rules (only the JDK integer and floating-point classes,
  `BigInteger` and `BigDecimal`; a zero `BigDecimal` is accepted at any scale, a non-zero one with
  more than 1024 decimal places is refused, and one with a scale below -12, so at least 10^13, is
  refused as out of range) matter to applications that call the handler with their own Java objects:
  JSON numbers arrive as those classes.

### Known issues / follow-ups

- **Static enrichment is still incomplete for the pattern shapes the candidate fallback cannot
  serve.** With the subject unresolved, the fallback is asked only if the predicate or the object
  resolves, and a literal-form object never resolves — so a pattern whose only constant is
  a literal object gets nothing from the fallback, and the two lookup routes then differ: on the
  composed route it contributes nothing at all, there being no root to build a forward view from
  unless a hinted key was honoured, while on the per-arrival route, which always has a root, it
  contributes whatever that element's rooted view supplies. Coverage also depends on the order in which
  the patterns are written, and a variable predicate that the stream seed or an earlier
  pattern has already bound does resolve and is served, so a shared-predicate join is unaffected.
  Two consequences are worth knowing before relying on this cycle's new rows, and both are named
  above where they arise: on the per-arrival lookup route, a query with **no** join key answers the
  first subject's facts for every solution of the arrival rather than each solution's own,
  outside the `Istream` equality that route otherwise states; and on the composed windowed route, a
  stream solution whose join key binds a **literal** drops every row for that solution, silently
  and SPARQL-wrong. The composed-route one predates this release untouched. The lookup-route
  divergence predates it too — the two routes already disagree on a ONE-statement element — but
  this cycle widened it: a multi-statement arrival now makes the rows, and with a literal object
  even their count, depend on the order the statements arrive in.
- **`SELECT DISTINCT` with `ORDER BY … LIMIT` can emit one projection more than once per arrival**
  — the fused top-K snapshots after every row rather than every arrival, so a better-ranked
  row carrying the same projection replaces the representative and re-emits it. This cycle made the
  shape reachable on the plain per-arrival route — it previously needed a `BIND` — but the fix is an
  evaluation-boundary change of the same kind as the per-arrival aggregate fold, so it is deferred
  rather than attempted here.
- **A time-scoped `SELECT DISTINCT` on the per-arrival routes keeps its keys for as long as event
  time stands still**. Under a `RANGE` or `SLIDE` window the scope evicts a key only when
  the high-water mark of the arrivals' event times has moved more than the window's extent past that
  key's newest occurrence. Every push at the same `eventTime` therefore enters the scope and nothing
  leaves it, neither at admission nor at the per-arrival boundary, so a stream whose events all
  carry one timestamp grows the scope by every distinct key it produces, and the first arrival that
  moves the high-water mark past the window then evicts that backlog at once. A `TRIPLES n` scope
  (and `[NOW]`) is bounded by count and unaffected. Pre-existing; found by a security review.
- **CypherQL `RETURN DISTINCT` parses and is then ignored**: duplicate rows are emitted.
- **CypherQL `RANGE` / `SLIDE` queries rank correctly but stay silent on a live stream**, because
  those window operators collect their source and fire on completion; of the three window forms
  only `TRIPLES` closes continuously. Pre-existing, pinned by a characterization test, and
  described above where the continuous `ORDER BY` / `LIMIT` work names it.
- **A stream-only `OPTIONAL` whose probe subject is an unprojected `BIND` target null-fills**
  instead of correlating; the `FILTER NOT EXISTS` twin of that shape is refused at
  registration rather than answering wrongly.
- **Per-arrival cost, now that one arrival can carry many solutions.** The lookup route evaluates
  each arrival's stream BGP twice, once in the admission gate and once building rows. On
  the composed route the static work bound is scoped to the whole arrival only when the static BGP
  shares **no** variable with the stream pattern; one that shares any — the predicate position
  counts, and a declared `operators.lookup.joinKeyVariable` neither makes a BGP joinable nor stops it being
  so — keeps a fresh budget per row, so one arrival's rows can each spend the whole
  allowance, and overflow there still ends the query rather than dropping the arrival. And several
  per-arrival quantities have no bound of their own: the element's forward-view cache, the
  candidate memo's extent, one enrichment traversal per distinct root of an element, and the
  binding width that decides what the live-set ceiling costs in bytes — none of these is a
  leak across arrivals; each is per-arrival work that a large element multiplies.
- **A variable repeated inside a single triple pattern** (`{ ?s ?s ?o }`) still binds
  inconsistently. The per-arrival routes now give the same answer as the windowed routes
  rather than a different wrong one, and that parity — not a fix — is what this cycle pins.
- **Multi-stream / multi-window: alpha.20's notes below still describe the capability picture, and
  this cycle does not revise it.** What it did change in that area is recorded under `### Changed`:
  the removed `IntervalJoinOperator.processingTimeJoin` entry point, and named windows now refused
  in the `CompiledQuery` constructor rather than in `execute()` — immediate for a direct
  `new CompiledQuery(...)` caller, which previously carried no such guard. One correction to that
  older entry while pointing at it: it calls `Kind.WINDOWED` a declared-but-unimplemented second
  kind that throws on dispatch, but `TemporalJoinSpec.Kind` has only ever declared `INTERVAL` —
  what existed was a dispatch guard for a value the JSON parser rejects first, and this cycle
  removed it as unreachable.
- **A `notify: true` result signal is written on the thread that delivers query results**.
  The write to the owning session's event stream is synchronous, so a client that opens its event
  stream and stops reading can stall result delivery for every query sharing that thread.
  Pre-existing: the broadcast this release replaced held the same thread until every session's
  write had finished, and narrowing the signal to the owning session left one such write.
- **No figure separates recovered events that were delivered from those correctly skipped**.
  After a restart, `persistence.appliedPosition` on `cqels://engine/status` advances through
  recovered records that were delivered or correctly skipped alike, and the engine's replay log
  counts every replayed record; an operator cannot read how many recovered events reached a stream's subscribers
  without registering a query.
- **String literals in CQELS-QL expressions do not decode the SPARQL escapes `\t \n \r \b \f`.**
  Only `\"` and `\'` are unescaped, so a literal such as `"a\nb"` compares as written, with a
  backslash and an `n`, not a newline; a `GROUP_CONCAT` separator is decoded the same way.
  Pre-existing.


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
  <version>2.0.0-alpha.21</version>
</dependency>
```

## Verifying this release

The signed manifest and its signature are served from the artifact repository;
the public key is served from a different origin, so verifying with it is not
circular. Take the key from the commit pinned below.

```bash
set -euo pipefail
BASE=https://maven.cqels.org/releases/supply-chain/2.0.0-alpha.21

curl -fsSLO https://raw.githubusercontent.com/cqels/CQELS4J/2320254702ddeea1b0abd96f8e89ac6b9fc32eec/cosign.pub
echo "36dd8daa9988f23eb40c4f3550fa7bdfa3796e5e58cce8d23b9cc6a99f47f30b  cosign.pub" | sha256sum --check --strict - || exit 1

curl -fsSLO "$BASE/SHA256SUMS" -O "$BASE/SHA256SUMS.bundle"
cosign verify-blob --key cosign.pub --bundle SHA256SUMS.bundle \
  --new-bundle-format=false --insecure-ignore-tlog SHA256SUMS
```

The URL is pinned to a commit rather than to `master`, so it keeps working
after a key rotation. `--insecure-ignore-tlog` is expected: these signatures
carry no transparency-log entry, which is precisely why the key's origin
matters. Full walkthrough: https://github.com/cqels/CQELS4J/blob/master/SUPPLY_CHAIN.md
