# Deep Review: 20260405-215736-fix-slice-negative-index

| | |
|---|---|
| **Date** | 2026-04-05 21:57 |
| **Repo** | [mikefarah/yq](https://github.com/mikefarah/yq) |
| **Round** | 1 |
| **Author** | Jan Dubois <jan@jandubois.com> |
| **Branch** | `fix-slice-negative-index` |
| **Commits** | `b18f62b0` Fix panic on negative slice indices that underflow after adjustment |
| **Reviewers** | Claude Opus 4.6, Codex GPT 5.4, Gemini 3.1 Pro |
| **Verdict** | **Merge with fixes** — the fix is correct but the regression test does not trigger the original panic |
| **Wall-clock time** | `18 min 8 s` |

---

## Executive Summary

This change clamps both slice index bounds to `[0, len(Content)]` after negative-to-positive adjustment, preventing the out-of-bounds panic reported by OSS-Fuzz. The clamping logic is correct and symmetric. The regression test, however, uses an expression (`.[0:-99999]`) that produces the same empty-sequence result with or without the fix — it never triggers the original panic, so it provides no protection against regression.

---

## Critical Issues

None.

---

## Important Issues

I1. **Regression test does not exercise the original panic** — `pkg/yqlib/operator_slice_test.go:101-111` (important, gap) [Claude Opus 4.6, Codex GPT 5.4, Gemini 3.1 Pro]

```go
{
	// Regression test for https://issues.oss-fuzz.com/issues/438776028
	skipDoc:    true,
	document:   `[a, b, c]`,
	expression: `.[0:-99999]`,
	expected: []string{
		"D0, P[], (!!seq)::[]\n",
	},
},
```

The expression `.[0:-99999]` only exercises second-index clamping (lines 54–55 of `operator_slice.go`). On unpatched code, `relativeSecondNumber` computes to `3 + (-99999) = -99996`, making the loop condition `0 < -99996` false — the loop terminates immediately without any out-of-bounds access. The test produces the same empty sequence with or without the fix.

The reported panic requires a first-index underflow: `.[-99999:3]` computes `relativeFirstNumber = 3 + (-99999) = -99996`, then the loop accesses `lhsNode.Content[-99996]`, triggering the panic. Gemini verified this by running the expression against unpatched code.

The fix also adds first-index clamping at lines 39–43 that has no test coverage.

Fix: add tests that exercise first-index underflow (which panics without the fix) and ideally both-index underflow:

```diff
 	{
 		skipDoc:    true,
 		document:   `[a, b, c]`,
 		expression: `.[0:-99999]`,
 		expected: []string{
 			"D0, P[], (!!seq)::[]\n",
 		},
 	},
+	{
+		skipDoc:    true,
+		document:   `[a, b, c]`,
+		expression: `.[-99999:3]`,
+		expected: []string{
+			"D0, P[], (!!seq)::- a\n- b\n- c\n",
+		},
+	},
+	{
+		skipDoc:    true,
+		document:   `[a, b, c]`,
+		expression: `.[-99999:-99998]`,
+		expected: []string{
+			"D0, P[], (!!seq)::[]\n",
+		},
+	},
```

---

## Suggestions

S1. **Extract shared clamping logic into a helper** — `pkg/yqlib/operator_slice.go:35-58` (suggestion, enhancement) [Claude Opus 4.6, Gemini 3.1 Pro]

```go
relativeFirstNumber := firstNumber
if relativeFirstNumber < 0 {
	relativeFirstNumber = len(lhsNode.Content) + firstNumber
}
if relativeFirstNumber < 0 {
	relativeFirstNumber = 0
} else if relativeFirstNumber > len(lhsNode.Content) {
	relativeFirstNumber = len(lhsNode.Content)
}
```

The same negative-adjustment-then-clamp pattern appears twice, once per index. A helper would remove duplication and make the intent self-documenting:

```go
func clampSliceIndex(index, length int) int {
	if index < 0 {
		index += length
	}
	if index < 0 {
		return 0
	}
	if index > length {
		return length
	}
	return index
}
```

Both call sites reduce to `relativeFirstNumber := clampSliceIndex(firstNumber, len(lhsNode.Content))`.

S2. **No type guard for non-sequence nodes** — `pkg/yqlib/operator_slice.go:27` (suggestion, gap) [Gemini 3.1 Pro]

```go
for el := context.MatchingNodes.Front(); el != nil; el = el.Next() {
	lhsNode := el.Value.(*CandidateNode)
```

`sliceArrayOperator` assumes `lhsNode` is a sequence. Slicing a scalar or mapping node silently produces a confusing result (an empty sequence tagged `!!str` or `!!map`). Gemini verified: `echo '"hello"' | yq eval '.[0:2]'` outputs `!!str []`. This is pre-existing and not worsened by the fix; it may be intentional. If not, adding a `lhsNode.Kind != SequenceNode` guard with a clear error would prevent confusion.

---

## Design Observations

**Strengths**

- The fix clamps both indices symmetrically, even though only the second index was reported. This prevents the same panic through the first-index path. *(in-scope)* [Claude Opus 4.6]
- Clamping uses `[0, len(Content)]` rather than `[0, len(Content)-1]`, correctly preserving half-open interval semantics. *(in-scope)* [Claude Opus 4.6]

---

## Testing Assessment

The existing test suite covers normal slicing, omitted first/second index, negative indices within range, indices beyond array length, and multi-array contexts. The new regression test covers the new second-index clamping path but does not exercise the original panic.

Untested scenarios ranked by risk:

1. **First-index underflow** (`.[-99999:3]`) — exercises lines 39–40, the only new code path without coverage. **(see I1)**
2. **Both indices underflow** (`.[-99999:-99998]`) — exercises both clamping paths simultaneously.
3. **First-index underflow on an empty sequence** (`[] | .[-99999:]`) — edge case combining empty array with underflow.

---

## Documentation Assessment

No documentation gaps. The commit message explains the root cause, fix, and links the OSS-Fuzz issue.

---

## Commit Structure

Single commit, single logical change. The message accurately describes the fix. Clean.

---

## Agent Performance Retro

### [Claude]

Claude read both changed files, ran `git blame`, ran the test suite, and searched the codebase for similar patterns. It correctly identified the missing first-index test coverage (I1) and the helper extraction opportunity (S1). It also noted two design strengths (symmetric clamping and correct half-open interval semantics).

Claude did not verify whether the existing test actually triggers the panic on unpatched code — it assumed the test was partially effective rather than wholly ineffective.

### [Codex]

Codex was the fastest agent (82s). It ran targeted `git blame` on both the fix and test lines, ran the test suite, and searched for related patterns. It identified the missing first-index test (I1) with a clear explanation and concrete fix suggestion including both first-index and both-index underflow tests.

Like Claude, Codex did not verify whether the existing test panics on unpatched code.

### [Gemini]

Gemini was the most thorough in verification, spending 344s and making 42 tool calls. It checked out the pre-fix commit and ran the test expression against unpatched code, confirming that `.[0:-99999]` does not panic. It also verified that `.[-99999:-99998]` does panic on unpatched code. This empirical verification was the review's most valuable contribution.

Gemini also found a pre-existing type-guard gap (S2) by testing `echo '"hello"' | yq eval '.[0:2]'` — an issue no other agent discovered.

Gemini never ran `git blame`, consistent with known quota limitations for the daily Pro model. However, it compensated by running code against both patched and unpatched versions — a stronger verification method for this review.

**Tool call highlights**

- **[Claude]**: Used `git blame` (1 call) to verify authorship. Ran tests (1 call). Searched for similar patterns with 4 grep calls. Efficient 11-call session.
- **[Codex]**: Used `git blame` (2 targeted calls on specific line ranges). Ran tests twice. Searched with 4 `rg` calls. Efficient 16-call session.
- **[Gemini]**: Never ran `git blame`. Made 14 `git checkout` calls to switch between patched and unpatched code — high volume but justified by the empirical verification approach. Ran 12 `echo | go run` commands testing various edge cases. Extensive exploration (42 calls total) that produced the review's key insight.

### Summary

| | Claude Opus 4.6 | Codex GPT 5.4 | Gemini 3.1 Pro |
|---|---|---|---|
| Duration | 143s | 82s | 344s |
| Findings | 0C 1I 1S | 0C 1I 0S | 0C 1I 2S |
| Tool calls | 11 (Read 3, Bash 4, Grep 4) | 16 (exec_command 14, write_stdin 2) | 42 (run_shell_command 35, read_file 4, grep_search 2, write_file 1) |
| Design observations | 2 | 0 | 0 |
| False positives | 0 | 0 | 0 |
| Downgraded | 0 | 0 | 1 (I→S) |
| Unique insights | 0 | 0 | 1 (S2) |
| Files reviewed | 2 | 2 | 2 |
| Coverage misses | 0 | 0 | 0 |

**Continuation threshold**: Claude (1I, 1S — qualifies), Codex (1I — qualifies), Gemini (2I, 1S — qualifies). All three would continue in thorough mode.

**Reconciliation**: Gemini I2 (missing type guard for non-sequence nodes): important → suggestion S2. The issue is pre-existing and unrelated to the negative-index fix; it does not meet the "important" bar for a change-scoped review.

Gemini provided the most value on this review. Its empirical verification — running the test expression against unpatched code — revealed that the regression test is not merely incomplete but wholly ineffective. Claude and Codex both identified the missing first-index coverage but missed that the existing test also fails to serve its stated purpose. Codex was the most efficient, delivering a focused review in under 90 seconds.

---

## Appendix: Original Reviews

### [Claude] — Pass 1

I now have a complete picture of both changed files. Let me write the review.

---

## Review: `fix-slice-negative-index` (b18f62b0)

| Attribute | Value |
|-----------|-------|
| **Verdict** | Approve with suggestions |
| **Risk** | Low |

### Executive Summary

This change correctly fixes an OSS-Fuzz-reported panic where negative slice indices with magnitude exceeding the array length cause out-of-bounds access. The clamping logic is sound and consistent between both index positions. The one gap is that the regression test only exercises second-index underflow, leaving the newly added first-index clamping untested.

---

### Findings

#### Critical Issues

None.

#### Important Issues

**I1. First-index underflow clamping is untested** — `operator_slice_test.go:101-111` *(important, gap)*

```go
	{
		// Regression test for https://issues.oss-fuzz.com/issues/438776028
		skipDoc:    true,
		document:   `[a, b, c]`,
		expression: `.[0:-99999]`,
		expected: []string{
			"D0, P[], (!!seq)::[]\n",
		},
	},
```

The regression test exercises only `.[0:-99999]`, which triggers clamping of `relativeSecondNumber` (lines 54–55). The fix also adds clamping for `relativeFirstNumber` (lines 39–40), but no test exercises that path. Before the fix, `.[-99999:3]` on `[a, b, c]` would produce `relativeFirstNumber = 3 + (-99999) = -99996`, which is passed directly to the loop at line 63, causing a panic identical to the reported one.

Fix: add a test case for first-index underflow:

```go
{
	skipDoc:    true,
	document:   `[a, b, c]`,
	expression: `.[-99999:3]`,
	expected: []string{
		"D0, P[], (!!seq)::- a\n- b\n- c\n",
	},
},
```

#### Suggestions

**S1. Extract shared clamping logic into a helper** — `operator_slice.go:35-43, 50-58` *(suggestion, enhancement)*

The same five-line pattern (negative adjustment, then clamp to `[0, len]`) is now duplicated for both indices:

```go
if relative < 0 {
    relative = length + original
}
if relative < 0 {
    relative = 0
} else if relative > length {
    relative = length
}
```

A small helper would remove the duplication and make the intent self-documenting:

```go
func clampSliceIndex(index, length int) int {
	if index < 0 {
		index += length
	}
	if index < 0 {
		return 0
	}
	if index > length {
		return length
	}
	return index
}
```

The call sites would simplify to:

```diff
-		relativeFirstNumber := firstNumber
-		if relativeFirstNumber < 0 {
-			relativeFirstNumber = len(lhsNode.Content) + firstNumber
-		}
-		if relativeFirstNumber < 0 {
-			relativeFirstNumber = 0
-		} else if relativeFirstNumber > len(lhsNode.Content) {
-			relativeFirstNumber = len(lhsNode.Content)
-		}
+		relativeFirstNumber := clampSliceIndex(firstNumber, len(lhsNode.Content))
```

(Same for `relativeSecondNumber`.)

---

### Design Observations

**Concerns**

- None in-scope. The clamping approach matches Python/jq semantics and is the standard fix for this class of bug.

**Strengths**

- The fix clamps both indices symmetrically, even though only the second index was reported. This prevents the same panic through the first-index path. *(in-scope)*
- The clamping uses `[0, len(Content)]` rather than `[0, len(Content)-1]`, correctly preserving the half-open interval semantics of slice bounds. *(in-scope)*

---

### Testing Assessment

The existing test suite covers: normal slicing, omitted first/second index, negative indices within range, indices beyond array length, and multi-array contexts. The new regression test covers the reported fuzzer case.

**Untested scenarios ranked by risk:**

1. **First-index underflow** (`.[-99999:3]`) — exercises lines 39–40, the only new code path without test coverage. **(see I1)**
2. **Both indices underflow** (`.[-99999:-99999]`) — should produce empty array. Low incremental risk since it combines two individually clamped paths.

---

### Documentation Assessment

No documentation gaps. The commit message is thorough — it explains the root cause, the fix, includes a reproducer, and links the OSS-Fuzz issue.

---

### Commit Structure

Single commit, single logical change. Message accurately describes the fix. Clean.

---

### Coverage Summary

| File | Status |
|------|--------|
| `pkg/yqlib/operator_slice.go` | I1, S1 |
| `pkg/yqlib/operator_slice_test.go` | I1 |

### [Codex] — Pass 1

### Executive Summary

This change closes the reported panic by clamping slice bounds after negative-index adjustment in `sliceArrayOperator()`, and the targeted slice test suite passes. The main issue is test coverage: the new code clamps both the first and second bounds, but the added regression test only exercises the second-bound path, leaving half of the fix unguarded.

### Findings

**Critical Issues**

None.

**Important Issues**

I1. **Missing regression coverage for the new first-bound clamp** — `pkg/yqlib/operator_slice_test.go:101-109` `(important, gap)`

```go
{
	// Regression test for https://issues.oss-fuzz.com/issues/438776028
	skipDoc:    true,
	document:   `[a, b, c]`,
	expression: `.[0:-99999]`,
	expected: []string{
		"D0, P[], (!!seq)::[]\n",
	},
},
```

The commit adds new lower/upper clamping for the first slice index at `pkg/yqlib/operator_slice.go:39-42`, but the only new test at `pkg/yqlib/operator_slice_test.go:101-109` exercises only the second-index path (`.[0:-99999]` hits `relativeSecondNumber`, not `relativeFirstNumber`). `git blame` confirms both the new clamp (`operator_slice.go:39-42`) and this regression test (`operator_slice_test.go:101-109`) were introduced in `b18f62b0`. Because the original bug came from an underflowed negative bound, leaving the mirrored first-bound case untested makes it easy to regress `.[-99999:2]` or `.[-99999:]` in a follow-up refactor without CI catching it.

Fix: add at least one regression case that drives `relativeFirstNumber < 0` after adjustment, and ideally one case where both bounds underflow.

```diff
@@
 	{
 		// Regression test for https://issues.oss-fuzz.com/issues/438776028
 		// Negative slice index that underflows after adjustment must not
 		// cause an out-of-bounds access.
 		skipDoc:    true,
 		document:   `[a, b, c]`,
 		expression: `.[0:-99999]`,
 		expected: []string{
 			"D0, P[], (!!seq)::[]\n",
 		},
 	},
+	{
+		skipDoc:    true,
+		document:   `[a, b, c]`,
+		expression: `.[-99999:2]`,
+		expected: []string{
+			"D0, P[], (!!seq)::- a\n- b\n",
+		},
+	},
+	{
+		skipDoc:    true,
+		document:   `[a, b, c]`,
+		expression: `.[-99999:-99998]`,
+		expected: []string{
+			"D0, P[], (!!seq)::[]\n",
+		},
+	},
```

**Suggestions**

None.

### Design Observations

**Concerns**

- `(in-scope)` The fix duplicates the clamp logic for the first and second bounds in `sliceArrayOperator()`. That is acceptable here, but it increases the chance of asymmetric future edits, which is exactly why the missing first-bound test matters.

**Strengths**

- `(in-scope)` Clamping after negative-to-positive adjustment is the right layer for this fix: it preserves existing slice parsing and only hardens the actual bounds used for `lhsNode.Content[i]`.

### Testing Assessment

Targeted validation:
- `go test ./pkg/yqlib -run TestSliceOperatorScenarios` passed.

Untested scenarios ranked by risk:
1. First bound underflows after adjustment, e.g. `.[-99999:2]`.
2. Both bounds underflow after adjustment, e.g. `.[-99999:-99998]`.
3. First bound underflows on an empty sequence, e.g. `[] | .[-99999:]`.

### Documentation Assessment

No documentation update looks necessary; this is a bug fix with test-only behavioural clarification.

### Commit Structure

The single commit is coherent and stands alone. The message accurately describes the code change.

### Acknowledged Limitations

None.

### Coverage Summary

- `pkg/yqlib/operator_slice.go` — **finding IDs:** I1
- `pkg/yqlib/operator_slice_test.go` — **finding IDs:** I1

### [Gemini] — Pass 1

### Executive Summary
This change fixes an out-of-bounds panic when negative slice indices underflow after adjustment, correctly clamping both starting and ending indices to the array bounds. However, the regression test added for this fix is ineffective and never triggered a panic on unpatched code, while a pre-existing gap allows slicing non-sequence types into conceptually invalid YAML nodes.

### Critical Issues

*(None)*

### Important Issues

I1. **Ineffective regression test** — `pkg/yqlib/operator_slice_test.go:101-111` (important, gap)

```go
		// Regression test for https://issues.oss-fuzz.com/issues/438776028
		// Negative slice index that underflows after adjustment must not
		// cause an out-of-bounds access.
		skipDoc:    true,
		document:   `[a, b, c]`,
		expression: `.[0:-99999]`,
		expected: []string{
			"D0, P[], (!!seq)::[]\n",
		},
```

The added regression test is a false positive for coverage. Because the first index `0` evaluates to a relative starting index of `0`, and `-99999` evaluated (before the fix) to an ending relative index of `-99996`, the internal loop `for i := 0; i < -99996; i++` terminates immediately without making any out-of-bounds access. This test correctly returns `[]` on the unpatched codebase without panicking. 

The panic caused by OSS-Fuzz specifically happened when the *first* index severely underflowed (e.g., `.[-99999:-99998]`), which sets `relativeFirstNumber` strictly less than 0 and triggers the panicking array access on the very first loop iteration. 

**Fix:** Update the test expression to one that definitively panics without the fix, such as `.[-99999:-99998]`.

I2. **Missing type check in array slice operator** — `pkg/yqlib/operator_slice.go:64-70` (important, gap)

```go
		var newResults []*CandidateNode
		for i := relativeFirstNumber; i < relativeSecondNumber; i++ {
			newResults = append(newResults, lhsNode.Content[i])
		}

		sliceArrayNode := lhsNode.CreateReplacement(SequenceNode, lhsNode.Tag, "")
		sliceArrayNode.AddChildren(newResults)
```

`sliceArrayOperator` unconditionally assumes `lhsNode` is a sequence. If `lhsNode` is a string scalar or mapping node, it blindly iterates over the node's underlying raw `Content` slice and wraps the output in a `SequenceNode` that improperly inherits the original node's tag (`!!str` or `!!map`). 

For example, `echo '"hello"' | yq eval '.[0:2]' -` outputs `!!str []`, creating a conceptually invalid sequence masquerading as a string.

**Fix:** Add a check `if lhsNode.Kind != SequenceNode` at the top of the function to return a clean error (e.g., `cannot slice scalar, expected array`). Alternatively, if `yq` aims to strictly mirror `jq`'s string slicing functionality, add a specific execution branch to handle `ScalarNode` string manipulation cleanly.

### Suggestions

S1. **Refactor duplicate clamping logic** — `pkg/yqlib/operator_slice.go:34-58` (suggestion, enhancement)

```go
		relativeFirstNumber := firstNumber
		if relativeFirstNumber < 0 {
			relativeFirstNumber = len(lhsNode.Content) + firstNumber
		}
		if relativeFirstNumber < 0 {
			relativeFirstNumber = 0
		} else if relativeFirstNumber > len(lhsNode.Content) {
			relativeFirstNumber = len(lhsNode.Content)
		}
```

The exact same bounds adjustment and clamping logic is applied to both `firstNumber` and `secondNumber` sequentially, creating nearly 20 lines of identical code. 

**Fix:** Extract this math into a testable helper function:

```go
func clampSliceIndex(index int, length int) int {
	relative := index
	if relative < 0 {
		relative = length + index
	}
	if relative < 0 {
		return 0
	}
	if relative > length {
		return length
	}
	return relative
}
```

### Coverage Summary

- `pkg/yqlib/operator_slice.go` — I2, S1
- `pkg/yqlib/operator_slice_test.go` — I1
