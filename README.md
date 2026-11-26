# expo-server-route-order-repro

Minimal repro for `expo-server`: a page route that matches broadly (a catch-all `app/[...missing].tsx`, or a dynamic `app/[slug].tsx`) is served instead of a more specific API route like `app/hello+api.ts`.

Versions: expo 58.0.26, expo-router 58.0.32, expo-server 58.1.0 (latest on npm), @expo/cli 58.1.21.

## Files

- `app/_layout.tsx` - a `<Slot />`
- `app/index.tsx` - home page
- `app/[...missing].tsx` - catch-all page that renders "Not found"
- `app/hello+api.ts` - `GET` returns `Response.json({ ok: true })`
- `app.json` - `web.output: "server"` and the `expo-router` plugin

## Steps

```sh
pnpm install
```

Dev server:

```sh
pnpm exec expo start --web --port 8099
curl -i http://localhost:8099/hello
```

Production style:

```sh
pnpm exec expo export -p web
pnpm exec expo serve --port 8099
curl -i http://localhost:8099/hello
```

## Expected

`GET /hello` returns `{"ok":true}` from `app/hello+api.ts`. `/hello` is a static match, `[...missing]` is a catch-all, so the API route should win.

## Actual

`GET /hello` returns `200 text/html` with the "Not found" page, both in dev and after `expo export` + `expo serve`.

`POST /hello` does reach the API route (it answers `405`, since only `GET` is exported), because the HTML routes are only tried for `GET` and `HEAD`.

If `app/[...missing].tsx` is deleted, `GET /hello` returns the JSON as expected.

## Other shapes

- `app/[slug].tsx` in place of the catch-all gives the same result for single segment API routes (`/hello`), but not for `/api/ping` or `/users/1`, which a single `[slug]` can't match.
- The catch-all shadows every API route, whatever its depth or file name (`/aaa`, `/zzz`, `/api/ping`, `/users/1` with `users/[id]+api.ts`).
- The order of the files doesn't matter. The order that matters is HTML routes before API routes.

## Cause

`createRequestHandler` in `expo-server/build/{cjs,mjs}/vendor/abstract.js` (`src/vendor/abstract.ts`) loops over `manifest.htmlRoutes` first and returns on the first regex match for `GET` and `HEAD`. Only when none match does it loop over `manifest.apiRoutes`. The two lists are never ranked against each other.
