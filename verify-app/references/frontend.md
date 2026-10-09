# Frontend surface

Use for UI, components, pages, and user flows.

## Pre-check gaps

In addition to the common gaps, record these:

- New `console.log` calls.
- Secrets exposed in client-side env vars (`NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*`).

## Test matrix

- **Happy path:** the main user flow, end to end.
- **UI states:** loading, empty, error (for example, the API returns 500 or the network is slow), and success.
- **Invalid input:** form validation messages, disabled submit, no crash on bad input.
- **Viewports:** desktop and phone width (about 375px).
- **Keyboard:** the flow can be completed with Tab and Enter, and focus is visible.
- **Regression:** one neighbouring page or component that shares the changed code.

## Start the app and pick a browser tool

Start the app the way step 1 found. Use, in order:

1. The start command from `.claude/verify.md`.
2. The project's own run skill.
3. The `run` skill.
4. Starting the dev server yourself and polling its URL.

Start the backend first if the frontend depends on it.

Use the first browser tool the session actually provides:

1. Claude in Chrome (`mcp__claude-in-chrome__*`)
2. The built-in browser (`mcp__Claude_Browser__*`)
3. A Playwright MCP server, or Playwright via a script in the scratchpad

Load the skill for that tool first if one exists. With no browser tool, fall back to fetching the HTML and the API calls over HTTP. The verdict for anything visual is then at best `INCONCLUSIVE`.

## Evidence

For each scenario:

- Take a screenshot or accessibility snapshot of the final state, and save it.
- Read the console. Errors and unhandled rejections count as failures, even if the UI looks fine.
- Check the network requests. The expected calls should fire with the right payloads, and none should fail with an unexpected 4xx or 5xx.
- Look at the screenshot yourself. Check the text, the layout, and anything overlapping or cut off; don't stop at "the element exists".

Write the exact steps taken to `steps.md` so the run can be replayed. Close the tabs you opened when done.

## Failure conditions

The verdict is `NOT VERIFIED` if any scenario renders wrongly, behaves wrongly, throws console errors, or makes a request that fails unexpectedly.
