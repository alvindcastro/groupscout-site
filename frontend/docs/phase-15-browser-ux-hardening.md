# Phase 15 Browser UX Hardening

## Scope

Phase 15 adds deterministic UX hardening evidence for the current dependency-free renderer. It verifies primary routes expose route-specific keyboard-focus labels, accessible names, rendered desktop/tablet/mobile modes, stable state regions, same-origin API metadata, and a text-containment policy without adding a browser dependency.

## UI Behavior

- Primary routes covered: Today, Leads, Lead Detail, Verification, Outreach, Pipeline, Analytics, and Alerts.
- Lead Inbox loading, error, and empty states remain stable and expose `role="status"` or `role="alert"` where appropriate.
- Focusable labels are derived from rendered route controls/actions and primary navigation, so non-lead routes cannot pass through navigation labels alone.
- The hardening report renders desktop, tablet, and mobile variants for every primary route and records the resulting layout modes.
- Browser API calls remain same-origin through `/api/*`.

## TDD Evidence

- Red: `node --test test/browser-ux-hardening.test.js` failed because `web/src/renderer/browserUxHardening.js` did not exist.
- Green: `node --test test/browser-ux-hardening.test.js` passed after adding `BROWSER_UX_HARDENING_CONTRACT` and `createBrowserUxHardeningReport(...)`.
- Refresh red on 2026-05-09: `node --test test/browser-ux-hardening.test.js` failed because route-specific focus labels and rendered responsive modes were not reported for every primary route.
- Refresh green on 2026-05-09: `node --test test/browser-ux-hardening.test.js` passed after extending the route shell and renderer to expose route controls/actions plus desktop, tablet, and mobile rendered modes.
- Broader verification: `npm test`.

## Out Of Scope

Real browser-engine focus traversal, screenshots, pixel checks, computed layout, and visual overlap detection are still out of scope until a deterministic browser harness such as Playwright is introduced.
