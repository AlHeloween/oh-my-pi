# Cache-breakpoint wire probe (standard)

Counterpart of `opencode/experiments/20260912_anthropic-cache/`. Both repos run
the same probe shape so the numbers are directly comparable.

## The standard

A conforming probe:

1. drives the **real** provider through its public entry with a stub `fetch`
   (never a hand-rolled request builder — the point is to see what production
   actually sends);
2. builds the system prompt from **real repo artifacts** (`AGENTS.md` and the
   agent prompt), so printed sizes are production sizes, not a toy;
3. prints exactly these four sections, in this order:

```
=== marker placement on the system blocks ===      idx | slot | chars | marker
=== cache_control blocks ON THE WIRE ===           where | idx | chars | ttl
    breakpoints on the wire: N  (Anthropic hard cap: 4)
=== static prefix anchored by each system breakpoint ===  breakpoint | anchoredChars | estTokens
    tools+system total / left OUTSIDE any static breakpoint (chars, %)
=== request headers (secrets redacted) ===
```

4. redacts any header whose name contains `key`, `token`, or is
   `authorization`, and never prints a credential value or length;
5. needs no credentials — a canned SSE envelope stands in for the response.

The headline number is the last line of section 3: **what share of the static
prefix (tools + system) is anchored by a breakpoint that survives a tail
reset.** That is the figure to compare across implementations.

## Status in this repo

`01_wire_shape.mts` is written but **has not run here**: this checkout has no
installed workspace (`node_modules/@oh-my-pi` is empty), so
`@oh-my-pi/pi-catalog/build` does not resolve and `@oh-my-pi/pi-natives` has no
built addon for win32-x64. The repo's own tests fail the same way — e.g.
`bun test packages/ai/test/anthropic-cache-refresh.test.ts` →
`Cannot find module '@oh-my-pi/pi-catalog/build'`.

To run:

```bash
bun install
bun --cwd=packages/natives run build
bun run experiments/20260912_cache-breakpoints/01_wire_shape.mts
```

## What is already established without running it

Read directly from `packages/ai/src/providers/anthropic.ts` (2026-09-12):

- `buildAnthropicSystemBlocks` (2896) never sets `cache_control`. The only
  writer is `applyPromptCaching` (3195), which marks the **last two messages**.
  Of the four breakpoints Anthropic allows, two are used and both sit on a
  moving target; tools + system have no anchor of their own.
- The anchor position depends on conversation shape: `hasTrailingAssistantPad`
  (3203-3210) shifts the window by one when the `"Continue."` pad is present.
- The first system block is `createClaudeBillingHeader(firstUserMessageText)` —
  request-derived content at prefix position 0, where any change invalidates
  everything after it.
- `applyCacheControlToLastBlock` (3179) correctly skips `thinking`,
  `redacted_thinking` and `fallback` blocks when walking backwards. This is a
  correctness detail the opencode side lacks.

The probe would add the byte counts and the anchored-share figure, and catch
anything the read missed.
