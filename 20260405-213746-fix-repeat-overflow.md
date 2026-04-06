# Deep Review: 20260405-213746-fix-repeat-overflow

| | |
|---|---|
| **Date** | 2026-04-05 21:37 |
| **Repo** | [mikefarah/yq](https://github.com/mikefarah/yq) |
| **Round** | 1 |
| **Branch** | `fix-repeat-overflow` |
| **Commits** | `2c37aa9d` Fix panic and OOM in repeatString for large repeat counts |
| **Reviewers** | Claude Opus 4.6, Codex GPT 5.4, Gemini 3.1 Pro |
| **Verdict** | **Merge with fixes** — integer overflow in the new size guard defeats the protection this commit introduces |
| **Wall-clock time** | `11 min 51 s` |

---

## Executive Summary

This commit replaces a fixed repeat-count ceiling (10 million) in `repeatString` with a result-size check (10 MiB), addressing two OSS-Fuzz panic reports. The size-based approach is the right design, but the new guard multiplies `len(stringNode.Value) * count` as native `int` values. On both 64-bit and 32-bit platforms, a sufficiently large `count` overflows the product to a negative or small positive number, bypassing the guard and reaching `strings.Repeat` — the exact panic path this commit exists to eliminate.

---

## Critical Issues

C1. **Integer overflow in size guard bypasses the check** — `pkg/yqlib/operator_multiply.go:159-161` [Claude Opus 4.6, Codex GPT 5.4, Gemini 3.1 Pro]

```go
maxResultLen := 10 * 1024 * 1024 // 10 MiB
if count > 0 && len(stringNode.Value)*count > maxResultLen {
    return nil, fmt.Errorf("result of repeating string (%v bytes) by %v would exceed %v bytes", len(stringNode.Value), count, maxResultLen)
}
target.Value = strings.Repeat(stringNode.Value, count)
```

`len(stringNode.Value)` and `count` are both `int`. The `parseInt` function at `lib.go:176` accepts values up to `math.MaxInt`, so `count` can reach `2^63 - 1` on 64-bit platforms. Multiplying a 2-byte string length by `2^62` produces `2^63`, which overflows `int64` to `math.MinInt64` (negative). Since negative values are always less than `maxResultLen`, the guard evaluates to `false`, and `strings.Repeat` panics with `"strings: Repeat output length overflow"` — the exact failure this commit aims to prevent.

Claude verified this experimentally: `strLen=2, count=4611686018427387904 → product=-9223372036854775808 → guard bypassed → panic`.

The same overflow occurs on 32-bit platforms (the project ships `yq_windows_386.exe` via `scripts/xcompile.sh`) at much smaller values. (critical, regression)

Fix: divide instead of multiply — division by a positive `count` cannot overflow:

```diff
 maxResultLen := 10 * 1024 * 1024 // 10 MiB
-if count > 0 && len(stringNode.Value)*count > maxResultLen {
+if count > 0 && len(stringNode.Value) > maxResultLen/count {
     return nil, fmt.Errorf("result of repeating string (%v bytes) by %v would exceed %v bytes", len(stringNode.Value), count, maxResultLen)
 }
```

When `count` is enormous, `maxResultLen/count` truncates to 0, so any non-empty string triggers the guard. When `count` is 0, the left side of `&&` short-circuits. This pattern is standard for overflow-safe size checks.

---

## Important Issues

I1. **No test for the integer overflow path** — `pkg/yqlib/operator_multiply_test.go` [Claude Opus 4.6, Codex GPT 5.4, Gemini 3.1 Pro]

All three regression tests use counts (99,999,999 and 100,000,001) far too small to trigger integer overflow on 64-bit. After fixing C1, add a test with a count large enough to confirm the division-based guard catches it:

```go
{
    // Verify the size guard is overflow-safe.
    skipDoc:       true,
    expression:    `"ab" * 4611686018427387904`,
    expectedError: "result of repeating string (2 bytes) by 4611686018427387904 would exceed 10485760 bytes",
},
```

This test would have caught C1 on the current code. (important, gap)

---

## Suggestions

S1. **No boundary test at exactly 10 MiB** — `pkg/yqlib/operator_multiply_test.go` [Codex GPT 5.4]

No test verifies that a repeated string whose output is exactly `10 * 1024 * 1024` bytes succeeds while one byte over is rejected. This boundary test would confirm the guard uses `>` (not `>=`) and that the result is produced. (suggestion, gap)

S2. **Grammar: "1 bytes"** — `pkg/yqlib/operator_multiply.go:161` [Claude Opus 4.6]

The format string `"(%v bytes)"` produces `"(1 bytes)"` for single-byte strings. Since this message is primarily diagnostic, this is cosmetic. One fix: use `"%v-byte"` which reads correctly for all values. (suggestion, regression)

---

## Design Observations

**Strengths**

- Replacing the count-based limit with a result-size limit aligns the guard with the actual allocation risk. A 1-byte string repeated 10 million times (10 MB) and a 10-byte string repeated 1 million times (10 MB) now receive identical treatment. [Claude Opus 4.6, Codex GPT 5.4, Gemini 3.1 Pro]
- The regression tests link to specific OSS-Fuzz issue URLs, making future failures easy to trace back to the original reports. [Claude Opus 4.6, Codex GPT 5.4]
- The old error message claimed a limit of "100 million" but the code checked `> 10000000` (10 million) — a 10x discrepancy. This change eliminates that confusion. [Claude Opus 4.6]

---

## Testing Assessment

Existing tests cover: negative count, count exceeding the limit for both short and long strings, and the two specific OSS-Fuzz inputs. Missing scenarios, ranked by risk:

1. **Integer overflow bypass** — a count near `math.MaxInt64` with a multi-byte string (see I1). This is the exact attack vector the fix targets.
2. **Boundary at the 10 MiB limit** — product exactly at the limit succeeds; product one byte over is rejected (see S1).

---

## Documentation Assessment

No documentation updates needed. The change is internal, and the test descriptions document the behaviour.

---

## Commit Structure

Single commit for a single fix — clean and appropriate. The commit message accurately describes the change.

---

## Agent Performance Retro

### [Claude]

Claude identified the integer overflow as critical and, uniquely, verified it experimentally by writing and running a Go program in `.scratch/` that demonstrated the overflow and resulting panic. This experimental verification elevated the finding from theoretical to proven. Claude also spotted the "1 bytes" grammar issue (S2), which no other agent raised. Traced `parseInt` in `lib.go` to confirm the input range, and searched for other `strings.Repeat` call sites and similar size limits in the codebase. Ran `git blame` 3 times to confirm regression classification.

**Tool call highlights**: 14 tool calls. 3 `git blame` invocations confirming the changed lines. 2 scratch files written and executed to reproduce the overflow experimentally — the most rigorous verification approach of the three agents.

### [Codex]

Codex found the same overflow issue but rated it important and framed it as 32-bit-only, missing that the same overflow occurs on 64-bit platforms at larger values. Codex ran the existing test suite (`go test -run TestMultiplyOperatorScenarios`) and attempted a 32-bit cross-compile (`GOARCH=386`), which failed on the macOS host. Checked `scripts/xcompile.sh` to confirm the project ships a 32-bit binary. Produced a clean, well-structured report in the shortest time. Ran `git blame` 3 times.

**Tool call highlights**: 20 tool calls (highest count). Distinctive: tried running tests with `GOARCH=386` to reproduce the 32-bit case, and checked `xcompile.sh` for build targets. Heavy use of `sed -n` and `nl` for targeted line reading.

### [Gemini]

Gemini found the overflow and rated it critical, matching Claude's assessment. Gemini also wrote scratch programs to verify the overflow, tested `parseInt` behaviour, and explored a pre-existing design concern about custom tag handling in `repeatString`. Raised the constant-extraction suggestion (not included in consolidated findings as YAGNI for a single-use local variable). Did not run `git blame`, so its classification of C1 as "gap" rather than "regression" was incorrect — the code was introduced by this commit. Modified the actual test file with `sed -i` to test overflow values, which is unusual in a review worktree.

**Tool call highlights**: 18 tool calls. No `git blame` usage (known Gemini limitation). 4 scratch programs written and executed. Used `sed -i` to mutate the test file in the worktree to test overflow scenarios — creative but atypical for a read-only review.

### Summary

| | Claude Opus 4.6 | Codex GPT 5.4 | Gemini 3.1 Pro |
|---|---|---|---|
| Duration | 167 s | 81 s | 218 s |
| Findings | 1C 0I 2S | 0C 1I 0S | 1C 1I 1S |
| Tool calls | 14 (Bash 7, Read 4, Grep 3) | 20 (exec 19, write_stdin 1) | 18 (shell 9, read 4, write 3, grep 2) |
| Design observations | 3 strengths | 2 strengths | 1 concern |
| False positives | 0 | 0 | 0 |
| Unique insights | S2 (grammar) | boundary test | custom tag concern |
| Files reviewed | 2/2 | 2/2 | 2/2 |
| Coverage misses | 0 | 0 | 0 |

**Reconciliation**: Codex rated the overflow as important (I1); consolidated as critical (C1) because Claude demonstrated the overflow on 64-bit, not just 32-bit. Claude rated the missing overflow test as S1 (suggestion); consolidated as I1 (important) because this test gap directly corresponds to the critical finding. Gemini classified C1 as "gap"; corrected to "regression" — the code was introduced by this commit per `git blame`. Gemini's S1 (extract constant) dropped — single-use local variable with an inline comment is sufficient.

All three agents independently identified the same core bug. Claude provided the strongest evidence through experimental verification. Codex was fastest and most thorough in tool usage but underrated the severity. Gemini explored the widest surface area, including pre-existing design concerns, but lacked `git blame` data for accurate classification.

---

## Review Process Notes

No skill improvements or repo context updates warranted. All three agents independently found the same critical issue from the inlined diff alone — the prompt guided them effectively for this change.

---

## Appendix: Original Reviews

### [Claude] — Pass 1

---

## Deep Code Review: `fix-repeat-overflow`

**Verdict: Request changes** — one critical integer overflow defeats the guard this change introduces.

### Executive Summary

This change replaces a fixed repeat-count ceiling (10 million) with a result-size check (10 MiB) in `repeatString`, aiming to prevent panics and OOM from fuzz-discovered inputs. The approach is sound, but the guard uses multiplication (`len * count`) that silently overflows on 64-bit (and 32-bit) platforms, allowing crafted inputs to bypass the check entirely and trigger the exact panic the change is meant to prevent.

---

### Findings

#### Critical Issues

**C1. Integer overflow in size guard bypasses the check** — `pkg/yqlib/operator_multiply.go:160` (critical, regression)

```go
maxResultLen := 10 * 1024 * 1024 // 10 MiB
if count > 0 && len(stringNode.Value)*count > maxResultLen {
```

`len(stringNode.Value)` and `count` are both `int`. On 64-bit platforms, `count` can be up to `math.MaxInt64` (via `parseInt` at `lib.go:176`). Multiplying a string length of 2 by a count of `2^62` produces `2^63`, which overflows `int64` to `math.MinInt64` (a negative number). Since negative values are always less than `maxResultLen`, the guard evaluates to `false`, and `strings.Repeat` is called with the enormous count, panicking with `"strings: Repeat output length overflow"`.

I verified this experimentally:

```
strLen=2  count=4611686018427387904  product=-9223372036854775808
product > maxResultLen: false   ← guard bypassed
strings.Repeat panicked: strings: Repeat output length overflow
```

Fix: use division instead of multiplication — division by a positive `count` cannot overflow:

```diff
 maxResultLen := 10 * 1024 * 1024 // 10 MiB
-if count > 0 && len(stringNode.Value)*count > maxResultLen {
-    return nil, fmt.Errorf("result of repeating string (%v bytes) by %v would exceed %v bytes", len(stringNode.Value), count, maxResultLen)
+if count > 0 && len(stringNode.Value) > maxResultLen/count {
+    return nil, fmt.Errorf("result of repeating string (%v bytes) by %v would exceed %v bytes", len(stringNode.Value), count, maxResultLen)
 }
```

When `count` is enormous, `maxResultLen/count` truncates to 0, so any non-empty string (`len > 0`) correctly triggers the guard. When `count` is 0, the left side of `&&` short-circuits. This pattern is standard for overflow-safe size checks.

#### Important Issues

None.

#### Suggestions

**S1. No test for the overflow path** — `pkg/yqlib/operator_multiply_test.go` (suggestion, gap)

All three error tests use counts (99,999,999 and 100,000,001) that don't overflow on 64-bit. After fixing C1, add a test with a count large enough to overflow the naive multiplication, confirming the division-based guard catches it:

```go
{
	// Verify the size guard is overflow-safe.
	skipDoc:       true,
	expression:    `"ab" * 4611686018427387904`,
	expectedError: "result of repeating string (2 bytes) by 4611686018427387904 would exceed 10485760 bytes",
},
```

**S2. Minor grammar: "1 bytes"** — `pkg/yqlib/operator_multiply.go:161` (suggestion, regression)

The error message format `"(%v bytes)"` produces `"(1 bytes)"` for single-byte strings. This is cosmetic, but since the message is user-facing, it could say `"(%v-byte)"` to be grammatically correct for all values:

```diff
-return nil, fmt.Errorf("result of repeating string (%v bytes) by %v would exceed %v bytes", len(stringNode.Value), count, maxResultLen)
+return nil, fmt.Errorf("result of repeating string (%v-byte) by %v would exceed %v bytes", len(stringNode.Value), count, maxResultLen)
```

Alternatively, leave it as-is — the message is primarily for diagnostics, not a polished UI string.

---

### Design Observations

**Strengths**

- Switching from a count-based limit to a result-size limit is the right design: it correctly handles both short strings repeated many times and long strings repeated fewer times, which the old fixed ceiling did not.
- The two regression tests tied to specific OSS-Fuzz issue URLs are good practice — they document *why* the guard exists and make the tests easy to trace back to the original reports.
- The old error message claimed a limit of "100 million" but the code checked `> 10000000` (10 million) — a 10x discrepancy. This change eliminates that confusion.

---

### Testing Assessment

The tests cover the core scenarios (negative count, count exceeding the limit for both short and long strings). Missing:

1. **Integer overflow bypass** (highest risk) — a count near `math.MaxInt64` with a multi-byte string, as described in S1. This is the exact attack vector the fix is meant to close.
2. **Boundary case: product exactly at limit** — e.g., a 10-byte string repeated 1,048,576 times (exactly 10 MiB). Verifies the guard uses `>` not `>=` and that the result is actually produced.

---

### Documentation Assessment

No documentation updates needed. The change is internal and the test descriptions adequately document the behaviour.

---

### Commit Structure

Single commit for a single fix — clean and appropriate. The commit message accurately describes the change and its motivation.

---

### Coverage Summary

| File | Status |
|------|--------|
| `pkg/yqlib/operator_multiply.go` | **C1, S2** |
| `pkg/yqlib/operator_multiply_test.go` | **S1** — reviewed, test coverage gap noted |

### [Codex] — Pass 1

### Executive Summary

This change replaces the old fixed repeat-count cap with a 10 MiB output-size cap in `repeatString`, and adds regression tests for the two reported fuzz cases. The size-based approach is directionally better, but the new guard introduced at `pkg/yqlib/operator_multiply.go:159-161` still performs unchecked `int` multiplication, so 32-bit builds can wrap the product and fall through to `strings.Repeat`, reintroducing the panic path the patch is supposed to eliminate.

### Findings

**Critical Issues**

None.

**Important Issues**

I1. **Overflow-prone size check still permits panic on 32-bit builds** — `pkg/yqlib/operator_multiply.go:159-163` (important, regression)

```go
maxResultLen := 10 * 1024 * 1024 // 10 MiB
if count > 0 && len(stringNode.Value)*count > maxResultLen {
	return nil, fmt.Errorf("result of repeating string (%v bytes) by %v would exceed %v bytes", len(stringNode.Value), count, maxResultLen)
}
target.Value = strings.Repeat(stringNode.Value, count)
```

The new guard at `pkg/yqlib/operator_multiply.go:159-161` multiplies `len(stringNode.Value)` and `count` as `int`s before comparing against the 10 MiB limit. `git blame` shows those lines were introduced in commit `2c37aa9d`. On 32-bit targets, `parseInt` in `pkg/yqlib/lib.go:176-184` explicitly accepts values up to `math.MaxInt`, and this repository still ships a 32-bit artefact via `scripts/xcompile.sh:23-31` (`yq_windows_386.exe`). That means an input such as `"ab" * 1500000000` can overflow the product at `pkg/yqlib/operator_multiply.go:160`, make the comparison false, and then reach `strings.Repeat` at `pkg/yqlib/operator_multiply.go:163`, which is exactly the panic path this patch is trying to prevent. The newly added tests at `pkg/yqlib/operator_multiply_test.go:240-244` and `pkg/yqlib/operator_multiply_test.go:696-707` only exercise 64-bit-safe products, so this regression is currently undetected.

Fix: avoid multiplying in `int` space. Either promote both operands to `int64` before the check, or use division so the comparison cannot overflow on any supported architecture.

```diff
 maxResultLen := 10 * 1024 * 1024 // 10 MiB
-if count > 0 && len(stringNode.Value)*count > maxResultLen {
+if count > 0 && len(stringNode.Value) > maxResultLen/count {
 	return nil, fmt.Errorf("result of repeating string (%v bytes) by %v would exceed %v bytes", len(stringNode.Value), count, maxResultLen)
 }
 target.Value = strings.Repeat(stringNode.Value, count)
```

Also add a regression test that targets the overflow shape rather than just a large but non-overflowing product; for example, gate it behind a 32-bit-only assertion or use a helper that validates the guard logic without depending on the host architecture.

**Suggestions**

None.

### Design Observations

**Strengths**

- Replacing the old fixed count cap with an output-size cap is the right abstraction for this operator: it aligns the guard with the actual allocation risk instead of penalising short strings with harmless repeat counts.
- The added tests in `pkg/yqlib/operator_multiply_test.go:696-707` correctly anchor the change to the two OSS-Fuzz reports, which makes future regressions easier to trace back to the original failures.

### Testing Assessment

1. The highest-risk missing case is a platform-width overflow scenario for the new guard, because `pkg/yqlib/operator_multiply.go:160` now depends on `int` arithmetic and the project still distributes a 32-bit binary.
2. There is no boundary test proving that a repeated string whose output is exactly `10 * 1024 * 1024` bytes is still accepted while `+1` byte is rejected.

### Documentation Assessment

No documentation updates appear necessary for this change. The behaviour is internal and the error text change is self-explanatory.

### Coverage Summary

- `pkg/yqlib/operator_multiply.go` — **I1**
- `pkg/yqlib/operator_multiply_test.go` — **I1**

### [Gemini] — Pass 1

### Executive Summary

This change replaces a hardcoded loop count limit with a 10 MiB string length limit to fix out-of-memory issues and panics when generating very large strings. While the transition from loop counting to memory bounding correctly addresses the root cause of the memory exhaustion, the implementation introduces a critical integer overflow gap that still allows `strings.Repeat` to panic, failing to fully resolve the fuzzing issue.

### Findings

#### Critical Issues

C1. **Integer overflow bypasses string length limit** — `pkg/yqlib/operator_multiply.go:159-161` (critical, gap)

```go
	maxResultLen := 10 * 1024 * 1024 // 10 MiB
	if count > 0 && len(stringNode.Value)*count > maxResultLen {
		return nil, fmt.Errorf("result of repeating string (%v bytes) by %v would exceed %v bytes", len(stringNode.Value), count, maxResultLen)
	}
```

The expression `len(stringNode.Value)*count` can overflow the signed `int` type when `count` is extremely large. In Go, `int` is 32-bit or 64-bit depending on the architecture. If a user or fuzzer passes a massive count (e.g., `math.MaxInt / 2`), the product overflows and wraps around to a negative number or a small positive number. When this overflowed value is `≤ maxResultLen`, the size check evaluates to `false` and is bypassed. The code then proceeds to `strings.Repeat(stringNode.Value, count)`, which internally checks for overflow and panics with `strings: Repeat output length overflow`, defeating the purpose of this PR. 

Fix: Re-write the size check using division instead of multiplication. Short-circuit evaluation ensures no division by zero occurs if the string is empty.

```diff
-	if count > 0 && len(stringNode.Value)*count > maxResultLen {
+	strLen := len(stringNode.Value)
+	if count > 0 && strLen > 0 && count > maxResultLen/strLen {
```

#### Important Issues

I1. **Missing test for integer overflow bounds** — `pkg/yqlib/operator_multiply_test.go:695-707` (important, gap)

```go
	{
		// Regression test for https://issues.oss-fuzz.com/issues/418818862
		// Large repeat count with a long string must not panic.
		skipDoc:       true,
		expression:    `"abc" * 99999999`,
		expectedError: "result of repeating string (3 bytes) by 99999999 would exceed 10485760 bytes",
	},
```

The new regression tests use `99999999` (~100 million). While this tests the intended limit logic correctly, it is far too small to trigger the integer overflow gap described in C1. To ensure the fix remains robust across both 32-bit and 64-bit architectures, the test suite must include a value large enough to overflow the product bounds check.

Fix: Add a test case with a massive repeat count that definitively overflows the check, ensuring the logic remains robust.

```diff
 		// Large repeat count with a long string must not panic.
 		skipDoc:       true,
 		expression:    `"abc" * 99999999`,
 		expectedError: "result of repeating string (3 bytes) by 99999999 would exceed 10485760 bytes",
 	},
+	{
+		// Test integer overflow protection in repeat string size check
+		skipDoc:       true,
+		expression:    `"abc" * 3074457345618258603`,
+		expectedError: "result of repeating string (3 bytes) by 3074457345618258603 would exceed 10485760 bytes",
+	},
```

#### Suggestions

S1. **Extract `maxResultLen` into a named constant** — `pkg/yqlib/operator_multiply.go:158` (suggestion, enhancement)

```go
	maxResultLen := 10 * 1024 * 1024 // 10 MiB
```

Defining the memory bounds inline makes it difficult to manage or track limits project-wide. 

Fix: Extract the magic number to a named constant at the package level, such as `const maxStringRepeatBytes = 10 * 1024 * 1024`, improving code clarity.

### Design Observations

#### Concerns

**Custom Tags Mapped as Strings Break Node Identification** (pre-existing)
When `repeatString` assigns `stringNode` and `intNode`, it relies entirely on `lhs.Tag == "!!str"` (`operator_multiply.go:142-148`). If the LHS is a custom tag (e.g. `!custom`) that `multiplyScalars` successfully guesses as a string (which resolves custom tags for operation dispatch but does *not* mutate the original `lhs.Tag`), `repeatString` erroneously jumps into the `else` branch. This assigns the integer to the string variable and the custom string to the integer variable. Calling `parseInt` on the custom string will fail and produce a runtime parsing error (`strconv.ParseInt: parsing "<custom_value>": invalid syntax`). This couples the `repeatString` implementation too tightly to strict YAML tag syntax, breaking behavior that `multiplyScalars` already normalized.
*Alternative approach*: Inspect the integer node directly (`rhs.guessTagFromCustomType() == "!!int"`) or pass down the guessed tags from `multiplyScalars` so `repeatString` does not have to evaluate strict standard tags a second time.

### Testing Assessment

1. (Highest risk) Missing an integer overflow bounds test to verify string multiplication limits do not panic.
2. (Moderate risk) Missing tests for repeating an empty string (`""`) with an extremely large count to confirm `maxResultLen` handles zero string length gracefully.

### Documentation Assessment

No major user-facing documentation updates are required, as `skipDoc: true` is properly used on internal limitation tests, preserving clean generation of the core operation documentation.

### Coverage Summary

- `pkg/yqlib/operator_multiply.go`: **C1, S1**
- `pkg/yqlib/operator_multiply_test.go`: **I1**
