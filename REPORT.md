# remotion: `@remotion/paths` chained `T` fix

## Branch

- Fork: `b84330808/remotion`
- Branch: `claude/funny-galileo-b8glbo`
- Commit: `0d8564a901863234fa373234e237fac3c65189b5`, one commit on top of `upstream/main` @ `3b7e7008a495094424859e9cf4f66f4844d2dfd9`
- Author/committer: `b84330808 <b84330808@gmail.com>`, trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`
- Files: `packages/paths/src/helpers/construct.ts`, `packages/paths/src/helpers/iterate.ts`, `packages/paths/src/test/get-length.test.ts`, `packages/paths/src/test/reduce-instructions.test.ts` (+43 / −4)

Nothing was opened, edited or commented upstream.

## PR title

```
`@remotion/paths`: Fix geometry of chained `T` commands after a non-quadratic command
```

(The shape matches the maintainer's merged #12547, "`@remotion/paths`: Fix geometry of relative quadratic paths". The prefix is the package's `name` from `packages/paths/package.json`.)

## PR body

```markdown
Per the [SVG spec](https://www.w3.org/TR/SVG2/paths.html#PathDataQuadraticBezierCommands), a `T`/`t` that does not follow a `Q`/`q`/`T`/`t` uses the current point as its control point, and the next `T` reflects that control point. Both code paths in `@remotion/paths` got the second `T` wrong:

- `construct()` (used by `getLength()`, `getPointAtLength()`, `getTangentAtLength()`, `getInstructionIndexAtLength()`, `evolvePath()`) reflected a stale control point instead: the one from an earlier `Q`, or `[0, 0]`. Because that point is absolute, the result also changed when the path was translated.
- `iterateOverSegments()` (used by `reduceInstructions()`, and through it `getBoundingBox()`, `cutPath()`, `reversePath()`, `scalePath()`, `warpPath()`, `interpolatePath()`) left the tracked control point `null` after `H`/`L`/`V`/`A`/`Z`/`M`, which was then read as `0`. After `C`/`S`, it reflected the cubic's second control point.

| Path | Chromium `getTotalLength()` | `getLength()` before | after |
| --- | --- | --- | --- |
| `M 0 0 H 10 T 20 10 T 30 0` | 45.925 | 44.206 | 45.925 |
| same after `translatePath(path, 100, 100)` | 45.925 | 160.758 | 45.925 |
| `M 0 0 C 0 10 10 10 10 0 T 20 10 T 30 0` | 55.925 | 54.206 | 55.925 |
| `M 0 0 Q 5 5 10 0 L 20 0 T 30 10 T 40 0` | 57.403 | 60.765 | 57.403 |

I also compared 2000 random paths mixing every command against Chromium's `getTotalLength()`. Before: `getLength()` disagreed on 300 and `getLength(serializeInstructions(reduceInstructions(...)))` on 296. After: 0 and 0.

The added tests compare each `T` chain with the equivalent explicit `Q` path.

I used Claude Code to write this change.
```

## Root cause

- `packages/paths/src/helpers/construct.ts`, `T` and `t` branches: when the previous command isn't quadratic, the segment is drawn as a line, which is correct. But `prev_point` is still updated with `2 * cur - prev_point`, using a stale `prev_point`. The fix sets it to `cur` in that case.
- `packages/paths/src/helpers/iterate.ts`, `case 'T'`: the control point was only reflected when `cpX`/`cpY` weren't `null`.
  - After `H`/`L`/`V`/`A`/`Z`/`M` they are `null`, so they stayed `null`. The next `T` in `remove-a-s-t-curves.ts` read them as `cpX as number`, so `null` became `0` and the point was reflected about the origin.
  - After `C`/`S` they hold the cubic's `cp2`, which was then reflected.
  - The fix reflects only when the previous segment is `Q` or `T`, and otherwise uses the current point.

## Fail before (source fix reverted, new tests kept)

`git stash push -- src/helpers/construct.ts src/helpers/iterate.ts`, then `bun test src/test/get-length.test.ts src/test/reduce-instructions.test.ts` in `packages/paths`:

```
bun test v1.4.4 (38afee6c8)

src/test/get-length.test.ts:
 5 | 	expect(getLength('M 0 0 L 100 0')).toBe(100);
 6 | });
 7 | 
 8 | test('Should get length of T after a T that follows a non-quadratic command', () => {
 9 | 	const expected = getLength('M 0 0 H 10 Q 10 0 20 10 Q 30 20 30 0');
10 | 	expect(getLength('M 0 0 H 10 T 20 10 T 30 0')).toBeCloseTo(expected);
                                                     ^
error: expect(received).toBeCloseTo(expected, precision)

Expected: 45.92523340395388
Received: 44.205799129001974

Expected precision: 2
Expected difference: < 0.005
Received difference: 1.719434274951908

      at <anonymous> (/home/user/remotion/packages/paths/src/test/get-length.test.ts:10:49)
(fail) Should get length of T after a T that follows a non-quadratic command [4.74ms]

src/test/reduce-instructions.test.ts:
34 | 	// @ts-expect-error
35 | 	expect(Math.round(simplified[1].cp2x) * 100).toEqual(20000);
36 | });
37 | 
38 | test('Should reduce T after a T that follows a non-quadratic command', () => {
39 | 	expect(reduceInstructions(parsePath('M 0 0 H 10 T 20 10 T 30 0'))).toEqual(
                                                                         ^
error: expect(received).toEqual(expected)

@@ -21,5 +21,5 @@
    {
-     "cp1x": 26.666666666666664,
+     "cp1x": 33.33333333333333,
      "cp1y": 16.666666666666664,
-     "cp2x": 30,
+     "cp2x": 36.666666666666664,
      "cp2y": 13.333333333333332,

- Expected  - 2
+ Received  + 2

      at <anonymous> (/home/user/remotion/packages/paths/src/test/reduce-instructions.test.ts:39:69)
(fail) Should reduce T after a T that follows a non-quadratic command [79.38ms]

 4 pass
 2 fail
 8 expect() calls
Ran 6 tests across 2 files. [229.00ms]
```

## Pass after

Same command with the fix restored:

```
bun test v1.4.4 (38afee6c8)

 6 pass
 0 fail
 10 expect() calls
Ran 6 tests across 2 files. [15.00ms]
```

## Other checks (in `packages/paths`, on the committed code)

| Check | Result |
| --- | --- |
| `bun test src` (whole package) | 53 pass, 0 fail |
| `bunx oxfmt src --check` | "All matched files use the correct format." |
| `bun run lint` (eslint) | 0 errors. 1 warning, already on `main`: a `TODO` comment in `cut-instruction.ts` |
| `bunx tsc --noEmit -p .` | passes |
| `bunx turbo run make --filter=@remotion/paths` | 3/3 tasks successful |

Root `bun run stylecheck` runs `lint` + `formatting` per package (done above for `@remotion/paths`) plus `checkskills`; no skills changed.

## Differential testing against Chromium

- Reference: Playwright Chromium, `SVGPathElement.getTotalLength()` / `getPointAtLength()`.
- Corpus: 2000 seeded random paths mixing `L l H h V v Q q T t C c S s A`, 1390 of them containing `T`/`t`.
- Tolerance: lengths within `max(0.5, 0.2% of length)`, points within `max(1, 0.5% of length)`.

| Check | Before | After |
| --- | --- | --- |
| `getLength(d)` vs Chromium | 300 mismatches | 0 |
| `getLength(serializeInstructions(reduceInstructions(parsePath(d))))` vs Chromium | 296 | 0 |
| `getPointAtLength()` at 25/50/75% (6000 samples) | 1259 | 443 |

All 443 remaining `getPointAtLength()` mismatches are in paths that contain an elliptical arc (`A`). None of the arc-free paths mismatch, including every `T` case. That's a separate issue, not part of this PR.

## Follow-up lead (not in this PR)

`getPointAtLength()` on an eccentric elliptical arc seems to map length to the arc's angle, not to arc length.

| Call | `@remotion/paths` | Chromium |
| --- | --- | --- |
| `getPointAtLength('M 0 0 A 50 10 0 0 1 100 0', L/4)` | (14.64, −7.07) | (23.79, −8.52) |
| same for the circle `A 50 50` | (14.64, −35.36) | (14.64, −35.36) |

Worth a look in `packages/paths/src/helpers/arc.ts` before opening anything; check for duplicates first.

## Duplicate search (all states, `remotion-dev/remotion`)

GitHub search via the GitHub MCP server, PRs open and closed, on 2026-10-30:

| Query | Hits | Relevant? |
| --- | --- | --- |
| PRs `paths quadratic T` | #12547, #12687, #10039 | No (see below) |
| PRs `getLength` | #12547, #12687, #12410, #11585, #10039, #6456 | No |
| PRs `reduceInstructions` | #12687, #12410, #2357 (2023 SVG parsing), #1793 (2023, added `reduceInstructions()`) | No |
| PRs `smooth quadratic` | #12547 | No |
| PRs `"@remotion/paths" control point` | #12687, #12547, #12410, #11581, #11579, #10039, … | No |
| Open PRs `paths` | 8 open PRs, none about `@remotion/paths` by title | No |
| Issues (semantic) "@remotion/paths smooth quadratic T command wrong control point getLength reduceInstructions" | none | — |
| Issues (semantic) "getLength returns wrong length for SVG path translatePath" | #3026 "getPointAtLength() return wrong point sometimes" (closed, old) | No |

- #12547 (merged as `6745d319a5`) fixed the `q` branch with `cpdx === 0 && cpdy === 0`, which built its line from the wrong fields. It's already in `upstream/main` and doesn't touch `T` / `prev_point` / `iterate.ts`.
- #12687 is our merged `getBoundingBox()` flat-tangent fix.
- `git log upstream/main -- packages/paths/src` shows no later change to `construct.ts` or `iterate.ts`.

## Notes for the local session

- Per the session instructions, the commit includes focused tests even though `.agents/skills/pr/SKILL.md` asks to keep agent-written tests out. #12645 and #12687 were merged with tests.
- `bun install` rewrote `bun.lock` locally. That change was reverted and isn't in the commit.
- To open the PR: `gh pr create --repo remotion-dev/remotion --head b84330808:claude/funny-galileo-b8glbo --title '<title above>' --body-file <body above>`.
