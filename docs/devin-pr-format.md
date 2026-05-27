# .devin

Containerized spawner that opens a Devin session when a GitHub issue in this
repo is labeled `devin`. Triggered by `.github/workflows/devin-on-issue.yml`;
prompt is built in `src/prompt.ts`.

## What a Devin PR looks like

The prompt requires every PR Devin opens to follow the structure below. The
sample below is what a reviewer should actually see — sections only appear
when they apply (e.g. no `Screenshots / GIFs` block for pure-backend changes,
no `Out of scope / follow-ups` if there genuinely are none).

---

**Title:** `fix(dashboard): close native filter dropdown on Esc keydown`

**Body:**

````markdown
## Issue
Closes https://github.com/jellybean0211/superset/issues/42

## Root cause
`NativeFilter.tsx` registers a click-outside handler to dismiss the popover
but never wires up a keydown listener. Pressing Esc was a no-op because the
underlying AntD `Select` only handles Esc when its internal input is focused,
and the filter wrapper steals focus on mount via `autoFocus`.

## Summary of change
- Attach a `keydown` listener on the filter container that calls the existing
  `onDropdownVisibleChange(false)` when the key is `Escape`.
- Listener is attached only while the dropdown is open and removed on
  close/unmount.

## Reproduction steps
1. Open any dashboard with a native filter (e.g. the "Video Game Sales" sample).
2. Click the filter trigger to open the dropdown.
3. Press `Esc`.
4. **Expected:** dropdown closes, focus returns to the trigger.
5. **Actual (before this PR):** dropdown stays open; only an outside click dismisses it.

## Blast radius
- `NativeFilter.tsx` is consumed by `FilterBar.tsx` and `FilterControl.tsx`.
  I grep'd `grep -r "NativeFilter" superset-frontend/src` — 6 import sites,
  all pass props through; none read `onDropdownVisibleChange` themselves.
- The new keydown listener is scoped to the filter container element via a
  ref, so it cannot bubble into or shadow listeners on the dashboard grid.
- Did not change the `Select` component or any AntD wrapper in
  `@superset-ui/core` — no cross-component impact.

## Alternatives considered
- **Patch AntD `Select` directly to handle Esc globally** — would change
  behavior for every Select in the app (SQL Lab, chart controls, etc.), which
  is out of scope and risky.
- **Use `onKeyDown` prop on the trigger button** — only fires when the trigger
  is focused; loses focus to the dropdown items once it opens, so Esc inside
  the open dropdown wouldn't fire.

## Confidence & unknowns
- `High confidence:` keydown fires and dropdown closes — covered by unit test.
- `Lower confidence:` I assumed the focus-stealing on mount is intentional and
  didn't change it. If a reviewer knows otherwise, that's a separate concern.
- `Did not verify:` touch devices (no `keydown` equivalent), screen reader
  behavior, RTL layouts, dashboards with >50 filters (performance of the
  per-filter listener at scale).

## Verification
- Added a unit test asserting Esc collapses the dropdown and a regression
  test asserting Esc is a no-op when the dropdown is already closed.
- Manually verified in `npm run dev` against a dashboard with three native
  filters.

## Test results
```
$ npm run test -- NativeFilter
PASS  src/dashboard/components/nativeFilters/NativeFilter.test.tsx
  ✓ closes dropdown on Escape keydown (38ms)
  ✓ ignores Escape when dropdown is closed (12ms)
  ✓ existing: opens on trigger click (24ms)

Test Suites: 1 passed, 1 total
Tests:       3 passed, 3 total
```
Pre-commit (`prettier`, `eslint`, `mypy`) — not run in sandbox; CI will exercise on push.

## Screenshots / GIFs
**Before:** ![before](https://.../before.gif) — Esc pressed, dropdown stays open.
**After:** ![after](https://.../after.gif) — Esc pressed, dropdown closes, focus returns to trigger.

## Out of scope / follow-ups
- Click-outside dismissal in `NativeFilter.tsx` swallows clicks on the
  dashboard's "Apply filters" button (separate bug, worth filing).
- The `autoFocus` behavior that steals focus on mount feels wrong but
  changing it touches accessibility — left for a dedicated PR.
````

---

## Why these sections

Reviewing an AI-generated PR is partly about checking whether the agent
understood the problem, not just whether the code compiles. Each section
exists to surface understanding (or its absence) without forcing the reviewer
to reconstruct it from the diff:

- **Reproduction steps** — validates the framing in seconds; separates "the
  bug" from "the test Devin wrote."
- **Blast radius** — answers "what else does this touch?" so the reviewer
  doesn't have to grep.
- **Alternatives considered** — pre-empts the "why didn't you just…" comment.
- **Confidence & unknowns** — the single highest-signal section for an AI PR;
  tells the reviewer where to focus attention.
- **Test results** — pasted command output, not a summary, so a fabricated
  pass is harder to hide.
- **Screenshots / GIFs** — required for UI changes only; the prompt says omit
  rather than write "N/A" for backend.
- **Out of scope / follow-ups** — surfaces known debt without scope creep; the
  prompt explicitly warns against inventing items to look thorough.

## Local dry-run

```bash
cd .devin
npm install
UPSTREAM_REPO=jellybean0211/superset \
TARGET_REPO=jellybean0211/superset \
ISSUE_NUMBER=42 \
ISSUE_TITLE="example" \
ISSUE_BODY="" \
ISSUE_URL=https://github.com/jellybean0211/superset/issues/42 \
  npx tsx src/index.ts --dry-run
```

Prints the exact prompt that would be sent to Devin without spawning a session.
