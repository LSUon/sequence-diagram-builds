# The Release Repository for the Sequence Diagram Generator for a university FYP project at the University of Newcastle, Australia
A simple public repository with the build outputs from currently private sequence diagram tool
# Tool's Readme below 
# Micronaut Sequence Diagram Generator

A CLI application that generates UML sequence diagrams from OpenTelemetry trace data (JSONL format) and source code. It can also generate OpenTelemetry instrumentation suggestions for Java codebases using large language models.

## Prerequisites

- Java 21 or higher
- Gradle 8.x (wrapper included)

## Installation

```bash
# Build the project
./gradlew build

# Or run with Shadow jar for a standalone executable
./gradlew shadowJar

# Or run the released jar
java -jar <jar-name>.jar -cli options here
```

## Usage

### Generate Sequence Diagrams

```bash
./gradlew run --args="-t <traces.jsonl> -var <codebase-directory>"
```

**Example:**
```bash
./gradlew run --args="-t ../otel-logs/traces.jsonl -var ../myapp/src/main/java -o ./output -v"
```

### Trace output: fingerprint groups by default

Trace mode groups structurally identical traces instead of merging everything into one
diagram. A 2,000-trace input normally collapses to roughly 20 scenario groups.

```bash
# Default: one diagram per fingerprint group (all member traces included)
./gradlew run --args="-t traces.jsonl --no-image"

# Old behaviour: one merged trace-diagram
./gradlew run --args="-t traces.jsonl --combine"

# Exact one-file-per-trace output
./gradlew run --args="-t traces.jsonl --group-by=trace"

# Bound a group and split at trace boundaries; the second value guards total arrows
./gradlew run --args="-t traces.jsonl --max-traces-per-group 500 --max-interactions-per-diagram 2000"

# Inspect or reuse the deterministic assignment
./gradlew run --args="-t traces.jsonl --write-groups traces.groups.json --no-image"
./gradlew run --args="-t traces.jsonl --groups-file traces.groups.json --no-image"
```

A fingerprint contains the normalized root operation and the ordered service/kind/class-or-method
tree. Span IDs, timings, attributes and status-only differences are excluded, so an error variant
stays in the same group; a structural error branch changes the tree and creates another group.

Group keys look like `group-post-checkout-a1b2c3d4`. A group over the trace cap is split by trace
boundary into `-001`, `-002`, and so on. `--groups-file` is validated against the current trace
fingerprint; use `--allow-stale-groups` only when the plan is intentionally reused.

Image rendering is bounded-parallel by default (`min(4, cores)` for SVG, `min(2, cores)` for PNG).
Loading, fingerprinting, mapping and refinement remain sequential. Use `--max-parallel-renders 1`
for fully sequential output or `--no-image` for PlantUML only. A render call that exceeds
`render.timeout-seconds` (default `600`) is abandoned and reported as an image failure; the
deterministic `.puml` file is still written. `--timings` logs each phase, queue wait, the maximum
in-flight renders, fallback and image-failure counts, heap use and process CPU time.

### Generate Instrumentation Suggestions

```bash
./gradlew run --args="-i -var <codebase-directory>"
```

For deterministic, non-AI instrumentation suitable for CI, use `--no-ai` and write
 the OTEL-compatible result to a file:

```bash
./gradlew run --args="--instrument --no-ai --codebase <codebase-directory> --instrumentation-output artifacts/instrumentation.otel"
```

The non-AI mode does not call an external provider. It returns every parsed class
that has methods, so it is an inventory/baseline rather than a semantic selection.

#### Durable, Resumable AI Instrumentation

For large codebases, use durable batching so work is checkpointed and can be resumed:

```bash
# Start a run and process at most 20 batches now
./gradlew run --args="--instrument --durable-instrument -var <codebase-directory> --max-batches-per-run 20"

# Resume the run (use the run id printed by the first command)
./gradlew run --args="--instrument --resume-run .sequencegen/ai-runs/<run-id> --max-batches-per-run 20"

# Merge completed results without calling AI again
./gradlew run --args="--instrument --resume-run .sequencegen/ai-runs/<run-id> --finalize-only"

# A batch that failed (timeout, connection error, unusable model output) is retried by the
# next resume, up to --max-batch-attempts. Reset exhausted batches to force another round:
./gradlew run --args="--instrument --resume-run .sequencegen/ai-runs/<run-id> --retry-failed-batches"
```

Durable runs store manifests, batch prompts, validated results, failures, and final artifacts under `.sequencegen/ai-runs/<run-id>/`.

A failed batch is not the end of the run: resume re-sends only the batches without a result,
records how many attempts each one has used, and stops retrying after `--max-batch-attempts`
(default 3). Timeouts are never retried inside a single invocation; raise
`AI_REQUEST_TIMEOUT_SECONDS` and resume instead.

You may change `AI_DEFAULT_MODEL` between invocations: the model does not shape batch
planning, so resume adopts the new model (completed batches keep their original results).
What still invalidates a resume is anything that changes the planned batches: source edits,
`--classpath`, token budgets, or `--batch-max-items`.

## AI Diagram Refinement

`--ai-refine` runs the trace diagram mapper, then lets a model improve the result without being
able to invent anything. It requires `--traces`.

```bash
# Default: native tool calling (V2)
./gradlew run --args="-t traces.jsonl --ai-refine"

# Portable: JSON envelope in plain text, for endpoints without reliable tool calling
./gradlew run --args="-t traces.jsonl --ai-refine --refiner v3"

# Add calls that exist in the source but were never observed in the trace
./gradlew run --args="-t traces.jsonl --ai-refine --refiner v4 -var ../myapp/src/main/java"

# Let the model suppress noisy interactions (evidence is kept in the audit)
./gradlew run --args="-t traces.jsonl --ai-refine --allow-ai-hide"

# Reuse one chunk's edits across structurally identical traces
./gradlew run --args="-t traces.jsonl --ai-refine --ai-refine-cluster-traces"
```

`--ai-refine` follows the active trace plan. By default it stacks every group into one baseline,
runs a single refinement (one shared participant pass, one shared chunk/chunk budget), then slices
the refined stack back into per-group artefacts: `group-*.puml` plus `group-*.ai.puml` (or
`.ai.partial.puml` when that group did not finish). Because the run is shared, participant labels
are consistent across groups and a failed chunk only makes its own group partial. With `--combine`
the run keeps the old `trace-diagram.puml` / `trace-diagram.ai.puml` pair.
`--refiner-compare` currently requires `--combine`.

A grouped run has one run id under `.sequencegen/ai-refine/`; no per-group child runs are created.
`groups.json` records the export groups and `layout.json` records which chunk contains which group,
so a resume can reconstruct the same plan and finish the remaining chunks.

### Watching a run

Progress is written to standard output while the model works: the run plan, one line per pass, and
one line per tool call the model makes, so a long run is never silent between the first and last
line.

```
INFO  ... - Refine run refine-20261020-143000: 7 chunk(s) plus the participant pass
INFO  ... - [global] pass started (4 turn(s) max, 0 element(s))
INFO  ... - [global] set_participant_label p3 -> label=orders
INFO  ... - [global] set_participant_label p4 -> label=OrderRepository
INFO  ... - [global] finish: named the participants from the loaded source
INFO  ... - [global] pass done: applied=4, rejected=0, turns=2
INFO  ... - [chunk 1/7] pass started (8 turn(s) max, 12 element(s))
INFO  ... - [chunk 1/7] view_source p3#listOrders -> ok
INFO  ... - [chunk 1/7] set_interaction_label e14 -> label=GET /orders
INFO  ... - [chunk 1/7] group_interactions e14,e15,e16 -> group Fetch orders as g3
WARN  ... - [chunk 1/7] group_interactions e20,e27 refused (NOT_CONTIGUOUS): handles are not contiguous siblings
INFO  ... - [chunk 1/7] pass done: applied=3, rejected=1, turns=4
INFO  ... - [chunk 2/7] reused from checkpoint
```

Refusals appear as they happen with their reason code, so a model that keeps asking for something it
cannot have is visible immediately, not only at the end of the run. `--verbose` adds the tools
offered for each pass, each call's arguments, and the turn counter; the default lines are not
repeated. The applied and refused operations are also persisted in `final/audit.md` for a run that
has finished.

Only tool names, handles, participant aliases and model-authored labels are printed: catalogue
labels, the rendered PlantUML body, the feedback diff and prompt text never reach the console, just
as they never reach the model.

### What the model is allowed to change

| Allowed | Not allowed |
| --- | --- |
| Relabel a participant or a message | Create or delete a participant |
| Group a contiguous call/return-closed range into `group`/`loop`/`par` | Create an arrow (except V4's allowlist) |
| Suppress a range into a fixed-format marker | Change a message's endpoints, type, or order |
| Remove the synthetic client trigger arrows | Add free-form notes |

The model never writes PlantUML. It refers to handles minted from the trace diagram, and every
operation is applied to a copy and checked against the structural invariant before it is kept, so
a reordering, a changed endpoint, or an undeclared addition is rejected. `par` requires overlapping
observed timestamps and `loop` requires an observed repeated sequence, so a fragment cannot assert
concurrency or repetition that the trace does not show.

Every layer the model can see is redacted — catalogue labels and core tokens, group names, the
rendered PlantUML body, the feedback diff, the participant table, and the persisted catalogue. Raw
SQL, URLs, query strings, hostnames, and credentials never reach the model, even though the exported
diagram keeps the original text.

### Resumable runs

Large diagrams are split at trace boundaries, and each chunk is checkpointed:

```bash
# Start, processing at most 20 new chunks now
./gradlew run --args="-t traces.jsonl --ai-refine --ai-refine-max-chunks 20"

# Resume the run (use the id printed by the first command)
./gradlew run --args="-t traces.jsonl --ai-refine --resume-refine-run .sequencegen/ai-refine/<run-id> --ai-refine-max-chunks 20"

# Assemble the result without calling the model again
./gradlew run --args="-t traces.jsonl --ai-refine --resume-refine-run .sequencegen/ai-refine/<run-id> --finalize-refine-only"
```

Run state lives under `.sequencegen/ai-refine/<run-id>/` and includes `groups.json` and
`layout.json` for grouped runs. A pass is only checkpointed when it
completed: a pass that ends without finishing (a transport error, unusable model output, or an
exhausted rejection budget) is rolled back and recorded as a retryable failure, bounded by
`--max-batch-attempts`. The rejection budget (`--ai-refine-max-rejections`) applies per pass — one
chunk, or the global participant pass — so refusals in one pass never make a later chunk fail. A
partial run writes `final/refined.partial.puml` and exports `trace-diagram.ai.partial.puml` rather
than the normal artefact unless `--allow-partial-ai-output` is given. `rejected.json` and
`final/audit.md` record every refused operation and the reason.

Transient transport failures (a dropped connection, a refused connect, a retryable HTTP status) are
retried with exponential backoff using the shared `ai.retry.*` settings, so one dead connection does
not cost a whole pass. Read timeouts are deliberately *not* retried: the server is usually still
generating, so resending would duplicate the work — raise `ai.request-timeout-seconds` instead.

A run interrupted by Ctrl-C, a kill or an out-of-memory error leaves its `run.lock` behind. The
next resume detects that the recorded process is no longer running, logs a warning and reclaims
the lock; a lock held by a live process is still refused.

If a run still fails, `failures/<chunk>.json` names the cause: the SDK exception class, its whole
cause chain, and the HTTP status and error body when the provider returned one. This matters because
the SDK's own message for a transport failure is usually just `Request failed`.

### Comparing the variants

```bash
./gradlew run --args="-t traces.jsonl --ai-refine --refiner-compare"
```

Runs V1, V2, V3 (and V4 when a codebase is loaded) on the same diagram with the same model and
writes a markdown report with per-variant operation counts and the resulting PlantUML.

## CLI Options

| Option | Description |
|---|---|
| `-t, --traces <path>` | Path to traces JSONL file (required for diagram mode) |
| `-var, --codebase <path>` | Path to Java source codebase directory (optional, enables class-level resolution) |
| `--classpath <path>` | Dependency classpath for source-code and instrumentation analysis (platform path separator) |
| `--source-only` | Force source-code-based diagram generation (ignore traces even if provided) |
| `--max-depth <n>` | Maximum source call depth to traverse (default: 20) |
| `--external-prefixes <prefixes>` | Comma-separated external package or label prefixes to suppress in source notes; defaults to Java/Jakarta standard libraries plus common framework noise (providing this replaces the default set) |
| `--combine` | Merge everything into one diagram. In trace mode this restores the old `trace-diagram` output. |
| `--group-by <mode>` | Trace grouping: `fingerprint` (default) or `trace`. Cannot be combined with `--combine`. |
| `--max-traces-per-group <n>` | Deterministic trace-boundary shard size (default: 500; `0` = unlimited) |
| `--max-interactions-per-diagram <n>` | Second guard: split a group at trace boundaries when it exceeds this many interactions (default: 100000; `0` = off). Applies to grouped and combined AI-refinement baselines as well. |
| `--groups-file <path>` | Import a group plan and validate it against the current trace index |
| `--write-groups <path>` | Write the active group plan as JSON |
| `--allow-stale-groups` | Accept a `--groups-file` plan whose trace fingerprint differs |
| `--no-image` | Write `.puml` files only |
| `--max-parallel-renders <n>` | Concurrent image renders (`0` = format-dependent default; `1` = sequential) |
| `--timings` | Log index/plan/export timings, queue wait, maximum in-flight renders, fallbacks, image failures, heap and CPU time |
| `--no-external-notes` | In source mode, do not render external (out-of-codebase) calls as notes |
| `--entry` | In source mode, which methods are diagram entry points: `all` (default) or `controllers` (controller/resource classes; mapping-annotated methods are preferred) |
| `--format` | Image output format: `png` (default) or `svg`. Very large diagrams fall back to `svg` automatically. |
| `-o, --output <path>` | Path to output directory (default: current directory) |
| `-v, --verbose` | Enable verbose output; with `--ai-refine` it also adds per-call tool detail |
| `-i, --instrument` | Switch to instrumentation suggestion mode (no trace input needed) |
| `--no-ai` | With `--instrument`: skip the AI call and output all classes that have methods in the otel format |
| `--instrumentation-output <path>` | With `--instrument`: write the OTEL-compatible result to a file |
| `--durable-instrument` | With `--instrument`: use durable, resumable batching instead of in-memory chunking |
| `--ai-run-root <path>` | Root directory for durable AI run state (default: `.sequencegen/ai-runs`) |
| `--resume-run <path>` | Resume a previous durable AI run by directory name or path |
| `--restart-ai-run` | Start a new durable AI run even if a matching prior run exists |
| `--max-batches-per-run <n>` | Maximum batches to process in this invocation (0 = unlimited) |
| `--batch-max-input-tokens <n>` | Maximum input tokens per AI batch (default: 100000) |
| `--batch-reserved-output-tokens <n>` | Tokens reserved for the AI response per batch (default: 20000) |
| `--batch-max-items <n>` | Maximum candidates per AI batch (default: 50) |
| `--allow-partial-ai-output` | Allow a partial durable run to produce output |
| `--retry-failed-batches` | With `--resume-run`: clear persisted failures so failed batches are attempted again |
| `--max-batch-attempts <n>` | Attempts per batch across resumes before it is terminal (default: 3) |
| `--finalize-only` | With `--instrument --resume-run`: merge completed batch results without calling AI |
| `--suppress-no-internal` | Suppress external DB/HTTP/messaging calls not from resolved internal spans |
| `-n, --trace-name <filter>` | Filter traces by span name (substring match) |
| `-svc, --service-filter <filter>` | Filter traces by service name (substring match) |
| `--duration-min <ms>` | Minimum span duration in milliseconds |
| `--duration-max <ms>` | Maximum span duration in milliseconds |

### Exit codes

| Code | Meaning |
|------|---------|
| `0` | Every requested artifact was written |
| `1` | The run failed: invalid options, unreadable traces or `--groups-file`, a diagram or image that could not be exported, or a failed AI run |
| `2` | The run finished but AI output is partial (`*.ai.partial` exported, or an incomplete durable instrumentation plan); resume it to finish |

## Published JAR distribution

The supported standalone distribution for another repository's CI pipeline is the
Shadow JAR. Release tags named `v<version>` are built and uploaded as GitHub releases in this repository
[LSUon/sequence-diagram-builds](https://github.com/LSUon/sequence-diagram-builds/releases/latest).
Every release includes a SHA-256 checksum.

To build the same artifact locally:

```bash
./gradlew test shadowJar -PreleaseVersion=0.1.0
sha256sum build/libs/sequence-diagram-generator-0.1.0-all.jar
```

A consumer pipeline should download and pin an exact release, verify the checksum,
and run it with Java 21. For a Gradle consumer, first build the project and create a
platform-separated dependency classpath, then invoke:

```bash
java -jar sequence-diagram-generator-0.1.0-all.jar \
  --instrument --no-ai \
  --codebase src/main/java \
  --classpath "$(cat build/instrumentation-classpath.txt)" \
  --instrumentation-output artifacts/instrumentation.otel
```

The generated file should be published as a Bitbucket pipeline artifact. The
consumer pipeline should initially treat changes to the generated list as advisory,
while failing on missing inputs, parser failures, or invalid output.

## AI Configuration

The AI client is created lazily and is only validated when `--instrument` is used. To avoid committing secrets, configure the AI via environment variables (preferred):

```bash
export AI_API_KEY=your-api-key
export AI_BASE_URL=http://localhost:8080
export AI_DEFAULT_MODEL=gpt-5.5-luna
export AI_MAX_PROMPT_TOKENS=50000
```

These values can also be set in `src/main/resources/application.properties` or a local `application-local.properties` file.

## Trace Filtering

Filter traces before diagram generation:

```bash
# Filter by service name
./gradlew run --args="-t traces.json -svc my-service"

# Filter by span name
./gradlew run --args="-t traces.json -n getUser"

# Filter by duration (only spans between 10ms and 500ms)
./gradlew run --args="-t traces.json --duration-min 10 --duration-max 500"
```

## Output

Trace mode writes one `.puml` and one image per active group, for example
`group-post-checkout-a1b2c3d4.puml` and `.png` (or `.svg`). `--combine` writes the single
legacy `trace-diagram.puml`. `--no-image` skips image files. Source mode keeps its existing
per-diagram names.

A failed image render does not delete the `.puml`; a group is reported as failed only when its
source could not be written or when every requested image backend failed. Diagram statistics are
logged after generation:

```
Diagram stats: 12 participants, 24 interactions, 3 groups, 5 traces, 342ms processing time
```

## Troubleshooting

### Out of Memory Errors

For large trace files, increase the JVM heap:

```bash
./gradlew run -Dorg.gradle.jvmargs="-Xmx8g" --args="-t large-traces.jsonl"
```

### Test Heap Size

The three optional multi-gigabyte trace fixtures are ignored by git. If they are present locally, run their benchmark-style test explicitly:

```bash
./gradlew test -PtestHeapSize=16g -Dtest.largeTraces.enabled=true
```

Normal tests use the tracked small fixtures and do not require this flag.

### AI Connection Issues

- Verify `ai.base-url` points to a running OpenAI-compatible API
- Check `AI_API_KEY` (or `ai.api-key` in local configuration) is set correctly
- Enable verbose logging: `-v` flag

