# Proposal: Evidence of Finding in Scan Output

- **Status:** Draft
- **Related issues:**
  - [kubescape/kubescape#1563 — Evidence of finding in output](https://github.com/kubescape/kubescape/issues/1563) (open)
  - [kubescape/kubescape#1737 — C-0012 rule support detailed object name](https://github.com/kubescape/kubescape/issues/1737) (open)
  - [kubescape/kubescape#1714 — Unexpected "Action Required" with 0 Failed Resources and no Evidence](https://github.com/kubescape/kubescape/issues/1714) (closed; cross-referenced in #1563)
- **Scope:** [`kubescape`](https://github.com/kubescape/kubescape) CLI, [`opa-utils`](https://github.com/kubescape/opa-utils), [`regolibrary`](https://github.com/kubescape/regolibrary) (phased), [`kubevuln`](https://github.com/kubescape/kubevuln) (phased)
- **Author:** Yugal Sadhwani (<yashsadhwani544@gmail.com>)

## 1. Summary

Add a first-class **Evidence** capability to Kubescape scan output so that every
failed control on every resource can be validated by a human (or auditor)
without re-reading the underlying manifest. Evidence means: the exact JSON-path
inside the resource that triggered the rule, the value at that path, and — when
the scan source is known — a pointer back to the source file (line number for
raw YAML, file name for Helm/Kustomize, `kubectl` command for live clusters).

The goal is not "more verbose output." The goal is that a user looking at any
single finding can answer two questions in under ten seconds:

1. **Why does Kubescape think this is failing?** (the matched field + value)
2. **Where do I go to fix it?** (file + line, or kubectl command)

## 2. Motivation

[Issue #1563](https://github.com/kubescape/kubescape/issues/1563) and related
reports ([#1737](https://github.com/kubescape/kubescape/issues/1737),
[#1714](https://github.com/kubescape/kubescape/issues/1714)) describe a
recurring blocker for adopting Kubescape in compliance and audit workflows:

- Auditors require **direct evidence** for each reported finding.
- False positives (e.g. C-0012 firing on Deployments that only use
  `valueFrom.secretKeyRef`, even though
  [`rule-credentials-in-env-var`](https://github.com/kubescape/regolibrary/blob/main/rules/rule-credentials-in-env-var/raw.rego)
  explicitly skips that case) are impossible to triage without seeing
  the matched env-var name and value.
- `--verbose` today expands the report to include passing resources, but
  for failing resources it still only shows *that* a control failed on a
  resource, not *which field* caused the failure. Users fall back to
  re-reading every manifest by hand.

The data needed to fix this **already exists** in the rule response — it is
simply not surfaced in the human-facing output.

## 3. Current state (what we already have)

Every rego rule emits an `AssistedRemediation` block on each failure
([`opa-utils/reporthandling/datastructuresv1.go` lines 42–48](https://github.com/kubescape/opa-utils/blob/master/reporthandling/datastructuresv1.go#L42-L48)):

```go
type AssistedRemediation struct {
    FailedPaths []string            // JSON-paths in the manifest that triggered the failure
    ReviewPaths []string            // JSON-paths a human should review
    DeletePaths []string            // JSON-paths to delete to remediate
    FixPaths    []armotypes.FixPath // JSON-paths + values to add to fix
    FixCommand  string              // optional literal remediation command
}
```

Survey of the rule library
([`regolibrary/rules/`](https://github.com/kubescape/regolibrary/tree/main/rules),
**275 classified rules** from the original survey; `master` has ~278
`rules/*/raw.rego` files as of 2026-07-21):

| `failedPaths` classification | Rule count |
|---|---|
| Always emits real paths (e.g. `spec.containers[0].env[2].name`) | **149** |
| Mixed (some emissions real, some `[]`) | **17** |
| Always emits `failedPaths: []` placeholder | **102** |
| Does not emit `failedPaths` at all | **7** |

Of the 109 classified rules with no real `failedPaths`, **35** still emit real
`reviewPaths`, `fixPaths`, or `deletePaths`. So the actual coverage picture is:

- **201 / 275 classified rules** emit at least one real evidence path somewhere.
- **74 / 275 classified rules** emit no path-level evidence at all — these are
  predominantly multi-resource cluster-config and RBAC controls where the
  triggering condition is a relationship across several objects.

The full K8s object that triggered the failure is also already attached to the
response as `AlertObject.K8SApiObjects`
([`opa-utils/reporthandling/datastructuresv1.go` line 50](https://github.com/kubescape/opa-utils/blob/master/reporthandling/datastructuresv1.go#L50)),
so resolving a path to a value requires no extra I/O.

What is missing is (a) a renderer that displays this data in the pretty
printer, (b) a resolver that walks the captured object using the path, (c) a
mapping back to the source file/line where applicable, and (d) a redaction
policy so evidence reports are safe to share.

## 3a. Prior art and dependencies

The evidence work is not starting from zero. Several merged and in-flight
changes already touch the data model, source-location capture, and path
semantics this proposal builds on. Failing to depend on them would create
parallel implementations that drift.

**Merged — evidence data model:**

- [kubescape #1402](https://github.com/kubescape/kubescape/pull/1402) +
  [opa-utils #139](https://github.com/kubescape/opa-utils/pull/139) — split
  `failedPath` into `deletePaths` and `reviewPaths`. Established the
  precedent of progressively enriching path semantics on `AssistedRemediation`.
- [opa-utils #168](https://github.com/kubescape/opa-utils/pull/168) — added
  `Source.HelmTemplateFile`, `HelmValuesPaths`, `HelmTemplateLine`. The
  Helm half of source-location capture is already in the schema.

**Merged — source-location & fix flow:**

- [kubescape #2083](https://github.com/kubescape/kubescape/pull/2083) — Helm
  provenance extractor (`core/cautils/helmprovenance`) + Helm-aware fix flow
  that prints `.Values.*` candidates instead of attempting to patch
  templates. Codifies the "do not fake Helm template lines" rule that this
  proposal inherits.
- [kubescape #2281](https://github.com/kubescape/kubescape/pull/2281) —
  segment-aware YAML path coverage helpers (`yamlPathCovers`,
  `plannedPathsFromExpressions`) in the fixhandler. The evidence path
  resolver must reuse the same segmenter; phase 1 lifts it into a shared
  package.

**Reverted — historical precedent:**

- [kubescape #1215](https://github.com/kubescape/kubescape/pull/1215)
  ("feat: Add the debugging ability for scanning Helm chart") was the prior
  attempt at mapping rendered-YAML lines back to Helm template lines, using
  MappingNode types and a per-line yqlib producer.
  [#1628](https://github.com/kubescape/kubescape/pull/1628) followed up
  with a crash fix in that mapping code.
  The whole approach was abandoned and its dead remnants cleaned up in
  [#1995](https://github.com/kubescape/kubescape/pull/1995). This is
  direct codebase history backing the no-line-faking constraint for
  Helm/Kustomize.

**Merged — correctness fix:**

- [kubescape #2311](https://github.com/kubescape/kubescape/pull/2311) —
  merged on 2026-05-26 and fixes a pre-seed bug where
  `ResourceAssociatedRule.Paths` for
  cluster-scoped resources (e.g. ClusterRoles) get clobbered across
  namespace iterations in large-cluster mode. The evidence implementation
  should build on that fix; no extra prerequisite is needed for this issue.

**Printer / `--verbose` precedent:**

- [kubescape #1320](https://github.com/kubescape/kubescape/pull/1320) — the
  "New output" PR that introduced the image-scan pretty printer with
  `--verbose` for detailed CVE view. Phase 6 (image-scan evidence) extends
  this printer.
- [kubescape #1932](https://github.com/kubescape/kubescape/pull/1932) —
  `--verbose` on `scan-images`. CLI-flag precedent for new verbose-style
  flags.

## 4. Goals and non-goals

### Goals

- Show, for every failed (resource, control) pair, the JSON-path(s) that
  triggered the rule and the resolved value at each path.
- Provide source-location pointers: file + line for raw YAML; file only for
  Helm/Kustomize (with an explicit caveat); a ready-to-run `kubectl` command
  for cluster scans.
- Surface evidence consistently across **pretty**, **JSON**, **SARIF**, **HTML**,
  and **JUnit** output formats.
- Default-redact resolved values for rules that match credentials via the
  existing `sensitiveValues` / `sensitiveKeyNames` posture control inputs
  (§5.4). Opt-in `--show-secrets` to reveal.
- Be honest about uncertainty: if a path cannot be resolved, say so explicitly
  rather than silently dropping it.

### Non-goals (initially)

- Image / CVE scan evidence (kubevuln). Different data shape, deferred to a
  later phase.
- Rewriting the 74 cluster-config / RBAC rules that emit no path-level
  evidence anywhere. These get a fallback rendering in phase 1 (snippets
  from `AlertObject` / `RelatedObjects`) and a regolibrary cleanup track in
  phase 5.
- Round-tripping evidence back into Helm chart source lines. Helm's render
  pipeline does not preserve template line numbers, and we will not pretend
  otherwise.

## 5. Proposed design

### 5.1 Data model

`opa-utils` already has the relevant building blocks. Rather than introduce a
parallel `Evidence` type, this proposal **extends what is already there**:

- `AssistedRemediation` (paths) — already on every `RuleResponse`.
- `reporthandling.Source` already carries source-location data and was extended
  by [opa-utils#168](https://github.com/kubescape/opa-utils/pull/168) with
  `HelmTemplateFile`, `HelmValuesPaths`, `HelmTemplateLine` for the Helm-fix
  work in [#2083](https://github.com/kubescape/kubescape/pull/2083).
- The `deletePaths` / `reviewPaths` split itself was added in
  [#1402](https://github.com/kubescape/kubescape/pull/1402) /
  [opa-utils#139](https://github.com/kubescape/opa-utils/pull/139), which
  establishes the precedent of progressively enriching path semantics.

Concrete changes:

```go
// opa-utils/reporthandling/datastructuresv1.go — extend Source
type Source struct {
    // ... existing fields, including the Helm ones from opa-utils#168 ...

    // KubectlCommand is set by the resource handler for cluster-scan resources.
    // Verbatim shell command that retrieves the relevant subtree, e.g.
    //   kubectl get deployment my-app -n prod -o yaml | yq '.spec.containers[0]'
    // Empty for non-cluster scans.
    KubectlCommand string `json:"kubectlCommand,omitempty"`

    // RawYamlLines maps the canonical form of a JSON-path string (as emitted
    // by rego in FailedPaths/ReviewPaths/etc.) to the 1-indexed line number in
    // File.
    // Populated only for SourceTypeYaml. Missing entries mean "unknown line";
    // we never guess. Backed by a yaml.v3 Node tree captured at parse time.
    RawYamlLines map[string]int `json:"rawYamlLines,omitempty"`
}
```

No new `Evidence` struct is added. The renderer consumes the existing
`RuleResponse.AssistedRemediation` (paths) + `Resource.Source` (provenance) +
the captured `AlertObject.K8SApiObjects` (for value resolution).

This avoids:
- A second source-of-truth for source-location data alongside `Source`.
- Backwards-incompatibility on the JSON report schema.
- Re-implementing fields that #2083 / opa-utils#168 already shipped.

Redaction is a report-serialization concern, not only a pretty-printer
concern: when a rule is classified sensitive by the predicate in §5.4, the
output sanitizer substitutes `"<redacted>"` for the resolved evidence value and
also scrubs the embedded raw objects that would otherwise carry the same literal
value. `--show-secrets` disables that sanitizer for users who explicitly need
literal values. The value-level substitution ships in phase 1 alongside the
pretty printer; embedded-object scrubbing follows in phase 3 (§6).

### 5.2 Path resolver

A small JSON-path walker that understands the exact subset rego emits:
dotted segments and `[N]` array indices. No JSONPath wildcards, no filters —
rego doesn't generate them.

**Reuse, don't reinvent:** [#2281](https://github.com/kubescape/kubescape/pull/2281)
already added segment-aware YAML path helpers (`yamlPathCovers`,
`plannedPathsFromExpressions`) in `core/pkg/fixhandler/fixhandler.go`. The
splitter / segmenter and canonicalizer must be **lifted into a shared package**
(e.g. `opa-utils/reporthandling/pathutil`) so the fixhandler, evidence
resolver, and YAML node walker share one definition of "path segment."
Independent implementations will drift and produce conflicting answers (e.g.
`spec.host` vs `spec.hostNetwork`) — exactly the bug class #2281 fixed.

The resolver does not compare raw path strings directly. Every rego-emitted path
and every key produced by the yaml.v3 node walk is parsed through `pathutil` and
serialized back to one canonical key before lookup. The original rule-emitted
path is still preserved for display, but `RawYamlLines` is keyed only by the
canonical form. Phase 1 owns this canonicalization contract so phase 2 does not
have to reverse-engineer multiple path dialects independently.

Behavior contract:

- Returns `(value, true)` on hit.
- Returns `(nil, false)` on any miss — caller renders as
  `value: <unresolved>` with the path still shown.
- Never panics on type mismatch (e.g. path expects array, object is a map).

### 5.3 Source-location capture

Done by the resource handler at scan time, not by re-reading files later
(the file may have changed between scan and report).

- **Raw YAML files:** parse with `gopkg.in/yaml.v3` to get a Node tree;
  walk it to populate `Source.RawYamlLines` keyed by the canonical JSON-path
  strings the resolver will look up. Done once at parse time; Node tree
  discarded afterward.
- **Helm:** **already done** by [#2083](https://github.com/kubescape/kubescape/pull/2083) +
  [opa-utils#168](https://github.com/kubescape/opa-utils/pull/168).
  `Source.HelmTemplateFile` and `Source.HelmValuesPaths` are populated by
  the `core/cautils/helmprovenance` extractor. The evidence renderer just
  reads these — no new scan-time work. The renderer also reuses the
  precedent established by #2083: **do not attempt to map rendered-YAML
  lines back to template lines**. The prior attempt
  ([#1215](https://github.com/kubescape/kubescape/pull/1215), with a
  follow-up crash fix in
  [#1628](https://github.com/kubescape/kubescape/pull/1628))
  was abandoned and the dead remnants cleaned up in
  [#1995](https://github.com/kubescape/kubescape/pull/1995).
- **Kustomize:** capture source file from `config.kubernetes.io/origin` when
  the rendered resource carries that annotation. This is best-effort: Kustomize
  only emits the annotation when origin annotations are enabled in
  `buildMetadata`, so the renderer must omit source rather than guess when the
  annotation is absent. No line.
- **Cluster:** populate `Source.KubectlCommand` from the resource's GVK +
  name + namespace + the path. No `File`/`Line`.
- **Git/repo scans:** same as raw YAML.

Note: reliable evidence on cluster-scoped resources (e.g. ClusterRole) in
large-cluster mode assumes the merged
[#2311](https://github.com/kubescape/kubescape/pull/2311) fix is present. No
additional defensive merge is planned for the evidence resolver.

### 5.4 Redaction policy

**Which rules are sensitive — decided from data that already exists.**

An earlier draft of this section keyed redaction off a `sensitive-data` entry in
rule metadata tags. That field does not exist: `rule.metadata.json` has no
`tags` key at all (the schema is `name` / `attributes` / `ruleLanguage` /
`match` / `configInputs` / `controlConfigInputs` / `ruleDependencies` /
`description` / `remediation` / `ruleQuery`), and the string `sensitive-data`
appears nowhere under `regolibrary/rules/` on `master`. A tag-based predicate
therefore selects nothing today, and inventing the tag would make this proposal
depend on a rule-library schema change — the same class of change §8 explicitly
defers.

**Decision: key the policy off the control inputs that already exist.** A rule
is treated as credential-bearing when its `controlConfigInputs` reference
either of the credential-matching posture control inputs:

- `settings.postureControlInputs.sensitiveValues`
- `settings.postureControlInputs.sensitiveKeyNames`

(together with their `sensitiveValuesAllowed` / `sensitiveKeyNamesAllowed`
allowlists, which the same rules consume to suppress known-safe matches).

This makes the policy data-driven and self-maintaining: any future rule that
matches credentials via these inputs is redacted automatically, with no Go-side
list to update. Kubescape already loads `rule.metadata.json` as part of the
downloaded policy, so the predicate is evaluable at scan time with no new I/O.

On `regolibrary` `master` the predicate resolves to exactly two rules:

| Rule | Control inputs consumed | Redacted? |
|---|---|---|
| `rule-credentials-in-env-var` (C-0012) | `sensitiveValues`, `sensitiveKeyNames` (+ both allowlists) | **yes** |
| `rule-credentials-configmap` | `sensitiveValues`, `sensitiveKeyNames` (+ both allowlists) | **yes** |

Two rules are deliberately **not** covered, and the reasons are worth recording
so they are not re-added later:

- `exposed-sensitive-interfaces-v1` is the third rule under `rules/` that
  references a `postureControlInputs.sensitive*` input, but the input it
  consumes is
  [`sensitiveInterfaces`](https://github.com/kubescape/regolibrary/blob/master/default-config-inputs.json)
  — a list of *workload name substrings* (`nifi`, `argo-server`, `kubeflow`,
  `kubernetes-dashboard`, `jenkins`, `prometheus-deployment`, `weave-scope-app`).
  Its evidence is a workload name and the LoadBalancer address that exposes it,
  not credential material. Redacting it would delete the finding's only useful
  content. Hence the predicate keys on `sensitiveValues`/`sensitiveKeyNames`
  specifically, not on the `sensitive*` prefix.
- `alert-mount-potential-credentials-paths` was on the earlier hardcoded list
  and is dropped. It declares no `controlConfigInputs` at all (and an empty
  `attributes` object), and its evidence is a `hostPath` mount path — the path
  *is* the finding. Redacting it removes the only actionable part.

The residual gap is a rule that surfaces credential material without consuming
either control input. Nothing under `rules/` does this today. Closing that gap
properly needs a first-class sensitivity marker in the rule schema, which is
the same regolibrary schema change §8 defers to phase 5; until it lands, the
control-input predicate is the whole policy and this document does not pretend
otherwise.

**What the sanitizer does.** For a rule matching the predicate, the output
sanitizer substitutes the resolved value with `"<redacted>"` before printing or
serializing, and also scrubs the same path in all embedded raw objects that can
be serialized into reports:

- `RuleResponse.AlertObject.K8SApiObjects`
- `RuleResponse.RelatedObjects[*].Object`
- `PostureReport.Resources[*].Object` when raw resources are included

If a sensitive path cannot be resolved in an embedded object, the sanitizer
drops that embedded object from default serialized outputs rather than shipping
raw credentials — except on the submit path, where omission is not available
and the fallback is a conservative scrub of the value-bearing subtree instead
(§5.4.1). A new CLI flag `--show-secrets` (default off) disables redaction. The
flag's help text spells out the risk: *"Includes literal credential values in
output; do not share resulting reports."*

The two halves ship in different phases: value-level substitution in the
evidence renderer lands in phase 1 with `--show-evidence`, embedded-object
scrubbing in phase 3 with the serialized emitters (§6).

### 5.4.1 Composition with the existing report-sanitization flags

Kubescape already ships three mechanisms for making a report safe to share, and
this proposal adds a fourth. Left undocumented, that is four partially
overlapping switches with undefined interactions. The intended composition:

| Existing mechanism | What it does today | Interaction with evidence redaction |
|---|---|---|
| `--omit-raw-resources` ([`printer/v2/utils.go` L217](https://github.com/kubescape/kubescape/blob/master/core/pkg/resultshandling/printer/v2/utils.go#L217): `if !data.OmitRawResources { report.Resources = finalizeResources(...) }`) | Drops `PostureReport.Resources` from serialized output entirely | Strictly stronger than the third scrub target above. When set, evidence redaction skips that target — the payload is already gone. It does **not** cover `AlertObject` / `RelatedObjects`, which the evidence sanitizer still scrubs. |
| `--hide` ([`scan.go` L184](https://github.com/kubescape/kubescape/blob/master/cmd/scan/scan.go#L184)) — "Replace sensitive report metadata with deterministic pseudonyms", applied via [`anonymizer.Apply`](https://github.com/kubescape/kubescape/blob/master/core/pkg/anonymizer/anonymizer.go#L10) | Rewrites resource names, namespaces, annotations, `sourcePath`, image references and container env values in the **session**, before results handling | Runs earlier in the pipeline than the evidence sanitizer, so the renderer sees already-pseudonymized data. `--hide` does not imply evidence redaction and evidence redaction does not imply `--hide`: they are orthogonal and both apply. Consequence to document in the flag help: under `--hide`, evidence `source:` lines and resolved values are pseudonyms, not literals. |
| `--encrypt` ([`scan.go` L185](https://github.com/kubescape/kubescape/blob/master/cmd/scan/scan.go#L185), [`core/pkg/reportcrypto`](https://github.com/kubescape/kubescape/blob/master/core/pkg/reportcrypto/crypto.go)) — same transformer pipeline via [`anonymizer.ApplyEncrypted`](https://github.com/kubescape/kubescape/blob/master/core/pkg/anonymizer/anonymizer.go#L44), reversible with `KUBESCAPE_MASTER_KEY` | Encrypts the same field set instead of pseudonymizing it; mutually exclusive with `--hide` ([`core/core/scan.go` L307/L343](https://github.com/kubescape/kubescape/blob/master/core/core/scan.go#L307)) | Same ordering as `--hide`. Because encryption is reversible and redaction is not, `--encrypt` must **not** be treated as satisfying the §7 bar: a `<redacted>` evidence value stays `<redacted>` after `kubescape decrypt`. |
| `--show-secrets` (new) | Disables evidence redaction | Overrides nothing else. It re-enables literal values in evidence but cannot resurrect payload dropped by `--omit-raw-resources`, nor un-pseudonymize `--hide` output. |

Note the overlap on the `--hide` row is not merely conceptual: the anonymizer
already carries its own hardcoded credential heuristics,
[`isSensitiveEnvName`](https://github.com/kubescape/kubescape/blob/master/core/pkg/anonymizer/container.go#L505)
and
[`isSensitiveEnvValue`](https://github.com/kubescape/kubescape/blob/master/core/pkg/anonymizer/container.go#L482),
whose pattern lists resemble but do not match `sensitiveKeyNames` /
`sensitiveValues` (e.g. the anonymizer knows `dsn` and `connectionstring`; the
control inputs know `eyJhbGciO` and `BEGIN \w+ PRIVATE KEY`). Phase 3 should
either converge the two on the control inputs or state explicitly why they stay
separate — shipping a third divergent credential-pattern list is the outcome to
avoid.

**Sharp edge: submitted reports.** `--omit-raw-resources` cannot be the
mechanism for submitted reports. It is rejected together with `--submit`
([`ErrOmitRawResourcesOrSubmit`](https://github.com/kubescape/kubescape/blob/master/cmd/scan/framework.go#L47),
enforced in [`cmd/scan/framework.go` L259](https://github.com/kubescape/kubescape/blob/master/cmd/scan/framework.go#L259)
and [`cmd/scan/control.go` L138](https://github.com/kubescape/kubescape/blob/master/cmd/scan/control.go#L138)),
and in submit mode it is explicitly ignored with a warning
([`core/core/scan.go` L67](https://github.com/kubescape/kubescape/blob/master/core/core/scan.go#L67):
`"omit-raw-resources flag will be ignored in submit mode"`). So for submitted
reports the raw objects always travel to the backend. Redaction there must be
value-level scrubbing applied to the object itself **before** submission, not
omission. This is the case where a leaked credential travels furthest, and
phase 3's acceptance criteria must include a submit-path test, not only a
serialized-file test.

### 5.5 CLI surface

- New flag: `--show-evidence` (alias `-E`) on `scan`, defaulting **off**.
  When on, the pretty printer renders an evidence block under each failed
  (resource, control) pair.
- `--verbose` / `-v` is left alone (compatibility). The current Kubescape CLI
  binds `-v` as a boolean, not a count, so `-vv` is not part of the initial
  surface; supporting it later would require explicit shorthand-count parsing.
- `--show-secrets` is independent from `--show-evidence`; it controls whether
  evidence-capable outputs keep or sanitize sensitive literal values.
- JSON / SARIF / HTML / JUnit always serialize the underlying path +
  resolved-value data after the redaction sanitizer runs — these consumers are
  programmatic and can choose to display or not. `--verbose` precedent for
  scan output and image scans is set by [#1320](https://github.com/kubescape/kubescape/pull/1320)
  and [#1932](https://github.com/kubescape/kubescape/pull/1932).

### 5.6 Pretty-printer rendering

For a finding on a Deployment failing C-0017 (read-only root filesystem):

```
✗ Deployment/prod/checkout-api  →  C-0017  Immutable container filesystem  [High]

  Evidence:
    spec.template.spec.containers[0].securityContext.readOnlyRootFilesystem
        value: <unset>
        source: charts/checkout/templates/deployment.yaml
                (Helm-rendered; original line not available)
    spec.template.spec.containers[1].securityContext.readOnlyRootFilesystem
        value: false
        source: charts/checkout/templates/deployment.yaml

  Fix:
    Set readOnlyRootFilesystem: true on both containers, or run:
      kubectl get deployment checkout-api -n prod -o yaml \
        | yq '.spec.template.spec.containers[].securityContext.readOnlyRootFilesystem = true'
```

For rules with no path-level evidence (the 74 cluster-config / RBAC rules
identified in §3), the evidence block falls back to a structured snippet of
the related objects:

```
  Evidence:
    (this control evaluates relationships across multiple resources)
    Subject:    ServiceAccount/kube-system/default
    Binding:    ClusterRoleBinding/cluster-admin-default
    Role:       ClusterRole/cluster-admin (verbs: *, resources: *)
    Inspect:    kubectl describe clusterrolebinding cluster-admin-default
```

## 6. Phased delivery

Each phase ships standalone value and is independently mergeable.

| Phase | Scope | Risk |
|---|---|---|
| **1** | Lift segmenter/canonicalizer from [#2281](https://github.com/kubescape/kubescape/pull/2281) into a shared `pathutil` package; build path resolver on top; pretty-printer block; `--show-evidence` flag; **value-level redaction for the §5.4 rule predicate plus `--show-secrets`**; cluster-scan `Source.KubectlCommand` extension; fallback renderer for the 74 no-path rules. | Medium — additive when the flag is off, but correctness depends on one shared canonical path representation. |
| **2** | Raw-YAML file:line capture via yaml.v3 Node tree; populate `Source.RawYamlLines` with canonical keys generated by `pathutil`. | Medium-high — touches scan-time code in the resource handler and must round-trip rego path variants to node-walk keys. |
| **3** | Extend the phase-1 sanitizer to serialized output: JSON/SARIF/HTML/JUnit emitters; scrub embedded raw objects (`AlertObject`, `RelatedObjects`, `PostureReport.Resources`) including on the submit path; document composition with `--omit-raw-resources` / `--hide` / `--encrypt` per §5.4.1. | Medium — must prove default serialized *and submitted* reports do not retain secret literals in `AlertObject`, `RelatedObjects`, or raw resource objects. |
| **4** | Wire existing `Source.HelmTemplateFile` / `HelmValuesPaths` (already populated by [#2083](https://github.com/kubescape/kubescape/pull/2083)) into the evidence renderer. Add Kustomize source-file capture. | Low for Helm (data is there); Medium for Kustomize (new). |
| **5** | regolibrary cleanup: convert the 74 no-path rules (and 17 mixed ones) to emit real paths or structured evidence fields. Continues the path-richness work begun in [#1402](https://github.com/kubescape/kubescape/pull/1402) / [opa-utils#139](https://github.com/kubescape/opa-utils/pull/139). | Long-tail, per-rule PRs. |
| **6** | kubevuln image-scan evidence (CVE + package + version + SBOM layer). Extends the verbose image-scan printer pattern from [#1320](https://github.com/kubescape/kubescape/pull/1320) / [#1932](https://github.com/kubescape/kubescape/pull/1932). | Separate repo, separate design doc. |

A user gets useful evidence the moment phase 1 lands; every subsequent
phase widens coverage without changing the surface.

**Why redaction is split across phases 1 and 3.** Because each phase is
independently mergeable, a phase-1-only release must already satisfy §7 bar #3.
An earlier draft put all redaction in phase 3, which meant phase 1 would print
the literal env-var value for a C-0012 finding with no sanitizer in front of it
— a shippable release violating the document's own reliability bar. The split
above is drawn along the surface each phase actually exposes:

- **Phase 1** exposes exactly one surface, the pretty-printer evidence block
  behind `--show-evidence`. It therefore needs exactly one thing: the
  value-level `"<redacted>"` substitution for rules matching the §5.4
  predicate, plus `--show-secrets` to opt out. That is a small, self-contained
  sanitizer sitting between the path resolver and the renderer.
- **Phase 3** extends that same sanitizer to the serialized formats and to the
  embedded raw objects. Phase 1 does not touch serialized output, so deferring
  the embedded-object scrubbing leaks nothing.

Concretely, `--show-secrets` is introduced in phase 1 rather than phase 3; the
flag's scope widens in phase 3 but its default and semantics do not change.

## 7. Reliability bar

Calling a column "Evidence" in a report seen by auditors raises the
correctness bar above normal CLI output. The implementation MUST:

1. Never silently drop a path it cannot resolve — always render
   `value: <unresolved>` so the user knows.
2. Never invent a source line. If line is unknown, omit it; do not guess.
3. Never include an unredacted sensitive value anywhere in default output.
   Redaction is the default, opt-out is explicit, and embedded raw objects must
   be scrubbed or omitted when they would otherwise carry the sensitive literal.
   This bar binds **per phase**, not only at the end of the sequence: no phase
   may ship an output surface without the sanitizer that covers it (§6).
   For submitted reports, "scrubbed or omitted" collapses to *scrubbed* —
   omission is unavailable on that path (§5.4.1).
4. Be deterministic: the same scan against the same input must produce
   byte-identical evidence blocks across runs.
5. Have golden-file tests for every rendering mode (pretty, JSON, SARIF,
   HTML, JUnit) covering: resolvable path, unresolvable path, redacted
   value, multi-path finding, placeholder-path fallback, cluster source,
   helm source, raw-yaml source with line — plus one case per flag
   interaction in §5.4.1 (`--omit-raw-resources`, `--hide`, `--encrypt`,
   `--show-secrets`) and a submit-path case asserting no credential literal
   survives in the submitted payload.

## 8. Open questions

- **Naming:** `--show-evidence` vs `--evidence`?
  Recommendation: `--show-evidence` as primary with `-E` shorthand. Reusing
  `-vv` is deferred unless Kubescape changes verbosity from a boolean into an
  explicit count or pre-parsed shorthand.
- **HTML report layout:** does the existing HTML template have room for a
  per-finding evidence panel, or does it need a redesign? Needs spike.
- **Multi-source resources:** a Deployment defined in raw YAML but applied
  to a cluster — which source wins? Recommendation: the scan input wins
  (the thing the user asked Kubescape to look at).
- **Performance:** path resolution is O(rules × resources × paths-per-rule).
  Expected to be negligible at typical scan sizes (<1s overhead on a
  10k-resource cluster); will benchmark in phase 1.
- **Regolibrary contract:** should we formalize an `evidence:` field in the
  rule response schema so phase-5 rules can emit structured evidence
  (e.g. matched substring + offset) rather than just paths? Probably yes,
  but defer the schema change until phase 5 is scoped.
- **Rule-level sensitivity marker:** the same phase-5 schema change is the
  natural home for an explicit per-rule sensitivity flag, which would replace
  the control-input predicate in §5.4 with a direct declaration and close the
  residual gap for rules that surface credentials without consuming
  `sensitiveValues` / `sensitiveKeyNames`. No such rule exists under `rules/`
  today, so this is a robustness improvement rather than a live gap — and it is
  deliberately *not* a prerequisite for phases 1–3. Recorded here so the
  control-input predicate is understood as the current whole policy, not as a
  stopgap for a field that already exists.

## 9. Alternatives considered

- **"Just point users at `--format json`."** Rejected: the JSON output is
  already the source of truth for downstream tooling, but the issue is
  specifically about humans reading CLI output during triage. Asking an
  auditor to `jq` through a 50MB JSON file is not a fix.
- **Per-rule custom evidence handlers in Go.** Rejected: 275 classified rules,
  maintenance burden, and most rules can be served by the generic
  path-resolver. Custom handling is reserved for the placeholder-path
  cluster-config rules in phase 5.
- **Generating a sidecar evidence file instead of inline rendering.**
  Rejected as primary: users want to *see* it while triaging. Sidecar is
  already covered by the JSON output.

