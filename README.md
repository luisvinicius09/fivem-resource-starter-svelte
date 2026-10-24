# starter-svelte

The `starter` resource with the NUI page rewritten in Svelte 5 and styled with Tailwind v4. Lua client
and TypeScript server are identical apart from names, so the two resources can run side by side.

| Part            | Language                          | Runs in                | Source                               |
|-----------------|-----------------------------------|------------------------|--------------------------------------|
| Game-side logic | Lua 5.4                           | FiveM client           | `client/main.lua`                    |
| Server logic    | TypeScript                        | FiveM server (Node)    | `server/main.ts` -> `dist/server.js` |
| UI              | Svelte 5 + Tailwind v4 + TypeScript | NUI (Chromium overlay) | `ui/` -> `dist/ui/`                |

Shared message types live in `shared/protocol.ts`. The Lua side mirrors them by hand.

## What is different from `starter`

- `ui/App.svelte` is the whole panel: state with `$state`, derived rows with `$derived`, events with
  `<svelte:window onmessage onkeydown>`, and Tailwind utility classes instead of a `<style>` block.
- `ui/app.css` is the Tailwind entry. Tailwind v4 is configured in CSS: `@import "tailwindcss"`, an
  `@source` that limits class scanning to `ui/`, and an `@theme` block that turns the design tokens
  (`panel`, `fg`, `muted`, `accent`, `edge`) into utilities such as `bg-panel` and `text-muted`.
- `build.mjs` runs Tailwind through PostCSS inside a small esbuild plugin, adds `esbuild-svelte`, and
  targets Chromium 103 so esbuild lowers JS and CSS to what FiveM's NUI can run.
- `npm run typecheck` runs `svelte-check` for the UI, because `tsc` cannot read `.svelte` files.
- Key F9 and command `/starter-svelte`, so they do not fight with `starter` on F7.

## Tailwind and FiveM's browser

FiveM's production NUI is Chromium 103. The CEF 144 upgrade exists but is on the canary channel.
Tailwind v4 officially targets Chrome 111+, and since 4.1 it ships fallbacks for colours and opacity
modifiers, so most utilities work on 103. A few rely on CSS that 103 does not have:

- `translate-*`, `rotate-*`, `scale-*` (the `translate`, `rotate`, `scale` properties, Chrome 104).
  Centre with flex or grid instead; `App.svelte` does this with a full-screen flex layer.
- `@container` queries and `has-*` variants (Chrome 105), `text-balance` (114), `starting:` (117),
  `field-sizing-*` (123).
- Default palette colours are `oklch`; esbuild converts them for the Chromium 103 target. Prefer the
  tokens in `@theme`, which are plain `rgb`/hex and need no conversion.

When CEF 144 reaches production, raise `target` in `build.mjs` and these limits disappear.

## How the pieces talk

    App.svelte  --fetch https://starter-svelte/<callback>-->  client/main.lua  --TriggerServerEvent-->  server/main.ts
    App.svelte  <--SendNUIMessage (window message)---------  client/main.lua  <--TriggerClientEvent--  server/main.ts

- NUI callbacks: `close`, `requestInfo` (registered in `client/main.lua`)
- Net events: `starter-svelte:server:getInfo`, `starter-svelte:client:info`
- Server export: `exports['starter-svelte']:getUptime()`

## Setup and day-to-day

1. `npm install`, then `npm run build`.
2. `ensure starter-svelte` in `server.cfg`.
3. Join, press F9 or type `/starter-svelte`.

`npm run watch` rebuilds `dist/` on every change, including Tailwind when a class is added in `ui/`;
then `restart starter-svelte` in the server console. Lua in `client/` needs no build step.
`npm run typecheck` checks server and UI. The Tailwind CSS IntelliSense extension for VS Code finds
`ui/app.css` on its own and completes the custom tokens.

## Deploying

Only `fxmanifest.lua`, `client/`, `ui/index.html` and `dist/` are needed at runtime. `node_modules/` is
build-time only. `dist/` is git-ignored, so either build on the server or commit it.

## Notes

- Svelte 5 compiles `<script lang="ts">` natively as long as the TypeScript is erasable (no `enum`, no
  `namespace`, no decorators). That is why there is no Svelte preprocessor in `build.mjs`.
- `@citizenfx/server` only publishes prerelease-tagged versions, which caret ranges never match. It is pinned exactly.
- `node_version '22'` selects Node 22 on recent artifacts; older ones ignore it and run Node 16.
- Keep per-frame work out of NUI. Send state when it changes, not every tick.
