# Deep Review: 20260405-211911-fix-map-equality-null-key

| | |
|---|---|
| **Date** | 2026-04-05 21:19 |
| **Repo** | [mikefarah/yq](https://github.com/mikefarah/yq) |
| **Round** | 1 |
| **Author** | Jan Dubois |
| **Branch** | `fix-map-equality-null-key` |
| **Commits** | `f7d884ae` Fix panic in recurseNodeObjectEqual when comparing maps with null keys |
| **Reviewers** | Claude Opus 4.6, Codex GPT 5.4, Gemini 3.1 Pro |
| **Verdict** | **Merge with fixes** — the one-line fix is correct, but the same bug persists in `containsObject` |
| **Wall-clock time** | `11 min 6 s` |

---

## Executive Summary

This branch replaces `findInArray` with `findKeyInMap` in `recurseNodeObjectEqual`, fixing an out-of-bounds panic when comparing maps whose keys include null values. The fix is correct, minimal, and well tested. All three reviewers independently identified the same parallel bug in `containsObject`, which uses `findInArray` on a MappingNode with a `%2` guard that prevents the panic but produces false negatives.

---

## Critical Issues

None.

---

## Important Issues

I1. **`containsObject` has the same `findInArray` bug** — `pkg/yqlib/operator_contains.go:49-51` (important, gap) [Claude Opus 4.6, Codex GPT 5.4, Gemini 3.1 Pro]

```go
lhsKeyIndex := findInArray(lhs, rhsKey)
log.Debugf("index is %v", lhsKeyIndex)
if lhsKeyIndex < 0 || lhsKeyIndex%2 != 0 {
    return false, nil
}
```

`containsObject` at line 49 searches a MappingNode's `Content` array with `findInArray`, which iterates by 1 and can match a key against a *value* at an odd index. The `%2` guard at line 51 prevents the out-of-bounds panic this branch fixes in `recurseNodeObjectEqual`, but it introduces false negatives: when a null key exists later in the map but a null *value* appears first, `findInArray` returns the value's odd index, the guard rejects it, and the function reports the key as missing.

Codex verified this live: `printf '? 1\n: ~\n? ~\n: x\n' | yq 'contains({~: "x"})'` returns `false` even though the map contains `null: x`.

`git blame` confirms this code predates the current change (Mike Farah, 2021, commit `2db8140d7`).

Fix: replace `findInArray` with `findKeyInMap` and remove the even-index guard.

```diff
-       lhsKeyIndex := findInArray(lhs, rhsKey)
+       lhsKeyIndex := findKeyInMap(lhs, rhsKey)
        log.Debugf("index is %v", lhsKeyIndex)
-       if lhsKeyIndex < 0 || lhsKeyIndex%2 != 0 {
+       if lhsKeyIndex < 0 {
            return false, nil
        }
```

---

## Suggestions

None.

---

## Design Observations

### Concerns

1. **Map lookups lack a single, safe primitive** (in-scope) [Claude Opus 4.6, Codex GPT 5.4, Gemini 3.1 Pro] — `findInArray` accepts any `*CandidateNode` without distinguishing sequences from maps. This makes it easy to use on a MappingNode by mistake, as both `recurseNodeObjectEqual` and `containsObject` demonstrate. Converting all map-key searches to `findKeyInMap` (or adding a type assertion to `findInArray`) would eliminate this class of bug rather than requiring each call site to remember the stride.

### Strengths

1. The fix reuses the existing `findKeyInMap` function rather than introducing new abstractions.
2. Both a low-level unit test (`TestRecurseNodeObjectEqual`) and an operator-level integration test (`. += .` with sequence keys) validate the fix from different angles.

---

## Testing Assessment

The new tests cover the reported crash well:
1. `lib_test.go:304-320` — directly verifies `recurseNodeObjectEqual` returns `false` for maps with different key types (null vs int).
2. `operator_add_test.go:531-541` — end-to-end regression test using the exact input shape from the OSS-Fuzz report.

Untested scenarios (ranked by risk):
1. `contains` with null keys where a null value appears earlier in the map (exercises the `containsObject` parallel bug from I1).

---

## Documentation Assessment

No updates needed. The regression test comments include the OSS-Fuzz issue link, which is appropriate documentation for a fuzz-discovered bug fix.

---

## Commit Structure

Single commit with a clear message stating what changed and why. The scope is tight: one bug fix with two regression tests.

---

## Agent Performance Retro

### [Claude]

Found I1 (`containsObject` gap), rated it Important. Provided a concrete false-negative example and traced the `%2` guard logic. Ran 4 `git blame` commands to confirm the code predates this change. Also grepped for all callers of `findInArray` and `findKeyInMap`, read `operator_omit.go` and `operator_add.go` to check for further instances, and ran the test suite to verify the new tests pass. Accurate, thorough, no false positives.

### [Codex]

Found I1, also rated Important. Distinguished itself by *live-verifying* the bug: ran `printf '? 1\n: ~\n? ~\n: x\n' | go run ./yq.go 'contains({~: "x"})'` and confirmed the false-negative output. Also read `operator_subtract.go` and `operator_contains_test.go` to check for parallel patterns. Ran `git blame` and the test suite. No false positives.

### [Gemini]

Found the same issue as C1 (Critical). Wrote test files in its worktree to verify the bug. Ran 1 `git blame` (expected — Gemini's Pro model quota limits blame calls). Also checked `operator_has.go` for related patterns. No false positives.

**Tool call highlights:**
- **Claude** — 4 `git blame` calls, systematic grep for `findInArray`/`findKeyInMap`/`recursiveNodeEqual`. Read 6 files beyond the 3 changed files.
- **Codex** — 2 `git blame` calls, 4 `rg` searches. Uniquely ran `go run ./yq.go` to verify the bug produces wrong output live.
- **Gemini** — 1 `git blame` call, 6 grep searches. Wrote 2 test files to verify the bug experimentally. Broader grep for `find` across the codebase.

### Summary

| | Claude Opus 4.6 | Codex GPT 5.4 | Gemini 3.1 Pro |
|---|---|---|---|
| Duration | 151s | 114s | 140s |
| Findings | 0C 1I 0S | 0C 1I 0S | 0C 1I 0S |
| Tool calls | 20 (Read 10, Bash 7) | 23 (exec_command 20) | 19 (grep 6, shell 6) |
| Design observations | 2 | 3 | 2 |
| False positives | 0 | 0 | 0 |
| Unique insights | 0 | 1 (live verification) | 0 |
| Files reviewed | 3/3 | 3/3 | 3/3 |
| Coverage misses | 0 | 0 | 0 |

**Downgraded:** Gemini rated the `containsObject` finding as Critical; consolidated to Important. The `%2` guard prevents the panic — the bug produces false negatives, not crashes. A pre-existing gap with no reported user impact does not meet the Critical bar.

**Reconciliation:** Gemini P1 containsObject finding: critical → important I1.

All three agents converged on the same finding with zero false positives. Codex's live verification added the most value by confirming the false-negative behaviour with a concrete command. Claude's systematic blame and grep coverage was the most thorough for attribution. Gemini's experimental test-writing approach was creative but produced no additional findings.

---

## Review Process Notes

No skill improvements or repo context updates warranted. All three agents found the same finding independently — the prompt guided them well. The small diff size made complete coverage straightforward.

---

## Appendix: Original Reviews

### [Claude] — Pass 1

I now have all the context needed. Here is the review.

---

## Executive Summary

This change fixes an index-out-of-bounds panic in `recurseNodeObjectEqual` caused by using `findInArray` (stride-1 iteration) to search a map's `Content` array (which stores alternating key-value pairs). The fix correctly replaces it with `findKeyInMap` (stride-2 iteration). Both a unit test and an end-to-end regression test are included.

**Verdict:** Approve — the fix is correct and well-tested. One parallel instance of the same bug exists in `containsObject` and should be addressed.

---

### Critical Issues

None.

### Important Issues

I1. **Same bug pattern in `containsObject`** — `pkg/yqlib/operator_contains.go:49-52` *(important, gap)*

```go
lhsKeyIndex := findInArray(lhs, rhsKey)
log.Debugf("index is %v", lhsKeyIndex)
if lhsKeyIndex < 0 || lhsKeyIndex%2 != 0 {
    return false, nil
}
```

`containsObject` at line 49 uses `findInArray` to search a MappingNode — the exact same incorrect usage this commit fixes in `recurseNodeObjectEqual`. The `%2 != 0` guard at line 51 prevents the out-of-bounds panic but produces false negatives: if a key matches a *value* at an odd index before matching the actual *key* at a higher even index, the function reports the key as missing.

Concrete example: `{a: null, null: b} contains {null: b}` should be `true`, but:
- `findInArray` finds `!!null` at index 1 (the *value* of key `a`), returns 1
- `1 % 2 != 0` → returns false
- The actual null *key* at index 2 is never checked

`git blame` confirms this code predates the current change (Mike Farah, 2021-09-15, commit `2db8140d7`), so it is not a regression. But it is the same class of bug, and the fix already exists in the codebase.

Fix: replace `findInArray` with `findKeyInMap` and remove the `%2` guard:

```diff
-		lhsKeyIndex := findInArray(lhs, rhsKey)
+		lhsKeyIndex := findKeyInMap(lhs, rhsKey)
 		log.Debugf("index is %v", lhsKeyIndex)
-		if lhsKeyIndex < 0 || lhsKeyIndex%2 != 0 {
+		if lhsKeyIndex < 0 {
 			return false, nil
 		}
```

### Suggestions

None.

---

### Design Observations

**Strengths**

- The fix is minimal and uses the existing `findKeyInMap` function, which was purpose-built for this iteration pattern. No new abstractions introduced.
- Both the direct unit test (`TestRecurseNodeObjectEqual`) and the integration test (`. += .` with sequence keys containing maps with null keys) are well-constructed and directly exercise the failure mode from the OSS-Fuzz report.

---

### Testing Assessment

The new tests cover the reported crash scenario well:

1. `lib_test.go:304-320` — unit test that directly verifies `recurseNodeObjectEqual` returns `false` for maps with different key types (null vs int) where the old code would match a null key against a null value and panic on the out-of-bounds `+1` access.
2. `operator_add_test.go:531-541` — end-to-end regression test using the exact input shape from the OSS-Fuzz report (sequence keys containing single-entry mappings with null vs non-null keys).

Untested scenarios (ranked by risk):

1. If I1 is fixed: `contains` with null keys and null values in the same map (exercises the `containsObject` parallel bug).

---

### Documentation Assessment

No documentation updates needed. The regression test comments include the OSS-Fuzz issue link, which is the appropriate documentation for a fuzz-discovered bug fix.

---

### Commit Structure

Single commit with a clear message that states what changed and why. The scope is tight — one bug fix with two test additions. Clean.

---

### Coverage Summary

| File | Status |
|---|---|
| `pkg/yqlib/lib.go` | **I1** (parallel bug in same file's `containsObject` caller); changed line reviewed, no issues |
| `pkg/yqlib/lib_test.go` | Reviewed, no issues |
| `pkg/yqlib/operator_add_test.go` | Reviewed, no issues |

### [Codex] — Pass 1

**Verdict:** Request changes

### Executive Summary

This change correctly fixes the panic in [`recurseNodeObjectEqual`](/private/tmp/review-codex-wzebKf/pkg/yqlib/lib.go#L74) by switching map-key lookup to [`findKeyInMap`](/private/tmp/review-codex-wzebKf/pkg/yqlib/lib.go#L64). The main remaining issue is that the same key-vs-value lookup mistake still exists in [`containsObject`](/private/tmp/review-codex-wzebKf/pkg/yqlib/operator_contains.go#L41), so the branch closes the crash in one path while leaving the same null-key miscomparison bug in another map operation. The new regression tests are good for the add/equality path, but they do not cover this parallel `contains()` failure mode.

### Findings

**Critical Issues**

None.

**Important Issues**

I1. **`containsObject` still searches map content as a flat array, so null keys can match null values and return false negatives** — [`pkg/yqlib/operator_contains.go:45-54`](/private/tmp/review-codex-wzebKf/pkg/yqlib/operator_contains.go#L45) (important, gap)

```go
for index := 0; index < len(rhs.Content); index = index + 2 {
	rhsKey := rhs.Content[index]
	rhsValue := rhs.Content[index+1]
	log.Debugf("Looking for %v in the lhs", rhsKey.Value)
	lhsKeyIndex := findInArray(lhs, rhsKey)
	log.Debugf("index is %v", lhsKeyIndex)
	if lhsKeyIndex < 0 || lhsKeyIndex%2 != 0 {
		return false, nil
	}
```

The fix in [`pkg/yqlib/lib.go:83`](/private/tmp/review-codex-wzebKf/pkg/yqlib/lib.go#L83) addresses the panic in `recurseNodeObjectEqual`, but `containsObject` still uses `findInArray` at line 49 against a map’s alternating key/value `Content`. Because `recursiveNodeEqual` treats any two `!!null` scalars as equal, a null key in `rhs` can match an earlier null **value** in `lhs`; the odd-index guard at lines 51-52 then returns `false` even when the real key exists later in the map. This is a concrete behavioural bug, not just a theoretical parallel pattern: `printf '? 1\n: ~\n? ~\n: x\n' | go run ./yq.go 'contains({~: "x"})'` currently returns `false`, although the map does contain the `null: x` entry.

Fix: use the same key-only lookup helper here, and add a regression test in [`pkg/yqlib/operator_contains_test.go`](/private/tmp/review-codex-wzebKf/pkg/yqlib/operator_contains_test.go) for a map with an earlier null value and a later null key.

```diff
for index := 0; index < len(rhs.Content); index = index + 2 {
 	rhsKey := rhs.Content[index]
 	rhsValue := rhs.Content[index+1]
 	log.Debugf("Looking for %v in the lhs", rhsKey.Value)
-	lhsKeyIndex := findInArray(lhs, rhsKey)
+	lhsKeyIndex := findKeyInMap(lhs, rhsKey)
 	log.Debugf("index is %v", lhsKeyIndex)
-	if lhsKeyIndex < 0 || lhsKeyIndex%2 != 0 {
+	if lhsKeyIndex < 0 {
 		return false, nil
 	}
 	lhsValue := lhs.Content[lhsKeyIndex+1]
```

**Suggestions**

None.

### Design Observations

**Concerns**

1. `(in-scope)` Map lookups are still implemented ad hoc across operators instead of being expressed as a single “key-only” primitive for mapping nodes. This branch fixed [`recurseNodeObjectEqual`](/private/tmp/review-codex-wzebKf/pkg/yqlib/lib.go#L74), but `containsObject` still had the same underlying assumption. Converting all mapping-node key searches to `findKeyInMap` or a small `lookupMapValue` helper would eliminate this entire class of key/value parity bugs rather than fixing individual call sites as fuzzing finds them.

**Strengths**

1. The change in [`pkg/yqlib/lib.go`](/private/tmp/review-codex-wzebKf/pkg/yqlib/lib.go#L83) is narrowly scoped and matches the actual root cause described in the fuzz report: key lookup on mapping content must advance by two.
2. Adding both a low-level regression in [`pkg/yqlib/lib_test.go`](/private/tmp/review-codex-wzebKf/pkg/yqlib/lib_test.go#L304) and an operator-level regression in [`pkg/yqlib/operator_add_test.go`](/private/tmp/review-codex-wzebKf/pkg/yqlib/operator_add_test.go#L531) is the right testing shape for this bug.

### Testing Assessment

1. `contains()` is untested for the same null-key/null-value collision that caused this bug family. A scenario like `? 1\n: ~\n? ~\n: x` with `contains({~: "x"})` should be added to [`pkg/yqlib/operator_contains_test.go`](/private/tmp/review-codex-wzebKf/pkg/yqlib/operator_contains_test.go).
2. No regression test currently exercises a multi-entry map where the matching key exists later in `Content` after an earlier value that compares equal. That ordering is what makes the `findInArray`/`findKeyInMap` distinction observable.

### Documentation Assessment

No documentation updates needed for the fixed equality path. If `containsObject` is corrected as well, no user-facing docs change should be required there either.

### Commit Structure (PR and branch reviews only)

The single commit `f7d884ae` is cohesive and the message accurately matches the diff.

### Coverage Summary

- [`pkg/yqlib/lib.go`](/private/tmp/review-codex-wzebKf/pkg/yqlib/lib.go) — Reviewed, no issues
- [`pkg/yqlib/lib_test.go`](/private/tmp/review-codex-wzebKf/pkg/yqlib/lib_test.go) — Reviewed, no issues
- [`pkg/yqlib/operator_add_test.go`](/private/tmp/review-codex-wzebKf/pkg/yqlib/operator_add_test.go) — Reviewed, no issues

### [Gemini] — Pass 1

### Executive Summary

This PR fixes a critical out-of-bounds panic in `recurseNodeObjectEqual` when comparing maps with null keys, resolving OSS-Fuzz issue 383860504. By replacing `findInArray` (which iterated by 1) with `findKeyInMap` (which iterates by 2), it ensures that key lookups only match against actual keys (even indices) rather than mistakenly matching values (odd indices) which could lead to an out-of-bounds index when accessing the corresponding value. The change is correct, well-tested, and fully addresses the reported vulnerability.

### Critical Issues

**C1. Incorrect `findInArray` usage in `contains` operator** — `pkg/yqlib/operator_contains.go:49-51` (critical, gap)

```go
		log.Debugf("Looking for %v in the lhs", rhsKey.Value)
		lhsKeyIndex := findInArray(lhs, rhsKey)
		log.Debugf("index is %v", lhsKeyIndex)
		if lhsKeyIndex < 0 || lhsKeyIndex%2 != 0 {
			return false, nil
		}
```

While this PR corrects the usage of `findInArray` for mapping nodes in `lib.go`, the same buggy pattern still exists in `containsObject`. Because `findInArray` searches every element of the `Content` array, it can mistakenly match a value instead of a key if the sought key appears earlier in the map as a value (e.g., `{"a": "b", "b": "c"}`). When this happens, `lhsKeyIndex` will be an odd number, causing the function to short-circuit and return `false` entirely instead of continuing to find the actual key at an even index.

Fix: Replace `findInArray` with `findKeyInMap` and remove the even-index check, bringing it inline with the fix in `recurseNodeObjectEqual`.

```diff
--- pkg/yqlib/operator_contains.go
+++ pkg/yqlib/operator_contains.go
@@ -46,9 +46,9 @@
 		rhsKey := rhs.Content[index]
 		rhsValue := rhs.Content[index+1]
 		log.Debugf("Looking for %v in the lhs", rhsKey.Value)
-		lhsKeyIndex := findInArray(lhs, rhsKey)
+		lhsKeyIndex := findKeyInMap(lhs, rhsKey)
 		log.Debugf("index is %v", lhsKeyIndex)
-		if lhsKeyIndex < 0 || lhsKeyIndex%2 != 0 {
+		if lhsKeyIndex < 0 {
 			return false, nil
 		}
 		lhsValue := lhs.Content[lhsKeyIndex+1]
```

### Important Issues

None.

### Suggestions

None.

### Design Observations

**Concerns**

- `findInArray` semantics on MappingNodes (in-scope): The root cause of the panic and the `contains` bug (C1) is that mapping node contents are flattened into a 1D slice rather than a strongly-typed structure or map. Because `findInArray` accepts `*CandidateNode` without checking if `Kind == SequenceNode`, it’s easy for developers to accidentally use it on `MappingNode`s. A type-guard or assertion in `findInArray` could prevent these categories of bugs entirely.

**Strengths**

- The regression tests are excellent. Including both a low-level AST node test (`TestRecurseNodeObjectEqual`) and a high-level integration test mapping to the user's workflow (`TestAddOperatorScenarios`) perfectly validates both the unit and the symptom.

### Testing Assessment

Test coverage is comprehensive for the changed code. The tests effectively simulate the crash condition using `!!null` keys and ensure the AST comparison properly steps over values.

1. **Untested Scenario (C1)**: The `contains` operator lacks a test case asserting behavior when a queried key appears as a value earlier in the target map. Adding a test like `contains({"a": "name", "name": "foo"}, {"name": "foo"})` would prevent regression on the `containsObject` logic.

### Documentation Assessment

No documentation gaps. The fix strictly addresses internal AST comparison semantics and has no user-facing API changes. 

### Coverage Summary

- `pkg/yqlib/lib.go` — Reviewed, no issues
- `pkg/yqlib/lib_test.go` — Reviewed, no issues
- `pkg/yqlib/operator_add_test.go` — Reviewed, no issues
