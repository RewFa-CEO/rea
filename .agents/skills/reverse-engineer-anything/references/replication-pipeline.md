# Replication pipeline: from observed behavior to a working reimplementation

Use this workflow when the user wants to reproduce a feature they are allowed to
analyze: their own project, an open-source project, or a target they have
explicit authorization for. It converts REA Evidence into a written
implementation spec, an independent reimplementation, and a behavior
comparison. It does not copy code, text, styling, or assets from the target.

## Boundaries before any analysis

State the boundary in the first reply and keep to it. Never analyze a target
that forbids reverse engineering, and never work against obfuscation or
encryption. Do not touch protected material: license-protected internals,
post-login private data, private APIs of third parties, or a commercial
product's complete features and resources. Reimplementation means understanding
behavior and writing your own version; it is not copying.

If the target description is ambiguous, ask before opening a session. A narrow
target yields a readable result; a broad one yields an unfalsifiable one.

## Stage 0 — Route the target and check this machine

Apply [Route the target first](#route-the-target-first) in the skill body to
pick the analysis route and its tools. Then confirm the host can actually run
that route: run `rea doctor` and read the failing checks. A missing analysis
engine is a host prerequisite, not an analysis result, so report it as a
prerequisite gap with the remediation the check names. Do not install engines
silently, and do not begin analysis whose route cannot execute.

Write down the route, the tools it needs, and which of those the host already
has. Anything missing is either fixed first or explicitly excluded from the
scope.

## Stage 1 — Analyze with evidence

Use the normal REA workflow for the chosen route. Every conclusion in the
specification must be traceable to Evidence. Use `record_unknown` for anything
you could not establish, and keep three states apart throughout: observed,
inferred, and unknown. If the evidence for a behavior is thin, say so in the
specification rather than closing the gap with a guess.

The specification inherits the analysis coverage. Do not claim a behavior the
analysis never observed.

## Stage 2 — Write the implementation spec

The spec is a plain-language contract, not a transcript of the analysis. Write
it so an implementer who never saw the target can build against it.

```text
Feature: <name in one line>
Input:    <what the user or caller supplies, and its type>
Process:  <ordered steps, each one observable>
Output:   <what the target produces, and its shape>
Edges:    <empty input, no match, over-long input, after-delete refresh, ...>
Parity:   <behaviors that must match the original exactly>
Unknown:  <behaviors not established by the analysis>
Evidence: <one line per claim, pointing at its evidence id>
```

Rules for the spec:

- One claim per line, one evidence reference per line. A claim without
  evidence is marked as inference or unknown.
- "Parity" lists only behaviors a test can distinguish. Do not list
  implementation details that cannot be observed from the outside.
- "Edges" is where the reimplementation usually fails. Include the empty
  result, the case-insensitive match, the character that must be trimmed, and
  whether a list refreshes after a delete.
- Keep the original's _behavior_, not its _structure_. "Results are ordered by
  creation time descending" is behavior; "it calls `sortBy()` in the store" is
  structure and must not appear.

## Stage 3 — Build your own version

Now write code, with normal coding tools, against the spec. The spec is the
contract; the original's source is not the input.

- Do not copy the target's code, comments, strings, styling, or assets.
  Reimplement the described behavior in your own stack and your own idiom.
- State the file layout and the test cases before writing implementation code,
  so the user can object to the shape of the solution first.
- Give each test the input, the expected output, and the spec line it covers.
  A test that does not map to a spec line is scope you invented.
- Run the tests and report which passed and which did not. Do not describe a
  version as working before the run.

## Stage 4 — Compare behavior against the original

A version that runs is not a version that matches. Drive the original and the
reimplementation with the same inputs and compare the outputs.

- Use the same input set for both, including the edge cases from the spec.
- Compare observable results only: output content, ordering, case handling,
  empty-result rendering, and what happens after a mutation.
- Report the differences as a table with one row per input, and mark each row
  as matching or differing. "Basically the same" is not a result.
- Sort the differences by impact, then fix them one at a time. Each fix gets
  its own test run. A single combined pass hides which change caused a
  regression.
- When the original cannot be observed, say that the comparison is partial and
  name the behaviors that remain unverified.

## Stage 5 — Check the result before it ships

PentAGI-style scanning is out of scope for this skill: it needs Docker, and
REA does not run containers. Use a self-run checklist instead and record the
result per item.

- Authentication: can a session be established without the intended
  credential, and can one be forged or replayed?
- Input handling: is every external input validated, length-bounded, and
  escaped at the point of use?
- Authorization: is every access checked against the caller's own scope, not
  only the resource's existence?
- File and upload handling: are paths confined, uploads type-checked and
  size-bounded, and temporary files removed?
- Secrets: are credentials, tokens, and keys kept out of source, logs, and
  client-visible responses?
- Dependency exposure: which third-party packages and versions ship, and which
  have known advisories?
- Configuration: are defaults safe on a fresh install, and do debug switches
  default to off?

For a system that will face the public internet, run a dedicated authorized
scanner against it separately. Treat a scanner's report as findings to
investigate, never as proof that the system is secure: dependency advisories,
weak credentials, and permission configuration need separate review, and a
scanner does not find business-logic flaws.

## Outputs

Keep the artifacts in one working directory so a later run can be compared
against an earlier one:

```text
<workspace>/
  01-target/          the authorized target, read-only
  02-analysis/        exported Evidence bundles and the analysis notes
  03-spec/            the implementation spec and its test cases
  04-rebuild/         your own implementation
  05-parity/          the behavior comparison table and its results
  06-security/        the pre-ship checklist and its findings
```

Never place the target's code in `04-rebuild/`. When a phase is not run, leave
its directory absent rather than filling it with placeholders; an empty
directory implies a result that does not exist.
